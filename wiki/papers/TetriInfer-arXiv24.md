---
type: paper
name: TetriInfer
full_title: "Inference without Interference: Disaggregate LLM Inference for Mixed Downstream Workloads"
authors: [Cunchen Hu, Heyang Huang, Liangliang Xu, Xusheng Chen, Jiang Xu, et al.]
venue: arXiv
year: 2024
tags: [lens/pd-serving, llm-serving, disaggregated-serving, scheduling, kv-cache]
source_pdf: "[[arxiv24-hu-tetriinfer.pdf]]"
source_md: "[[arxiv24-hu-tetriinfer]]"
review_status: needs-review
evidence_level: full-text
last_reviewed: 2026-10-29
---

# TetriInfer：面向混合工作负载的 P/D 分离与调度（arXiv 2024）

> **原题**：Inference without Interference: Disaggregate LLM Inference for Mixed Downstream Workloads

> **一句话总结**：不同请求的输入、输出长度导致阶段内部及阶段之间的干扰；TetriInfer 将 prefill 分块、分离 P/D 实例，并预测输出长度以调度 decode，在所测混合负载上改善平均延迟和资源效率。[^paper]

**阅读状态**：依据 2024 年 arXiv v1 全文，回看图 11–19；原文仍为预印本，模拟网络和小规模测试床限制结论。未运行代码或复现实验。[^paper]

## 问题与动机

不同应用的输入和输出长度相差很大。短 prompt 的交互请求与长 prompt 的摘要任务混在一起时，后者可能增加前者的排队；prefill 与 decode 共批也可能拖慢生成；不同输出长度的 decode 请求集中在同一实例还会形成负载热点（§2.2、图 3–5）。[^paper]

TetriInfer 把问题拆成三个层次：先控制单次 prefill 的工作量，再把两阶段交给不同实例，最后根据预测的资源占用选择 decode 实例与批次。它与同期的 [[DistServe-OSDI24]]、[[Splitwise-ISCA24]] 同属早期阶段分离工作，但评估设置和重点不同。[^paper]

## 关键观察 / 隐含假设

- **请求之间也有干扰。** 论文在所测场景中分别观察 prefill–prefill、prefill–decode 和 decode–decode 干扰；这说明消除跨阶段干扰后，仍需考虑阶段内部的调度（§2.2）。[^paper]
- **未来输出长度会影响资源预留。** decode 的 KV cache 随生成增长，若长输出请求集中到一个实例，显存和处理时间都可能失衡；预测不准时，提前预留也可能浪费或不足（§3.3–3.4）。[^paper]
- **阶段容量需求会变化。** 系统允许实例在 prefill 与 decode 角色之间切换，前提是能够停止接收新任务并处理完在途工作（§3.5）。[^paper]

## 核心方法

### 一条请求如何经过两个实例

集中式控制面接收请求，向 prefill 实例分配工作，并收集和广播实例负载。prefill 侧将请求切成固定大小的块，使用局部调度策略安排这些块；预填充生成的 KV 被交给 decode 实例，后者连续生成输出。论文将分块与阶段分离组合：同一个 prefill 实例只处理 prefill，decode 不与其共同组成混合批次（§3.1–3.3、图 6–7）。[^paper]

作者比较 FCFS、短作业优先和长作业优先等 prefill 排队方式。在图 16 的设置下，分块加 FCFS 相对 vLLM 基线降低平均延迟，短作业优先进一步降低平均等待；优先短请求可能使长请求等待更久，平均值本身不能说明公平性（§5.2.1）。[^paper]

### 输出长度预测用于哪些决策

系统在 prefill 节点运行小模型，预测目标模型的输出长度区间。预测不是直接决定生成内容，而是用来估计请求在 decode 阶段可能占用的内存和时间；prefill 节点据此将请求分散到不同 decode 实例，减少长输出集中造成的热点（§3.3.2、§3.4）。[^paper]

decode 实例内部还比较不同显存预留策略。论文的动态预留根据预测的剩余长度估计后续空间，试图避免显存反复紧张和请求换入换出。图 18 中，实际预测精度下该策略的平均 JCT 与 vLLM 贪心策略相近；使用理想预测时才出现更明显的改善。这个对照说明策略收益受预测质量约束（§5.2.3）。[^paper]

论文使用 OPT-125M 预测 OPT-13B 的输出长度，在 ShareGPT 上按 100、200、400 token 的区间划分时报告准确率 58.9%、74.9%、85%。区间越粗越容易命中，但也提供更少的长度信息；不能把分类准确率等同于逐 token 精确预测（§5.2.2、图 17）。[^paper]

### P/D 数量如何变化

除增加实例之外，控制面也可在固定资源内切换角色。P→D 时停止向该实例发送新 prefill，等待队列排空后修改角色；D→P 时通知 prefill 实例停止向该 decode 发送请求，再完成切换。论文报告修改角色本身约 5–7 ms，**不包含动态排空时间**；后者取决于在途请求（§3.5、图 10）。[^paper]

