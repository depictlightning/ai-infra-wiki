---
type: paper
name: PRISM
full_title: "PRISM: Parametrically Refactor Inference for Speculative Decoding Draft Models"
authors: [Xuliang Wang, Yuetao Chen, Maochan Zhen, Fang Liu, Xinzhou Zheng, Xingwu Liu, Hong Xu, Ming Li]
venue: MLSys
year: 2026
tags: [llm-inference, speculative-decoding, draft-model, sglang, area/llm-inference]
source_pdf: "[[414fd191b3246a19a55741b938380136.pdf]]"
source_md: "[[414fd191b3246a19a55741b938380136]]"
review_status: complete
evidence_level: full-text
last_reviewed: 2026-10-28
---

# PRISM：按草稿步分配参数的投机解码（MLSys 2026）

> **原题**：PRISM: Parametrically Refactor Inference for Speculative Decoding Draft Models

> **一句话总结**：投机解码中较远的草稿 token 更难被接受，PRISM 为不同草稿阶段分配参数以增加有效容量；在 A800、batch 1 的所测任务中，相对 EAGLE-2 平均生成吞吐提高 14.21%（§3、§4.1）。[^paper]

**版本与阅读状态**：使用 MLSys 论文集提供的正式版 PDF 与独立补充文件，均经 MinerU 解析；仓库旧版 PDF 与正式版不同。已阅读正文、附录并回看主要图表。本文 PRISM 与 [[Prism-OSDI26]] 是不同系统。[^paper][^supplement]

## 问题与动机

投机解码先由草稿模型生成候选，再由目标模型并行验证；草稿越准，可接受的连续 token 越多，但更大的草稿也可能增加延迟。论文研究如何在推理时激活相近参数量的条件下提高草稿有效容量（§1–2）。[^paper]

作者观察到更远的草稿步通常有更低接受率。PRISM 依据草稿阶段改变使用的参数路径，而非简单堆叠更多层并在每个草稿步都执行完整堆叠（§3.1、图 1–2）。[^paper]

## 关键观察 / 隐含假设

- **草稿步难度并不均匀**：距离当前目标状态越远，作者测得的接受率越低；分配更有针对性的参数可能改善后续候选，但具体规律受任务和模型影响（§3.1、图 1）。[^paper]
- **提高接受长度不必然提高吞吐**：额外参数、KV 和验证开销必须小于减少目标模型调用的收益；高并发下瓶颈变化也会改变结果（§4.1、§4.3）。[^paper]
- **专用模块需要足够训练信号**：论文观察三模块版本在较大数据规模下未必优于两模块版本，作者将训练样本分散和后续步噪声列为可能原因，而非已证明的唯一因果（§4.2、图 7）。[^paper]

## 核心方法

PRISM 草稿模型由 token 嵌入、输出头和多个处理模块构成；每个模块包含特征融合层与 Transformer 层。目标模型预填充或验证后的隐藏状态进入草稿，后续草稿步根据预先设计的映射切换处理模块并传递状态（§3.2、图 3）。[^paper]

主要实验采用两个模块：一个处理草稿的预填充，另一个处理后续解码步；共享 KV 在模块间传递。训练先用单模块预热，再复制权重并切换模块继续训练（§3.2、§4）。[^paper]

论文将实现集成到 SGLang，并用相同训练数据训练比较草稿模型；其提速仍依赖目标模型验证候选，不能把草稿直接输出描述成与目标模型自动等价（§3.3–4）。[^paper]

## 设计取舍

- **更大可用容量换额外驻留**：所测 PRISM 比单层草稿多占用 576 MB GPU 内存；内存紧张时需计入批次容量损失（§4.1）。[^paper]
- **步级专门化换训练样本分散**：模块增加可能提高表达能力，也可能让各模块获得更少的有效训练信号。
- **接受率换验证与草稿成本**：较深或较宽的候选树影响验证工作量；吞吐需要与接受长度同时测量。

## 实验与结果

主要实验使用单张 A800 80 GB，batch 为 1；目标模型为 LLaMA-2-Chat-7B 和 LLaMA-3-Instruct-8B。草稿树为 6 步、4 分支，最多验证 16 个 token；也提供 H800 对照（§4、表 4–5）。[^paper]

