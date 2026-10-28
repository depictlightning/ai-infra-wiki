---
type: theme
topic: LLM-Training
title: LLM Training
theme_kind: area
member_tag: area/llm-training
paper_count: 20
first_generated: 2026-10-27
last_updated: 2026-10-28
tags: [topic-overview, llm-training]
---

# LLM Training

聚焦 LLM 预训练与 RL 后训练的执行系统。并行、资源调度、权重同步、数据供给、恢复与调试放在本页内部组织。

## 纳入范围

只收录直接处理大模型训练流程、状态或调试问题，且已有全文笔记的工作。通用加速器配额、GPU 工具或其他模型训练，不为了补齐分类而纳入。RL rollout 作为训练流程的一部分在本页阅读。

## 核心论文

### 并行执行与训练效率

- [[Tessera-OSDI26]] — 联合规划异构 MoE 训练的流水分段与通信重叠。
- [[Hetu-v2-OSDI26]] — 以分层异构 SPMD 与图切换表达异构大模型训练。
- [[Kareus-OSDI26]] — 联合优化训练通信时机、SM 份额和 GPU 频率。
- [[AXLearn-MLSys26]] — 通过模块化配置、硬件适配与运行时支持跨硬件大模型训练。
- [[ProTrain-MLSys26]] — 联合搜索训练状态驻留、激活卸载与重计算配置。

### RL 后训练与权重同步

- [[RLinf-OSDI26]] — 将 RL 工作流映射为时间复用、空间流水或混合执行。
- [[RollArt-OSDI26]] — 拆分 agentic RL 阶段，按任务特征选择异构资源。
- [[DynaRL-OSDI26]] — 在 rollout、推理和 trainer 之间动态重分资源。
- [[Weave-OSDI26]] — 交错多个 RL 作业的互补阶段，减少资源空闲。
- [[Seer-OSDI26]] — 优化同步 RL 中的 rollout 分块、上下文调度和成组草稿。
- [[TensorHub-SOSP26]] — 用已有 worker 的权重副本支持弹性同步与传输。

### 数据供给与故障恢复

- [[Chen-LLMDataPipelines-OSDI26]] — 优化大规模 LLM 预训练的数据转换、checkpoint 复制与读取。
- [[RobustRL-OSDI26]] — 按角色恢复 RL worker，保留有效 rollout 进度。
- [[TrainMover-OSDI26]] — 为训练迁移预热运行时和通信组，缩短切换停机。
- [[AEGIS-OSDI26]] — 在生产 LLM 训练中以在线传感和异步确认检测 SDC。
- [[AITurbo-FAST26]] — 利用分组写入、去重和计算互连降低训练检查点 I/O 等待。
- [[AdaCheck-FAST26]] — 识别训练状态冗余，并用梯度和远端更新提高检查点频率。

### 调试与仿真

- [[OpGuard-OSDI26]] — 逐位对齐大模型训练执行，定位首个异常算子边界。
- [[SDCHunter-OSDI26]] — 保存真实 LLM 训练触发条件，诊断 SDC 故障 GPU。
- [[CrystalLLM-SOSP26]] — 用少量 GPU 采集、回放并调试大规模训练执行。

## 阅读提示

先区分预训练、同步 RL 与异步 RL，再看瓶颈发生在哪个阶段。MFU、单步时间、端到端训练吞吐、有效训练时间和故障恢复停机分别衡量不同问题；仿真的规模展示也需与有真机对照的误差测量分开。

模型执行优化见 [[LLM-Inference]]，面向用户请求的服务调度见 [[LLM-Serving]]。
