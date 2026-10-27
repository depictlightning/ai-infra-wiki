---
type: theme
topic: Offloading-and-Tiered-Storage
theme_kind: lens
member_tag: lens/offloading-and-tiered-storage
paper_count: 14
first_generated: 2026-10-27
last_updated: 2026-10-27
tags: [topic-overview, optimization-method]
---

# 卸载与分层存储

本专题按优化方法组织站内论文。它可与 [[KV-Cache-Systems]]、[[LLM-Serving]] 等按研究对象组织的专题交叉；一篇论文可属于多个方法专题。

## 纳入范围

把状态、数据或处理工作放到 CPU、主机内存、CXL、SSD、其他节点等层级，并显式处理层间访问与容量。单纯的显存内重新分配不计入。核心名单按现有论文笔记人工确认，标签用于导航，不自动决定成员。只有摘要的论文标为「仅公开摘要」。

## 核心论文

- [[Chen-LLMDataPipelines-OSDI26]]
- [[Cocoon-OSDI26]]
- [[CoPilotIO-OSDI26]]
- [[DirectKV-OSDI26]]
- [[DualPath-SIGCOMM26]]
- [[ECHO-OSDI26]]
- [[FlowANN-OSDI26]]
- [[KAIROX-OSDI26]]
- [[KTransformers-SOSP25]]
- [[Nixie-OSDI26]]
- [[Strata-OSDI26]]
- [[Umap-OSDI26]]
- [[Wang-LocalMoEInference-OSDI26]]
- [[Weave-OSDI26]]

## 主题导读

[[ECHO-OSDI26]] 与 [[DirectKV-OSDI26]] 都把 KV 放到主机侧，但前者按需取回选中条目，后者利用高速互连直接读取；[[Strata-OSDI26]] 管理多层缓存中的命中和加载，[[FlowANN-OSDI26]] 分层放置图边。节省 GPU 驻留常会改变搬运量与关键路径，不能只看显存数字。
