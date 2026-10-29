---
type: theme
topic: Speculative-Decoding
title: Speculative Decoding
theme_kind: lens
member_tag: lens/speculative-decoding
candidate_tags: [speculative-decoding]
paper_count: 9
first_generated: 2026-10-29
last_updated: 2026-10-29
tags: [topic-overview, speculative-decoding, llm-inference]
---

# Speculative Decoding

## 无损性证明（四句话）

1. **核心是一个恒等式：**`p(x) = min(p(x), q(x)) + max(0, p(x) - q(x))`，目标模型分布 `p` 被拆成草稿分布 `q` 可安全贡献的部分和必须由目标模型补足的残差。
2. **接受路径贡献第一部分：**从 `q` 抽到 token `x` 后，以 `min(1, p(x)/q(x))` 接受，因此抽到且接受 `x` 的概率正好是 `min(p(x), q(x))`。
3. **拒绝路径补上残差：**总拒绝概率为 `β = 1 - Σ_x min(p(x), q(x))`，拒绝后按 `r(x) ∝ max(0, p(x) - q(x))` 重采样，贡献正好是 `max(0, p(x) - q(x))`。
4. **两部分相加逐点等于 `p`，所以单步无损；**每个位置都使用真实前缀下的条件分布 `p(· | x_{<t})`，由链式法则可知整段序列仍服从目标模型的自回归分布。

> 拒绝后不能直接按 `p` 重抽，因为接受路径已经贡献了 `min(p,q)`，再抽一次 `p` 会重复计数；残差 `max(0, p-q)` 正是由恒等式唯一确定的补足项。这里的无损性指精确分布算法，在实际浮点实现中仍可能有数值误差。

## 纳入范围

聚焦通过草稿生成、并行验证、拒绝采样或自推测减少目标模型串行解码轮次的方法，也收录直接研究这类机制的评测与系统调度论文。核心成员须有本站全文笔记；仅使用推测执行但不符合该分布校正机制的工作不自动纳入。

## 核心论文

- [[DFlash-ICML26]] — 将目标模型多层特征注入轻量草稿模型，以块扩散并行生成候选。
- [[DSpark-arXiv26]] — 依据置信度分配半自回归草稿的验证预算。
- [[SpecDecodeBench-MLSys26]] — 量化不同草稿方法的接受分布、实际吞吐与理想化收益。
- [[SparseSpec-MLSys26]] — 用同一模型的稀疏注意力生成草稿，再以完整注意力核验。
- [[PRISM-MLSys26]] — 重构草稿模型的参数执行路径，提高接受长度与吞吐。
- [[TiDAR-MLSys26]] — 结合扩散式并行草稿与自回归验证；质量结论针对适配后的模型。
- [[Dataflow-MLSys26]] — 在数据流加速器上执行草稿与目标模型的推测解码。
- [[Sereno-OSDI26]] — 调整草稿抢占和验证批次，在移动设备上控制内存带宽争用。
- [[Seer-OSDI26]] — 在强化学习 rollout 中按组共享上下文并自适应安排推测解码。

## 阅读提示

上述论文优化对象不同：算法接受率、草稿生成成本、目标模型验证、设备争用与 rollout 长尾不能用单一加速比直接比较。[[SpecDiff2-MLSys26]] 的全文笔记仍标为待复核，暂列为候选而非核心成员。

完整推理系统脉络见 [[LLM-Inference]]；强化学习训练场景见 [[LLM-Training]]。
