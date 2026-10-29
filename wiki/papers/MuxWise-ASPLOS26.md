---
type: paper
name: MuxWise
full_title: "Towards High-Goodput LLM Serving with Prefill-decode Multiplexing"
authors: [Yukang Chen, Weihao Cui, Han Zhao, Ziyi Xu, Xiaoze Fan, Xusheng Chen, Yangjie Zhou, Shixuan Sun, Bingsheng He, Quan Chen]
venue: ASPLOS
year: 2026
tags: [lens/pd-serving, llm-serving, scheduling, gpu-sharing, kv-cache]
source_pdf: "[[asplos26-chen-muxwise.pdf]]"
source_md: "[[asplos26-chen-muxwise]]"
review_status: needs-review
evidence_level: full-text
last_reviewed: 2026-10-29
---

# MuxWise：在 GPU 内复用预填充与解码计算资源（ASPLOS 2026）

> **原题**：Towards High-Goodput LLM Serving with Prefill-decode Multiplexing

> **一句话总结**：在多轮请求复用大量 KV、解码延迟要求严格的条件下，MuxWise 将一个实例内的 GPU 计算单元动态分给 prefill 和 decode，让它们共享 KV 内存并并发执行；收益依赖阶段负载、竞争和具体 SLO。[^paper]

**版本与阅读状态**：会议论文信息经 ASPLOS 2026 日程核对；本页依据作者公开的 arXiv v3 全文，并回看图 12、14、17–18 及表 3。该版本印有会议引用信息，但未逐项与 ACM 出版文件比对。尚未复现实验。[^paper][^program]

## 问题与动机

一个服务实例既要处理新请求的 prefill，也要为已有请求持续 decode。[[SarathiServe-OSDI24]] 一类分块方法将长 prefill 切短，与 decode 交错或融合，以限制单次等待。但块太小可能降低计算利用率；在复用上下文很长时，即便新增 token 很少，注意力仍要读取已有 KV，缩小块未必能控制 token 间延迟（§2.3、图 6）。[^paper]

[[DistServe-OSDI24]] 和 [[Splitwise-ISCA24]] 代表的实例级分离允许两阶段独立配置，但需要跨实例交接状态，并把显存划成不同的 KV 池。对多轮请求来说，能否继续命中此前的上下文影响很大；作者在图 5 中用缓存容量与命中率分析支持这个动机，不能将该曲线视作所有生产系统的实测结论。[^paper]

MuxWise 把分配粒度降到单个 GPU 内的流式多处理器（SM）：两个阶段各占一部分计算单元，仍留在同一进程和同一 KV 池中。它不需要固定数量的 P 实例和 D 实例，但要处理共享显存带宽竞争、两阶段完成时间不同和调度开销。[^paper]

## 关键观察 / 隐含假设

- **复用上下文改变资源需求。** 作者在 8 张 A100 上部署 Llama-70B，改变复用上下文长度，观察到 prefill 满足首 token 延迟要求所需计算资源增加，而 decode 所需算力对该变量不那么敏感；两阶段 KV 显存需求仍可能很大（§2.2、图 3）。这是特定批量与延迟目标下的观察，不是 prefill 永远计算受限的定律。[^paper]
- **划分 SM 不等于隔离所有资源。** prefill 和 decode 仍共享显存带宽。作者在 A100、H100 和不同 Llama 配置上画像，测得 decode 竞争减速从接近零到约 30%，且随划分变化（§3.3.1、图 11）。因此调度不能只看每个阶段单独运行的延迟。[^paper]
- **共享池需要共同部署。** 统一 KV 池有利于阶段及请求之间复用，但隐含前提是两阶段能在同一实例与支持相应划分机制的 GPU 上运行，并有足够显存承载模型和活跃 KV。单实例方案不能自动解决模型或缓存超出单实例容量的问题。[^paper]

## 核心方法

### SM 分配和共享 KV

系统以 NVIDIA GreenContext 把 CUDA stream 绑定到不同 SM 集合，并在同一进程中共享内存。调度器根据负载重新选择两个阶段的 SM 划分。论文使用 16 SM 的配置粒度，在所测 A100 和 H100 上分别形成六、七种配置；这不是把显存带宽同步切成相同的比例（§3.2–3.3）。[^paper]

这与固定 P:D 实例数不同：同一请求完成 prefill 后进入本实例的 decode 批次，不因阶段切换必须跨实例发送 KV；多个请求的不同阶段可以同时运行。如果两阶段在带宽上竞争，SM 分得再准确也不能保证实际延迟等于独立运行延迟。[^paper]

### 按层执行与异步合批

朴素并发仍会出现空档：长 prefill 启动或执行期间，decode 可能已经结束；decode 批次全部结束时，已经发出的长 prefill kernel 也不能立即抢占。MuxWise 将 prefill 划到 Transformer 层的粒度，在 decode 运行期间发出适量层；decode 结束后，可以让后续尚未发出的 prefill 层使用新的资源划分（§3.2、图 9–10）。[^paper]

