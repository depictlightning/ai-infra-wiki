---
type: theme
topic: Quantization-and-Low-Precision
theme_kind: lens
member_tag: lens/quantization-and-low-precision
paper_count: 1
first_generated: 2026-10-27
last_updated: 2026-10-27
tags: [topic-overview, optimization-method]
---

# 量化与低精度计算

本专题按优化方法组织站内论文。它可与 [[KV-Cache-Systems]]、[[LLM-Serving]] 等按研究对象组织的专题交叉；一篇论文可属于多个方法专题。

## 纳入范围

以更低数值精度减少权重、激活或计算成本，或针对量化运算设计执行路径。仅使用 FP8 数据格式作为实验配置不自动算作本专题。核心名单按现有论文笔记人工确认，标签用于导航，不自动决定成员。只有摘要的论文标为「仅公开摘要」。

## 核心论文

- [[ADAngel-OSDI26]]

## 主题导读

[[ADAngel-OSDI26]] 针对任意权重/激活精度组合选择计算映射。其收益还来自不同 kernel 与离线画像，不能把全部加速归因于降低 bit-width。其他论文若只采用现成 FP8 权重而不研究精度选择或低精度执行，暂留在相应系统专题。
