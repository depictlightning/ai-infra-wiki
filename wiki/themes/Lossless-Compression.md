---
type: theme
topic: Lossless-Compression
theme_kind: lens
member_tag: lens/lossless-compression
paper_count: 1
first_generated: 2026-10-27
last_updated: 2026-10-27
tags: [topic-overview, optimization-method]
---

# 无损压缩

本专题按优化方法组织站内论文。它可与 [[KV-Cache-Systems]]、[[LLM-Serving]] 等按研究对象组织的专题交叉；一篇论文可属于多个方法专题。

## 纳入范围

压缩后能精确恢复原始数据，并明确处理编码、解码或存储表示成本。准确率约束下的模型合并不属于无损压缩。核心名单按现有论文笔记人工确认，标签用于导航，不自动决定成员。只有摘要的论文标为「仅公开摘要」。

## 核心论文

- [[TensorDex-SOSP26]]

## 主题导读

目前明确符合的是 [[TensorDex-SOSP26]]：以 tensor 为粒度选择参考并保存无损差分。[[SANDHI-SOSP26]] 的模型合并受准确率约束，不能据此认定为位级无损。当前只有一篇可确认成员，后续扩充时须核对恢复语义。
