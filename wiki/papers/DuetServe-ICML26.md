---
type: paper
name: DuetServe
full_title: "DuetServe: Harmonizing Prefill and Decode for LLM Serving via Adaptive GPU Multiplexing"
authors: [Lei Gao, Chaoyi Jiang, Hossein Entezari Zarch, Daniel Wong, Mark D. Hill, Murali Annavaram]
venue: ICML
year: 2026
tags: [lens/pd-serving, llm-serving, scheduling, gpu-sharing, latency-slo]
source_pdf: "[[icml26-gao-duetserve.pdf]]"
source_md: "[[icml26-gao-duetserve]]"
review_status: needs-review
evidence_level: full-text
last_reviewed: 2026-10-29
---

# DuetServe：在 GPU 内动态隔离预填充与解码（ICML 2026）

> **原题**：DuetServe: Harmonizing Prefill and Decode for LLM Serving via Adaptive GPU Multiplexing

> **一句话总结**：固定 token 预算难以反映长上下文注意力成本；DuetServe 在预测 token 间延迟超标时动态划分 GPU 的 SM，让两个阶段并发执行，在所测 Mooncake 工作负载下相对 vLLM 达到约 1.3 倍请求吞吐（§5.2）。[^paper]

**阅读状态**：作者托管的会议版 PDF 已由 MinerU Flash 提取文本；图表及附录尚未视觉复核，故保留待审状态。[^paper]

## 问题与动机

同一 GPU 混合阶段时，长 prefill 可能增加 decode 的 token 间隔；跨 GPU 分离则可能带来副本、交接和容量失衡。论文探索在单个 GPU 内按需隔离执行资源。

## 关键观察 / 隐含假设

- 相同 token 预算下，注意力计算和 KV 读取仍随请求上下文长度变化，单靠 token 数难以预测迭代延迟（§3、图 1）。
- 所测 H100 上，一部分 SM 即可达到较高内存带宽，而计算吞吐随 SM 数变化；作者据此提出两阶段并发的机会（§3、图 3）。内存带宽仍是共享资源，SM 隔离不等于完整硬件隔离。

## 核心方法

调度器先按分块 prefill 策略形成批次，再用考虑注意力和缓存访问的性能模型预测迭代延迟。如果预测会超过 TBT 目标，就把 prefill 与 decode 分到不同 SM 集合并并发执行；否则继续共享执行。系统使用 libsmctrl 控制内核可见的 SM，并通过提前调度和异步派发减少同步（§4）。[[SarathiServe-OSDI24]] 主要用时间上的分块限制干扰；DuetServe 额外按需进行 GPU 内空间划分。

## 设计取舍

### 延迟预测器看到了什么

系统把算子分为按 token 数变化的线性等操作、按请求序列长度变化的注意力操作，以及并行通信。预测时综合各操作的计算量、内存访问量、被分配 SM 下的计算吞吐与 HBM 带宽，并将各层估计相加（§4.1）。这解释了为何相同 token 预算下，缓存历史长度不同仍可能产生不同 decode 时间。[^paper]

预测需要预先画像不同 SM 分配下的计算和带宽曲线。附录 A 指出，在较少 SM 的 decode 配置上模型倾向高估延迟；高估会留下空闲，而低估可能导致 decode 分得的资源不足。预测是决策依据，不是每次迭代的硬实时保证。[^paper]

### 两个阶段如何并发

若预测混合批次会超出 TBT 约束，调度器枚举 decode 分区，剩余 SM 交给 prefill；它同时选择一次 prefill 期间执行多少个 decode 步，比较两边预计耗时，最大化估计 token 吞吐并要求 decode 步耗时满足 TBT 约束（§4.2）。两边结束时间不一致时仍会有空闲，且目标函数中的输入 token 与输出 token 共同计数，不等同于请求级 goodput。[^paper]

运行时为两个阶段使用不同 CUDA stream；decode 使用预录制的 CUDA Graph，先预分配多个未来步骤的 KV 空间并准备元数据，连续派发多个 decode 步，减少逐步等待 CPU 同步的开销。prefill 因输入形状动态而逐个派发内核（§4.3）。预留空间和多步执行的收益要结合额外显存、已完成请求处理和预测偏差衡量。[^paper]

### 结果随输出长度变化

