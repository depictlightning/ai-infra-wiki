---
type: theme
topic: Operator-and-Compiler-Optimization
theme_kind: lens
member_tag: lens/operator-and-compiler-optimization
paper_count: 15
first_generated: 2026-10-27
last_updated: 2026-10-27
tags: [topic-overview, optimization-method]
---

# 算子、编译与执行加速

本专题按优化方法组织站内论文。它可与 [[KV-Cache-Systems]]、[[LLM-Serving]] 等按研究对象组织的专题交叉；一篇论文可属于多个方法专题。

## 纳入范围

通过算子实现、编译映射、融合、运行时执行图或单设备内核布局提高执行效率。纯粹的集群放置不计入。核心名单按现有论文笔记人工确认，标签用于导航，不自动决定成员。只有摘要的论文标为「仅公开摘要」。

## 核心论文

- [[ADAngel-OSDI26]]
- [[Ave-SOSP26]]
- [[CoPilotIO-OSDI26]]
- [[DirectKV-OSDI26]]
- [[GraCE-OSDI26]]
- [[KTransformers-SOSP25]]
- [[MeldKernel-SOSP26]]
- [[MoonBright-OSDI26]]
- [[MPK-OSDI26]]
- [[SANI-OSDI26]]
- [[Syncopate-OSDI26]]
- [[TileLoom-OSDI26]]
- [[Twill-OSDI26]]
- [[VTC-OSDI26]]
- [[Wang-LocalMoEInference-OSDI26]]

## 主题导读

[[GraCE-OSDI26]]、[[MPK-OSDI26]] 与 [[Twill-OSDI26]] 分别处理执行图、巨型内核和流水调度；[[VTC-OSDI26]] 则消除不必要的中间数据搬运。共同问题是如何把理论上的并行性落实到设备执行，但限制可能来自寄存器、同步、访存或形状变化。
