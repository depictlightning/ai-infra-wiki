---
type: theme
topic: Scheduling-and-Resource-Management
theme_kind: lens
member_tag: lens/scheduling-and-resource-management
paper_count: 29
first_generated: 2026-10-27
last_updated: 2026-10-27
tags: [topic-overview, optimization-method]
---

# 调度与资源管理

本专题按优化方法组织站内论文。它可与 [[KV-Cache-Systems]]、[[LLM-Serving]] 等按研究对象组织的专题交叉；一篇论文可属于多个方法专题。

## 纳入范围

决定请求、任务或模型在何时、何处、以多少资源运行，包含批次、弹性、放置与共享；只提供性能画像而不做资源决策的论文不纳入。核心名单按现有论文笔记人工确认，标签用于导航，不自动决定成员。只有摘要的论文标为「仅公开摘要」。

## 核心论文

- [[ACDC-SOSP26]]
- [[Alibaba-ASI-OSDI26]]
- [[BatchGen-OSDI26]]
- [[Bedrock-WLM-SOSP26]] — 仅公开摘要。
- [[DCP-OSDI26]]
- [[DiFlow-SOSP26]]
- [[DynaRL-OSDI26]]
- [[EcoServe-OSDI26]]
- [[EnerTune-SOSP26]] — 仅公开摘要。
- [[Hetu-v2-OSDI26]]
- [[Janus-SOSP26]]
- [[KAIROX-OSDI26]]
- [[Kareus-OSDI26]]
- [[LMetric-OSDI26]]
- [[MorphX-SOSP26]] — 仅公开摘要。
- [[Murakkab-OSDI26]]
- [[Nixie-OSDI26]]
- [[OpenTela-OSDI26]]
- [[Prism-OSDI26]]
- [[Quota-Marketplace-OSDI26]]
- [[RLinf-OSDI26]]
- [[RollArt-OSDI26]]
- [[SANI-OSDI26]]
- [[Seer-OSDI26]]
- [[Sereno-OSDI26]]
- [[SPEX-OSDI26]]
- [[Tessera-OSDI26]]
- [[Wang-LocalMoEInference-OSDI26]]
- [[Weave-OSDI26]]

## 主题导读

[[LMetric-OSDI26]]、[[DCP-OSDI26]] 与 [[Janus-SOSP26]] 分别调请求、阶段和模型副本；[[DynaRL-OSDI26]]、[[RollArt-OSDI26]]、[[Weave-OSDI26]] 处理训练工作流阶段间的资源变化。调度尺度不同，评估时须同时看等待、资源占用和 SLO。
