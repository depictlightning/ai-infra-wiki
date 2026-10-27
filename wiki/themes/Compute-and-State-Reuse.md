---
type: theme
topic: Compute-and-State-Reuse
theme_kind: lens
member_tag: lens/compute-and-state-reuse
paper_count: 13
first_generated: 2026-10-27
last_updated: 2026-10-27
tags: [topic-overview, optimization-method]
---

# 计算与状态复用

本专题按优化方法组织站内论文。它可与 [[KV-Cache-Systems]]、[[LLM-Serving]] 等按研究对象组织的专题交叉；一篇论文可属于多个方法专题。

## 纳入范围

复用已有的前缀、权重、KV、中间表示或恢复状态，避免重复生成或重复装载；需说明被复用的状态和一致性条件。核心名单按现有论文笔记人工确认，标签用于导航，不自动决定成员。只有摘要的论文标为「仅公开摘要」。

## 核心论文

- [[ACDC-SOSP26]]
- [[Anchor-SOSP26]] — 仅公开摘要。
- [[Cocoon-OSDI26]]
- [[ECHO-OSDI26]]
- [[HySparse-arXiv26]]
- [[HySparse2-arXiv26]]
- [[LMetric-OSDI26]]
- [[llmovoice-SOSP26]]
- [[SANDHI-SOSP26]] — 仅公开摘要。
- [[Strata-OSDI26]]
- [[TensorDex-SOSP26]]
- [[TensorHub-SOSP26]]
- [[UnlimitedOCR2-TechReport26]]

## 主题导读

[[ACDC-SOSP26]] 与 [[LMetric-OSDI26]] 依赖请求前缀复用，[[HySparse-arXiv26]] 和 [[HySparse2-arXiv26]] 在模型结构中共享 KV，[[TensorDex-SOSP26]] 在模型仓库中复用参考 tensor。[[llmovoice-SOSP26]] 检索并复用跨轮会话状态，处理的是应用上下文，不是 GPU KV cache。[[Anchor-SOSP26]] 保留故障后仍有效的显存状态，其边界取决于设备本身是否健康。这些复用发生在不同生命周期。
