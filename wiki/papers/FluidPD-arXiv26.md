---
type: paper
name: FluidPD
full_title: "FluidPD: In-Place Elasticity for SLO-Aware Prefill-Decode Disaggregated LLM Serving"
authors: [Kartik Ramesh, Kaidi Fu, Zihan Zheng, Jiahuan Yu, Fabio Oliveira, Carlos Costa, Minjia Zhang]
venue: arXiv
year: 2026
tags: [lens/pd-serving, llm-serving, disaggregated-serving, autoscaling, scheduling]
source_pdf: "[[arxiv26-ramesh-fluidpd.pdf]]"
source_md: "[[arxiv26-ramesh-fluidpd]]"
review_status: needs-review
evidence_level: full-text
last_reviewed: 2026-10-29
---

# FluidPD：在既有 GPU 上弹性调整预填充与解码能力（arXiv 2026）

> **原题**：FluidPD: In-Place Elasticity for SLO-Aware Prefill-Decode Disaggregated LLM Serving

> **一句话总结**：固定 P/D 配比遇到短时突发和持续流量变化时可能失衡；FluidPD 让 decode 实例有条件地接手部分 prefill，并在压力持续时切换实例角色，在单机八卡实验中改善所测 SLO 达成率。[^paper]

**阅读状态**：依据 2026-10-02 arXiv v1 的本地 PDF 和 MinerU 全文；已回看 PDF 图 9–11，其他图表和实现细节仍需复核。该工作是预印本，以下数字为作者报告，未复现实验。[^paper]

## 问题与动机

分离部署中，prefill 实例产生完成预填充的请求，decode 实例消耗这些请求并持续生成 token。配置静态的 4P4D 并不意味着四张卡始终足以应对 prefill：代码请求输入较长、输出较短时 prefill 可能排队，而对话请求输出较长时 decode 可能成为瓶颈。请求长度、前缀命中率和到达率变化都会改变有效需求。[^paper]

常规扩容可以在总容量不足时增加实例，但新增实例涉及检测、资源供应与模型加载；如果系统只是阶段之间分配失衡，尚有空闲 GPU，增加总 GPU 也不是唯一选择。论文区分短时突发与持续的阶段需求变化，并尝试在既有部署内分别处理（§III–IV）。[^paper]

## 关键观察 / 隐含假设

- **同一配比对不同流量可能表现相反。** Llama-3.1-8B、八张 A100 的分阶段实验中，5P3D 在对话阶段的总体 SLO 达成率为 86.93%，代码阶段为 3.40%；6P2D 在对应阶段为 6.64% 和 99.40%（§III.A、图 2）。这是实验设定下的例子，不代表线上必然应使用这些配比。[^paper]
- **空闲必须足够及时且可用。** 让 decode 侧分担 prefill 依赖该侧有计算余量与 KV 容量；若总负载饱和，或 decode 已接近 TPOT 目标，借用能力本身可能带来干扰（§IV.B、§V.B）。[^paper]
- **角色切换快于重新部署，但不是瞬时完成。** 运行中的请求需要排空；切换时间取决于当前阶段及未完成请求。模型常驻和引擎支持角色转换是设计前提（§IV.C、§V.H）。[^paper]

## 核心方法

### 两种时间尺度的弹性

系统先将请求分配给一对 prefill 和 decode 实例。短暂出现 prefill 压力时，**FluidToken** 将选定请求的 prompt 切成前缀和后缀：prefill 实例先计算前缀并传输其 KV，配对的 decode 实例再完成后缀的 prefill，然后进入正常 decode。这样减少 prefill 队列中的工作，不要求立刻改变实例角色；prompt 计算被重新分配，并没有凭空消失（§IV.B、图 6–7）。[^paper]

例如某请求包含 1000 个输入 token，若策略选择后缀占 20%，则 prefill 侧处理前 800 个 token，decode 侧在收到前缀 KV 后处理余下 200 个。这只是说明数据流的例子；实际比例由压力、预测耗时和 decode 的空闲预算共同决定。[^paper]

**FluidRole** 用于持续失衡：选取一个运行中的实例，经过停止接收新任务、排空在途工作及角色切换的过程，将其从 P 转为 D 或从 D 转为 P。模型权重保留在实例内，避免重新加载；切换仍需等待安全边界。论文用阈值、持续观察和冷却期避免瞬时压力使角色反复切换（§IV.C、图 8）。[^paper]

### 如何决定借多少、何时切换

预填充压力指数 PPI 比较队列中请求的预测完成时间与剩余 TTFT 预算。PPI 大于 1 表示按照当前队列和预测执行时间，该请求有超时风险。解码压力指数 DPI 则分别考虑活跃 decode 工作量、KV 已占用和为已接纳请求保留的空间。两类信号帮助控制器区分 prefill 本身拥塞与 decode 无法接纳新请求两种情况（§IV.B）。[^paper]

FluidToken 只在配对的 decode 实例有余量时分担后缀，并依据预测的 decode 迭代时间约束后缀长度。论文的贪心策略优先分配能减少更多预测 TTFT 违约的工作；其判断依赖离线画像。预测错误或并发访存干扰可能使实际延迟不同于估计，不能把预算约束当成严格实时保证（§IV.B、算法 1）。[^paper]

FluidRole 汇总持续压力，选择可释放容量的实例，并等待它完成当前工作再改变角色。P→D 主要等待当前 prefill 和未完成的 KV 传输；D→P 要等待活跃 decode 请求结束，因此受输出长度影响。不同于 [[DistServe-OSDI24]] 的画像和部署规划，FluidPD 的重点是已有配置运行之后的适应；与 [[TaiChi-arXiv25]] 的请求级流动相近，但其角色切换直接改变 P/D worker 数量。[^paper]

