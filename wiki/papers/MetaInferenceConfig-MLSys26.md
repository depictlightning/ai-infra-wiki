---
type: paper
name: Meta Inference Configuration
full_title: "Optimizing Deployment Configurations for LLM Inference"
authors: [Sung Min Cho, Jaewon Lee, Chunqiang Tang, Yejin Lee, Geonhwa Jeong, Anca Agape]
venue: MLSys
year: 2026
tags: [llm-serving, deployment, performance-modeling, parallelism, moe, area/llm-serving]
source_pdf: "[[c7e1249ffc03eb9ded908c236bd1996d.pdf]]"
source_md: "[[c7e1249ffc03eb9ded908c236bd1996d]]"
review_status: complete
evidence_level: full-text
last_reviewed: 2026-10-29
---

# Meta 推理部署配置：结合工作负载与硬件搜索并行方案（MLSys 2026）

> **原题**：Optimizing Deployment Configurations for LLM Inference

> **一句话总结**：Meta 以算子实测画像构建轻量模拟器，联合搜索硬件、并行、批次和运行时配置，并总结在线与离线 LLM 推理部署中的权衡。[^paper]

**版本与阅读状态**：已通读与会议正式 PDF 一致的 MinerU 正文，核对模拟器、在线配置表和 MoE 并行/路由图。会议网页列出完整作者并使用上述题名，PDF 标题带 *Challenges and Insights*、署名为 Meta Inference Team；会议页面未列独立补充材料。[^paper]

## 问题与动机

LLM 服务配置不仅包括模型和 GPU，还涉及 TP、PP、EP、CP、DP、批量、量化以及 prefill/decode 是否分离。同一个配置在短请求、长上下文、在线聊天与离线生成中可能有不同最优结果。枚举所有组合并逐一实机测试成本很高（§1–2）。[^paper]

论文的目标是在给定工作负载和延迟约束下筛选高吞吐配置，而不是提出一种适合所有模型的固定并行策略。[^paper]

## 关键观察 / 隐含假设

- **性能画像有覆盖范围**：模拟器以不同算子、形状和硬件的实测结果插值；新硬件或未采样形状需要外推，精度不一定相同（§3.2–3.4）。[^paper]
- **平均性能不等于尾延迟**：论文报告所测情况通常在约 ±5% 误差内，但网络抖动等因素使 P99 更难预测；图表不构成全场景误差保证（§3.4）。[^paper]
- **输入分布影响结果**：生产输入、输出长度和上下文变化大，用单一平均长度做配置选择可能忽略长尾（§2.1）。[^paper]
- **吞吐必须连同资源口径一起看**：案例中的 `QPScluster` 按 256 张加速卡归一化，不能直接当作单机实测 QPS（§4.1）。[^paper]

## 核心方法

### 从算子画像组合到端到端配置

系统先对 GEMM、attention、AllReduce、All-to-All 等按输入形状做微基准，构建多维插值模型；根据模型、并行切分和请求形态组装算子，再沿关键路径累加计算、通信和运行时开销，估算 prefill 与 decode 延迟（§3.2）。[^paper]

随后按显存容量和 TTFT、TTIT 约束剪枝，搜索剩余硬件、运行时、批量和并行组合；分离式系统分别优化两阶段，并按处理速率估算所需 GPU 比例。论文还描述对推测解码接受率、MoE 路由和功耗配置的扩展建模（§3.3）。[^paper]

### 针对阶段和目标选择并行度

在线服务受 TTFT/TTIT 约束时，prefill 和 decode 可以使用不同的并行方案：前者需要平衡计算与通信，后者往往更受权重与 KV 读取影响。离线任务的延迟约束较宽松时，两阶段可能都倾向较深流水线和大批量（§4.2–4.3）。[^paper]

更多 TP 可以降低单次操作延迟，但跨设备通信增加，不保证单位 GPU 吞吐也提高；需在满足 SLO 后比较系统整体容量。[^paper]

### MoE 路由和互连

跨节点纯 TP 可能受到 AllReduce 影响，EP 将专家分布到不同设备并通过 All-to-All 路由 token，可能在所测拓扑下更有效。专家负载不均也有阶段差异：prefill 中热点专家可能形成计算长尾；decode 在部分专家未激活时，偏斜反而可能减少权重读取（§4.5）。[^paper]

