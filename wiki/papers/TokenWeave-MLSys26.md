---
type: paper
name: TokenWeave
full_title: "TokenWeave: Efficient Compute-Communication Overlap for Distributed LLM Inference"
authors: [Raja Gond, Nipun Kwatra, Ramachandran Ramjee]
venue: MLSys
year: 2026
tags: [llm-inference, tensor-parallelism, communication-overlap, kernel-fusion, area/llm-inference]
source_pdf: "[[e4da3b7fbbce2345d7772b0674a318d5.pdf]]"
source_md: "[[e4da3b7fbbce2345d7772b0674a318d5]]"
review_status: complete
evidence_level: full-text
last_reviewed: 2026-10-28
---

# TokenWeave：让 LLM 张量并行的通信与计算重叠（MLSys 2026）

> **原题**：TokenWeave: Efficient Compute-Communication Overlap for Distributed LLM Inference

> **一句话总结**：TokenWeave 按 GPU 执行波次拆分 token，并融合通信与 RMSNorm，让一个分片的通信与另一个分片的计算重叠，同时减少张量并行中重复的归一化计算和内存访问。[^paper]

**阅读状态**：已阅读与会议正式 PDF 一致的 MinerU 全文，核对方法、实验条件和消融结果。[^paper]

## 问题与动机

张量并行将模型计算分到多张 GPU，但每层需要同步中间结果。作者在 8 张 H100 上测得，AllReduce 通信占所测模型推理延迟的 9%–23%，RMSNorm 又占 4%–9%（图 1）。将 batch 拆小可给通信和计算创造重叠机会，但拆分本身可能降低 GPU 利用率，抵消省下的通信时间。[^paper]

低延迟服务常使用较小的 prefill chunk；因此作者关心的不只是「能否重叠」，而是约 1K token 的单次迭代中，拆分和通信占用 SM 的代价是否仍可接受（§1–2）。[^paper]

## 关键观察 / 隐含假设

- **通信后仍有重复工作**：常规张量并行先 AllReduce，再在每张 GPU 上对相同结果做 RMSNorm；合并通信和归一化有机会减少重复计算及 HBM 读写（§3–4）。[^paper]
- **拆分边界影响执行波次**：两个等长分片未必比一个整体运行得快；实际收益依赖 kernel 形状和 SM 可用量（§4.1）。[^paper]
- **重叠依赖硬件和负载**：NVSwitch、NVSHARP / Multimem、模型类型及 batch 大小都会影响通信成本和可隐藏的计算时间（§4–5）。[^paper]

## 核心方法

### 按波次拆分 token

TokenWeave 将一次迭代的 token 分成两个子批次，通过独立的计算流和通信流交错执行。注意力存在前缀依赖，系统采用 chunked attention，并保证被拆开序列的前缀先执行，不能任意交换两个分片（§4）。[^paper]

拆分点由 GPU 波次决定。论文举例：300 个 CTA 在 132 个 SM 上原需三波；平分为 150 和 150 则共需四波，而拆为 132 和 168 仍是三波。这里假定每个 CTA 独占一个 SM，是解释波次量化的简化模型，并非所有 kernel 的真实占用情况（§4.1.1）。[^paper]

### 重排与融合通信、残差和归一化

作者把 AllReduce 分解成 ReduceScatter 与 AllGather：先按 token 归约，让每张 GPU 对自己负责的完整 hidden vector 执行残差相加与 RMSNorm，再收集结果。这样每个 token 的归一化不必在所有 GPU 重复计算。单独调用多次通信仍有中间写回成本，因此实现将归约、归一化和多播结合到定制 kernel，利用 Multimem/NVSHARP 并尽量在寄存器中处理（§4.2–4.3）。[^paper]

该通信 kernel 在所测 8×H100 系统上使用约 2–8 个 SM，给并行计算留出资源；这依赖对应 GPU 和互连能力，不能当作所有 GPU 集群的通用配置（§4.3）。[^paper]

## 设计取舍

- **重叠换拆分开销**：当分片太小或计算本身受内存限制时，额外调度和波次可能超过隐藏的通信时间。
- **融合换可移植性**：优化依赖特定模型操作顺序、定制 kernel 和硬件互连；支持其他归一化算子或硬件需要重新实现和评测。
- **前向吞吐与服务体验不同**：较大 prefill chunk 能提高吞吐，也可能推高 token 间延迟；论文中的收益须结合 chunk 配置理解。

