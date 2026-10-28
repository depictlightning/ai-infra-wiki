---
type: paper
name: ScaleSearch
full_title: "Search Your Block Floating Point Scales!"
authors: [Tanmaey Gupta, Hayden Prairie, Shirley Wu, Reyna Abhyankar, Qingyang Wu, et al.]
venue: MLSys
year: 2026
tags: [llm-inference, kv-cache, attention, quantization, fp4, area/llm-inference, area/kv-cache-systems]
source_pdf: "[[633b0e871a48d542280c3ad03928e60d.pdf]]"
source_md: "[[633b0e871a48d542280c3ad03928e60d]]"
review_status: complete
evidence_level: full-text
last_reviewed: 2026-10-28
---

# ScaleSearch：搜索低位宽浮点块的量化尺度（MLSys 2026）

> **原题**：Search Your Block Floating Point Scales!

> **一句话总结**：ScaleSearch 在块最大值确定的默认量化尺度附近搜索，使块重建误差更低；论文进一步将其用于 NVFP4 注意力和 KV cache，但质量、算子效率和端到端实验分别来自不同设置。[^paper]

**阅读状态**：已阅读会议正式版 MinerU 全文和独立补充文件，并核对关键质量、量化开销和注意力吞吐表图。[^paper][^supplement]

## 问题与动机

块浮点量化为一小组数共享尺度，常见做法根据块中最大绝对值选择尺度，使最大元素落在可表示范围内。但让最大值精确映射不保证整个块的均方误差最小；尤其 NVFP4 的尺度本身有尾数位，可以在邻近的可表示尺度中选择更好的值（§3–4）。[^paper]

这一问题同时影响 LLM 权重后训练量化、注意力计算和 KV 表示。论文还提出 ScaleSearchAttention，将尺度搜索与低精度注意力和混合精度 KV 组合（§4.2）。[^paper]

## 关键观察 / 隐含假设

- **最小块 MSE 不等于最优任务准确率**：优化的是量化重建误差，模型输出还受误差传播与任务分布影响，需要独立质量评测（§4–5）。[^paper]
- **搜索范围换额外工作**：更宽的候选范围可能降低误差，但也会增加量化时间；论文的主要 NVFP4 设置搜索默认尺度附近的有限偏移（§4.1、§5.3）。[^paper]
- **低位宽执行依赖硬件**：NVFP4 Tensor Core 是论文注意力执行路径的条件，不能将其吞吐直接套用于不支持该路径的设备（§3、§4.2）。[^paper]

## 核心方法

### 搜索块尺度

先按照块最大绝对值获得默认尺度，再在尺度表示中应用有限范围的相邻偏移。对每个候选尺度执行量化与重建，计算块内平方误差并选择最小者。论文用合成高斯数据和 Llama Key 状态观察候选偏移分布，主要 NVFP4 实验使用 −2 到 +6 的偏移范围；它是有限范围搜索，并非每次遍历所有可能尺度（§4、算法 1）。[^paper]

### ScaleSearchAttention

方法对 Q、K、V 和注意力概率矩阵应用低精度表示，并为 Q/K 使用变换以减轻异常值与量化误差。量化后的矩阵乘采用 NVFP4 Tensor Core 和 FP32 累加。初始 KV 块与尚未填满的最新块保持较高精度，其余块存为 NVFP4；混合部分分别计算并合并（§4.2、图 8）。[^paper]

由此可减少 KV 存储和部分矩阵乘成本，但保留 FP16 块、尺度及其他状态仍占空间；低精度计算不表示完整模型所有运算都是 FP4，也不保证输出与原模型一致。[^paper]

## 设计取舍

- **多次试算换较低误差**：更多候选尺度提高量化成本，是否值得取决于目标质量与执行路径。
- **变换和混合精度换精度恢复**：需要处理额外变换、缓存和两种矩阵乘，而非纯粹替换一个尺度。
- **优化内核换硬件约束**：支持 NVFP4 的设备和矩阵布局影响收益与可移植性。

## 实验与结果

论文分别评估后训练量化、语言模型注意力质量、注意力核吞吐和视频模型延迟。这些实验使用的模型、算法组合和指标不同，不能将某一项的收益直接当作全部场景的端到端收益（§5）。[^paper]

