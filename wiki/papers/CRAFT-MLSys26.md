---
type: paper
name: CRAFT
full_title: "CRAFT: Fine-Grained Cost-Aware Expert Replication For Efficient Mixture-of-Experts Serving"
authors: [Adrian Zhao, Zhenkun Cai, Zhenyu Song, Lingfan Yu, Haozheng Fan, Jun Wu, Yida Wang, Nandita Vijaykumar]
venue: MLSys
year: 2026
tags: [llm-serving, moe, expert-parallelism, load-balancing, gpu-memory]
source_pdf: "[[3a7f9e485845dac27423375c934cb4db.pdf]]"
source_md: "[[3a7f9e485845dac27423375c934cb4db]]"
review_status: needs-review
evidence_level: full-text
last_reviewed: 2026-10-28
---

# CRAFT：按收益分配 MoE 专家副本（MLSys 2026）

> **原题**：CRAFT: Fine-Grained Cost-Aware Expert Replication For Efficient Mixture-of-Experts Serving

> **一句话总结**：均匀地为每层复制专家可能挤占 KV cache 却收效甚微；CRAFT 按层估计负载均衡收益并分配副本，在 8 节点实验中相对 EPLB 的平均有效吞吐提高 1.14 倍（§5.2、图 8）。[^paper]

**版本与阅读状态**：使用 MLSys 论文集提供的 12 页正式版 PDF，并在本次调用 MinerU 生成文本；已阅读正文和单独发布的附录并核对主要图表。正式主 PDF 在结论处结束，补充文件含附录但未见正文参考文献表；仓库旧版不同，暂保留待复核状态。[^paper][^supplement]

## 问题与动机

专家并行将 MoE 专家分布到多个 GPU，但 token 的路由可能高度偏斜。只重新放置专家未必能平衡极热专家；复制热点专家可以分摊负载，却消耗 GPU 内存，与 KV cache 争夺服务容量（§2）。[^paper]

CRAFT 关注各层专家的负载分布不同、复制收益也不同这一问题。它与 [[DeepSeek-EPLB]] 讨论的专家负载均衡属于相邻问题；具体策略和实验不能直接视为同一套实现。[^paper]

## 关键观察 / 隐含假设

- **层之间偏斜不同**：论文 DE8 示例的第 20 层最热专家负载约为均值的 2.5 倍，第 51 层超过 27 倍；副本应优先用于放置难以解决的高偏斜层（§3.2、图 4）。[^paper]
- **更多副本的边际收益递减**：论文的所测配置中，进一步增加副本可能只带来很小的平衡改善，同时占用更多可用于 KV cache 的内存（§3.3、图 5）。[^paper]
- **历史路由分布有预测价值**：系统用采样统计制定计划，并可周期性重新平衡；分布快速变化时，统计和重配置的时效与成本都需要考虑（§4.3、§5.6）。[^paper]

## 核心方法

CRAFT 先收集每层专家负载，估计不同副本数量下的均衡收益；用户可指定预算，或由系统选取每份副本收益较高的预算。随后使用动态规划，将总副本数分配给各层以最大化预估均衡收益（§4.2.1、图 7）。补充文件给出收益估计、动态规划和容量分配的伪代码；候选副本数按 2 的几何级数选取，因而是受限候选空间中的优化（附录 A）。[^paper][^supplement]

由于各层副本数不同，系统再为设备分配容量，并采用交错分配和容量感知的贪心专家放置，控制设备和节点间的额外容量不均（§4.2.2）。它优化的是副本与放置计划，不改变模型权重或路由器训练。[^paper]

## 设计取舍

- 副本更少可增加 KV 空间，但过少可能重新出现热点 GPU 和通信长尾。
- 每层分别优化提高预算灵活性，也增加容量分配、计划求解和重平衡复杂度。
- 路由分布采样反映过去的请求，计划对未来突发负载的效果需要通过在线实验验证。

## 实验与结果