论文比较均衡、独立随机和生产流量校准的专家激活模型；中等批量下模型预测差异明显，说明默认均匀路由可能误估性能。[^paper]

## 设计取舍

- **搜索速度与现实误差**：模拟可覆盖大量组合，最终选择仍需实测核对网络、队列、缓存和尾部延迟。
- **专用与通用资源**：异构 P/D 部署可利用不同硬件优势，但增加容量规划、调度和跨池运维复杂度。
- **稀疏计算与通信**：MoE 减少激活计算不自动减少总成本；热点、专家权重和网络拓扑决定实际收益。

## 实验与结果

论文 §4 明确说明案例结果来自模拟框架；GPU-A/B/C 与 Next-Gen 等平台按文中设定比较，部分 MoE 模型或未来平台为假设配置。生产部署经验与这些模拟数据应分别理解。[^paper]

| 结果 | 条件与边界 |
|---|---|
| 在线运行时 | 表 6 的 Llama3-70B、GPU-A 配置中，分离式与连续批处理的归一化集群 QPS 分别为 356 和 198，约 1.80×；两者使用不同并行与 batch 配置，这是模拟搜索结果。 |
| 部署经验 | 作者报告多数在线服务迁移到分离式运行时后获得约 30% 容量节省；论文没有在该段给出完整线上对照设计，不应与表 6 的模拟倍数混算。 |
| 跨节点 MoE | 图 13 的 Llama 4 Maverick decode 模拟中，TP8 为 618 QPScluster，TP16 为 329，TP4DP4-TP4EP4 为 899；这是特定模型、硬件及批量配置。 |
| 假设 MoE | 图 12 的 405B dense 与等总参数 MoE 是性能建模对照；作者明确说明假设 MoE 并不代表可直接使用且质量等价的模型。 |
| 异构成本 | 论文估算为不同阶段选择硬件可带来 15–25% TCO 改善；这依赖成本模型，不是普遍适用的实测节省。 |

## 论断—证据表

| 论断 | 证据 | 评测边界 | 置信度 |
|---|---|---|---|
| 配置最优解依赖工作负载、硬件和 SLO | §3–4、表 6–8 | 搜索空间与画像质量影响结果 | 中高 |
| EP 在所测跨节点 MoE 配置中可优于纯 TP | 图 12–13 | 通信拓扑、路由与 batch 会改变比较 | 中 |
| 模拟器可准确预测任意部署的 P99 | §3.4 | 作者指出抖动等影响尾延迟，未提供普遍保证 | 低 |

## 批判性分析

### 论证链条

论文以实测算子构建预测模型，再搜索多维部署配置，结合不同工作负载展示阶段并行、硬件异构和 MoE 的影响。其价值是帮助提出候选配置，而不是绕过最终负载测试。[^paper]

### 假设压力测试

未覆盖的算子形状、新硬件、复杂通信争用或专家热点漂移会降低预测可靠性。不同硬件价格、利用率和集群拓扑也会改变异构部署的 TCO；模拟中的假设 MoE 不应被当作质量对等的模型。

### 实验可信度

文章报告模拟与部分实测的对齐，也结合生产观察，但大量案例仍为模拟；特别是未来平台、假设模型和跨不同硬件的成本结论需要明确区分。只给出平均误差无法证明所有候选和尾部指标均准确。[^paper]

## 局限与后续工作

- 用真实请求分布和候选配置测量 P95/P99、队列、通信拥塞及缓存命中，校准模拟器。
- 在具体硬件成本和模型质量约束下，比较分离、共置及异构 P/D 的总成本。

## 相关

- [[LLM-Serving]]：服务配置、阶段并行与资源规划。
- [[PragmaticDisaggregation-MLSys26]]：同样研究阶段分离和资源比例，采用不同模拟框架；其图文倍数待复核。
- [[MoEServingTax-MLSys26]]：从专家权重读取、路由及 kernel 拆解 MoE 推理成本。

## 参考文献

[^paper]: Sung Min Cho 等. 《Optimizing Deployment Configurations for LLM Inference》. MLSys，2026。[原文](https://proceedings.mlsys.org/paper_files/paper/2026/hash/97dc07f1253ab33ee514f395a82fa7cc-Abstract-Conference.html)。
