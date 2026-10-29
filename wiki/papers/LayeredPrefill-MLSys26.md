---
type: paper
name: Layered Prefill
full_title: "From Tokens to Layers: Redefining Stall-Free Scheduling for MoE Serving with Layered Prefill"
authors: [Gunjun Lee, Jiwon Kim, Jaiyoung Park, Younjoo Lee, Jung Ho Ahn]
venue: MLSys
year: 2026
tags: [llm-serving, moe, prefill, scheduling, energy-efficiency]
source_pdf: "[[c0f460c6d63599ea870ba9db63dc96a9.pdf]]"
source_md: "[[c0f460c6d63599ea870ba9db63dc96a9_1-12]]"
review_status: needs-review
evidence_level: full-text
last_reviewed: 2026-10-29
---

# Layered Prefill：沿模型层调度 MoE 预填充（MLSys 2026）

> **原题**：From Tokens to Layers: Redefining Stall-Free Scheduling for MoE Serving with Layered Prefill

> **一句话总结**：把新请求的 prefill 按模型层分组，分散到连续的 decode 迭代中，尝试减少小 token chunk 对 MoE 专家权重的重复读取。[^paper]

**版本与阅读状态**：会议正式 PDF 与仓库旧版不同，已对正式版使用 MinerU 分两段解析并通读正文、参考文献，回看方法图及关键表格。论文的能耗分母在方法说明和表格标题中不一致，暂标待复核；会议页面未列独立补充材料。正式版后半段解析见 [[c0f460c6d63599ea870ba9db63dc96a9_13-24]]。[^paper]

## 问题与动机

共置服务中，长 prefill 可能拖慢已有请求的 decode。常见 chunked prefill 将长 prompt 沿 token 切成小块，每轮混合一块 prefill 和 decode，以限制每次迭代的时间。[^paper]

对于 MoE，同一 prompt 的每个 chunk 都要经过全部层；多个 chunk 可能反复加载相同专家权重。chunk 越小，单次专家计算中的 token 越少，权重复用不足和读取次数增加可能抵消稀疏激活的收益。论文关注的是这一结构与服务延迟约束之间的冲突（§2–3）。[^paper]

## 关键观察 / 隐含假设

- **专家路由与批量有关**：论文在 Qwen3-30B-A3B 上测量，不同 decode 批量覆盖的专家比例变化明显，无法假设每个 chunk 都只加载固定少量专家（表 1）。[^paper]
- **层间中间状态可保留**：新的 prefill 在多个迭代间推进，系统必须保存未完成请求的激活、位置和 KV；显存占用与调度开销需要计算。
- **服务约束依赖实际迭代时间**：每一轮都包含 decode 不代表 TBT 必然达标；prompt 太长或层组太大时，同轮 prefill 仍可造成延迟。
- **收益依赖模型和负载**：论文的 dense Qwen3-8B 消融中该策略慢于 chunked prefill，不能推广成所有模型的通用替代（§5.13）。[^paper]

## 核心方法

### 从按 token 切分改为按层推进

将模型划成连续的层组。每次服务迭代中，一个指定层组执行新请求 prefill 和已有请求 decode，其余层组只执行 decode；下一轮将新请求的 prefill 推进至下一个层组，直至完成全部层（§4.1–4.2、图 1）。[^paper]

例如四个层组对应四轮：第一轮新请求在第 1 组执行，第二轮在第 2 组执行，依此类推；已有 decode 每轮仍穿过全部模型层。与一个 prompt 拆成多个 token chunk、每个 chunk 都经过所有层不同，纯分层方案让该 prompt 对每层进行一次 prefill，从而避免由 token 切块引入的多次专家权重读取。[^paper]

### 层组数与输入分块

组数越多，每轮的新 prefill 层数通常越少，有利于降低 TBT 压力，但新请求需要更多轮才能完成 prefill。论文按输入长度选组数：`max(1, ceil(L / 512))`，意在与 512 token 的 chunk 基线对齐工作粒度；它是实验启发式，并非所有模型和上下文下的最优配置（§4.4）。[^paper]

极长输入还可以同时使用较大的 token chunk 与层组调度；一旦重新切成多个 chunk，每层仍可能在不同 chunk 之间重新加载权重，因此不能把“每层只执行一次”无条件应用于组合方案（§4.3）。[^paper]

## 设计取舍

- **TTFT 与 TBT**：更多层组通常提高 prefill 完成所需轮数，却能降低单轮对 decode 的干扰；请求排队与长度分布同样重要。
- **专家复用与中间状态**：整段 prompt 在一个层组内处理可增加复用，但长输入的中间激活和 KV 更大。
- **小 chunk 与计算效率**：放松 TBT 后，chunk 基线可增大 token 块；论文观察到两种方法收益逐渐接近，不能忽略这种调参对照。

## 实验与结果

