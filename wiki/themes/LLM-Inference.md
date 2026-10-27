---
type: theme
topic: LLM-Inference
title: LLM 推理专题
theme_kind: area
member_tag: area/llm-inference
candidate_tags: [llm-inference, llm-serving, moe-inference, agentic-inference, speculative-decoding, sparse-attention, offline-serving]
paper_count: 29
first_generated: 2026-10-27
last_updated: 2026-10-27
tags: [topic-overview, llm-inference, llm-serving]
---

# LLM 推理专题

围绕一次 LLM 调用从输入到输出的执行过程，组织模型结构、算子、权重放置、KV cache、解码、批处理和性能观测。可以从这里找问题，再进入 [[KV-Cache-Systems]] 或具体优化方法专题。

## 纳入范围

核心成员需要直接研究 LLM 的推理路径或服务运行时，并已有全文笔记。模型结构论文、系统实现和生产经验可以并列，但各自证据类型须保留。RL 工作只有在提出具体生成、KV 或解码方法时，才以其推理子系统纳入。

[[LLM-Serving]] 更集中于路由、批次、模型副本、资源放置与服务约束；本页同时覆盖模型内部和设备上的执行。两个专题可重叠，成员仍按各自问题人工确认。通用训练框架、模型仓库或只在 LLM 上测过的基础工具不自动进入本页。只有摘要的工作单列为候选，不计入 `paper_count`。

## 核心论文

### 模型结构与注意力

- [[HySparse-arXiv26]] — 以混合完整/稀疏层共享块索引和 KV；证据主要是模型质量与缓存规模。
- [[HySparse2-arXiv26]] — 以跨解码器桥接和两级 KV 共享缩短历史缓存构建路径；FLOPs 估算需与实测推理延迟区分。

### 内核、低精度与异构执行

- [[ADAngel-OSDI26]] — 为量化权重/激活选择设备与形状适配的计算映射。
- [[KTransformers-SOSP25]] — CPU 执行路由专家，GPU 执行注意力等部分，并优化量化内核与跨设备协调。
- [[Wang-LocalMoEInference-OSDI26]] — 结合流式 prefill、CPU FP8 GEMV、SmallEP 与节点内 P/D 路径服务大型 MoE。
- [[KAIROX-OSDI26]] — 预测活跃神经元组，在线平衡 CPU/GPU 放置与计算。
- [[MeldKernel-SOSP26]] — 在解码 attention、MoE 等算子中组织跨 SM 协作、融合与即时归约。
- [[MPK-OSDI26]] — 用 persistent mega-kernel 执行 LLM 张量图，减少算子边界与启动开销。
- [[UEP-OSDI26]] — 以 CPU proxy 支持细粒度专家通信；推理与训练案例分别评测。

### 权重、KV 与设备驻留

- [[HiSparse-arXiv26]] — 完整稀疏 KV 留在主机，以热缓存、融合召回及共享索引预取控制解码驻留。
- [[ECHO-OSDI26]] — 为原生稀疏注意力保留主机完整 KV，并按所选 token 召回及预取。
- [[DirectKV-OSDI26]] — 在高速 CPU–GPU 互连上直接读取主机 KV，消除显存暂存。
- [[Strata-OSDI26]] — 处理分层前缀缓存的小页搬运、布局转换与加载调度。
- [[DualPath-SIGCOMM26]] — 利用解码侧存储网卡为长上下文预填充供给历史 KV。
- [[Prism-OSDI26]] — 统一多模型权重与 KV 的显存气球，并协调空间和时间共享。
- [[Nixie-OSDI26]] — 在消费级 GPU 上协同安排应用执行、整工作集驻留与分层迁移。

### 解码、推理树与输出约束

- [[DSpark-arXiv26]] — 半自回归草稿结合置信度验证预算，优化多 token 生成。
- [[Sereno-OSDI26]] — 用弹性推测解码、验证批次和让出点控制移动设备带宽争用。
- [[SPEX-OSDI26]] — 提前探索可能的推理树分支，隐藏 reward 等待；对象是分支级推测。
- [[LLM-42-SOSP26]] — 通过固定形状核验和回滚为部分请求提供确定性输出。
- [[Seer-OSDI26]] — 在同步 RL 的 rollout 子系统中做分块迁移、上下文调度和成组推测解码。

### 批处理、流水与服务运行时

- [[DCP-OSDI26]] — 在流水并行中按延迟约束调整预填充块与解码顺序。
- [[EcoServe-OSDI26]] — 错峰安排多个实例的 prefill/decode，减少阶段干扰与 KV 跨实例传输。
- [[LMetric-OSDI26]] — 在请求路由中同时考虑前缀复用与实例负载。
- [[Janus-SOSP26]] — 在生产多模型负载中联合协调稳定放置、弹性副本和截止时间。
- [[ACDC-SOSP26]] — 针对离线任务选择并行度、缓存顺序及推测解码策略。
- [[BatchGen-OSDI26]] — 在执行中让出和重组序列，提升 MoE 批处理及长尾完成效率。
- [[OpenTela-OSDI26]] — 用去中心化控制面把异构 HPC 资源组织成可发现的 LLM 服务。