实验采用实例空闲一分钟后切换的策略。这个规则可以减少闲置，但没有证明它对所有突发时长和输出长度都合适；与 [[FluidPD-arXiv26]] 使用压力信号分别应对短时和持续失衡的设计可对照阅读。[^paper]

## 设计取舍

- **隔离与交接**：独立阶段减少共批干扰，但增加模型副本和 KV 传输需求；低带宽网络可能抵消收益。
- **预测与开销**：预测有助于分散 decode 负载，却需要额外模型计算；短区间预测不准、长区间信息较粗都会限制调度收益。
- **角色切换与排空**：保留进程和模型能缩短切换动作，但在途请求完成之前资源不能立即交给另一阶段。

## 实验与结果

测试床为单机四张 32 GB V100，主模型 OPT-13B、预测模型 OPT-125M，基线为 vLLM。论文使用 ShareGPT 样本构造四类输入输出长度组合及混合负载；其中输入超过 512 token 为重 prefill，输出超过 128 token 为重 decode（§5）。[^paper]

**网络证据边界**：实现只支持间接 socket 传输；端到端评估通过预先准备 KV、发送元数据并按目标带宽等待的方式模拟 200 Gbps RoCE 和 300 GB/s NVLink。图中的两种网络曲线不是在相应网络上实际传输完整 KV 的结果（§4–5.1）。[^paper]

- 轻 prefill、重 decode 的 128 请求实验中，作者报告平均 TTFT 降低 97%、平均 JCT 降低 47%、资源使用时间降低 38%，性能成本比达到 vLLM 的 2.4 倍（§5.1、图 12）。这里资源使用时间是实例使用时长的汇总，不是相同卡数下的直接吞吐加速。[^paper]
- 混合负载的 128 请求实验中，作者报告平均 TTFT、JCT 和资源使用时间分别降低 85%、50%、21%，性能成本比达到 1.9 倍（§5.1、图 15）。[^paper]
- 重 prefill、轻 decode 实验中，作者报告平均 TTFT 和 JCT 分别降低 9% 和 23%，但资源使用增加 43%，vLLM 的性能成本比高 14%。这是对“分离总能降低成本”的反例（§5.1、图 13）。[^paper]
- 预测模型与目标模型并发的压力测试中，作者报告目标模型平均 prefill 延迟增加约 10%、吞吐降低约 12%；它不应被当作所有部署的固定成本（§5.2.2、图 17）。[^paper]

## 论断—证据表

| 论断 | 证据 | 边界 | 置信度 |
|---|---|---|---|
| 阶段分离和调度改善所测混合负载 | §5.1、图 15 | 四张 V100、OPT-13B、模拟交接网络 | 中 |
| 预测质量影响 decode 预留收益 | §5.2.3、图 18 | 实际预测与理想预测的特定对照 | 中 |
| 分离不一定提高成本效率 | §5.1、图 13 | 重输入、轻输出负载 | 中 |
| 角色切换动作较快 | §3.5、图 10 | 5–7 ms 不包含排空 | 中 |

## 批判性分析

### 论证链条

论文从多类干扰出发，分别用分块、阶段隔离和预测调度回应，再按请求类型比较正负结果。实验中分块、队列策略、预测和分离同时变化；端到端差距不应全部归因于 P/D 分离。

### 假设压力测试

如果请求的输出长度难以预测，decode 侧预留和负载均衡可能失效；如果网络慢于模拟条件或与其他传输争用，KV 交接可能拉长 TTFT 和完成时间。长请求迟迟不结束也可能使角色切换等待显著超过 5–7 ms。

### 实验可信度

测试床确实运行模型计算，但网络交接采用预装 KV 和延迟模拟；使用小模型、四卡及固定数量请求也限制向大模型和在线持续到达推广。论文主要报告平均 TTFT、JCT 和资源时间，不能直接证明逐请求或 P99 SLO 保证。

### 系统性缺陷

集中控制面的故障恢复、预测模型漂移、真实网络拥塞和长时间角色抖动未在所述实验中得到充分验证。短作业优先对长请求尾延迟和公平性的影响也需要单独评估。

## 局限与后续工作

- 以真实 P/D KV 传输替代模拟等待，扫描带宽和拥塞，比较成本效率及 TTFT。
- 固定分离部署，分别关闭输出长度预测、decode 预留和跨实例均衡，量化每项贡献。
- 在重尾输出和突发负载下测量完整排空时间、角色切换频率与长请求尾延迟。

## 相关

- [[PD-Serving]]、[[DistServe-OSDI24]]、[[Splitwise-ISCA24]]、[[FluidPD-arXiv26]]

[^paper]: Hu 等. “Inference without Interference: Disaggregate LLM Inference for Mixed Downstream Workloads.” arXiv，2024。[原文](https://arxiv.org/abs/2401.11181v1)；本地全文 [[arxiv24-hu-tetriinfer]]。
