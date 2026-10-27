---
type: theme
topic: KV-Cache-Systems
theme_kind: area
member_tag: area/kv-cache-systems
candidate_tags: [kv-cache]
paper_count: 8
first_generated: 2026-10-23
last_updated: 2026-10-27
tags: [topic-overview, kv-cache, llm-serving]
---

# KV cache 系统专题

长上下文推理会反复使用历史 KV cache。以下论文从模型结构、缓存驻留、数据搬运和请求分配等位置处理这个问题。本页先给出阅读路线；更细的实验对照与设计空间留待后续补充。

## 纳入范围

本专题收录以 **KV cache 的跨层共享、驻留、搬运或复用调度** 为主要问题的论文。`kv-cache` 标签用于发现候选，`## 核心论文` 是人工确认的成员名单；同样带有该标签、但主要研究别的问题的论文不会自动加入。成员可以同时属于其他专题。模型结构论文和运行时系统论文的评测指标不同，不能直接比较吞吐收益。

## 核心论文

- [[UnlimitedOCR2-TechReport26]] — 面向长文档 OCR，限制视觉参考范围，并通过页面分块与边界页 KV 复用控制显存驻留。
- [[HySparse2-arXiv26]] — 通过跨解码器桥接生成各层 KV，结合 token 级稀疏选择和强制局部窗口缩短预填充路径。
- [[HySparse-arXiv26]] — 让稀疏注意力层复用前一完整层的块索引与 KV，局部滑窗另存 KV；论文测了模型质量，未测服务性能。
- [[Strata-OSDI26]] — 处理分层缓存命中后的碎片搬运、布局转换和加载时机。
- [[ECHO-OSDI26]] — 为原生稀疏注意力模型把大部分 KV cache 放入主机内存，按需预取被选 token。
- [[DirectKV-OSDI26]] — 在高带宽 CPU–GPU 互连上直接读取主机内存中的 KV，减少显存暂存区。
- [[LMetric-OSDI26]] — 在请求路由时同时考虑前缀复用与实例负载。
- [[DualPath-SIGCOMM26]] — 利用解码节点的存储网卡读取旧 KV，再经计算网络送往预填充节点。

## 相关博客

- [[DeepSeek-V4.1-Flash-Serving]] — DeepSeek 对 Block KV 与 SWA KV 采用不同存储寿命，并说明共享 3FS 的前缀复用与取回路径。

## 延伸阅读

- [2026年了，关于大模型kv cache的优化有什么新的方法吗？](https://www.zhihu.com/question/2081079592870212699) — 用户推荐的 KV cache 讨论入口；回答摘要已核对，全文待读。

## 主题导读

先问 **模型本身必须生成和保存多少 KV**。[[HySparse-arXiv26]] 在模型预训练时安排少数完整注意力层，后续稀疏层共享其长期 KV；[[HySparse2-arXiv26]] 进一步让后半模型的完整层从前半隐藏状态分别投影出各自的 KV，从而缩短预填充缓存构建路径。第二代对照实验还改变了 KV 头数与维度，报告的缓存差异不能全部归因于跨层桥接。两篇都改变了模型架构，其成本数据不能直接当作线上服务加速。[^hysparse][^hysparse2]

[[UnlimitedOCR2-TechReport26]] 面向多页 OCR，通过视觉块选择减少单步读取，并用页面分块、边界页复用和旧块释放控制 KV 驻留；其跨页上下文能力受分块范围限制。它与 HySparse 系列的模型、工作负载和测量指标不同。

再看 **KV 在哪里**。[[ECHO-OSDI26]] 用主机内存扩大可容纳的上下文，[[DirectKV-OSDI26]] 探索 GPU 直接访问主机内存。两篇都依赖具体模型与硬件：前者的稀疏注意力决定预取哪些 token，后者的高速 CPU–GPU 互连决定直读是否划算。

再问 **命中后怎样送达计算设备**。[[Strata-OSDI26]] 处理单条分层路径上的小页传输，[[DualPath-SIGCOMM26]] 在预填充与解码两侧分担外部存储读取。[[LMetric-OSDI26]] 则在请求进入实例前权衡前缀复用与排队负载。它们优化的瓶颈位置不同；后续比较应统一工作负载、硬件拓扑与延迟指标，不能直接排列论文报告的加速倍数。

## 参考文献

[^hysparse]: Yizhao Gao 等. 《HySparse: A Hybrid Sparse Attention Architecture with Oracle Token Selection and KV Cache Sharing》. arXiv 计算机科学预印本，2026-02-03。[原文](https://arxiv.org/abs/2602.03560v1)。
[^hysparse2]: Jianyu Wei 等. 《HySparse2: Hybrid Sparse Attention with Two-Level KV Sharing》. arXiv，2026-09-22。[原文](https://arxiv.org/abs/2609.26368v1)。