## 设计取舍

- **快速借用与干扰**：借用 decode 的空闲处理 prefill 能缩短其他请求排队，但会占用 decode 算力和 KV 空间；高负载下 TPOT 可能恶化。
- **稳定切换与响应速度**：等待压力持续和实例排空有助于减少抖动、保留请求，却会延迟新配比生效；长输出尤其影响 D→P。
- **固定总量与扩容**：就地调整应对阶段错配；当两侧都缺容量时，仍需要扩容或其他准入策略。

## 实验与结果

作者在单节点 8×A100-80GB、每实例一张卡上测试 Llama-3.1-8B、Qwen3-14B 和 Qwen3-30B-A3B。工作负载由 Azure code 与 conversation 轨迹构造：staged 将三段各 1 万请求串接以制造持续变化，mixed 取 1 万条混合请求观察突发；按受控速率回放，并非完整一周生产流量原样回放（§V.A）。[^paper]

总体 SLO 达成率是同时满足 TTFT 与 TPOT 门槛的请求比例；主实验 TPOT 门槛为 100 ms，三个模型的 TTFT 门槛分别为 2、3、1 秒。静态基线 SGLang、vLLM、Dynamo 从 4P4D 开始；FluidPD 也从 4P4D 开始但可在八卡内调整。作者报告相对静态 SGLang，staged 和 mixed 的总体达成率分别最高增加 94.6 和 74.3 **个百分点**（§V.B、图 9）。这些是特定负载点的达成率差，不是吞吐倍数。[^paper]

在另一组扩容对照中，双方起始均为 3P3D；FluidPD 固定使用六卡，Dynamo 可以扩至八卡。Llama-3.1-8B、mixed、20 请求/秒下，总体达成率分别为 97.6% 和 9.3%；作者报告 Dynamo 扩容后的 worker 在约 200 秒后就绪。具体优势取决于扩容配置、可用 GPU 和突发持续时间，不能概括为所有扩容系统均有相同延迟（§III.C、§V.B、图 4、9）。[^paper]

### 消融与代价

图 11 在 Llama-3.1-8B、4P4D 起始配置下比较静态基线、仅 FluidToken、仅 FluidRole 和完整系统。staged、30 请求/秒时总体 SLO 达成率分别约为 41%、52%、78%、80%；mixed、35 请求/秒时分别约为 56%、79%、61%、81%。持续变化主要由角色切换受益，短时混合突发则更依赖借用；两组结果对应不同流量，不能将数字直接相减归因于普遍的组件贡献。[^paper]

作者报告在线控制每轮 CPU 侧开销低于 0.50 ms；P→D 约 1 秒，D→P 最多约 12 秒（§V.H）。这不是集群级最坏情况保证。模型与 GPU 的画像需事先建立；论文报告大部分所测点的预测相对误差为 2%–10%，在线重新校准留作后续工作（§V.G）。[^paper]

## 论断—证据表

| 论断 | 证据 | 边界 | 置信度 |
|---|---|---|---|
| 固定配比随请求类型变化而失衡 | §III.A、图 2 | 单机八卡、指定工作负载 | 中 |
| 两种机制分别应对不同持续时间的压力 | §V.D、图 11 | 一种模型及两组回放负载 | 中 |
| 原地调整改善所测达成率 | §V.B、图 9 | 最多八卡、固定 SLO 与所选基线 | 中 |

## 批判性分析

### 论证链条

论文先用配比对照说明工作负载变化会改变瓶颈，再用借用后缀和角色转换处理不同时间尺度，最后用消融与排队时间线检验。图 9 的总体改善大多来自 TTFT；高负载下部分 prefill 移到 decode 会使 TPOT 达成率下降，说明借用并非无成本。

### 假设压力测试

DPI 和后缀预算依赖当前 KV 余量、批量与延迟画像。长上下文、热点前缀、不准确的剩余输出估计或并发网络流量都可能改变可借用空间。工作负载在切换完成前再次反转时，排空和冷却时间可能使配置短期落后于需求；这属于基于机制的推断。[^paper]

### 实验可信度

代码和对话请求来自生产轨迹，但 staged 是人工组织的三段负载，实验在单个八卡节点内进行。论文说明没有直接比较 TaiChi、TokenScale 和 [[Libra-NSDI26]]，原因是评估时未取得公开实现；Dynamo 扩容对照的 GPU 预算和配置也与静态对照不同。单机结果不能证明跨节点或大规模故障场景下同样有效（§V.A）。[^paper]

### 系统性缺陷

角色转换等待请求完成，未知剩余输出长度会放大 D→P 等待；过载时同时出现两侧压力，就地调配无法增加总算力。论文未给出跨节点网络争用、故障恢复和多租户公平性的系统实验。[^paper]

## 局限与后续工作

- 在相同 GPU 总量、模型和 SLO 下比较静态搜索、请求级借用、原地角色切换与扩缩容，记录每种方法的生效时间和违约请求。
- 控制输出长度尾部、KV 占用与负载反转频率，测量 D→P 排空时间、角色抖动和 TPOT 违约。
- 扩展至多节点，计入 KV 交接、网络拥塞、模型并行与实例重配成本。

## 相关

- [[PD-Serving]]、[[DistServe-OSDI24]]、[[TaiChi-arXiv25]]、[[Mooncake-FAST25]]

[^paper]: Ramesh 等. “FluidPD: In-Place Elasticity for SLO-Aware Prefill-Decode Disaggregated LLM Serving.” arXiv，2026。[原文](https://arxiv.org/abs/2610.06917v1)；本地全文 [[arxiv26-ramesh-fluidpd]]。
