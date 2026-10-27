---
type: theme
topic: Sparsification
theme_kind: lens
member_tag: lens/sparsification
paper_count: 7
first_generated: 2026-10-27
last_updated: 2026-10-27
tags: [topic-overview, optimization-method]
---

# 稀疏化与加速

本专题按优化方法组织站内论文。它可与 [[KV-Cache-Systems]]、[[LLM-Serving]] 等按研究对象组织的专题交叉；一篇论文可属于多个方法专题。

## 纳入范围

通过筛掉当前步骤不必处理的数据、候选或连接来减少计算与访问；只有工作明确使用选择或剪除机制才纳入。核心名单按现有论文笔记人工确认，标签用于导航，不自动决定成员。只有摘要的论文标为「仅公开摘要」。

## 核心论文

- [[Drs-NAS-OSDI26]]
- [[ECHO-OSDI26]]
- [[Helmsman-OSDI26]]
- [[HySparse-arXiv26]]
- [[HySparse2-arXiv26]]
- [[KAIROX-OSDI26]]
- [[UnlimitedOCR2-TechReport26]]

## 主题导读

[[HySparse-arXiv26]] 与 [[HySparse2-arXiv26]] 改变模型注意力及跨层 KV 构造；[[ECHO-OSDI26]] 利用原生稀疏注意力减少 GPU 中需驻留的历史 KV。[[Drs-NAS-OSDI26]] 跳过候选架构的完整训练，[[Helmsman-OSDI26]] 根据查询预测扫描范围。它们删减的是不同对象，不能用同一个加速倍数比较。
