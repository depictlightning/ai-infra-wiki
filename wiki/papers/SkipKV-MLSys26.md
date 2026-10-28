---
type: paper
name: SkipKV
full_title: "SkipKV: Selective Skipping of KV Generation and Storage for Efficient Inference with Large Reasoning Models"
authors: [Jiayi Tian, Seyedarmin Azizi, Yequan Zhao, Erfan Baghaei Potraghloo, Sean McPherson, et al.]
venue: MLSys
year: 2026
tags: [llm-inference, reasoning, kv-cache, cache-eviction, activation-steering]
source_pdf: "[[45c1f6a8cbf2da59ebf2c802b4f742cd.pdf]]"
source_md: "[[45c1f6a8cbf2da59ebf2c802b4f742cd]]"
review_status: needs-review
evidence_level: full-text
last_reviewed: 2026-10-28
---

# SkipKV：按句子压缩推理轨迹的 KV（MLSys 2026）

> **原题**：SkipKV: Selective Skipping of KV Generation and Storage for Efficient Inference with Large Reasoning Models

> **一句话总结**：SkipKV 结合句子相似性引导的 KV 淘汰、激活引导和批次分组，减少推理模型的缓存与输出长度；它会改变生成路径，论文一个核心集合定义与文字存在待核对的不一致。[^paper]

**阅读状态**：已阅读会议正式版 MinerU 全文及附录，核对核心公式、吞吐表和 vLLM 实验。式 3 的比较范围与正文解释不一致，论文链接的作者仓库目前只有 README，无法用实现核实，暂保留待复核状态。[^paper][^code]

## 问题与动机

长推理链产生大量 KV 和输出 token。已有逐 token 淘汰方法可能移除推理过程中的局部数字或上下文，使生成轨迹变长；固定预算批处理中，长度不一导致的 padding 还会占据有效 KV 容量。作者据此研究以句子为单位识别重复，并减少后续无效生成（§1、§3）。[^paper]

论文展示了淘汰后出现重复验证的案例和相关统计；这为设计提供动机，但单个例子和相关性不能独立证明被删 token 是所有过度推理的唯一原因（§3、图 3–5）。[^paper]

## 关键观察 / 隐含假设

- **相似不等于可安全删除**：重复语句可能包含必要条件或数值；语义分数、保留预算和任务类型会影响准确率（§4.1）。[^paper]
- **引导会改变输出**：steering 修改隐藏状态，淘汰也改变模型可见历史；不能声称与完整 KV 的输出逐 token 等价（§4.2）。[^paper]
- **分组依赖批次条件**：按输入长度预排序适合已有样本的静态批处理；论文的 vLLM 连续批处理实验没有使用分组（§4.3、附录 A.3）。[^paper]

## 核心方法

### 句子与 token 联合评分

系统按生成分隔符记录句子范围，用某层隐藏状态的句内均值形成句向量，识别高度相似的句子；再结合 token 注意力重要性、KV 冗余和句子相似分数排序，在缓存达到预算时优先淘汰分数低的 token。论文每 128 个解码步进行一次压缩；句子分数偏向整段删除，但剩余预算不足时仍可能进行 token 级淘汰（§4.1、附录 A.2）。[^paper]

生成序列与淘汰后的 KV 数组不再具有相同位置。系统维护从原始句子范围到当前缓存范围的映射，并在每次淘汰后更新，避免把句子分数应用到错误位置（§4.1、图 7）。[^paper]

**公式疑点**：正式版式 3 写作 `i ≤ j`，而正文称识别相似句对后将较早句子 i 标记、保留较晚句子 j。如果按式 1 的余弦相似度并允许 i = j，非零句向量与自身相似度为 1，低于 1 的阈值下会把自身计为重复；式 3 又用向量点积表达相似度，未在该式交代归一化。论文并未明确排除对角线。仅凭文字不能确定实现是否实际排除自身比较；公开仓库目前没有代码供核对。[^paper][^code]

### 自适应 steering 与批次分组

作者用 MATH 训练集中的 1000 个样本构造执行与非执行思路的平均隐藏状态差，在指定层和分隔符处注入向量；根据已观察到的非执行思路数量动态提高强度。论文将这种区分视为统计关联，而不声称所有反思都无用（§4.2）。[^paper]

静态批处理中，按 prefill 长度排序并组合相近样本，以减少 padding 占据固定 KV 预算。此方法需要可提前组织样本；附录 vLLM 连续批处理采用分页 KV，没有使用该分组优化（§4.3、附录 A.3）。[^paper]

