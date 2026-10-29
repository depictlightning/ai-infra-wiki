---
type: paper
name: DistServe
full_title: "DistServe: Disaggregating Prefill and Decoding for Goodput-optimized Large Language Model Serving"
authors: [Yinmin Zhong, Shengyu Liu, Junda Chen, Jianbo Hu, Yibo Zhu, Xuanzhe Liu, Xin Jin, Hao Zhang]
venue: OSDI
year: 2024
tags: [lens/pd-serving, llm-serving, disaggregated-serving, scheduling, kv-cache]
source_pdf: "[[osdi2024-zhong-yinmin.pdf]]"
source_md: "[[osdi2024-zhong-yinmin]]"
review_status: needs-review
evidence_level: full-text
last_reviewed: 2026-10-29
---

# DistServe：分离预填充和解码以优化有效吞吐（OSDI 2024）

> **原题**：DistServe: Disaggregating Prefill and Decoding for Goodput-optimized Large Language Model Serving

> **一句话总结**：共置使两阶段互相干扰且并行配置耦合；DistServe 分离阶段并搜索资源和放置方案，在 ShareGPT 的 OPT 实验中相对 vLLM 支持 2.0–4.6 倍请求速率（§6.2）。[^paper]

**阅读状态**：MinerU Flash 提取全文文本但未导出图像；已回看 PDF 的图 8–10，其余图表和附录尚未复核。[^paper]

## 问题与动机

共置批次中的长 prefill 拖慢 decode，decode 又可能增加 prefill 延迟；同时两阶段被迫采用相同的硬件和并行策略。论文把满足 TTFT 与 TPOT 要求的请求比例作为约束，比较单位 GPU 可承受的最大请求速率。

## 关键观察 / 隐含假设

- §2.3 的批处理实验显示混入一个 prefill 会增加 decode 批次时间；增长幅度取决于 prompt 长度和批量。
- §3 指出独立阶段的批处理与并行选择不同，但阶段分离需要把 KV 移交给 decode；高效性依赖网络带宽和合理放置。

## 核心方法

系统为 prefill 和 decode 分别配置实例，并由编排层路由请求和发送 KV。放置算法结合阶段性能画像、SLO 和拓扑搜索并行度及资源分配（§4–5）。跨节点使用 NCCL，同节点采用异步复制。该方案与 [[SarathiServe-OSDI24]] 在同一实例中缩小 prefill 干扰的思路不同。

## 设计取舍

### 阶段为何需要不同配置

论文对 OPT-13B 的画像发现，在所测 A100 上单条 512 token 的 prefill 已可让计算趋于饱和，继续合批可能主要增加等待；decode 则可借助较大批量分摊读取权重的代价，但批量受活跃请求的 KV 显存和 TPOT 约束（§3.1–3.2）。这是具体模型和设备的画像结果，不是任何部署的固定阈值。[^paper]

对 66B 模型的两 GPU 实验，作者比较一条请求跨 GPU 并行与多个独立副本：低到达率时，缩短单请求执行有利于 TTFT；负载升高后，排队和吞吐逐渐重要。请求长度、跨设备通信以及 SLO 会改变交叉位置（§3.1、图 4）。[^paper]

### 从画像到放置

配置搜索先考虑每阶段在不同并行度和批量下的能力，再确定两阶段资源数与放置。其目标是在给定 SLO 达成率下提高每 GPU 的可服务速率；单纯为 prefill 增加 GPU 可能让 decode 成为新的瓶颈。节点内与节点间带宽不同，放置必须将 KV 交接开销计入，而不是只看各阶段的孤立吞吐（§4）。[^paper]

具体实现由前端接收请求，编排层选择两个阶段实例、传递 KV 并返回结果；每个实例以 Ray 管理 GPU worker。跨节点发送走 NCCL，同节点采用异步复制（§5）。所测低节点亲和方案尽量让对应流水线阶段在同一节点交接，减少对 25 Gbps 跨节点链路的依赖；这意味着测到的低传输延迟不能自动外推到任意跨节点配对（§6.3、图 10）。[^paper]

### 实验口径如何读

