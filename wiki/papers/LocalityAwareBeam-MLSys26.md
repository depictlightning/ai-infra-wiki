---
type: paper
name: Locality-Aware Beam Scheduling
full_title: "Locality-Aware Beam Scheduling for Efficient Test-Time Compute with a Consumer-grade GPU"
authors: [Hsing-Ti Wang, Hung-Tso Shiao, Chia-Lin Yang]
venue: MLSys
year: 2026
tags: [llm-inference, kv-cache, beam-search, offloading, prefetching, area/llm-inference, area/kv-cache-systems]
source_pdf: "[[44f683a84163b3523afe57c2e008bc8c.pdf]]"
source_md: "[[44f683a84163b3523afe57c2e008bc8c]]"
review_status: complete
evidence_level: full-text
last_reviewed: 2026-10-28
---

# Locality-Aware Beam Scheduling：面向消费级 GPU 的分支推理与 KV 调度（MLSys 2026）

> **原题**：Locality-Aware Beam Scheduling for Efficient Test-Time Compute with a Consumer-grade GPU

> **一句话总结**：当多个生成分支的 KV cache 放不进 GPU 时，论文按组连续推进多个 token，并让共享前缀的分支尽量同组，减少主机与 GPU 之间的 KV 重复搬运。[^paper]

**阅读状态**：已通读与会议正式版一致的 MinerU 正文，并核对方法图、端到端性能图和 KV 搬运量图；会议页面未列出独立补充材料。[^paper]

## 问题与动机

推理时增加搜索分支，可让较小模型进行更多候选探索，但每个活跃分支都需要历史 KV。消费级 GPU 的显存可能足以容纳模型权重，却不足以同时放下所有分支的 KV；此时逐层卸载方案在每次生成 token 时反复从主机取回缓存，PCIe 等待迅速累积（§1–3）。[^paper]

本文采用 **step-wise beam search**：每轮先从保留的 beam 扩展出候选序列，各候选生成多个 token 后评分，最后筛选并进入下一轮。它与每生成一个 token 就全局筛选的标准 token-wise beam search 不同；改变 step 长度既影响缓存重用，也改变候选被评分和剪枝的时机。[^paper]

## 关键观察 / 隐含假设

- **时间局部性**：同一分支连续生成多个 token，会反复使用相同历史 KV；如果将该组保留在显存中，就无需每个 token 都重新加载（§4.1）。[^paper]
- **前缀局部性**：由同一祖先扩展出的多个分支共享前缀，合理分组可避免每个组重复加载这部分缓存（§4.2）。[^paper]
- **权重能够常驻 GPU**：评测的模型权重适合放入显存。本文优化的是 KV 不足，不解决模型权重本身无法放入设备的问题（§6.1、§7）。[^paper]
- **缓存压力足够大**：如果全部 KV 已经放得下，卸载几乎不触发，实验中不同策略运行时间接近（图 9）。[^paper]

## 核心方法

### 按组推进 token

系统按当前可用显存估计每组能够容纳的分支数，把分支分组。一个组的 KV 从主机装入 GPU 后，连续生成本轮剩余 token，再切换到另一组；本轮各组执行结束后统一评分、剪枝和扩展。新增 KV 写回主机，以便后续轮次继续使用（§4.1）。[^paper]

例如一轮生成 32 个 token，逐 token 换组可能反复取回同一历史；按组连续执行则让这些访问发生在同一次驻留期间。组间串行和生成顺序变化可能影响并行效率与随机状态，不能仅凭调度图宣称与其他搜索实现逐 token 相同。

### 利用分支共同前缀

分支构成一棵共享历史的树。系统从一个未分配分支开始，贪心加入与当前组已有前缀重叠最大的分支，直到达到显存预算，再构建下一组。共享 KV 在一组内部只需加载一次；跨组相同前缀仍可能重复加载。贪心算法避免枚举所有分组，但不保证全局最优（§4.2、算法 1）。[^paper]

### 平衡分组与预取

简单地把前面各组装满，可能让最后一组很小，留下无法充分利用的显存。论文先保持所需组数不变，再均衡每组分支数，将腾出的空间用于预取下一组部分 KV，以重叠 I/O 和计算（§5）。[^paper]

这种选择也有代价：分组执行会反复启动计算和读取权重；预取受当前组的显存余量和 PCIe 带宽限制，不能认为所有搬运都被隐藏。[^paper]

## 设计取舍