## 设计取舍

- **压缩换模型质量风险**：淘汰永久移除历史，无法像主机保存完整 KV 的方法那样回读。
- **更短输出换语义变化**：steering 可能减少重复验证，也可能抑制对某些难题有价值的检查。
- **排序换在线约束**：静态预排序减少 padding，但在线请求存在到达时间、等待和公平性约束。

## 实验与结果

论文评测 R1-Qwen-7B、R1-Qwen-14B 和 R1-Llama-8B，任务包括 AIME-24、MATH-500、GSM8K 和 LiveCodeBench。预算比例以各模型与数据集完整 KV 的平均生成长度为分母，不能直接当成全部设备显存的节省比例（§5、图 9）。[^paper]

| 结果 | 条件与边界 |
|---|---|
| 分组减少 padding 占用 | MATH-500、R1-Qwen-7B、预算 768：平均有效预算从 630 增至 759，准确率从 77.8% 增至 83.4%；这是 5.6 个百分点（表 3）。 |
| 句子评分和 steering 的消融并非每列都改善 | AIME-24、预算 4096 时，仅加句子评分的总 token 长度从 R-KV 的 11403 增至 11819；完整方案降至 7988（表 2）。 |
| GSM8K 静态批次吞吐 | 单 A100 40 GB，batch 50 时 SkipKV 为 22.7 samples/min，R-KV 为 13.7，约 1.66 倍；batch 100 时分别为 25.4 和 20.0，约 1.27 倍（表 1）。 |
| 跨批次对 FullKV 的比较 | 表 1 中 FullKV 的 batch 50 为 2.64 samples/min，SkipKV 的 batch 100 为 25.4，比例约 9.6；表内 FullKV 的 batch 10 达 4.07，对照 SkipKV 的 25.4 约为 6.2。以上都是所列配置，不能作为同批次加速（表 1）。 |
| vLLM 连续批处理 | GSM8K、预算 512、batch 64：SkipKV 44 分钟、84.4% 准确率；R-KV 51 分钟、73.8%。没有使用静态 batch grouping（附录 A.3、表 4）。 |

## 论断—证据表

| 论断 | 证据 | 评测边界 | 置信度 |
|---|---|---|---|
| 联合机制可改善所测推理任务的质量与缓存权衡 | §5、表 2–3 | 特定模型、任务和预算；公式实现存在疑点 | 中 |
| 减少 padding 有助于静态批处理 | §4.3、表 3 | 不直接适用于论文的 vLLM 连续批处理实验 | 中高 |
| 比较 9.6 倍代表同批次服务加速 | 表 1 | 该比例对应不同 batch 的两个数据点 | 低 |

## 批判性分析

### 论证链条

方法把句子完整性、推理方向和批内 padding 放在同一方案；消融说明完整组合有收益，但各机制的收益随预算和任务变化，不能把改善全归因于句子级淘汰（表 2）。[^paper]

### 假设压力测试

重复表述可能包含必要中间结论，强制偏向执行式思路也可能伤害需要验证的任务。分隔符与模型隐藏层选择均具有工作负载依赖；尤其代码、长文或不同语言的分段应单独验证。[^paper]

### 实验可信度

论文给出不同预算、模型及 vLLM 实验，但部分指标由不同 batch 比较得出；句子比较集合的对角线处理尚无公开实现可确认，影响对算法细节的精确复现。[^paper][^code]

## 局限与后续工作

- 核对作者实现或勘误中式 3 的自比较、向量归一化和句子淘汰细节。
- 在相同准确率、请求到达及预算下，对比真实在线服务的吞吐、尾延迟和永久淘汰的质量损失。

## 相关

- [[KV-Cache-Systems]]：KV 驻留与复用；本篇暂不计入核心成员，待公式核验。
- [[HiSparse-arXiv26]]：保留完整主机 KV 的稀疏访问，和永久淘汰的语义边界不同。

## 参考文献

[^paper]: Jiayi Tian 等. 《SkipKV: Selective Skipping of KV Generation and Storage for Efficient Inference with Large Reasoning Models》. MLSys，2026。[原文](https://proceedings.mlsys.org/paper_files/paper/2026/hash/45c1f6a8cbf2da59ebf2c802b4f742cd-Abstract-Conference.html)。
[^code]: SkipKV 作者. 《SkipKV》. GitHub。[原文](https://github.com/TTTTTTris/SkipKV)。
