---
type: paper
name: Splitwise
full_title: "Splitwise: Efficient Generative LLM Inference Using Phase Splitting"
authors: [Pratyush Patel, Esha Choukse, Chaojie Zhang, Aashaka Shah, Íñigo Goiri, Saeed Maleki, Ricardo Bianchini]
venue: ISCA
year: 2024
tags: [lens/pd-serving, llm-serving, disaggregated-serving, heterogeneous-computing, kv-cache]
source_pdf: "[[isca24-patel-splitwise.pdf]]"
source_md: "[[isca24-patel-splitwise]]"
review_status: needs-review
evidence_level: full-text
last_reviewed: 2026-10-29
---

# Splitwise：分离推理阶段并匹配异构硬件（ISCA 2024）

> **原题**：Splitwise: Efficient Generative LLM Inference Using Phase Splitting

> **一句话总结**：利用 prefill 和 decode 对计算、内存与功耗的不同需求，Splitwise 分配专用及混合机器池；等功耗模拟中，A100 分离配置相对 A100 共置基线达到 2.15 倍吞吐（§VI.B）。[^paper]

**阅读状态**：MinerU Flash 输出全文文本但未导出图像；已回看 PDF 的图 14–15、18–20，其余图表和模拟细节尚未复核。[^paper]

## 问题与动机

GPU 的计算能力和内存带宽并非按相同比例增长，两个推理阶段的硬件需求也不同。将阶段放在不同机器上，可以分别配置资源，并探索在 prefill 和 decode 使用不同代 GPU；代价是搬运 KV 和保留额外模型副本。

## 关键观察 / 隐含假设

- 作者对生产对话和代码工作负载及 A100/H100 进行阶段画像；prompt 与 token 阶段在批量、内存使用和功耗上不同（§III）。观察依赖所测模型、并发和 GPU。
- 异构资源要有足够快的互连，且请求长度分布足以使阶段池保持平衡；论文用混合池吸收负载变化（§IV）。

## 核心方法

集群设置 prompt、token 和混合池。调度器按负载将请求安排到合适的池；混合机器可根据需要参与两个阶段。prefill 机器按层发送 KV，与后续计算重叠；接收端依据信号开始处理（§IV、§V.A）。[[DistServe-OSDI24]] 同样分离阶段，但重点是 SLO 约束下的阶段并行与放置搜索。

## 设计取舍

### 为什么引入混合池

集群调度器维护 prompt 池、token 池和混合池。专用池分别按阶段优化批量和硬件，混合池用于吸收资源不能整除以及负载变化带来的闲置；负载较高时混合池可以承担两阶段，减少资源碎片（§IV.A、§VI.B）。因此 Splitwise 不是简单把全部 GPU 静态切成两个固定池。[^paper]

异构配置将不同代 GPU 用于不同阶段，出发点是计算能力、内存带宽、功耗和价格增长并不同步。论文表 I 的 A100/H100 比较是当时的硬件与价格参数；对新的 GPU、能源价格和采购方式需要重新画像与计算。[^paper]

### KV 交接与实际延迟

作者在 vLLM 上实现逐层发送：一层 KV 产生后即可经 MSCCL++ 单边传输，不必等待整个 prefill 结束；每个请求用信号同步接收端，并合并连续块以减少传输次数（§IV.C、§V.A）。图 14 报告逐层发送剩余未重叠时间在所测 A100/H100 上约为 8/5 ms。[^paper]

交接对用户的影响不能只看传输量：图 15 的两机、无批处理实验中，作者报告其逐层方案增加约 0.8% 端到端延迟，但第二 token 延迟增加 16.5%；串行传输的第二 token 延迟开销更高。该实验使用高速 InfiniBand，不能保证拥塞网络下仍有相同表现。[^paper]

### 模拟中的资源预算