### 性能观测与诊断

- [[StriaTrace-OSDI26]] — 以同步边界、GPU 事件与异常 trace 定位生产推理慢步骤。

## 相关博客

- [[vLLM-Hybrid-HiSparse]] — 汇集 vLLM Team 的官方博客、设计文档与实现入口，说明原始 HiSparse 和 vLLM 动态驻留策略的关系。
- [[DeepSeek-V4.1-Flash-Serving]] — 转述 DeepSeek 的生产推理部署经验，覆盖阶段分离、路由、KV 生命周期与执行调度。
- [[DeepSeek-EPLB]] — 引用 kaiyuan 的 EPLB 讲解，并对照 DeepSeek 官方代码理解专家复制与放置。

三篇博客的原作者、原文和阅读范围见各自页面；它们不计入论文数。

## 仅摘要候选

- [[SANDHI-SOSP26]] — 多模型组件共享；尚不能还原具体合并与运行时协议。
- [[StreamInfer-SOSP26]] — MoE 长尾与通信 barrier；队列和两阶段通信细节仍受摘要证据限制。
- [[Bedrock-WLM-SOSP26]] — 全球模型服务的准入、分类、路由与编排，当前只有架构线索。
- [[LMTracer-SOSP26]] — 生产 LLM 性能剖析；探针协议、开销口径与评测配置尚缺全文证据。

## 阅读路线

1. **先明确执行阶段和瓶颈**。从 [[DCP-OSDI26]]、[[EcoServe-OSDI26]] 看 prefill/decode 的阶段安排，再用 [[StriaTrace-OSDI26]] 思考如何定位实际慢步骤。阶段边界、请求等待和设备内核是不同观测层。
2. **检查容量与数据路径**。[[HySparse-arXiv26]]、[[HySparse2-arXiv26]] 改变 KV 的构造和共享；[[HiSparse-arXiv26]]、[[ECHO-OSDI26]]、[[DirectKV-OSDI26]]、[[Strata-OSDI26]] 处理既有状态的驻留或供数。进一步读 [[KV-Cache-Systems]]、[[Offloading-and-Tiered-Storage]] 与 [[Prefetching-and-Access-Prediction]]。
3. **再看 token 和分支怎样推进**。[[DSpark-arXiv26]] 的逐 token 草稿/验证、[[SPEX-OSDI26]] 的树分支推测和 [[LLM-42-SOSP26]] 的确定性核验分别回答不同问题。[[Speculative-Decoding-and-Multi-Token]] 已把邻近机制单列。
4. **深入单设备与异构执行**。从 [[ADAngel-OSDI26]]、[[KTransformers-SOSP25]] 和 [[Wang-LocalMoEInference-OSDI26]] 进入 [[Quantization-and-Low-Precision]]、[[Operator-and-Compiler-Optimization]] 与 [[Communication-and-Parallelism]]，检查算术强度、数据格式及跨设备代价。
5. **最后扩展到负载与资源**。在线场景读 [[LMetric-OSDI26]]、[[Janus-SOSP26]]，离线任务读 [[ACDC-SOSP26]]、[[BatchGen-OSDI26]]；完整服务视角见 [[LLM-Serving]] 与 [[Scheduling-and-Resource-Management]]。

## 比较时保留的边界

模型质量、预填充 FLOPs、单步内核延迟、整批完成时间、峰值吞吐与在线 SLO 分别代表不同结果。比较前记录模型、输入/输出长度、并发或到达率、硬件和主机预算、缓存精度、基线以及是否包含预填充。本文的分组是阅读判断，不表示组内结果已按同一条件归一化。

保持所选 KV 不变、保持模型分布、允许准确率损失与保证逐位确定性也需分别说明。HiSparse 的无 I/O 上界使用无效输出，KTransformers 的可选专家延迟和 HySparse2 的结构变化也有各自的行为与质量边界；比较时应回对应论文页核对。

## 相邻专题与材料

- **系统与项目**：[[LLM-Serving]]、[[KV-Cache-Systems]]、[[vLLM-Serving]]、[[DeepSeek]]。
- **通用执行基础**：[[GraCE-OSDI26]]、[[Twill-OSDI26]]、[[Syncopate-OSDI26]]，按其实际覆盖的模型和内核范围阅读。
- **应用侧上下文**：[[llmovoice-SOSP26]] 管理语音会话语义状态；[[UnlimitedOCR2-TechReport26]] 面向多页 OCR 的视觉参考与分块 KV。
- **训练中的推理阶段**：[[RLinf-OSDI26]]、[[DynaRL-OSDI26]]、[[Weave-OSDI26]] 提供 RL 组件执行与资源分配的背景，不据此把整套训练框架视为解码算法。
