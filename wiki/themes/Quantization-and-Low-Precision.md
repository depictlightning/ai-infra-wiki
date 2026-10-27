---
type: theme
topic: Quantization-and-Low-Precision
theme_kind: lens
member_tag: lens/quantization-and-low-precision
paper_count: 3
first_generated: 2026-10-27
last_updated: 2026-10-27
tags: [topic-overview, optimization-method]
---

# 量化与低精度计算

本专题按优化方法组织站内论文，可与 [[KV-Cache-Systems]]、[[LLM-Serving]] 等系统领域专题交叉。以下是本站依据论文页「核心方法」作的归类；明确的副机制也可纳入，理由在条目中说明。

## 纳入范围

收录精度选择、量化表示或专门针对低精度格式设计的计算路径。仅把 FP8 作为实验配置不构成成员依据；为已有 FP8/INT4 格式开发专用内核则属于低精度执行研究。核心名单由人工确认，标签不自动决定成员；「仅公开摘要」表示尚未取得全文。

## 核心论文

- [[ADAngel-OSDI26]] — 面向任意权重/激活精度组合，选择 Padding、Split 或 Bitwise 内核。
- [[KTransformers-SOSP25]] — 为分组量化 INT4/INT8 权重设计布局及 AMX/AVX-512 执行路径。
- [[Wang-LocalMoEInference-OSDI26]] — 专门设计 AVX-512 FP8 GEMV 与 INT4 转换融合，避免完整上转换权重。

## 主题导读

[[ADAngel-OSDI26]] 在不同 bit-width 与计算映射间选择；[[KTransformers-SOSP25]] 与 [[Wang-LocalMoEInference-OSDI26]] 研究量化权重或原生 FP8 的 CPU 执行。后者的 FP8 GEMV 在寄存器中转换并用 BF16 指令累加，不能据此说 CPU 具备原生 FP8 乘加硬件。
