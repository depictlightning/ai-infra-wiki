---
type: blog
title: "SGLang HiCache: Fast Hierarchical KV Caching with Your Favorite Storage Backends"
author: Zhiqiang Xie
published: 2025-09-10
source_url: https://www.lmsys.org/blog/2025-09-10-sglang-hicache/
reading_status: reviewed
tags: [sglang, llm-serving, kv-cache, storage-hierarchy, lens/kv-cache-offloading]
last_updated: 2026-10-30
---

# SGLang HiCache：分层前缀缓存的工程实践

> **原文**：[SGLang HiCache: Fast Hierarchical KV Caching with Your Favorite Storage Backends](https://www.lmsys.org/blog/2025-09-10-sglang-hicache/) · Zhiqiang Xie · 2025-09-10

本文为原博客的中文整理。原文说明 SGLang 如何把可复用前缀从 GPU 显存扩展到 CPU 内存和外部存储，并介绍数据搬运和缓存控制策略。下文涉及的性能均为原作者或原文列出的社区报告，未在本工作区复现。[^blog]

## 它解决什么问题

SGLang 的 RadixAttention 使用 GPU 内的前缀树管理可复用 KV。多轮对话和长文档增长时，旧前缀可能被逐出 GPU；后续请求若再用这些前缀，就需要重新 prefill。HiCache 用 HiRadixTree 索引 GPU、主机内存中的前缀位置，并由缓存控制器负责较慢层的备份、查询和恢复。外部后端可以提供磁盘或远端存储（原文 Why Hierarchical KV Caching Matters）。[^blog]

典型读取路径是：先匹配已有前缀；若 GPU 有数据则直接复用，若主机有数据则加载到 GPU；外部存储命中时先读取到主机，再送到 GPU。是否等待外部预取完成由策略决定。命中意味着找到可复用的数据，实际执行仍需承担传输、排队和同步时间。[^blog][^design]

## 怎么管理搬运

GPU 注意力内核倾向按模型层组织 KV，存储读写则更适合按页聚合不同层的数据。HiCache 在 GPU 维持按层布局，主机与存储采用按页布局，并在搬运时转换。原博客报告 GPU 辅助 I/O 内核的 CPU–GPU 搬运吞吐最高为常规 `cudaMemcpyAsync` 路径的 3 倍；该数值是原文所测路径的上限，未给出足以推广到所有硬件的完整条件。[^blog]

CPU 命中后可以按层加载，使下一层传输与当前层计算重叠；外部存储访问时间更长且不稳定，控制器可利用排队时间预取，并在请求进入执行时选择继续等待、超时或采用尽力而为的结果。原文还描述写穿、按命中次数选择性写穿、写回三种策略，分别体现备份流量、保留机会和慢层容量的取舍。[^blog]

博客介绍的外部存储接口包括 `get`、`exist`、`set`，并列出 Mooncake、3FS、NIXL 和本地文件后端。接口职责不等于各后端具有相同一致性、持久化或性能；实际行为应查看固定版本的实现。[^blog]

## 和 Strata、HiSparse 的关系

[[Strata-OSDI26]] 是围绕 SGLang 分层上下文缓存的论文：GPU 辅助 I/O、布局和预取与博客有明确重合，论文还更系统地分析缓存加载感知调度及实验边界。论文的 SGLang-HiCache 对照组是其评测时选择的基线，不能等同于所有版本的 HiCache。[^blog][^strata]

[[SGLang-HiSparse]] 则针对活跃请求的原生稀疏注意力解码，将完整历史放在主机并按选择结果取回。两者均跨 GPU 和 CPU 管理 KV，但需要分别看请求生命周期、访问频率与系统配置，不能把两篇博客的性能数字互换。[^hisparse]

## 原文实验与边界

原文报告两类复现入口：8×H20-3e、DeepSeek-R1 与 3FS 的长上下文测试，以及 8×H800、Qwen3-235B 与 Mooncake 的多轮测试。博客概述最高 6 倍吞吐和最多 80% TTFT 降低，具体曲线、请求率和基线应以对应图表和命令为准；原文也引用 Novita AI 与 Ant Group 的部署观察，它们不是同一组实验。[^blog]

使用时应固定 SGLang 提交、后端、页面大小、主机池和预取／写入策略；新版本对混合注意力或其他状态的支持不能由 2025 年博客直接推断。[^design]

## 相关

- [[KV-Cache-Offloading]]、[[KV-Cache-Systems]]
- [[Strata-OSDI26]]、[[LMCache-arXiv25]]、[[Mooncake-FAST25]]、[[SGLang-HiSparse]]

## 参考文献

[^blog]: Zhiqiang Xie. 《SGLang HiCache: Fast Hierarchical KV Caching with Your Favorite Storage Backends》. LMSYS，2025-09-10。[原文](https://www.lmsys.org/blog/2025-09-10-sglang-hicache/)。
[^design]: SGLang. 《HiCache in SGLang》. [设计说明](https://www.sglang.io/blog/hicache)，访问于 2026-10-30。
[^strata]: Zhiqiang Xie 等. 《Strata: Hierarchical Context Caching for Long Context Language Model Serving》. OSDI，2026。[论文](https://www.usenix.org/conference/osdi26/presentation/xie-zhiqiang)。
[^hisparse]: Zhiqiang Xie、Zhangheng Huang、Tingwei Huang. 《HiSparse: Turbocharging Sparse Attention with Hierarchical Memory》. LMSYS，2026-04-10。[原文](https://www.lmsys.org/blog/2026-04-10-sglang-hisparse/)。