| 结果 | 条件与基线 |
|---|---|
| 相对普通自回归解码加速超过 2.4 倍 | A800、batch 1 的所测任务；该结果不等于相对 EAGLE-2 的加速（§4.1）。 |
| 生成吞吐相对 EAGLE-2 平均提高 14.21% | 所测草稿激活参数量相近，跨任务的平均值；相对 HASS 平均提高 6.10%（§4.1、表 4）。 |
| 平均接受长度相对 EAGLE-2 提高 14.09% | 相对 HASS 提高 5.69%；接受长度和吞吐是不同指标（§4.1）。 |
| 多模块消融出现非单调结果 | 三模块在较小数据量有优势，随数据增大被两模块超过；作者提出训练信号分散等解释（§4.2、图 7）。 |
| 补充实验的接受长度并非逐项最高 | 2 张 RTX 4090、LLaMA-3-8B、HumanEval、温度 0：Standard 的接受长度为 5.93，PRISM 为 5.44；吞吐分别为 171.76 和 252.18 tokens/s（补充文件表 7）。 |

补充文件文字声称 PRISM 在所有基准上均取得最佳接受长度，但上述表 7 中 Standard 更高；因此接受长度与吞吐应分别依据具体数据判断。补充文件图 10 的数据规模实验采用 4 步加 1 个 bonus token、4 分支和最多验证 16 个 token；其 PRISM* 变体使用目标模型三个层的隐藏状态，不能和正文不同树配置无条件合并。[^supplement]

## 论断—证据表

| 论断 | 证据 | 评测边界 | 置信度 |
|---|---|---|---|
| 草稿步可使用不同参数路径 | §3.1–3.2、图 1–3 | 所测草稿架构和目标模型 | 中高 |
| 专门化改善所测草稿效率 | §4.1–4.2、表 4–5 | 单 GPU、主要 batch 1 和指定草稿树 | 中高 |
| 更多处理模块不保证更优 | §4.2、图 7 | 两模块和三模块的所测训练设置 | 中高 |

## 批判性分析

### 论证链条

接受率随草稿步降低为分步专门化提供动机；作者比较相近激活参数量及简单堆叠草稿，检验收益是否只来自增加参数。与基线的吞吐差距明显小于相对普通自回归解码的倍数，比较时必须区分。[^paper]

### 假设压力测试

高并发、长上下文或更强目标模型可能改变草稿与验证的资源占比；论文的主要实验以 7B/8B、batch 1 为主，不能据此保证大型服务的有效吞吐同幅提升。[^paper]

### 实验可信度

论文使用多个基准和同一草稿训练数据，并报告额外显存。批量增大时投机解码加速下降，图 8 展示了最高 batch 32 的结果；候选树敏感性也有图 9。论文集另提供附录 B 的不同候选树扩展实验，配置与正文实验必须分别记录（§4.2–4.3、补充文件图 10）。[^paper][^supplement]

### 系统性缺陷

多个草稿模块、KV 传递和预训练增加集成复杂度；训练开销、主机/设备容量以及与不同调度器的交互应纳入部署成本。[^paper]

## 局限与后续工作

- 在不同并发、输入长度与候选树大小下联合测量接受长度、TPOT、显存和 SLO 达标吞吐。
- 对不同时期生成的草稿负载测量每步接受率，验证固定步骤与参数映射是否持续有效。

## 相关

- 同会议：[[MLSys-2026]]。
- 相邻研究：[[DSpark-arXiv26]]、[[Sereno-OSDI26]]。

## 参考文献

[^paper]: Xuliang Wang 等. 《PRISM: Parametrically Refactor Inference for Speculative Decoding Draft Models》. MLSys，2026。[原文](https://proceedings.mlsys.org/paper_files/paper/2026/hash/414fd191b3246a19a55741b938380136-Abstract-Conference.html)。
[^supplement]: Xuliang Wang 等. 《PRISM: Parametrically Refactor Inference for Speculative Decoding Draft Models》补充材料. MLSys，2026。[原文](https://proceedings.mlsys.org/paper_files/paper/2026/file/414fd191b3246a19a55741b938380136-Supplemental-Conference.pdf)。
