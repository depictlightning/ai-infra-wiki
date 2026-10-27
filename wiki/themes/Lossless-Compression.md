---
type: theme
topic: Lossless-Compression
theme_kind: lens
member_tag: lens/lossless-compression
paper_count: 2
first_generated: 2026-10-27
last_updated: 2026-10-27
tags: [topic-overview, optimization-method]
---

# 无损压缩

本专题按优化方法组织站内论文，可与 [[KV-Cache-Systems]]、[[LLM-Serving]] 等系统领域专题交叉。以下是本站依据论文页「核心方法」作的归类；明确的副机制也可纳入，理由在条目中说明。

## 纳入范围

收录能精确恢复原始表示的编码、差分、去重或紧凑布局。必须说明无损保证覆盖的对象；模型输出质量近似不变、搜索召回相近和语义摘要均不等于位级无损。核心名单由人工确认，标签不自动决定成员；「仅公开摘要」表示尚未取得全文。

## 核心论文

- [[FlowANN-OSDI26]] — 用组内局部 ID 和互补行布局精确表示图边，减少编号字节与 padding。
- [[TensorDex-SOSP26]] — 对 tensor 精确去重，再对原始位模式做可还原的无损差分编码。

## 主题导读

[[TensorDex-SOSP26]] 压缩模型权重，[[FlowANN-OSDI26]] 压缩图边编号与布局。后者的无损性只针对原有图边的存储编码，不代表向量 PQ 或近似搜索本身无损。[[SANDHI-SOSP26]] 允许准确率约束下的组件合并，因此属于状态共享，不列入无损压缩。
