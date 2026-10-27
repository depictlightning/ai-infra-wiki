---
type: theme
topic: Speculative-Decoding-and-Multi-Token
theme_kind: lens
member_tag: lens/speculative-decoding-and-multi-token
paper_count: 6
first_generated: 2026-10-27
last_updated: 2026-10-27
tags: [topic-overview, optimization-method]
---

# 投机解码与多 Token 生成

本专题按优化方法组织站内论文。它可与 [[KV-Cache-Systems]]、[[LLM-Serving]] 等按研究对象组织的专题交叉；一篇论文可属于多个方法专题。

## 纳入范围

通过生成候选、提前执行后续分支或批量验证缩短串行生成路径；同时记录候选失败、回滚和验证的成本。核心名单按现有论文笔记人工确认，标签用于导航，不自动决定成员。只有摘要的论文标为「仅公开摘要」。

## 核心论文

- [[ACDC-SOSP26]]
- [[DSpark-arXiv26]]
- [[LLM-42-SOSP26]]
- [[Seer-OSDI26]]
- [[Sereno-OSDI26]]
- [[SPEX-OSDI26]]

## 主题导读

[[DSpark-arXiv26]]、[[Seer-OSDI26]] 与 [[ACDC-SOSP26]] 调整草稿或验证的工作量；[[SPEX-OSDI26]] 提前探索可能的推理分支。[[LLM-42-SOSP26]] 的验证目标是确定性，[[Sereno-OSDI26]] 利用草稿阶段形成设备让出点。因此“使用 speculation”并不意味着评测目标相同。
