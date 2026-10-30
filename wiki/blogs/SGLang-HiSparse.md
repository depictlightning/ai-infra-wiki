---
type: blog
title: "HiSparse: Turbocharging Sparse Attention with Hierarchical Memory"
author: "Zhiqiang Xie, Zhangheng Huang, Tingwei Huang"
published: 2026-04-10
source_url: https://www.lmsys.org/blog/2026-04-10-sglang-hisparse/
reading_status: reviewed
tags: [sglang, llm-inference, kv-cache, offloading, sparse-attention, lens/kv-cache-offloading]
last_updated: 2026-10-30
---

# SGLang HiSparse：稀疏注意力的分层内存实践

> **原文**：[HiSparse: Turbocharging Sparse Attention with Hierarchical Memory](https://www.lmsys.org/blog/2026-04-10-sglang-hisparse/) · Zhiqiang Xie、Zhangheng Huang、Tingwei Huang · 2026-04-10

本页是 SGLang 官方博客的中文整理。详细论文机制与评测另见 [[HiSparse-arXiv26]]；该论文是 2026 年 8 月的版本，博客与论文不是同一次实验，也不能将论文数据直接当成博客配置的结果。[^blog][^paper]

## 问题与机制

原生稀疏注意力在每个解码步骤只使用索引器选出的少量历史 KV，但没有被本次选中的历史仍可能在未来需要。若将全部历史留在 GPU，并发和上下文长度仍受显存容量限制。HiSparse 将不常用的历史放到主机，GPU 保留有限的热缓冲；被索引器选中但缓冲未命中的项需要从主机加载。[^blog]

官方博客介绍的专用 CUDA 内核会检查选中项、按 LRU 选择可替换槽、更新页表并从主机取回缺失项。缓冲容量与替换策略影响未命中数量，同时主机容量、带宽和同步成本也影响结果；不能只统计 GPU 的 KV 空间。[^blog]

## 博客里的实验

在 GLM-5.1-FP8、32K 输入、8K 输出、单机 8×H200 的 P/D 混部测试中，原文称 256 并发时吞吐超过不启用 HiSparse 的基线 3 倍；低并发时额外加载反而产生开销。原文另给出两台 H20 节点的 P/D 分离测试，报告所测长上下文设置下最高 5 倍吞吐。两组数字使用不同部署，不能相互替代。[^blog]

缓冲区实验使用 DeepSeek-V3.2、LongBenchV2 和 top-k 为 2048 的选择，博客对比 2048 与 4096 槽及替换策略。容量增大和 LRU 改变同时发生，不能把未命中变化全部归因于某一种策略。[^blog]

## 与相邻工作的关系

- [[SGLang-HiCache]] 的重点是保存和恢复可复用前缀；HiSparse 的重点是运行中稀疏注意力的历史 KV 工作集。它们都使用分层内存，目标和访问模式不同。[^blog][^hicache]
- [[ECHO-OSDI26]] 也研究原生稀疏模型的主机侧 KV 管理；ECHO 的预取可以预测，但在注意力计算前补齐真实选中项。两者的硬件与实现不同，论文倍数不能直接排序。[^echo]
- [[vLLM-Hybrid-HiSparse]] 是 vLLM 的运行时方案：显存足够时可保留普通 GPU 驻留页，压力增加时逐步转向混合驻留。博客中的模型、实现和实验应与 SGLang 版本分别阅读。[^vllm]

博客发布时将 HiSparse 描述为实验性功能，并列出 DeepSeek-V3.2、GLM-5.1 等模型；当时的支持列表不应作为当前版本的完整兼容性保证。原生稀疏模型本身与完整注意力模型不同，保持它的选择语义不等于恢复完整注意力结果。[^blog]

## 相关

- [[KV-Cache-Offloading]]、[[HiSparse-arXiv26]]、[[SGLang-HiCache]]
- [[ECHO-OSDI26]]、[[vLLM-Hybrid-HiSparse]]

## 参考文献

[^blog]: Zhiqiang Xie、Zhangheng Huang、Tingwei Huang. 《HiSparse: Turbocharging Sparse Attention with Hierarchical Memory》. LMSYS，2026-04-10。[原文](https://www.lmsys.org/blog/2026-04-10-sglang-hisparse/)。
[^paper]: Zhiqiang Xie 等. 《HiSparse: Scaling Sparse-Attention Decoding with Hierarchical KV Cache Management》. arXiv:2608.07009v1，2026。[原文](https://arxiv.org/abs/2608.07009v1)。
[^hicache]: Zhiqiang Xie. 《SGLang HiCache: Fast Hierarchical KV Caching with Your Favorite Storage Backends》. LMSYS，2025。[原文](https://www.lmsys.org/blog/2025-09-10-sglang-hicache/)。
[^echo]: Guangda Liu 等. 《ECHO: Efficient KV Cache Offloading with Lossless Prefetching for Serving Native Sparse Attention LLMs》. OSDI，2026。[论文](https://www.usenix.org/conference/osdi26/presentation/liu-guangda)。
[^vllm]: vLLM Team. 《GLM 5.3 Optimizations, Part 1: Hybrid HiSparse Offloading in vLLM》. 2026-09-08。[原文](https://vllm.ai/blog/2026-09-08-glm53-part1-hybrid-sparse-offloading)。
