---
type: theme
topic: Scheduling-and-Resource-Management
theme_kind: lens
member_tag: lens/scheduling-and-resource-management
paper_count: 33
first_generated: 2026-10-27
last_updated: 2026-10-27
tags: [topic-overview, optimization-method]
---

# 调度与资源管理

本专题按优化方法组织站内论文，可与 [[KV-Cache-Systems]]、[[LLM-Serving]] 等系统领域专题交叉。以下是本站依据论文页「核心方法」作的归类；明确的副机制也可纳入，理由在条目中说明。

## 纳入范围

收录在请求、批次、模型、工作流或集群层面决定何时、何处、以多少资源运行的工作，包含验证批量和缓存加载调度。算子内部的指令流水主要放在编译与执行专题。资源分配率、显存占用和设备执行利用率需分别判断。核心名单由人工确认，标签不自动决定成员；「仅公开摘要」表示尚未取得全文。

## 核心论文

- [[ACDC-SOSP26]] — 按任务选择并行度、前缀顺序及在线空隙中的离线执行计划。
- [[Alibaba-ASI-OSDI26]] — 用 IPC 去碎片、拓扑放置和 SpotGPU 回收预留资源。
- [[BatchGen-OSDI26]] — 在协程让出点重组批次，并把空闲 GPU 分给尾部序列。
- [[Bedrock-WLM-SOSP26]] — 公开摘要描述全球准入、分类、路由和服务资源编排。（仅公开摘要）
- [[DCP-OSDI26]] — 按 SLO 动态选择预填充块，并重排解码以减少流水气泡。
- [[DiFlow-SOSP26]] — 调度模型级 DAG、复用副本，按请求选择 GPU 并行度。
- [[DSpark-arXiv26]] — 按前缀存活概率与吞吐曲线分配请求间的验证预算。
- [[DynaRL-OSDI26]] — 根据组件进度和队列压力在 rollout、推理与 trainer 间迁移资源。
- [[EcoServe-OSDI26]] — 跨实例错峰安排 prefill/decode，并调节请求路由与窗口。
- [[EnerTune-SOSP26]] — 摘要提出联合选择放置、频率和批次的能耗感知 GPU 共享。（仅公开摘要）
- [[Hetu-v2-OSDI26]] — 用异构分片和在线图切换适应设备、负载与故障变化。
- [[Janus-SOSP26]] — 以不同周期处理稳定放置、弹性副本和请求截止时间。
- [[KAIROX-OSDI26]] — 在线平衡 CPU/GPU 神经元组的驻留与计算分工。
- [[Kareus-OSDI26]] — 联合选择通信启动点、SM 份额及 GPU 频率。
- [[LLM-42-SOSP26]] — 把确定性请求的候选组成固定形状验证批次，权衡回滚与核验开销。
- [[LMetric-OSDI26]] — 结合前缀命中和当前负载选择请求目标实例。
- [[MorphX-SOSP26]] — 摘要提出按画像和可变形内核调整 GPU 多任务资源。（仅公开摘要）
- [[Murakkab-OSDI26]] — 按 SLO 与画像选择工作流配置、模型、实例和路由。
- [[Nixie-OSDI26]] — 统一控制 kernel admission、整应用驻留和切换迁移。
- [[OpenTela-OSDI26]] — 跨异构资源发现服务端点，并做模型放置与路由。
- [[Prism-OSDI26]] — 联合管理多模型内存气球、GPU group 放置与 slack 调度。
- [[Quota-Marketplace-OSDI26]] — 按市场权重与团队需求动态分配训练加速器配额。
- [[RLinf-OSDI26]] — 在 temporal、spatial 与 hybrid 执行计划中分配 RL 组件资源。
- [[RollArt-OSDI26]] — 按阶段选择异构资源，异步推进 trajectory 与 reward。
- [[SANI-OSDI26]] — 按 big/LITTLE 线程速度动态拆分、合并和迁移任务。
- [[Seer-OSDI26]] — 拆分并迁移 rollout chunk，利用组长估计降低长尾。
- [[Sereno-OSDI26]] — 以草稿让出点、验证 batch 和短暂停顿调节设备争用。
- [[SPEX-OSDI26]] — 调配推理树各分支及不同 query 的空闲 GPU 并行度。
- [[Strata-OSDI26]] — 推迟 transient-prefix 命中，平衡加载/计算量并用 decode 填 I/O 气泡。
- [[StreamInfer-SOSP26]] — 摘要提出逐层 token 队列与整理调度以减轻 MoE 长尾。（仅公开摘要）
- [[Tessera-OSDI26]] — 联合选择流水分段及细粒度重叠计划。
- [[Wang-LocalMoEInference-OSDI26]] — 按提示长度与 decode 负载选择 CPU、单/双 GPU 和节点内 P/D 路径。
- [[Weave-OSDI26]] — 把多个 RL 作业的互补阶段组织进协同执行组。

## 主题导读

[[LMetric-OSDI26]] 决定请求路由，[[DSpark-arXiv26]] 分配实例内验证预算，[[Strata-OSDI26]] 联合考虑缓存就绪与计算量，[[Janus-SOSP26]] 管模型放置和弹性。它们处于不同尺度；论文里的 allocation/occupancy 提高不能直接解释成 SM/Tensor Core 利用率提高。
