---
type: theme
topic: LLM-Serving
title: LLM Serving
theme_kind: area
member_tag: area/llm-serving
paper_count: 19
first_generated: 2026-10-23
last_updated: 2026-10-28
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
- [[BatchLLM-MLSys26]] — 提前识别离线批次中的共享前缀，并按组重排与混合 prefill/decode。
- [[Stream2LLM-MLSys26]] — 在检索上下文流式到达时重叠 prefill，并按资源压力调整请求与 KV 抢占。
- [[SparseSpec-MLSys26]] — 混合多个请求的稀疏草稿与完整核验，减少批量长推理的资源波动。

### 路由、资源与数据路径

- [[LMetric-OSDI26]] — 在前缀复用与实例负载之间选择请求目标。
- [[Janus-SOSP26]] — 协调生产多模型的稳定放置、弹性副本与截止时间。
- [[Prism-OSDI26]] — 联合管理多模型权重、KV 显存气球与执行空隙。
- [[DualPath-SIGCOMM26]] — 在预填充与解码节点间分担外部 KV 存储读取。
- [[OpenTela-OSDI26]] — 在异构 HPC 资源之上组织可发现的 LLM 服务。
- [[Bidaw-FAST26]] — 协同请求调度和分层 KV 缓存替换，减少多轮对话的存储等待。
- [[MAIO-FAST26]] — 按模型加载轨迹预取权重，缩短推理服务冷启动。
- [[MorphServe-MLSys26]] — 按负载调整层精度，将释放的显存分配给 KV cache。
- [[BreakingTheIce-MLSys26]] — 拆解并预测 vLLM 引擎启动成本，为扩缩容提供测量依据。
- [[HELIOS-MLSys26]] — 根据提前退出画像选择模型和加载层数，权衡资源、吞吐与输出质量。
- [[ContextPilot-MLSys26]] — 对齐并调度共享上下文的请求，提高前缀命中机会。

### 生产观测

- [[StriaTrace-OSDI26]] — 定位在线 LLM 推理中的慢步骤和跨实例关键路径。

## 相关博客

- [[DeepSeek-V4.1-Flash-Serving]] — 引用 DeepSeek 的生产部署说明，覆盖阶段分离、路由、KV 生命周期和扩缩容。
- [[Zartbot-DualPath]] — zartbot 的 DualPath 讲解书签；仅确认原文入口，全文尚未核对。

## 阅读提示

在线服务比较 TTFT、TPOT、SLO 与有效吞吐，离线任务比较整批完成时间和资源成本。先确定请求形态及约束，再判断调度、复用或弹性是否有收益；不同论文的最高加速比不直接排名。

执行机制见 [[LLM-Inference]]，训练任务的资源组织见 [[LLM-Training]]。
