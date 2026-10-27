---
type: theme
topic: Communication-and-Parallelism
theme_kind: lens
member_tag: lens/communication-and-parallelism
paper_count: 17
first_generated: 2026-10-27
last_updated: 2026-10-27
tags: [topic-overview, optimization-method]
---

# 通信与并行优化

本专题按优化方法组织站内论文。它可与 [[KV-Cache-Systems]]、[[LLM-Serving]] 等按研究对象组织的专题交叉；一篇论文可属于多个方法专题。

## 纳入范围

通过并行划分、网络传输、通信语义或计算通信重叠改善多设备协同；仅在单设备本地调度的工作不纳入。核心名单按现有论文笔记人工确认，标签用于导航，不自动决定成员。只有摘要的论文标为「仅公开摘要」。

## 核心论文

- [[ACDC-SOSP26]]
- [[DCP-OSDI26]]
- [[DiFlow-SOSP26]]
- [[DualPath-SIGCOMM26]]
- [[EcoServe-OSDI26]]
- [[Hetu-v2-OSDI26]]
- [[Kareus-OSDI26]]
- [[RollArt-OSDI26]]
- [[StreamInfer-SOSP26]] — 仅公开摘要。
- [[Syncopate-OSDI26]]
- [[TensorHub-SOSP26]]
- [[Tessera-OSDI26]]
- [[TrainMover-OSDI26]]
- [[UCCL-Tran-OSDI26]]
- [[UEP-OSDI26]]
- [[Wang-LocalMoEInference-OSDI26]]
- [[Weave-OSDI26]]

## 主题导读

[[Tessera-OSDI26]] 与 [[Kareus-OSDI26]] 研究通信何时与计算重叠，[[UCCL-Tran-OSDI26]] 与 [[UEP-OSDI26]] 处理传输层或专家通信，[[DCP-OSDI26]] 和 [[Hetu-v2-OSDI26]] 改变并行划分。并行增加计算资源时，也会引入额外通信与同步，应一起核算。