调度器以 CUDA event 查询完成状态，继续异步启动工作，并在观察到 prefill 完成时把请求并入 decode 批次，减少等待两侧同时完成的空档。层级边界也为长请求让出位置：若短请求可能错过其首 token 目标，系统可在不使长请求自身违约的预测条件下抢占一次，不支持递归抢占。这里的抢占是调度尚未执行的层，不是中断已在 GPU 上运行的 kernel（§3.2、§3.4）。[^paper]

### 先估计独立延迟，再给竞争留余量

作者针对模型和机器离线画像，用上下文长度、批量、SM 配置等因素分别拟合 prefill 和 decode 的独立运行延迟。decode 侧再按网格采样结果增加竞争减速余量；运行期采集的数据可用于更新该余量。论文报告独立预测的最大偏差分别为 prefill 8.16%、decode 8.84%；这是画像覆盖范围内的测量，不是对未见负载的误差上界（§3.3）。[^paper]

精细的竞争全组合画像成本高，论文称其分析性画像花了一周；用于初始化的粗网格约 7000 个采样点，每个模型—机器组合可在 12 小时内收集。新的硬件、模型、kernel 或上下文分布都可能要求重新画像。论文用采样区间内的最大减速作保守估计，并不构成任意未来负载的数学最坏情况证明。[^paper]

### 每轮先给 decode 足够的 SM

调度器在 prefill 批次完成和每次 decode 迭代结束时重新决策：依据预测与竞争余量，为 decode 选能满足 token 间延迟目标的最少 SM，再把其余 SM 分给 prefill；同时根据两个阶段的预计持续时间决定发出多少 prefill 层。图 12 的示例在 decode 批量从 16 增至 20 后，把 decode 份额从 60% 调至 80%。这个比例是示例，不是推荐的固定配置。[^paper]

作者明确优先考虑 decode SLO；通常让 prefill 尽快完成，只有发生抢占时才检查相关 prefill 请求的目标。论文并未为全部请求的 TTFT 提供硬保证（§3.4.1–3.4.2）。超过实例容量时，还需要上层路由、排队或扩容；本文没有实现集群级配比控制。[^paper]

## 设计取舍

- **复用与独立配置**：共享模型和 KV 池有助于复用并减少跨实例交接，但单实例内两阶段受同一显存容量与带宽约束。
- **更细调度与开销**：按层发出 prefill 便于响应 decode 和短请求，但需要更多事件、调度和多种 CUDA Graph；论文测得图相关显存开销为 6.2%，最细层级启动额外开销不超过 1.5%（§4.5）。[^paper]
- **保守余量与利用率**：竞争保护有助于控制 decode 尾延迟，余量过大可能把更多 SM 留给 decode；低估则可能使其超时。运行期更新有助适应，但未证明能覆盖所有突发状态。[^paper]
- **严格目标与宽松目标**：作者将方法定位于严格 token 间延迟；对于宽松或无延迟约束的离线任务，论文也承认它可能没有优势（§5）。[^paper]

## 实验与结果

主要平台为 8 张 NVLink 互连的 A100 80GB，另测 8 张 H100 和 8 张 H200；主要模型为 Llama-8B、70B，也测试 Qwen3-235B MoE。实现基于 SGLang 0.4.10post2。对照包括调优 token 预算的 SGLang 分块 prefill、NanoFlow、LoongServe 和固定 1:1 的 SGLang-PD；对照的并行配置不同，SGLang-PD 每侧 TP=4，而 MuxWise 主配置 TP=8（§4.1）。[^paper]

- **真实轨迹端到端**：真实 Conversation 与 Tool&Agent 轨迹缩放到单服务器负载。8 张 A100、Llama-70B、Conversation 下，图 14(c) 中 MuxWise 的 P99 TTFT 为 17.9 秒、P99 TBT 为 54.8 毫秒；SGLang-PD 分别为 32.2 秒和 40.6 毫秒。后者 token 间延迟更短，说明收益不是每个指标都领先。部分基线进入不稳定状态，作者计算平均加速时排除了不稳定结果（§4.2、图 13–14）。[^paper]
- **增加到达率时的容量**：Tool&Agent 请求内容来自轨迹，到达时间改为泊松过程。Llama-70B 满足 P99 TBT 目标的峰值 goodput，MuxWise 分别是分块 prefill、LoongServe、SGLang-PD 的 3.06、2.62、1.62 倍；NanoFlow 在该配置下即便小块也未满足目标，论文未给其倍率（§4.2.3、图 15）。论文此处以 TBT 达标评估容量，不应改写成同时满足统一 TTFT 与 TBT 的请求比例。[^paper]
- **不同负载的划分**：在 LooGLE 长输入负载下，大部分 SM 给 prefill；OpenThoughts 长输出负载下，多数 SM 给 decode；ShareGPT 居中。对稳定请求率画出的配置变化较小；论文报告真实轨迹突发区间 30 秒内使用过全部六种配置（§4.4.1、图 18）。[^paper]
- **单卡边界**：单张 A100、Llama-8B 和 ShareGPT 下，MuxWise 相对分块 prefill 的 goodput 为 1.2 倍，同时 TBT 接近；该收益小于部分长上下文结果（§4.3.1）。[^paper]
- **空档的解释**：论文报告 MuxWise 与分块方法的 GPU stream 空档时间占比分别为 7.7% 和 4.5%；这是时间维度的指标，不衡量 kernel 占用的并行资源是否充分利用。作者认为额外空档主要出现在没有待处理 prefill 的 decode 阶段，不能简单据此判断效率更差（§4.4.2）。[^paper]