主要实验为 2×H100、TP、Qwen3-30B-A3B 和 GPT-OSS-20B，流量采用泊松到达；ShareGPT 平均输入 2,340 token，arXiv 摘要数据平均输入 9,194 token（§5.1）。[^paper]

| 结果 | 实验条件与边界 |
|---|---|
| 时延 | Qwen、arXiv、1.3 请求/秒，表 6 的平均 TTFT 从 2.80 秒降至 1.24 秒，p99 从 8.65 秒降至 4.10 秒；不能把该降幅套到所有负载。 |
| 专家读取 | Qwen、100 请求的统计中，arXiv 从 35.6 TB 降至 21.7 TB，ShareGPT 从 28.5 TB 降至 25.1 TB；分别约下降 39% 和 12%（表 7）。 |
| 能耗 | 表 8 的 Qwen 数值从 56.6 降至 51.7 mJ/tok 是相同 1.3 请求/秒下约 9% 的下降；从 56.6 到 44.2 的约 22% 对应方案提高到 1.6 请求/秒，负载不同。分母口径存在下文所述疑点。 |
| 层组权衡 | Qwen、arXiv、1.4 请求/秒，层组数从 2 增至 16，平均 TTFT 从 0.480 秒增至 1.27 秒，p99 TBT 从 138 ms 降至 35.7 ms（表 11）。 |
| dense 反例 | Qwen3-8B、单 H100、arXiv、1.5 请求/秒，平均 TTFT 为 1.45 秒，对比 chunked 的 0.955 秒；平均 TBT 也从 16.2 ms 增至 20.3 ms（§5.13）。 |

**待复核的指标定义**：§5.1 将每 token 能耗定义为总 GPU 能耗除以输入与输出 token 之和；§5.6 和表 8 标题却写作每输出 token 能耗。作者当前公开仓库的 README 也采用输入与输出之和，但这不能单独证明论文表 8 的实际计算脚本及实验版本；因此不擅自认定表格的绝对单位分母。[^paper][^code]

正文 §5.7 称表 9 所有配置的 TBT 均保持或改善，但表中部分 p99 TBT 上升：例如 Qwen3-235B 的 ShareGPT 配置从 70.2 ms 增至 88.5 ms。应按具体条件看 TTFT 与尾延迟，而不是引用“所有 TBT 均改善”。[^paper]

## 论断—证据表

| 论断 | 证据 | 评测边界 | 置信度 |
|---|---|---|---|
| 分层调度可减少所测 MoE 专家读取 | §5.4、表 7 | 模型、长度分布和 token chunk 策略影响收益 | 中高 |
| 可改善所测工作负载的延迟权衡 | 表 6、9、11 | 不保证所有配置的 TBT 均下降 | 中高 |
| 论文表格的 mJ/tok 明确只按输出 token 计算 | §5.1、表 8 | 两处定义冲突，需查看实现或作者说明 | 低 |

## 批判性分析

### 论证链条

论文从小块 prefill 的专家读取放大出发，将执行粒度改为层组，再用权重读取、SLO、能耗和层组消融支持这一机制。数据支持特定 MoE 设置下的改善；把不同请求率的能耗对比视作完全同负载的纯机制收益则不成立。[^paper]

### 假设压力测试

专家权重缓存命中、量化、不同路由分布及更长上下文都可能改变读取与计算瓶颈。组数公式随输入长度增加，而层数有限；论文将不均匀层组和更复杂多 GPU 环境列为后续工作，不能假定其覆盖所有长度与架构。[^paper]

### 实验可信度

作者提供多个模型、GPU 和 dense 反例，且比较不同层组数；但表 8 的能耗分母和表 9 的 TBT 全面改善说法需要进一步核对。分离式服务比较仅在两卡上分别分给 P、D，不能代表其他集群规模与资源比例。[^paper]

## 局限与后续工作

- 查验能耗统计代码或作者说明，确定表 8 的 token 分母；核对实际实现对超长 prompt 与层组数的上限处理。
- 按相同到达率、模型、硬件和 SLO，对比经过调优的 chunk 大小、层组及组合策略，并测量中间激活和尾延迟。

## 相关

- [[MoEServingTax-MLSys26]]：分析 MoE 推理中专家权重和计算调度成本；本文进一步改变 prefill 的调度维度。
- [[BEAM-MLSys26]]：联合调节 chunk 和 GPU 频率节能，关注对象和能耗基线不同。
- [[LLM-Serving]]：服务阶段调度；本文待复核能耗口径后再决定是否列入核心成员。

## 参考文献

[^paper]: Gunjun Lee 等. 《From Tokens to Layers: Redefining Stall-Free Scheduling for MoE Serving with Layered Prefill》. MLSys，2026。[原文](https://proceedings.mlsys.org/paper_files/paper/2026/hash/c0f460c6d63599ea870ba9db63dc96a9-Abstract-Conference.html)。
[^code]: scale-snu. 《Layered Prefill》. GitHub。[原文](https://github.com/scale-snu/layered-prefill)。