| 结果 | 条件与边界 |
|---|---|
| Llama 3.1 70B 的 WikiText-2 PPL | 原始精度 2.5554，naive FP4 3.4000，ScaleSearchAttention 2.6348；比 naive FP4 降低约 0.77，但仍高于原始精度（表 3）。 |
| Llama 3.1 8B 的消融 | 完整方法 PPL 为 5.4977，去掉 ScaleSearch 为 5.5024，去掉混合精度 KV 为 5.5768；完整方法收益来自多个组件（表 5）。 |
| 块量化耗时 | 高斯矩阵 2048×2048，FP32→NVFP4 基线 0.0258 ms；搜索 −1 至 1 为 0.0328 ms，搜索 −2 至 6 为 0.0449 ms，分别约 1.27 和 1.74 倍量化时间（表 6）。 |
| 注意力吞吐 | RTX 5090、32K 序列，SageAttention3 加 ScaleSearch 在所测因果 attention 达到原 SageAttention3 吞吐的 97.5%；这是注意力核比较，不是完整 ScaleSearchAttention 的 LLM 服务吞吐（图 9、§5.3）。 |

论文 §4.1 写合成实验 MSE 从 0.0990 降至 0.0066，并称降低约 25%；这两个数字相减约降低 93%，且图 3 的起点约为 0.009，而非 0.099。原文数字、图与百分比存在不一致，这里不自行替作者更正。[^paper]

论文的端到端延迟表测试的是 Mochi 和 CogVideoX 视频生成，不是 LLM；语言模型 PPL 表中的 SageAttention3 由作者模拟，因为其提供的代码在因果场景数值不稳定（§5.2–5.3）。[^paper]

## 论断—证据表

| 论断 | 证据 | 评测边界 | 置信度 |
|---|---|---|---|
| 限定范围的尺度搜索能减少所测块量化误差 | §4、算法 1、图 3 | 正文 MSE 数字有笔误或口径问题；分布和搜索范围影响效果 | 中 |
| 组合方法改善所测语言模型低精度注意力质量 | §5.2、表 3–5 | 对照含模拟实现；PPL 不代表全部任务质量 | 中高 |
| 该方案已证明 LLM 服务端到端吞吐提升 | §5.3 | 核吞吐和视频端到端实验不等于 LLM 服务测试 | 低 |

## 批判性分析

### 论证链条

论文先展示默认尺度与块误差的关系，再比较低精度质量和额外搜索开销。对语言模型，完整 ScaleSearchAttention 还包含变换和混合精度 KV；表 5 表明不能将其整体质量改善全部归因于尺度搜索。[^paper]

### 假设压力测试

如果模型值分布或块大小改变，最优偏移分布可能偏离有限搜索范围；面向低延迟单请求的场景也可能更敏感于搜索与混合精度管理成本。论文没有证明所有模型和 GPU 都有相同收益。[^paper]

### 实验可信度

论文提供无量化、naive FP4 与其他低精度对照，也公开量化开销；不过 SageAttention3 在因果语言任务中是模拟版本，硬件吞吐图并非完整线上服务；合成 MSE 的数字矛盾需要额外核验。[^paper]

## 局限与后续工作

- 核对合成误差数值及搜索范围在不同模型、上下文中的稳定性。
- 在实际 LLM 推理引擎中测量混合精度 KV 占用、TPOT、吞吐与任务质量。

## 相关

- [[LLM-Inference]]、[[KV-Cache-Systems]]：低精度注意力与 KV。
- [[Kitty-MLSys26]]：Key 通道混合位宽和缓存布局，目标与量化粒度不同。

## 参考文献

[^paper]: Tanmaey Gupta 等. 《Search Your Block Floating Point Scales!》. MLSys，2026。[原文](https://proceedings.mlsys.org/paper_files/paper/2026/hash/633b0e871a48d542280c3ad03928e60d-Abstract-Conference.html)。
[^supplement]: Tanmaey Gupta 等. 《Search Your Block Floating Point Scales!》补充材料. MLSys，2026。[原文](https://proceedings.mlsys.org/paper_files/paper/2026/file/633b0e871a48d542280c3ad03928e60d-Supplemental-Conference.pdf)。
