---
type: theme
topic: LLM-Serving
theme_kind: area
member_tag: area/llm-serving
candidate_tags: [llm-serving, llm-inference, offline-serving, foundation-model-serving]
paper_count: 10
first_generated: 2026-10-23
last_updated: 2026-10-23
tags: [topic-overview, llm-serving, scheduling]
---

# LLM Serving 系统专题

LLM 服务系统要把到来的请求分配给模型实例，并在延迟、吞吐、资源成本和输出约束之间作选择。本页先按决策发生的位置组织已读论文；在线与离线任务的指标不同，后续再补完整的实验对照。

## 纳入范围

收录以 **LLM 请求的路由、批处理、资源放置、状态传输或服务结果约束** 为主要系统问题的论文。`llm-serving`、`llm-inference` 等标签用于发现候选，核心成员仍需按正文问题和实验人工确认。本页先选有全文笔记的论文；仅摘要的工作保留为候选，不据其推断设计细节。同一论文可以同时属于 [[KV-Cache-Systems]] 等专题。

## 核心论文

### 请求路由与在线执行

- [[LMetric-OSDI26]] — 路由时结合前缀 KV 复用与实例负载。
- [[DCP-OSDI26]] — 在流水线并行中按延迟约束调整预填充切块和解码顺序。
- [[EcoServe-OSDI26]] — 让多个实例错峰执行预填充与解码，控制阶段干扰。
- [[DSpark-arXiv26]] — 在实例内生成半自回归草稿，并随并发负载调整每个请求的验证长度。

### 模型、显存与数据路径

- [[Janus-SOSP26]] — 在生产多模型负载中协调模型放置和弹性副本。
- [[Prism-OSDI26]] — 联合管理模型权重与 KV cache 的显存占用。
- [[DualPath-SIGCOMM26]] — 借用解码节点的存储网卡读取长上下文 KV cache。

### 离线任务与服务约束

- [[ACDC-SOSP26]] — 根据离线任务特征选择并行度、前缀顺序和推测解码策略。
- [[BatchGen-OSDI26]] — 在执行中重组批次，并用空闲 GPU 缩短长尾任务的完成时间。
- [[LLM-42-SOSP26]] — 为要求可复现输出的请求增加验证与回滚路径。

## 相关博客

- [[DeepSeek-V4.1-Flash-Serving]] — DeepSeek 生产推理系统：分阶段部署、路由与扩缩容、KVCache 和解码调度；已阅读正文，图表待核对。
- [[Zartbot-DualPath]] — zartbot 的 DualPath 讲解；目前仅记录原文入口，博客正文尚未核对。关联论文：[[DualPath-SIGCOMM26]]。

## 主题导读

在线服务首先要区分 **请求放到哪里** 与 **实例内怎样执行**。[[LMetric-OSDI26]] 决定路由目标；[[DCP-OSDI26]] 和 [[EcoServe-OSDI26]] 调整预填充、解码的执行时机；[[DSpark-arXiv26]] 按候选 token 的预期接受率分配目标模型的验证批量。[[Janus-SOSP26]] 与 [[Prism-OSDI26]] 把决策扩到模型副本和显存容量，[[DualPath-SIGCOMM26]] 则处理长上下文请求从外部存储读回 KV 的网络瓶颈。这些方法涉及不同时间尺度和资源，不能仅凭单篇的吞吐倍数判断能否叠加。

离线任务更关注一批请求何时全部完成、每块 GPU 产出多少。[[ACDC-SOSP26]] 使用任务已知信息改变执行计划，[[BatchGen-OSDI26]] 在运行中重新组合请求；它们的作业完成时间和在线论文的 TTFT、SLO 达标率不能直接互换。[[LLM-42-SOSP26]] 还表明部分服务要保证重复推理的一致输出，此时调度收益需连同验证与回滚成本一起计算。