论文中的 90% 是请求同时满足相应 TTFT 与 TPOT 目标的达成率；图上横轴的 per-GPU rate 把服务速率按 GPU 数归一化。代码补全、对话和摘要使用不同的请求长度与 SLO，因此“7.4 倍请求速率”和“12.6 倍更严格 SLO”分别来自不同设置，不能相乘或当成同一个实验点（§6.1–6.2）。[^paper]

独立配置降低阶段耦合，却引入 KV 网络流量、额外副本及 P/D 资源匹配问题。画像和搜索依赖目标硬件与请求分布；网络拥塞或流量变化会使静态结果失准。

## 实验与结果

- 4 节点、32 张 A100，节点间 25 Gbps；OPT-13B/66B/175B，ShareGPT、HumanEval、LongBench；到达过程为泊松采样（§6.1）。
- 在 90% 请求同时达到各自阶段 SLO 的目标下，ShareGPT 上单位 GPU 请求速率相对 vLLM 提高 2.0–4.6 倍；对 DeepSpeed-MII 提高 1.6–7.4 倍（§6.2、图 8）。
- OPT-66B 的 LongBench 摘要实验中，相对 vLLM 请求速率提高 4.3 倍；这是特定 SLO 和负载下的对比，不能当作普遍收益（§6.2、图 9）。
- 作者在所测配置中对比模拟器与真实系统，报告 SLO 达成率误差小于 2%；覆盖范围有限（§6.4、表 2）。

### 配置搜索的消融与限制

作者用模拟器比较默认 vLLM、枚举并行策略的 vLLM++、受节点内放置约束的 DistServe-Low，以及假设跨节点高带宽、搜索约束较少的 DistServe-High。OPT-66B 的 ShareGPT 设置中，vLLM++ 未优于其默认策略；High 的更大搜索空间优于 Low（§6.4、图 11）。vLLM 缺少该实验所需的跨操作并行、实际测试床也没有 High 假设的跨节点带宽，所以这一组消融是模拟而非实机结果。[^paper]

算法运行时间为分钟级，作者将其视为部署前或重新部署时的开销（§6.5）。如果流量变化比搜索、模型加载和实例重配更快，原配置可能在适应之前失衡；该暂态没有由离线搜索时间单独覆盖。作者也指出，离线吞吐优先或仅有少量 GPU 的环境中，共置方案可能更合适（§7）。[^paper]

## 论断—证据表

| 论断 | 证据 | 边界 | 置信度 |
|---|---|---|---|
| 分离及独立配置提高所测 SLO 下的有效吞吐 | §6.2、图 8–9 | OPT、A100 与旧基线 | 中 |
| 放置模拟器可预测所测配置的达成率 | §6.4、表 2 | 仅验证有限配置 | 中 |

## 批判性分析

### 论证链条

干扰实验解释分离动机，阶段画像支撑配置搜索，再用真实系统和模拟消融测试；各组件的贡献仍受所选基线与互连限制。

### 假设压力测试

长上下文、更慢网络、动态流量与模型切换可能提高传输或容量失衡成本；这些情况不能从单一倍数推断。

### 实验可信度

请求长度来自公开数据集，但到达为合成过程；SLO 是作者针对任务设置的目标。实验不等于生产尾延迟保障。

### 系统性缺陷

论文未充分评估故障恢复、网络竞争以及频繁调整阶段比例的运维成本。

## 局限与后续工作

- 在相同 GPU 总数和网络预算下与 [[EcoServe-OSDI26]] 和 [[SarathiServe-OSDI24]] 比较尾延迟、KV 交接和利用率。
- 检查真实突发流量下两侧队列增长、KV 占用与角色重配置成本。

## 相关

- [[PD-Serving]]、[[DualPath-SIGCOMM26]]、[[PragmaticDisaggregation-MLSys26]]

[^paper]: Zhong 等. “DistServe: Disaggregating Prefill and Decoding for Goodput-optimized Large Language Model Serving.” OSDI 2024。[论文](https://www.usenix.org/conference/osdi24/presentation/zhong-yinmin)；本地全文 [[osdi2024-zhong-yinmin]]。