## 实验与结果

原型基于 vLLM 0.8.5 V1、PyTorch 2.6 和 CUDA 12.4；主要实验在 8×H100 DGX 上使用 Llama-3.3-70B、Qwen2.5-72B 和 Mixtral-8x22B，并使用 ShareGPT、arXiv 与固定长度负载。论文关闭 prefix caching，端到端计时不包含 CPU detokenization（§5.1）。[^paper]

| 结果 | 条件与边界 |
|---|---|
| 混合 prefill/decode 吞吐约提升 1.19× | 8×H100、dense 模型与论文负载；chunk 默认为 2K，对比 vLLM-Multimem。改变 chunk 时 Llama-3.3-70B 获得约 1.14–1.26×（§5.2.1、图 11–12）。 |
| 单次 prefill 前向延迟加速 1.16–1.28× | 8×H100、dense 模型、1K token 起，速度比为基线延迟除以 TokenWeave 延迟；不是端到端请求延迟降低同样比例（§5.2.2、图 13）。 |
| 仅融合的收益 | 不做重叠的 fuse-only 在所测模型得到 1.04–1.09× 加速；Mixtral 的 1K/2K 负载中，完整拆分反而有开销，适合仅启用融合（§5.2.2、§5.2.5）。 |
| 与 TileLink 对比 | 由于 TileLink 未接入同一服务栈，实验只比较 Llama 单层；不能把单层倍数当成端到端倍数（§5.2.3）。 |
| B200 小批次 decode | 附录 C 的 8×B200 实验仅启用融合，加速约 1.01–1.05×；该实验使用 vLLM 0.14.1，不能与 H100 结果视为完全同配置对照。 |

论文还设置 vLLM-nocomm：移除通信后的性能参照**不能产生正确模型输出**。TokenWeave 在部分设置快于它，是因为同时减少了 RMSNorm 重复计算和访存，不能据此说通信具有负开销或基线是正确系统（§5.1–5.2）。[^paper]

## 论断—证据表

| 论断 | 证据 | 评测边界 | 置信度 |
|---|---|---|---|
| 波次感知拆分可降低小批次拆分的额外成本 | §4.1、图 9 与图 17 | 依赖具体 kernel 形状和 GPU 占用 | 中高 |
| 融合和重叠可改善所测张量并行推理 | §5.2、图 11–13、16 | 特定 vLLM 版本、模型与 NVSwitch 系统 | 中高 |
| 全部模型及小批次都适合完整重叠 | §5.2.2 | Mixtral 小负载存在反例 | 低 |

## 批判性分析

### 论证链条

论文先测量通信和归一化开销，再用波次感知拆分、归一化重排和融合共同降低成本。fuse-only 与完整实现的消融有助于区分融合和重叠的贡献，但这些收益会随模型、批次和硬件改变。[^paper]

### 假设压力测试

TP 度数降低时通信占比也可能下降；MoE 的 token 分散到不同专家后，小分片更容易受内存瓶颈影响。若通信和计算争用 SM、内存带宽或互连，理论上的并行也可能产生额外延迟，应按目标负载测量。[^paper]

### 实验可信度

作者提供真实 trace、不同 chunk 和融合消融；但 TileLink 是单层对比，NanoFlow 使用不同服务栈并移植到 H100，跨系统相对收益并不等价于同栈端到端比较。评估线上部署还需考虑 prefix cache、detokenization、排队与尾延迟。[^paper]

## 局限与后续工作

- 在目标 GPU、互连与现有 vLLM 版本上重新验证正确性、图编译兼容性以及不同 TP 度数的收益。
- 对比不同 prefill/decode 比例、MoE 路由和动态请求到达下的吞吐、TTFT 与 TPOT，而非只看单轮前向。

## 相关

- [[LLM-Inference]]：算子融合、通信与并行优化。
- [作者代码](https://github.com/microsoft/tokenweave)：了解实现依赖与后续变更。

## 参考文献

[^paper]: Raja Gond 等. 《TokenWeave: Efficient Compute-Communication Overlap for Distributed LLM Inference》. MLSys，2026。[原文](https://proceedings.mlsys.org/paper_files/paper/2026/hash/73ba81c7b25134a559c8a9c39ec1a4c3-Abstract-Conference.html)。
