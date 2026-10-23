---
type: theme
topic: KV-Cache-Systems
theme_kind: area
member_tag: area/kv-cache-systems
candidate_tags: [kv-cache]
paper_count: 5
first_generated: 2026-10-23
last_updated: 2026-10-23
tags: [topic-overview, kv-cache, llm-serving]
---

# KV cache 系统专题

长上下文推理会反复使用历史 KV cache。以下论文从存放位置、读取路径和请求分配三个位置处理这个问题。本页先给出阅读路线；更细的实验对照与设计空间留待后续补充。

## 纳入范围

本专题收录以 **KV cache 的存放、搬运或复用调度** 为主要系统问题的论文。`kv-cache` 标签用于发现候选，`## 核心论文` 是人工确认的成员名单；同样带有该标签、但主要研究别的问题的论文不会自动加入。成员可以同时属于其他专题。

## 核心论文

- [[Strata-OSDI26]] — 处理分层缓存命中后的碎片搬运、布局转换和加载时机。
- [[ECHO-OSDI26]] — 为原生稀疏注意力模型把大部分 KV cache 放入主机内存，按需预取被选 token。
- [[DirectKV-OSDI26]] — 在高带宽 CPU–GPU 互连上直接读取主机内存中的 KV，减少显存暂存区。
- [[LMetric-OSDI26]] — 在请求路由时同时考虑前缀复用与实例负载。
- [[DualPath-SIGCOMM26]] — 利用解码节点的存储网卡读取旧 KV，再经计算网络送往预填充节点。

## 主题导读

先问 **KV 在哪里**。[[ECHO-OSDI26]] 用主机内存扩大可容纳的上下文，[[DirectKV-OSDI26]] 探索 GPU 直接访问主机内存。两篇都依赖具体模型与硬件：前者的稀疏注意力决定预取哪些 token，后者的高速 CPU–GPU 互连决定直读是否划算。

再问 **命中后怎样送达计算设备**。[[Strata-OSDI26]] 处理单条分层路径上的小页传输，[[DualPath-SIGCOMM26]] 在预填充与解码两侧分担外部存储读取。[[LMetric-OSDI26]] 则在请求进入实例前权衡前缀复用与排队负载。它们优化的瓶颈位置不同；后续比较应统一工作负载、硬件拓扑与延迟指标，不能直接排列论文报告的加速倍数。
