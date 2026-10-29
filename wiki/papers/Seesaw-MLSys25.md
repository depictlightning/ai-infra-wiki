---
type: paper
name: Seesaw
full_title: "Seesaw: High-throughput LLM Inference via Model Re-sharding"
authors: [Qidong Su, Wei Zhao, Xin Li, Muralidhar Andoorveedu, Chenhao Jiang, Zhanda Zhu, Kevin Song, Christina Giannoula, Gennady Pekhimenko]
venue: MLSys
year: 2025
tags: [lens/pd-serving, llm-serving, offline-inference, parallelism, kv-cache]
source_pdf: "[[cbc4ab80cd77aa0eb87da062fbcddb46.pdf]]"
source_md: "[[cbc4ab80cd77aa0eb87da062fbcddb46]]"
review_status: needs-review
evidence_level: full-text
last_reviewed: 2026-10-29
---

# Seesaw：通过模型重分片优化两阶段离线吞吐（MLSys 2025）

> **原题**：Seesaw: High-throughput LLM Inference via Model Re-sharding

> **一句话总结**：在低带宽多 GPU 环境中，prefill 和 decode 可能偏好不同的并行方式；Seesaw 在同一组 GPU 上切换 TP/PP 配置，并缓冲 KV 以摊薄切换成本，所测 PCIe 实验相对最佳固定并行 vLLM 平均达到 1.36 倍吞吐（§6.2）。[^paper]

**阅读状态**：依据 MLSys 2025 会议版 PDF 和 MinerU 全文，回看图 5–7、10–12；其余图表和源码未逐项复核。该文主要研究吞吐导向的离线推理，未复现实验。[^paper]

## 问题与动机

当模型跨多张 GPU 运行时，TP 需要各 GPU 在每层交换结果，PP 则将不同层放到流水线阶段。prefill 一次处理较多输入 token，跨卡通信可能很重；decode 每步 token 少，PP 为填充流水线而切分的微批可能反复读取模型权重。一个固定的 TP/PP 配置因此未必同时适合两阶段，尤其在 PCIe 等相对低带宽互连上（§1、§3.1）。[^paper]

一种选择是为两个阶段分别分配 GPU，但小集群要为两个池各存一份模型，且 GPU 数量和显存以整数粒度约束配比。论文图 4 用 70B 模型和八张 40 GiB GPU 举例：每侧至少四张才能存下权重，配比被限制为 4P4D。作者针对离线吞吐场景，让同一组 GPU 随阶段变换并行配置（§3.2）。[^paper]

## 关键观察 / 隐含假设

- **两阶段并行瓶颈不同。** 在论文测量中，prefill 的 TP 集合通信占比较高，而 decode 的 PP 微批会增加重复读权重的开销；结论依赖模型、批量、GPU 和互连（§3.1、图 1、3）。[^paper]
- **阶段切换必须摊薄。** 模型权重和 KV 的分布随 TP/PP 配置改变；如果持续批处理不断交替两阶段，重分片可能频繁发生，抵消计算收益（§4.1–4.2）。[^paper]
- **系统有 CPU 内存和离线等待空间。** 缓冲大量已预填充请求需要主机内存，也使请求在阶段切换前等待；这符合本文吞吐优先的目标，不等于在线 TTFT 满足保证（§4.2、§6.1）。[^paper]

## 核心方法

### 为什么重分片不只是改一个配置数字

假设 prefill 时两张 GPU 各持有不同的层，decode 改用 TP 后，两张 GPU 需要持有各层不同的权重分片；对应的 KV 布局也必须随之变化。系统从 CPU 内存重新装载所需权重分片，并借助 CPU 共享内存重新组织 KV。切换具有真实的数据移动成本，不能假定瞬时完成（§4.1、图 5）。[^paper]

Seesaw 动态调整的是 TP 与 PP；论文没有动态调整 DP，因为 DP 改变模型副本数及显存中权重与 KV 的比例，会增加内存管理和数据移动复杂度。阶段内选何种配置仍取决于目标硬件和工作负载（§4.1）。[^paper]

### 如何避免每接一个新请求就切换

系统在 prefill 阶段连续处理请求，把生成的 KV 暂存到 CPU；当 CPU KV 缓冲装满后，转换并行配置并进入 decode。decode 期间请求逐步完成，CPU 中缓存的 KV 被换入 GPU 以维持批量；当 CPU KV 缓冲清空后，系统再切回 prefill（§4.2、图 6）。这样以主机内存和等待时间换取较少阶段切换及较大的 decode 批量。[^paper]

KV 在 CPU 共享存储中以可供多个 GPU 访问的形式组织。prefill 侧按原来的分片写出，decode 侧按新配置读取所需分片；因此 KV 换出与换入也承担布局变换。调度器和每个 GPU 的 worker 之间通过队列传递工作（§4.2、§5.1、图 7）。[^paper]

### 如何掩盖数据移动

prefill 期间，刚生成的 KV 在 decode 之前不会使用，可以把 GPU 到固定页内存的复制与后续计算重叠，再从固定页内存复制到共享内存。decode 侧由后台线程预取，KV 到达后再通知调度器把请求加入批次。异步执行能减少可见等待，前提是复制吞吐和可用 CPU 内存跟得上请求消耗；它并不消除 CPU–GPU 字节数（§5.2、图 8）。[^paper]

## 设计取舍

