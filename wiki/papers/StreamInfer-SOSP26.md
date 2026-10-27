---
type: paper
name: StreamInfer
full_title: "StreamEP: Straggler-Tolerant MoE Decoding without Communication Barriers"
authors: [Yizhuo Liang, Shaoyu Wang, Jaeyong Song, Yanqi Zhou, Geon-Woo Kim, et al.]
venue: SOSP
year: 2026
tags: [moe-inference, expert-parallelism, communication, stragglers, lens/communication-and-parallelism, mechanism/communication-overhead, lens/scheduling-and-resource-management]
source_pdf: ""
source_md: ""
review_status: abstract-only
evidence_level: abstract
last_reviewed: 2026-10-22
---

# StreamEP / StreamInfer：消除 MoE 解码通信屏障（SOSP 2026）

> **原题**：StreamEP: Straggler-Tolerant MoE Decoding without Communication Barriers

> **证据边界**：未取得全文；仅据会议题录和公开摘要。这里用论文题名 StreamEP 指问题与贡献，用摘要中的系统名 StreamInfer 指实现，二者不能混成两篇。设计与评测数字未核对图表。

## 问题与动机

MoE 专家并行解码需要将 token 分发到远端专家并收回结果。传统全局通信屏障会让较慢专家或链路拖住全部请求；在商品网络和多请求混合时尤其明显。摘要介绍 StreamInfer：每层维护 token 队列，用碎片整理调度器重新组织待处理 token，并以两阶段通信器配合普通 NCCL，让已就绪的 token 继续推进。它与 [[MeldKernel-SOSP26]] 处理单 GPU 内不同 SM 的长尾不同，关注跨 GPU 专家并行的等待。

## 关键观察 / 隐含假设

- 传统跨专家屏障把全层进度绑在最慢分支上，异步队列的价值取决于专家执行和网络时延有足够偏斜。
- 多请求并发提供可交错 token；若只有一条短请求，队列可能没有可用的替代工作。

## 核心方法

公开摘要提到每层 token 队列、碎片整理调度和基于普通 NCCL 的两阶段通信。现有证据不足以说明队列何时重排、如何保证输出次序或处理故障，因此此处只记录设计轮廓。

## 实验与结果

公开摘要称在 **16 张 L40S** 和 **16 张 A100**、**200 Gbps** 网络的配置上，相对传统专家并行解码方案，吞吐最高 **1.7 倍**。这是最高值；没有全文无法确认模型大小、专家偏斜、输出长度、批次规模、P99 延迟及通信开销。

## 批判性分析

### 论证链条

去屏障与抗长尾在机制上吻合；1.7 倍峰值吞吐能说明可行性，但不能单独证明队列和两阶段通信各自的贡献。

### 假设压力测试

低并发、专家负载均衡良好或网络非常快时，跨请求交错空间可能小；短序列还可能被队列管理开销主导。

### 实验可信度

两类 16-GPU 商品硬件有现实意义，但摘要缺少偏斜分布、对照实现、尾延迟和正确性消融。

- 队列能否减少屏障等待，取决于请求间有足够可交错 token；低并发时收益可能下降。
- 两阶段通信会增加状态与故障恢复复杂度，尤其是 token 乱序与专家状态一致性。摘要没有给出这些细节。
- 公开论文列表中有一处 StreamEP 摘要误配到另一篇 MoE 论文，因此这里只使用标题相符的摘要线索；全文缺失前不据此推断具体协议证明。

## 局限与后续工作

- 获得全文后复核专家偏斜、并发、输出长度对 1.7 倍峰值的影响，特别测 P99 和 token 次序正确性。

## 相关

- **同会议**：[[SOSP-2026]]；**相邻系统**：[[MeldKernel-SOSP26]]、[[Janus-SOSP26]]。
- **会议记录**：[SOSP DOI](https://doi.org/10.1145/3830418.3843869)。
