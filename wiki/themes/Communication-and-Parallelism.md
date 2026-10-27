---
type: theme
topic: Communication-and-Parallelism
theme_kind: lens
member_tag: lens/communication-and-parallelism
paper_count: 19
first_generated: 2026-10-27
last_updated: 2026-10-27
tags: [topic-overview, optimization-method]
---

# 通信与并行优化

本专题按优化方法组织站内论文，可与 [[KV-Cache-Systems]]、[[LLM-Serving]] 等系统领域专题交叉。以下是本站依据论文页「核心方法」作的归类；明确的副机制也可纳入，理由在条目中说明。

## 纳入范围

收录并行划分、传输组织、通信语义和计算通信重叠的明确设计。既可优化正常执行，也可优化恢复时的连接和权重同步；条目须说明发生阶段。片上指令流水主要归编译与执行专题。核心名单由人工确认，标签不自动决定成员；「仅公开摘要」表示尚未取得全文。

## 核心论文

- [[ACDC-SOSP26]] — 依据任务 token 形态选择 TP/DP/EP 组合。
- [[DCP-OSDI26]] — 优化流水并行的预填充块与解码顺序。
- [[DiFlow-SOSP26]] — 以 GPU 原生传输和逐请求并行度连接模型级微服务。
- [[DualPath-SIGCOMM26]] — 借用解码侧存储网卡，并以计算网络和 QoS 组织 KV 供数。
- [[EcoServe-OSDI26]] — 通过实例内 KV 保留与错峰阶段安排避免 P/D 跨实例 KV 传输。
- [[Hetu-v2-OSDI26]] — 以 HSPMD 描述异构并行分片，并解析 collective 或重分片传输。
- [[Kareus-OSDI26]] — 联合选择计算通信重叠时机和通信 SM 配额。
- [[RLinf-OSDI26]] — 将宏观 RL 调用转成空间流水、时间复用或混合并行计划。
- [[RobustRL-OSDI26]] — 恢复时使用 UCX 动态重连，并通过权重 relay 续传和更新。
- [[RollArt-OSDI26]] — 分离 RL 角色并异步传递 trajectory、reward 与版本化权重。
- [[StreamInfer-SOSP26]] — 摘要以逐层流式 token 和两阶段通信减轻 MoE barrier 长尾。（仅公开摘要）
- [[Syncopate-OSDI26]] — 把逻辑通信 chunk 与计算 tile 协同排布，选择低层传输方式。
- [[TensorHub-SOSP26]] — 按源副本与网络拓扑组织 RDMA 权重复制。
- [[Tessera-OSDI26]] — 联合规划异构流水分段和细粒度通信重叠。
- [[TrainMover-OSDI26]] — 在迁移前准备增量通信组，把重连切换留在短暂停机阶段。
- [[UCCL-Tran-OSDI26]] — 将拥塞控制、路径选择及部分可靠性放入主机传输引擎。
- [[UEP-OSDI26]] — 将 GPU 发起的专家传输命令交给可移植 CPU proxy 执行。
- [[Wang-LocalMoEInference-OSDI26]] — 为小规模 PCIe GPU 设计 SmallEP，重组 dispatch/combine 流量。
- [[Weave-OSDI26]] — 通过分层权重同步与迁移连接不同 RL 作业的阶段执行。

## 主题导读

[[Tessera-OSDI26]] 与 [[Kareus-OSDI26]] 调整计算通信重叠，[[UCCL-Tran-OSDI26]] 与 [[UEP-OSDI26]] 改造传输组织。[[TrainMover-OSDI26]] 和 [[RobustRL-OSDI26]] 关注恢复阶段的通信组准备或动态重连，不能将其停机改善解释成正常迭代带宽收益。