作者基于 SGLang，评估 DeepSeek-R1-671B 和 Kimi-K2-1000B，输入按 4096 token 分块，输出固定 256 token；使用 AWS p4de 节点、每节点 8 张 A100 80 GB、6/8/12 节点配置，在线到达按泊松过程建模（§3.1、§5.2）。[^paper]

| 结果 | 条件与基线 |
|---|---|
| 平均有效吞吐分别提高 1.15 倍和 1.12 倍 | 8 节点，CRA8 对比 EPLB；分别为 DeepSeek-R1 和 Kimi-K2，单点最高 1.2 倍（§5.2、图 8）。 |
| 副本数分别少 7.25 倍和 7.5 倍 | 同一 8 节点比较；节省的内存可给 KV cache，需与最终吞吐一起理解（§5.2）。 |
| 6 节点时 EPLB 的有效吞吐平均低于仅放置基线 46% | 论文将此归因于副本挤占有限的 KV 空间；CRA8 在相应比较中仍高于基线 1.14 倍（§5.4）。 |
| 动态分布测试 | 作者合成昼夜变化和突发混合，并按约 20 分钟重平衡；这不是长期生产流量实测（§5.6、图 10）。 |
| 初始化分析 | 所测配置中约 10 秒；论文说在线重平衡可与 CPU 计算重叠，但完整重配置成本仍依赖实现和负载（§5.5）。 |

## 论断—证据表

| 论断 | 证据 | 评测边界 | 置信度 |
|---|---|---|---|
| 不同层的复制收益差异大 | §3.2–3.3，图 3–6 | 论文模型和输入分布 | 中高 |
| 细粒度预算可改善有效吞吐 | §4、§5.2，图 8 | SGLang、两种 MoE 模型及所测节点数 | 中高 |
| 分布变化后可周期性重平衡 | §4.3、§5.6，图 10 | 合成工作负载；20 分钟重平衡周期 | 中 |

## 批判性分析

### 论证链条

层级负载和副本收益解释了为什么统一复制会浪费内存；有效吞吐曲线把内存节省连接到服务容量。仅比较专家负载均衡度不足以推导端到端吞吐，论文的 KV 容量及排队测量补上了这一环。[^paper]

### 假设压力测试

请求域突然改变时，历史路由统计可能失效；更频繁重平衡也会产生复制和迁移代价。是否仍优于其他在线策略需要在真实时变流量中测量。[^paper]

### 实验可信度

两种大 MoE 模型与多个节点规模直接对应 LLM 服务，但输入和输出长度固定、到达时间建模且时变实验为合成轨迹；不能将结果直接外推为所有生产业务的长期收益。独立补充文件提供算法和附加实验；主文件与补充文件均未见正文参考文献表，相关引文仍需进一步核验。[^paper][^supplement]

### 系统性缺陷

每层动态规划、设备容量分配和副本切换增加控制面复杂度；模型版本更新、多租户隔离以及故障期间副本一致性仍需核验。[^paper]

## 局限与后续工作

- 在真实时变请求与突发域切换下，记录副本调整频率、迁移流量、P99 和有效吞吐。
- 将副本预算与 KV cache 预算联合作为动态决策变量，比较固定预算和按负载调整的成本与 SLO。

## 相关

- 同会议：[[MLSys-2026]]。
- 专题材料：[[DeepSeek-EPLB]]、[[DeepSeek]]。

## 参考文献

[^paper]: Adrian Zhao 等. 《CRAFT: Fine-Grained Cost-Aware Expert Replication For Efficient Mixture-of-Experts Serving》. MLSys，2026。[原文](https://proceedings.mlsys.org/paper_files/paper/2026/hash/3a7f9e485845dac27423375c934cb4db-Abstract-Conference.html)。
[^supplement]: Adrian Zhao 等. 《CRAFT: Fine-Grained Cost-Aware Expert Replication For Efficient Mixture-of-Experts Serving》补充材料. MLSys，2026。[原文](https://proceedings.mlsys.org/paper_files/paper/2026/file/3a7f9e485845dac27423375c934cb4db-Supplemental-Conference.pdf)。
