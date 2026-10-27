---
type: theme
topic: LLM-Inference
title: LLM Inference
theme_kind: area
member_tag: area/llm-inference
paper_count: 16
first_generated: 2026-10-27
last_updated: 2026-10-27
tags: [topic-overview, llm-inference]
---

# LLM Inference

聚焦模型在一次调用中怎样执行：注意力与 KV、低精度内核、CPU/GPU 分工、解码及输出语义。具体方法作为本页小节，不再各建一个专题。

## 纳入范围

只收录以 LLM 执行路径为主要问题、已有全文笔记的论文。请求路由、模型副本和服务资源放到 [[LLM-Serving]]，RL 训练整体流程放到 [[LLM-Training]]。通用 GPU 工具仅出现 LLM 实验，不足以进入核心名单。

## 核心论文

### 注意力与 KV

- [[HySparse-arXiv26]] — 在模型中共享重要块索引与跨层 KV。
- [[HySparse2-arXiv26]] — 以两级 KV 共享缩短历史缓存构建路径。
- [[HiSparse-arXiv26]] — 用主机完整历史和 GPU 热缓存控制稀疏解码驻留。
- [[ECHO-OSDI26]] — 对原生稀疏注意力做 KV 卸载与精确召回。
- [[DirectKV-OSDI26]] — 让注意力内核直接读取高速互连上的主机 KV。
- [[Strata-OSDI26]] — 优化分层 KV 的搬运、布局转换与加载时机。

### 内核、低精度与异构执行

- [[ADAngel-OSDI26]] — 按量化精度、形状和设备选择矩阵乘映射。
- [[KTransformers-SOSP25]] — 协调 CPU 路由专家与 GPU 注意力，优化量化执行。
- [[Wang-LocalMoEInference-OSDI26]] — 结合流式 prefill、CPU FP8 内核与小规模专家并行。
- [[KAIROX-OSDI26]] — 预测活跃神经元组并平衡 CPU/GPU 执行。
- [[MeldKernel-SOSP26]] — 在解码 attention、MoE 等内核中组织跨 SM 合作。
- [[MPK-OSDI26]] — 将 LLM 推理张量图编译成 persistent mega-kernel。

### 解码与输出语义

- [[DSpark-arXiv26]] — 以半自回归草稿和置信度调度目标模型验证。
- [[Sereno-OSDI26]] — 用弹性推测解码控制移动设备上的带宽争用。
- [[SPEX-OSDI26]] — 在推理树中提前执行可能的后续分支。
- [[LLM-42-SOSP26]] — 按固定形状核验候选，为指定请求提供确定性输出。

## 相关博客

- [[vLLM-Hybrid-HiSparse]] — vLLM Team 的动态稀疏 KV 驻留方案与实现资料。
- [[DeepSeek-EPLB]] — 引用 kaiyuan 的讲解，结合 DeepSeek 官方代码理解专家复制与放置。

## 阅读提示

先从 KV 或解码问题进入，再读对应内核实现。模型架构改变、KV 放置、逐 token 草稿、树分支推测和确定性核验有不同的行为与质量边界；比较时保留模型、输入/输出长度、并发、硬件、精度与基线条件。

缓存细节见 [[KV-Cache-Systems]]，DeepSeek 资料见 [[DeepSeek]]。vLLM 的基础论文可从 [PagedAttention](https://arxiv.org/abs/2309.06180) 开始，[[vLLM-Hybrid-HiSparse]] 汇集了相关设计与实践资料。