集群设计主要由离散事件模拟器评估，输入包括硬件画像、生产轨迹的输入输出长度分布、泊松到达、机器数量和价格功耗参数。每个设计需同时满足 TTFT、TBT、端到端时间的 P50、P90、P99 共九项目标；这些目标以无竞争 A100 请求为参照（§V.B、表 VI）。比较等功耗、等成本、等吞吐设计时，必须分别看机器数量、GPU 代际和空间约束，不能把不同图中的倍数混用。[^paper]

异构部署允许权衡吞吐、采购成本、功耗与机房空间，但旧 GPU 数量可能增加，阶段容量也可能失衡。按层传输减轻可见延迟，却依赖快速互连和实现细节。

## 实验与结果

- 作者在 Azure 的 A100 和 H100 机器上实现 KV 传输，并用画像驱动的事件模拟器探索集群配置；模拟与硬件实验是不同证据（§V）。
- 层级传输的未重叠部分在所测 A100/H100 机器上约为 8/5 ms；传输开销相对于 prompt 计算低于 7%（§VI.A、图 14）。
- 等功耗、对话轨迹模拟中，Splitwise-AA 相对 Baseline-A100 在相同功耗和成本下达到 2.15 倍吞吐（§VI.B、图 18）。
- 把为代码流量配置的异构集群用于对话流量时，部分异构方案吞吐相对专门配置下降 7%（§VI.D、图 20）。

### 工作负载与模拟边界

生产轨迹的代码服务输入长度中位数为 1500 token、输出中位数为 13；对话服务对应 1020 和 129（§III.A）。这使两种服务的阶段需求不同，也解释了为什么不能只用一个固定 P/D 比例。画像时作者将轨迹缩小到每秒两个请求：对话服务有 60%–70% 的时间只运行不超过 20 个活跃 token，代码服务超过 20% 的时间只运行一个 token（§III.B）。这是缩放后的单机画像，并非所有生产集群的利用率。[^paper]

等功耗对话实验中，Splitwise-AA 以同代 A100 分离阶段达到相对 A100 共置基线 2.15 倍吞吐；混合 H100/A100 的 Splitwise-HA 为 1.18 倍吞吐、成本低 10%。异构方案不必然比同代方案更快，其价值取决于功耗、成本和空间约束（§VI.B、图 18）。高负载离线任务中混合池可能承接全部机器，系统趋近共置基线，论文也报告该场景的吞吐成本效率没有同样优势（§VI.E）。[^paper]

## 论断—证据表

| 论断 | 证据 | 边界 | 置信度 |
|---|---|---|---|
| 逐层交接可隐藏大部分所测传输时间 | §VI.A、图 14 | 高速 InfiniBand 与所测模型 | 中 |
| 分离资源池可以提高相同预算下的模拟吞吐 | §VI.B、图 18 | 画像驱动模拟、指定价格与功耗 | 中 |

## 批判性分析

### 论证链条

实机画像与传输实验支撑模拟器的输入；大规模成本和吞吐结论主要来自模拟，不能等同于完整集群实测。

### 假设压力测试

电价、硬件租赁价格、网络带宽和请求长度分布变化，都可能改变最优机器组合。

### 实验可信度

生产轨迹提供长度分布，扩容比较仍依赖合成到达与性能模型；作者报告模型误差，但无法覆盖所有生产扰动。

### 系统性缺陷

论文指出失败时可从头重启请求，安全高效的 KV 检查点恢复不在设计范围（§IV.E）。

## 局限与后续工作

- 使用相同成本、功耗和空间口径重新评估当前 GPU，以及网络争用下的交接开销。
- 测量突发到达时混合池切换带来的排队和尾延迟。

## 相关

- [[PD-Serving]]、[[EcoServe-OSDI26]]、[[PragmaticDisaggregation-MLSys26]]

[^paper]: Patel 等. “Splitwise: Efficient Generative LLM Inference Using Phase Splitting.” ISCA 2024。[论文](https://www.microsoft.com/en-us/research/wp-content/uploads/2023/12/Splitwise_ISCA24.pdf)；本地全文 [[isca24-patel-splitwise]]。
