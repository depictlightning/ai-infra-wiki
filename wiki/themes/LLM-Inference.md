---
type: theme
topic: LLM-Inference
title: LLM Inference
theme_kind: area
member_tag: area/llm-inference
paper_count: 31
first_generated: 2026-10-27
last_updated: 2026-10-28
tags: [topic-overview, llm-inference]
---

# LLM Inference

聚焦模型在一次调用中怎样执行：注意力与 KV、低精度内核、CPU/GPU 分工、解码及输出语义。具体方法作为本页小节，不再各建一个专题。

## 纳入范围

只收录以 LLM 执行路径为主要问题、已有全文笔记的论文。请求路由、模型副本和服务资源放到 [[LLM-Serving]]，RL 训练整体流程放到 [[LLM-Training]]。通用 GPU 工具仅出现 LLM 实验，不足以进入核心名单。

## 核心论文

本专题共 31 篇核心论文，按论文去重计数：

- [[HySparse-arXiv26]]、[[HySparse2-arXiv26]]、[[HiSparse-arXiv26]]、[[ECHO-OSDI26]]
- [[DirectKV-OSDI26]]、[[Strata-OSDI26]]、[[ADAngel-OSDI26]]、[[KTransformers-SOSP25]]
- [[Wang-LocalMoEInference-OSDI26]]、[[KAIROX-OSDI26]]、[[MeldKernel-SOSP26]]、[[MPK-OSDI26]]
- [[DSpark-arXiv26]]、[[Sereno-OSDI26]]、[[SPEX-OSDI26]]、[[LLM-42-SOSP26]]
- [[SolidAttention-FAST26]]、[[FlexiCache-MLSys26]]、[[SparseSpec-MLSys26]]、[[Dataflow-MLSys26]]、[[IntAttention-MLSys26]]、[[PRISM-MLSys26]]、[[ProfInfer-MLSys26]]、[[Kitty-MLSys26]]、[[ScaleSearch-MLSys26]]、[[OPKV-MLSys26]]、[[TokenWeave-MLSys26]]、[[SpecDecodeBench-MLSys26]]、[[MoEServingTax-MLSys26]]、[[ContextPilot-MLSys26]]、[[LocalityAwareBeam-MLSys26]]

## 按优化方法查找

以下按方法交叉整理本专题的核心论文；同一论文可出现在多个方向，因此各方向的数量不相加。空标题表示现有核心论文中暂未确认合适的成员。

### 01 稀疏化加速

- [[HySparse-arXiv26]]、[[HySparse2-arXiv26]] — 通过稀疏选择与跨层共享减少注意力处理范围。
- [[HiSparse-arXiv26]]、[[ECHO-OSDI26]] — 利用已有稀疏选择管理 KV 驻留和回载，不改变选择器语义。
- [[KAIROX-OSDI26]] — 预测活跃神经元组，仅计算和搬运所需组。
- [[SolidAttention-FAST26]] — 以块级稀疏选择控制从 SSD 读取的 KV。
- [[FlexiCache-MLSys26]] — 按注意力头选择高关注 KV 页面，结合稳定性控制重排频率。
- [[SparseSpec-MLSys26]] — 以完整核验的注意力分数选择下一轮稀疏草稿所需 KV。
- [[OPKV-MLSys26]] — 将稀疏选择的 KV 重新聚合成页，减少召回搬运。

### 02 无损压缩

### 03 量化与低精度计算

- [[ADAngel-OSDI26]] — 针对权重和激活精度组合选择矩阵乘映射。
- [[KTransformers-SOSP25]] — 为量化专家权重设计 CPU 布局和执行路径。
- [[Wang-LocalMoEInference-OSDI26]] — 使用专门的 FP8 GEMV 和量化转换融合优化 CPU MoE 执行。
- [[IntAttention-MLSys26]] — 用整数查表与量化归一化减少端侧注意力的类型转换开销。
- [[Kitty-MLSys26]] — 动态提高部分 Key 通道精度，并以低位宽页布局降低 KV 容量。
- [[ScaleSearch-MLSys26]] — 搜索低位宽块尺度并应用于量化注意力与 KV。

### 04 算子、编译与执行加速

- [[MoEServingTax-MLSys26]] — 分析专家计算拆分、填充与权重读取对前向延迟的影响。
- [[ADAngel-OSDI26]] — 构造低精度矩阵乘内核，并按形状和设备选择映射。
- [[KTransformers-SOSP25]]、[[Wang-LocalMoEInference-OSDI26]] — 分别优化 CPU 专家内核、融合与同步。
- [[MeldKernel-SOSP26]] — 通过跨 SM 协作与即时归约减少阶段间写回。
- [[MPK-OSDI26]] — 将 LLM 张量图编译为 persistent mega-kernel。
- [[Dataflow-MLSys26]] — 在数据流加速器上循环化解码层、跨样本流水并下放调度。
- [[ProfInfer-MLSys26]] — 关联端侧推理算子、线程和性能计数器，辅助定位执行瓶颈。
- [[TokenWeave-MLSys26]] — 融合张量并行中的通信、残差与归一化，减少重复计算和访存。
- [[DirectKV-OSDI26]]、[[HiSparse-arXiv26]]、[[Strata-OSDI26]] — 分别优化远端 KV 读取、召回和搬运布局的内核。

### 05 计算与状态复用

