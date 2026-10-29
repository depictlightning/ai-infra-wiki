---
type: paper
name: MAC-Attention
full_title: "MAC-Attention: a Match--Amend--Complete scheme for fast and accurate attention computation"
authors: [Jinghan Yao, Sam Ade Jacobs, Walid Krichene, Masahiro Tanaka, Dhabaleswar K Panda]
venue: MLSys
year: 2026
tags: [llm-inference, long-context, attention-reuse, kv-cache, gpu-kernel]
source_pdf: "[[7398289396de403d7d0505ed791e704a.pdf]]"
source_md: "[[7398289396de403d7d0505ed791e704a]]"
review_status: needs-review
evidence_level: full-text
last_reviewed: 2026-10-29
---

# MAC-Attention：复用相似查询的历史注意力结果

> **原题**：MAC-Attention: a Match--Amend--Complete scheme for fast and accurate attention computation

> **一句话总结**：当当前 query 与近期某次 query 相似时，复用其历史前缀的注意力摘要，重新计算靠近匹配点的短区间和新尾部，以减少长上下文 decode 的 KV 读取。[^paper]

**阅读状态**：已阅读会议正式版 MinerU 正文与附录，核对设计图、结果表和消融图；辅助缓存的显存统计与窗口配置描述需复核。会议页面未列独立补充材料。[^paper]

## 问题与动机

长上下文 decode 每生成一个 token 都可能重新读取很长的 KV 前缀。压缩或选择 KV 可以减少访问，但可能影响后续查询能看到的信息。MAC-Attention 保留完整 KV，尝试复用历史注意力计算结果，减少同一请求连续生成中的重复读取（§1–3）。[^paper]

保留 KV 不等于输出严格无损：历史摘要由旧 query 计算，用来近似当前 query 对历史前缀的注意力；只是需要时仍可退回完整注意力。[^paper]

## 关键观察 / 隐含假设

- **查询重复有局部性**：最近一段 query 中可能存在与当前 query 相似者；若未命中，执行完整注意力。命中率取决于模型、层、任务和阈值。
- **位置编码会影响相似性**：论文在 RoPE 前匹配以提高命中；两个位置的 RoPE 后 query 不一定同样接近，因此匹配本身不构成精确等价证明。[^paper]
- **短区间修正有经验前提**：如果误差集中在匹配位置附近，重算该区间可改善精度；长距离依赖或分散注意力可能需要更大区间。
- **计算缓存也有成本**：每请求需要近期 query 环形缓存和对应的注意力摘要；匹配、调度及更新开销可能抵消短上下文的收益。[^paper]

## 核心方法

### Match：查找相似 query

每个请求、注意力头维护最近 $K$ 个 RoPE 前 query 及其摘要，使用 L2 距离在窗口内寻找候选，并根据阈值 $\tau$ 判断能否复用。L2 同时考虑方向和向量大小；没有合适候选时回退完整注意力。[^paper]

### Amend：修正边界

设当前 token 为 $m$、匹配位置为 $p$，历史摘要只保留到 $p-r$，匹配点附近宽度为 $r$ 的区间改用当前 query 重新计算。摘要更新与主路径可以通过辅助 CUDA stream 重叠，但仍需确保依赖完成后才供下一次使用（§3–4）。[^paper]

### Complete：合并前缀和尾部

历史前缀缓存保存加权 value 总和与 softmax 归一化统计；当前 query 重新计算从 $p-r+1$ 到 $m$ 的区间，然后通过数值稳定的在线合并得到注意力输出。公式中的前缀摘要是旧 query 的近似结果；合并公式本身的稳定性不消除这种近似误差。[^paper]

## 实验与结果

论文使用 SGLang 0.4.9、FlashInfer 0.2.7 和 H100 SXM5，默认主要改变 decode 注意力路径。基准涵盖 LongBench v2、RULER 和长生成任务 LongGenBench；常用搜索窗口 1,024、修正带 256。[^paper]