## 论断—证据表

| 论断 | 证据 | 评测边界 | 置信度 |
|---|---|---|---|
| 单 GPU 内的动态阶段划分可改善严格 TBT 下的峰值容量 | 图 15、表 5 | A100、Tool&Agent 请求及泊松到达，基线并行配置不同 | 中 |
| 复用上下文和共享资源竞争会改变分配决策 | 图 3、图 11、图 18 | 特定 Llama、GPU 和画像空间，带宽不能独立划分 | 中 |
| 按层调度、事件查询降低朴素并发的停顿 | 图 19、§4.4.2–4.5 | 两组工作负载及所测配置；不能外推为完全消除空档 | 中 |

## 批判性分析

### 论证链条

论文从复用上下文下的显存与计算需求出发，引出共享池与可调 SM 分配；再通过竞争画像、层级调度和消融实验解释实现路径。两组真实轨迹提供了比纯合成输入更强的动机，但被缩放到单服务器，不能直接作为集群生产部署结果。图 17 进一步覆盖不同输入输出形态，但这些请求的到达由泊松过程生成（§4.2–4.3）。[^paper]

### 对照和指标

固定 1:1 的 SGLang-PD 是明确的阶段分离基线，却不足以证明所有动态配比系统都会落后；论文也没有直接比较本 wiki 中的 [[FluidPD-arXiv26]] 或 [[Libra-NSDI26]]。NanoFlow 使用与分块方法相同的 token 预算，而论文指出其原方案需要较大预算才能获得高效率：这有助在严格 TBT 约束下比较，但应连同配置一起解读。[^paper]

论文主要使用 P99 TBT 确定 goodput，而不是对每个请求同时约束 TTFT 和 TBT；平均值、P50、P99 和逐 token TBT 也不可混同。论文整体平均 2.20 倍的摘要结果有其所选实验和基线范围，不应移植为任意系统的预期收益。[^paper]

### 迁移到新系统的风险

带宽竞争余量由样本得到，并依赖模型和机器；极端上下文、显存压力、未覆盖 kernel 组合和突发可能改变误差。层级切换还受非抢占 GPU kernel 和 CUDA Graph 配置数量限制。多租户隔离、故障恢复、跨节点流量与集群调度不是本文实验的核心，不能据此断定生产环境已经解决这些问题。[^paper]

## 局限与后续工作

- 在同一 GPU、模型和工作负载下比较 MuxWise、[[DuetServe-ICML26]] 与分块方案，统一 P99 TBT、TTFT 和双目标请求达标口径，测量不同竞争余量的代价。
- 把请求上下文复用率、显存占用、网络带宽和副本数一起纳入评测，比较单实例空间复用与跨实例阶段分离的边界；不能只比较 SM 比例和 P:D GPU 数。
- 给定画像之外的上下文分布或新 kernel，测量保护余量的更新速度、SLO 违约和预留资源，验证其是否足够稳健。
- 在集群实验中测量单实例方案与阶段分离组合时的路由和 KV 生命周期。论文讨论了这一可能性，但没有提供对应的大规模实证（§5）。[^paper]

## 相关

- **专题**：[[PD-Serving]]、[[LLM-Serving]]。
- **同 GPU 分配**：[[DuetServe-ICML26]] 也研究动态 SM 划分；两篇的调度、目标和评测配置不同，不能直接用摘要倍率排名。
- **混合与分离**：[[SarathiServe-OSDI24]]、[[Splitwise-ISCA24]]、[[DistServe-OSDI24]]。
- **公开实现线索**：论文附录提供 [作者的 SGLang 分支](https://github.com/ykcombat/sglang/tree/slo_config) 和复现实验入口；[SGLang 官方博客](https://www.lmsys.org/blog/2025-09-28-pdmux/) 由重合作者介绍其 PD-Multiplexing 早期实现。博客与本文具体版本和实验结果应分别核对。[^paper]

[^paper]: Chen et al., [Towards High-Goodput LLM Serving with Prefill-decode Multiplexing, arXiv v3](https://arxiv.org/abs/2504.14489v3), 2026. 本地全文见 frontmatter。
[^program]: [ASPLOS 2026 会议日程](https://www.asplos-conference.org/asplos2026/program/index.html)。