- [[LocalityAwareBeam-MLSys26]] — 将共享前缀的分支分组，减少重复加载 KV。

- [[ContextPilot-MLSys26]] — 对齐检索文档顺序并去重，增加前缀缓存复用机会。
- [[HySparse-arXiv26]]、[[HySparse2-arXiv26]] — 复用跨层 KV、索引或投影输入。
- [[HiSparse-arXiv26]]、[[ECHO-OSDI26]] — 复用热缓存命中的 KV，避免重复从主机取回。
- [[Strata-OSDI26]] — 复用多层前缀缓存及同批请求的共同加载结果。
- [[Sereno-OSDI26]] — 复用 n-gram 草稿候选，仍由目标模型验证。
- [[Wang-LocalMoEInference-OSDI26]] — 节点内 prefill 和 decode 共享一份 DRAM 权重。

### 06 投机解码与多 Token 生成

- [[SpecDecodeBench-MLSys26]] — 比较不同草稿方法的吞吐、接受分布与理想化收益。
- [[DSpark-arXiv26]] — 使用半自回归草稿，并按置信度分配验证预算。
- [[SparseSpec-MLSys26]] — 使用同一模型的稀疏注意力生成草稿，完整注意力核验。
- [[Dataflow-MLSys26]] — 利用数据流执行加速草稿与目标模型的推测解码。
- [[PRISM-MLSys26]] — 为不同草稿阶段分配参数路径，提高所测草稿接受长度和吞吐。
- [[Sereno-OSDI26]] — 调整草稿与验证批次，以控制移动设备上的带宽争用。
- [[SPEX-OSDI26]] — 邻近机制：推测执行推理树的后续分支，不等同于逐 token 投机解码。
- [[LLM-42-SOSP26]] — 邻近机制：通过候选生成与固定形状核验保证确定性，目标不是减少生成轮次。

### 07 调度与资源管理

- [[DSpark-arXiv26]]、[[Sereno-OSDI26]] — 分别调节验证预算及草稿、验证的执行时机。
- [[SparseSpec-MLSys26]] — 在请求间混合草稿与核验，并以主机卸载控制 KV 容量。
- [[SPEX-OSDI26]] — 在推理树分支和请求间分配空闲 GPU 并行度。
- [[LLM-42-SOSP26]] — 将确定性请求组成固定形状验证批次。
- [[KAIROX-OSDI26]]、[[Wang-LocalMoEInference-OSDI26]] — 分别调整 CPU/GPU 工作分工与执行路径。
- [[Strata-OSDI26]] — 联合安排缓存加载与计算，利用 decode 填补 I/O 等待。

### 08 卸载与分层存储

- [[LocalityAwareBeam-MLSys26]] — 分组连续生成多个 token，降低主机与 GPU 间的 KV 搬运。

- [[HiSparse-arXiv26]]、[[ECHO-OSDI26]] — 在主机保存完整 KV，并按需管理 GPU 热缓存。
- [[DirectKV-OSDI26]] — 从 GPU 直接读取高速互连上的主机 KV。
- [[Strata-OSDI26]] — 管理 GPU、主机与外部存储之间的 KV。
- [[SolidAttention-FAST26]] — 将完整 KV 存在 SSD，按需加载所选块。
- [[FlexiCache-MLSys26]] — 将稳定头的完整 KV 存在主机，按需回载新选页面。
- [[KTransformers-SOSP25]]、[[KAIROX-OSDI26]]、[[Wang-LocalMoEInference-OSDI26]] — 在 CPU/GPU 之间分配专家或神经元组的存储与计算。

### 09 预取与访问预测

- [[HiSparse-arXiv26]] — 在共享索引层组中提前回载所需 KV。
- [[ECHO-OSDI26]] — 利用跨层稀疏选择相关性提前取回 KV。
- [[Strata-OSDI26]] — 在排队期间将 SSD 命中的 KV 预取到 DRAM。
- [[SolidAttention-FAST26]] — 根据上一次选择预测并预取下一步的 KV 块。
- [[KAIROX-OSDI26]]、[[Wang-LocalMoEInference-OSDI26]] — 分别预测活跃组或提前准备下一专家权重。

### 10 通信与并行优化

- [[MoEServingTax-MLSys26]] — 比较 TP 与 EP 中专家计算、长尾及通信的成本。
- [[TokenWeave-MLSys26]] — 按 GPU 波次拆分 token，使张量并行的计算和通信重叠。
- [[Wang-LocalMoEInference-OSDI26]] — 面向少量 PCIe GPU 优化专家并行的 dispatch 和 combine 流量。

## 相关博客

- [[vLLM-Hybrid-HiSparse]] — vLLM Team 的动态稀疏 KV 驻留方案与实现资料。
- [[DeepSeek-EPLB]] — 引用 kaiyuan 的讲解，结合 DeepSeek 官方代码理解专家复制与放置。

## 阅读提示

先从 KV 或解码问题进入，再读对应内核实现。模型架构改变、KV 放置、逐 token 草稿、树分支推测和确定性核验有不同的行为与质量边界；比较时保留模型、输入/输出长度、并发、硬件、精度与基线条件。

缓存细节见 [[KV-Cache-Systems]]，DeepSeek 资料见 [[DeepSeek]]。vLLM 的基础论文可从 [PagedAttention](https://arxiv.org/abs/2309.06180) 开始，[[vLLM-Hybrid-HiSparse]] 汇集了相关设计与实践资料。