- **阶段专用配置与重分片**：分别选择 TP/PP 可降低某阶段瓶颈，但切换时必须迁移权重与 KV。
- **大缓冲与请求等待**：CPU 缓冲和较长阶段窗口减少切换次数，也占用主机内存并增加请求等待时间。
- **离线吞吐与在线延迟**：论文的主要目标是离线处理速率；不能只根据吞吐结果推断 TTFT、TPOT 或逐请求 SLO。

## 实验与结果

论文使用 A10、L4 和 A100，多 GPU 测试；模型包括 15B、CodeLLaMA-34B 和 LLaMA2-70B，数据来自 ShareGPT 与 arXiv 摘要，分别抽取 2000 与 500 个请求。每 GPU 配置 80 GiB CPU 内存。基线 vLLM 扫描固定并行配置并选择最佳结果（§6.1–6.2）。[^paper]

- PCIe 系统中，A10 的吞吐几何平均为固定并行 vLLM 的 1.45 倍、最高 1.78 倍；L4 几何平均 1.29 倍、最高 1.52 倍；论文报告综合平均 1.36 倍（§6.2、图 10）。这些是离线吞吐比，不是端到端延迟或 SLO 达成率。
- 图 12 的 CodeLLaMA-34B、摘要任务和四张 A10 案例中，PP4 的 prefill 表现较好，TP4 的 decode 较好；Seesaw 在两个阶段分别采用这两种配置。图中 TP4 和 PP4 对照关闭了 chunked prefill，用于展示阶段时间；带 chunked prefill 的固定配置是 TP2PP2（§6.3）。[^paper]
- 图 11 中，A100 PCIe 的摘要和对话任务相对对应的 vLLM 结果，作者报告吞吐提高 46% 和 30%；NVLink 下对话任务报告 13%。不同互连条件导致收益变化（§6.4）。[^paper]
- 图 14 的互连带宽扫描不是在全部带宽上实机运行，而是修改采集到的 all-reduce 时间后预测吞吐；不能当成全范围网络实测（§6.5）。[^paper]

### 与 P/D 数量分配的关系

Seesaw 不为 prefill 和 decode 建立两个固定实例池，因此没有直接求出 P/D 实例数量。它回答相邻问题：当资源太少、阶段复制和整数配比造成浪费时，同一批 GPU 能否在时间上轮换并针对阶段改变并行方式？它适合作为 [[DistServe-OSDI24]]、[[Splitwise-ISCA24]] 的离线吞吐对照，不可据此判断在线混合或分离方案的 SLO 排名。[^paper]

## 论断—证据表

| 论断 | 证据 | 边界 | 置信度 |
|---|---|---|---|
| 两阶段可能偏好不同并行策略 | §3.1、图 1、3 | 所测模型、批量和互连 | 中 |
| 重分片提高所测离线吞吐 | §6.2、图 10 | 预先选择最佳固定 vLLM 配置，A10/L4 | 中 |
| CPU 缓冲减少切换频率 | §4.2、图 6–7 | 需要主机内存，增加等待 | 中 |

## 批判性分析

### 论证链条

作者先解释 TP 与 PP 在两阶段的开销差异，再指出空间分离在小规模集群中的副本和整数配比限制，最后以重分片、CPU KV 缓冲和离线吞吐实验回应。由于完整系统同时引入布局、预取和调度机制，最终吞吐收益不能全部归因于单一组件。

### 假设压力测试

如果互连很快，TP 在 prefill 的通信劣势可能缩小；如果主机内存不足或 CPU–GPU 链路与其他数据传输争用，换入换出可能暴露在关键路径上。较短输出也可能使后台预取难以充分重叠（§5.2、§6.5）。[^paper]

### 实验可信度

实验覆盖多型号 GPU 和两类数据集，但目标是吞吐而非用户交互延迟；带宽扫描部分依赖预测。正文 §6.1 将 70B 模型列为 LLaMA2-70B，而 §6.4 一处写作 LLaMA3-70B，图 11 本身没有明确模型标注；引用该结果时应保留这一型号不一致，避免替作者选择版本。[^paper]

### 系统性缺陷

CPU KV 缓冲带来主机内存消耗和跨层级数据移动；较长阶段窗口可能延迟新请求。论文没有充分证明负载突发、取消请求、故障恢复和多租户公平性，也没有把离线吞吐收益转换为在线尾延迟保障。

## 局限与后续工作

- 固定 GPU 和 CPU 内存预算，比较重分片、固定配置及 P/D 分离的吞吐、首次响应时间和数据移动量。
- 扫描输入输出长度、CPU–GPU 带宽与缓存容量，记录阶段窗口、GPU 空闲和实际重分片时间。
- 在在线突发与长尾输出下测量等待和 SLO 达成率，确定该离线方法是否需要不同切换策略。

## 相关

- [[PD-Serving]]、[[DistServe-OSDI24]]、[[Splitwise-ISCA24]]、[[BatchLLM-MLSys26]]

[^paper]: Su 等. “Seesaw: High-throughput LLM Inference via Model Re-sharding.” MLSys 2025。[会议论文](https://proceedings.mlsys.org/paper_files/paper/2025/hash/cbc4ab80cd77aa0eb87da062fbcddb46-Abstract-Conference.html)；本地全文 [[cbc4ab80cd77aa0eb87da062fbcddb46]]。
