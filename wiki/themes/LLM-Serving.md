---
type: theme
topic: LLM-Serving
title: LLM Serving
theme_kind: area
member_tag: area/llm-serving
paper_count: 10
first_generated: 2026-10-23
last_updated: 2026-10-27
tags: [topic-overview, llm-serving]
---

# LLM Serving

聚焦怎样把 LLM 执行组织成服务：请求批次、阶段调度、模型放置、弹性、数据路径与运行观测。

## 纳入范围

核心论文需要直接研究 LLM 服务系统，并已有全文笔记。单次模型执行的算法和内核归 [[LLM-Inference]]；通用集群管理和应用工作流不因调用了 LLM 就进入本页。

## 核心论文

### 批处理与阶段调度

- [[DCP-OSDI26]] — 按延迟约束调整流水并行的预填充块与解码顺序。
- [[EcoServe-OSDI26]] — 错峰安排多个实例的 prefill/decode，控制阶段干扰。
- [[ACDC-SOSP26]] — 按离线任务组织并行配置、缓存顺序和执行计划。
- [[BatchGen-OSDI26]] — 在执行中重组序列批次，并用空闲 GPU 缩短长尾。

### 路由、资源与数据路径

- [[LMetric-OSDI26]] — 在前缀复用与实例负载之间选择请求目标。
- [[Janus-SOSP26]] — 协调生产多模型的稳定放置、弹性副本与截止时间。
- [[Prism-OSDI26]] — 联合管理多模型权重、KV 显存气球与执行空隙。
- [[DualPath-SIGCOMM26]] — 在预填充与解码节点间分担外部 KV 存储读取。
- [[OpenTela-OSDI26]] — 在异构 HPC 资源之上组织可发现的 LLM 服务。

### 生产观测

- [[StriaTrace-OSDI26]] — 定位在线 LLM 推理中的慢步骤和跨实例关键路径。

## 相关博客

- [[DeepSeek-V4.1-Flash-Serving]] — 引用 DeepSeek 的生产部署说明，覆盖阶段分离、路由、KV 生命周期和扩缩容。
- [[Zartbot-DualPath]] — zartbot 的 DualPath 讲解书签；仅确认原文入口，全文尚未核对。

## 阅读提示

在线服务比较 TTFT、TPOT、SLO 与有效吞吐，离线任务比较整批完成时间和资源成本。先确定请求形态及约束，再判断调度、复用或弹性是否有收益；不同论文的最高加速比不直接排名。

执行机制见 [[LLM-Inference]]，训练任务的资源组织见 [[LLM-Training]]。