| 实验 | 结果与边界 |
|---|---|
| 任务质量 | 表 1 的 Llama-3.1-8B 在 LongBench v2 上：完整注意力 29.0，MAC 30.2；RULER 为 79.8 与 78.8。测试分数接近不意味着生成内容逐 token 相同。 |
| 结束率 | 同表的 8B LongGenBench 完成率从 93.1% 到 90.0%，模型评审准确率从 34.7 到 38.2；应同时看完成率和准确率，而非只挑选增加的数字。 |
| 注意力路径 | 表 3 的 120K、1% KV 预算下，FlashInfer 注意力 234.2 μs、MAC 注意力路径 62.9 μs，约 3.72×；表题的 end-to-end 指注意力路径，不是完整模型 decode。 |
| 完整 decode | 图 7 报告随上下文和跳过比例变化的整体 decode 延迟，128K、99% 跳过的示例约 2.61×；其他模型层和运行时开销限制整体收益。 |
| 修正消融 | 图 4 按复用间隔和带宽展示误差；更宽的修正带可降低所测误差，但更宽也会增加重新读取量。 |
| Prefill 扩展 | 附录表 8 将该机制用于 prefill 和 decode：若允许全局匹配，多个设置整体分数低于完整注意力的 29.0；论文称这是探索性实验，而非常规生产配置。 |

表 1 中的 KV 百分比表示相对完整注意力的读取预算减少，不能当成原始 KV 缓存容量减少；表 3 的 KV 百分比表示保留访问预算，两个表的列含义相反。[^paper]

## 论断—证据表

| 论断 | 证据 | 边界 | 置信度 |
|---|---|---|---|
| 相似 query 的摘要可减少长前缀重复读取 | 图 2、表 3 | 命中与区间修正依赖任务和阈值 | 中高 |
| 所有输出与完整注意力完全相同 | §3、表 1 | 摘要由旧 query 生成，仅是近似；部分基准分数下降 | 低 |
| 额外缓存的相对开销很小 | §4.1、§5.13、表 7 | query 与向量摘要具体计入方式待核对 | 中低 |

## 批判性分析

### 论证链条

方法利用临近 query 的相似性复用前缀摘要，再修正局部高权重区域并计算新增尾部。质量表、误差热图和路径计时分别展示可行性；这不等于对每个 query 或全部长上下文任务提供精确保证。[^paper]

### 假设压力测试

RoPE 前距离小并不直接约束不同位置 RoPE 后的 query 距离；对于位置敏感、长距离匹配或注意力质量集中在修正区间外的输入，历史摘要误差可能累积。命中率和跳过比例高也不自动说明输出质量高，附录 prefill 实验给出了需要额外约束的场景。[^paper]

### 实验可信度

论文 §5.13 将辅助开销主要归因于 query ring，并称摘要只需少量标量；但 §3–4 定义的摘要同时包含按 value 维度的加权向量与归一化统计。表 7 的百分比是否完整包含摘要向量及其数据类型，需要核对实现；不能仅凭文字确定实际显存。论文也称实验窗口固定为 1,024，但图 6 图注写 512，应保留具体图表配置。[^paper]

## 局限与后续工作

- 核对摘要向量、数值稳定状态和每请求环形缓存的实际显存，并在不同 batch 和上下文长度下计入更新开销。
- 测量不同位置编码、长距离匹配和注意力分散情况下的输出误差，必要时限制匹配距离或回退完整计算。
- 将注意力路径速度、完整 decode 延迟和任务质量放在同一配置中比较，避免混合不同预算。

## 相关

- [[HiSparse-arXiv26]]：通过选择与分层驻留减少所需 KV 访问；MAC 保留完整 KV 并近似复用历史注意力摘要。
- [[BLASST-MLSys26]]：基于当前 QK 分数跳过低贡献块；两者均是近似，跳过依据与保存的状态不同。

## 参考文献

[^paper]: Jinghan Yao 等. 《MAC-Attention: a Match--Amend--Complete scheme for fast and accurate attention computation》. MLSys，2026。[原文](https://proceedings.mlsys.org/paper_files/paper/2026/hash/7398289396de403d7d0505ed791e704a-Abstract-Conference.html)。