- **较长 step 与搜索行为**：较长 step 提供更多驻留复用，但降低跨分支筛选频率；性能对照不是任务质量对照。
- **前缀相似性与分组容量**：同组共享历史可以节省搬运，但不能超过实际 GPU 预算，贪心结果也依赖分支树形态。
- **搬运量与总耗时**：搬运字节减少不自动等于等比例加速；计算、分组和仍无法重叠的 I/O 都占时间。

## 实验与结果

主要实验为 RTX 4090（24 GB）与 128 GB 主机内存，PCIe Gen4 ×8；模型为 OPT-6.7B、LLaMA-2-7B 和 Qwen-7B。输入 128 token、输出 1,920 token，beam width 为 2，总候选分支为 4–64，step 长度为 32、64 或 128 token；与逐层 KV 卸载基线比较（§6.1）。[^paper]

| 结果 | 实验条件及含义 |
|---|---|
| 端到端时间 | 图 9 中 OPT-6.7B、64 分支、step 32 的完整方案相对逐层卸载为 **9.72×**；小分支、缓存可全部驻留时则接近 1×。该基线并非所有现代推理框架的统一代表。 |
| KV 传输量 | 图 10(a) 中 OPT-6.7B、64 分支的 inter-token 策略相对逐层卸载分别为 **3.7%、1.8%、0.9%**，对应 step 32、64、128；这是传输量比值，不是端到端时间比值。 |
| 前缀复用 | 图 10(b) 将分支共同前缀纳入分组后，所测配置的传输量进一步下降；例如 OPT-6.7B、64 分支从约 2,000 GB 降到 450 GB。 |
| 等待构成 | 图 11 的 OPT-6.7B、64 分支、step 32 下，未被重叠的 I/O 占比由约 86% 降至约 10%；不能将该比例直接当作所有设备上的吞吐提升。 |

图 7 的搬运量来自分析模型；上表的图 9–11 属于论文实验图，二者证据性质不同。论文正文列举的加速上限与图 9 中更长 step 的部分柱值并不完全一致，以上只引用图中可直接核对的具体配置，不把某个数字当成全实验的统一上限。[^paper]

## 论断—证据表

| 论断 | 证据 | 评测边界 | 置信度 |
|---|---|---|---|
| 分组连续生成能减少 KV 换入 | §4.1、图 10(a) | 分支较多且显存不足；step 长度影响搜索 | 高 |
| 按共享前缀分组进一步减少搬运 | §4.2、图 10(b) | 收益取决于实际共享程度和分组预算 | 中高 |
| 可以普遍提升 LLM 推理质量 | §6–7 | 评测主要报告性能，未提供任务质量和等价性实验 | 低 |

## 批判性分析

### 论证链条

论文从层级卸载的重复数据传输出发，用时间局部性减少同一组在连续 token 之间的换入，再用前缀局部性避免同组分支重复加载。传输量、端到端时间和 I/O 时间分解互相支持，但实验相对于具体逐层卸载基线，不能直接推广为对生产系统的统一优势。[^paper]

### 假设压力测试

若 prompt 很长、分支间共享少、权重不能完全常驻 GPU，或者可用显存随其他任务变化，原有分组和预取收益可能下降。长 step 也会推迟剪枝，可能浪费已无希望分支的计算；其对最终答案质量的影响需要按任务验证。[^paper]

### 实验可信度

论文用多个模型、分支数和步长比较，并包含额外消费级 GPU 实验，但主要结果限定于单机、特定 PCIe 配置和固定输入输出长度。它未证明对在线多用户调度、不同搜索或 verifier，以及不同 GPU 与互连均有相同收益。[^paper]

## 局限与后续工作

- 在相同任务准确率约束下比较 step 长度、搜索宽度与端到端成本，并验证随机采样与调度顺序的可复现性。
- 对照现代推理引擎的 KV 管理，在不同显存预算、动态分支及多请求下测量主机内存、传输量和延迟。

## 相关

- [[LLM-Inference]]、[[KV-Cache-Systems]]：推理过程与 KV 分层管理。
- [[SpecDecodeBench-MLSys26]]：研究多 token 投机解码的评测；本文研究搜索分支的缓存调度，机制不同。
- [[HiSparse-arXiv26]]：同样使用主机与 GPU 的分层 KV，但面向稀疏注意力选择，不能直接等同于 beam 前缀分组。

## 参考文献

[^paper]: Hsing-Ti Wang, Hung-Tso Shiao, Chia-Lin Yang. 《Locality-Aware Beam Scheduling for Efficient Test-Time Compute with a Consumer-grade GPU》. MLSys，2026。[原文](https://proceedings.mlsys.org/paper_files/paper/2026/hash/c74b624843218d9b6713fcf299d6d5e4-Abstract-Conference.html)。