附录表 2 固定输入 4096 token，将输出从 64 增至 1024、2048：DuetServe 相对 vLLM 的请求吞吐倍数分别是 1.28、1.11、1.04，平均 TBT 分别从 170 降至 105 ms、55 降至 50 ms、45 降至 44 ms。该结果说明在论文的合成设置中，prefill 占比下降时收益减小；不能只引用最好的 1.28 倍。[^paper]

附录表 3 还给出单节点八张 H100 的 Qwen3-32B 实验：Dynamo 和 DuetServe 分别为 5.69 与 8.02 请求/秒，平均 TBT 分别为 23.1 与 104.7 ms。前者的 token 间延迟更低，后者吞吐更高；这不是同时优化所有延迟指标的证明，且对照受角色重配置成本影响。[^paper]

动态划分避免永久占用专用 GPU，但需要可靠的延迟预测，并依赖特定 GPU 与驱动机制；两个分区仍可能竞争 HBM 和其他共享资源。

## 实验与结果

- 主实验使用两张 NVLink 相连的 H100、Qwen3-8B/14B，比较 vLLM、SGLang 和 Dynamo 等配置（§5.1）。
- Mooncake 轨迹、Qwen3-8B、到达率 5 请求/秒时，vLLM 输出 2.14 请求/秒，DuetServe 为 2.72 请求/秒；这是输出吞吐，不应直接称为逐请求双 SLO goodput（§5.2、图 6）。
- 作者在附录比较静态 SM 配比与动态方案，并测试性能预测误差及不同输入输出长度；这些实验仍需结合图表核对（附录 A、图 8–9）。

### 负载、指标与比较口径

主实验中的 Azure Code、Azure Conversation 和 Mooncake Conversation 分别含 19366、8819 和抽样的 1000 条请求；表 1 给出的平均输入长度分别为 2047、1155、12035 token。作者按泊松过程重新生成到达，报告平均 TTFT、平均 TBT 和完成请求数除以总服务时间。平均 TBT 可能掩盖个别长间隔，不能直接证明 P99 或逐请求 SLO 达标（§5.1、表 1）。[^paper]

两 GPU 的分离基线为固定 1P+1D；GPU 更多时可以使用其他比例，论文自身也在附录讨论了这一限制。八 GPU 比较里 Dynamo 发生角色重配置，作者报告该过程约 40 秒；DuetServe 的吞吐收益可能同时反映配置切换成本和执行方式，不能直接归因于 SM 划分一个组件（附录 B、表 3）。[^paper]

## 论断—证据表

| 论断 | 证据 | 边界 | 置信度 |
|---|---|---|---|
| 动态 SM 划分可在所测负载下改善吞吐和 token 间延迟 | §5.2、图 6 | H100、Qwen3 和特定引擎版本 | 中 |
| 性能预测有助于选择划分 | §4、附录 A、图 8–9 | 测试覆盖的算子和 GPU | 中 |

## 批判性分析

### 论证链条

作者由固定预算无法覆盖注意力成本出发，构造包含注意力的预测器，再以动态划分与静态划分对照；图表及具体误差范围尚未复核。

### 假设压力测试

不同模型、内核、GPU 及并发访存会改变预测精度；当两阶段都受带宽限制时，SM 划分可能不能消除干扰。

### 实验可信度

主要实验规模是单机两 GPU；论文附录还报告单机八 GPU 实验。跨节点、大规模部署和阶段配比变化仍不能从中直接推断。

### 系统性缺陷

使用底层 SM 屏蔽机制带来驱动及平台兼容性问题；论文未充分覆盖多租户隔离、故障恢复和部署升级成本。

## 局限与后续工作

- 测量不同上下文长度下预测误差与 TBT 违约率，并记录 HBM 竞争。
- 在多节点相同总 GPU 预算下，比较动态 SM 划分与可调整 P/D 比例的分离方案。

## 相关

- [[PD-Serving]]、[[SarathiServe-OSDI24]]、[[DistServe-OSDI24]]、[[MuxWise-ASPLOS26]]（同样研究 GPU 内 SM 划分，评测口径与实现机制不同）

[^paper]: Gao 等. “DuetServe: Harmonizing Prefill and Decode for LLM Serving via Adaptive GPU Multiplexing.” ICML 2026。[会议页](https://proceedings.mlr.press/v306/gao26m.html)；本地全文 [[icml26-gao-duetserve]]。
