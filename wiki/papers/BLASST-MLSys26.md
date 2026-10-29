---
type: paper
name: BLASST
full_title: "BLASST: Dynamic BLocked Attention Sparsity via Softmax Thresholding"
authors: [Jiayi Yuan, Cameron Shinn, Kai Xu, Jingze Cui, George Klimiashvili, Guangxuan Xiao, Perkz Zheng, Bo Li, Yuxin Zhou, Zhouhai Ye, Weijie You, Tian Zheng, Dominic Brown, Pengbo Wang, Markus Hoehnerbach, Richard Cai, Julien Demouth, John D. Owens, Xia Hu, Song Han, Timmy Liu, Huizi Mao]
venue: MLSys
year: 2026
tags: [llm-inference, sparse-attention, long-context, gpu-kernel, kv-cache]
source_pdf: "[[c6ee784cbe46d854843e4c883a3321ef.pdf]]"
source_md: "[[c6ee784cbe46d854843e4c883a3321ef]]"
review_status: needs-review
evidence_level: full-text
last_reviewed: 2026-10-29
---

# BLASST：用在线 Softmax 统计跳过低贡献注意力块

> **原题**：BLASST: Dynamic BLocked Attention Sparsity via Softmax Thresholding

> **一句话总结**：先计算 query 与 key 的块分数，再利用运行中的最大值判断是否跳过该块的 softmax、注意力与 value 乘法，以及 decode 中相应的 value 读取。[^paper]

**阅读状态**：已解析并阅读会议正式版正文和附录，回看架构、性能与误差界；附录的重归一化误差推导待复核。旧稿摘要的 prefill 数字与正式版不同，本文以正式版为准。[^paper]

## 问题与动机

长上下文注意力即便使用 FlashAttention 分块，仍需处理大量 query–key 交互和 value 数据。既有稀疏方法可能需要预先选择块、额外模型或只覆盖某一推理阶段。BLASST 尝试复用分块在线 softmax 本来就会产生的统计量，在计算过程中判断低贡献块（§1–3）。[^paper]

这是**近似注意力**：跳过的块不参与结果，并不保证与完整注意力逐元素相同。训练感知稀疏是论文的可选扩展，基础推理方案不要求重新训练。[^paper]

## 关键观察 / 隐含假设

- **判定发生在 QK 之后**：必须先读取 key 并计算当前块的 query–key 分数与局部最大值，不能说该方法跳过了所有 QK 计算。
- **小分数是近似贡献**：当前块最大分数相对运行最大值足够小，表示块中 softmax 权重被压低；大量小块累积仍可能影响输出。
- **阶段瓶颈不同**：作者的 prefill 内核依旧预取 value，主要节省计算；decode 内核根据判定跳过部分 value 读取。[^paper]
- **阈值依赖数据和上下文**：为目标稀疏度做标定可改善所测长度的稳定性，但同一比例不保证所有任务和输入都保留质量。

## 核心方法

### 基于运行最大值的跳块

按块计算 $QK^\top$，维护运行最大分数 $m$；若当前块最大分数 $\tilde m$ 满足 $\tilde m-m<\ln\lambda$，则跳过该块后续工作。算法依据已经计算出的真实分数做判定，不需要额外的代理选择器，但仍有判定、归约和同步开销（算法 1、§4）。[^paper]

Prefill 内核保留 value 的预取，跳过该块指数、归约及注意力–value 矩阵乘；decode 内核先批量计算多个 key 分数，再有条件加载通过判定的 value，减少等待造成的流水线空洞（图 3–4）。[^paper]

### 阈值标定与可选训练

作者观察到相同阈值在不同上下文长度产生的稀疏度不同，拟合 $\lambda=\alpha e^{\beta s}/L$，以模型、目标稀疏度 $s$ 和上下文长度 $L$ 设定阈值。标定使用约 1,000 条不同长度的 RULER 样本；从目标稀疏度预测质量和速度仍是经验关系（§3.2、§5.1）。[^paper]

额外实验在微调时启用同样的稀疏注意力，使模型适应被跳过的块；这改变了模型训练，不应与无需训练的基础方案混淆。[^paper]

## 实验与结果

作者主要评估 Llama-3.1-8B、Qwen3-8B，补充更大模型和 DeepSeek-R1 MLA；内核运行在 B200、H200。表 5 的加速相对密集注意力内核，图 5 则是服务框架中的性能，二者不能互换。[^paper]

| 实验 | 结果与边界 |
|---|---|
| Prefill 内核 | B200、batch=1、64K：实测稀疏度 71.9% 时加速 1.52×；H200、71.0% 时同为 1.52×。正式版摘要是 1.52×，不是旧稿的 1.62×。 |
| Decode 内核 | B200、batch=148、32K：73.2% 稀疏度时 1.48×；H200、batch=128、16K：70.5% 时 1.40×。硬件、长度和 batch 不同，不能直接将两行当作同配置对比。 |
| 服务端结果 | 图 5 使用 Qwen3-30B-A3B、TensorRT-LLM、并发 64，平均输入约 10K、输出约 6 token；文中报告 TTFT 和 TPOT 约 1.1×，不是 1.52× 内核加速。 |
| 任务质量 | 表 2 中 Llama-3.1-8B、prefill 目标稀疏度 75% 的 RULER-32K 为 91.67，密集基线为 92.33；这是特定评测下的分数，不能推出所有输出无损。 |
| 阈值标定 | 表 6 的 50% 目标下，固定阈值从 4K 的 23.09% 增至 64K 的 74.63%；标定后为 54.20% 和 48.75%。正文称平均误差 1.2%，表中五个绝对偏差的平均约 2.20 个百分点，聚合口径需核对。 |
| 组合与消融 | 表 7 将 XAttention prefill 与 BLASST decode 组合；在所测 Qwen 8B 上 RULER-16K 从密集 93.22 到 92.89，不能把组合质量理解成单个方法的收益。 |

推理基准包含不同采样协议：长上下文任务使用贪心解码，推理任务采用多次采样和投票。比较质量时需要保留这些条件（§5.1）。[^paper]

## 论断—证据表

| 论断 | 证据 | 边界 | 置信度 |
|---|---|---|---|
| 已计算的分数可用于决定是否跳过后续块运算 | 算法 1、图 3–4 | QK 与 key 读取仍需执行 | 高 |
| 特定内核和服务配置能获得加速 | 表 5、图 5 | 内核与服务指标、设备和负载不同 | 中高 |
| 附录给出的误差界直接覆盖重归一化后的真实输出 | 附录 B | 被保留块的归一化变化是一阶项，需进一步核对 | 低 |

## 批判性分析

### 论证链条

论文从块级 softmax 最大值出发设计条件跳过，再针对 prefill 的计算和 decode 的带宽分别安排流水线，最后用内核与服务实验展示收益。表 2、图 5 给出质量与效率的联合观察，但不同模型和硬件下的最佳点不能直接拼接。[^paper]

### 假设压力测试

附录 B 先把误差写成被跳过 value 的贡献，随后称保留块重归一化仅带来 $O(\delta^2)$ 修正。设完整注意力对跳过块的质量为 $\delta$、两部分加权均值分别为 $\mu_S,\mu_R$，实际重归一化输出为 $\mu_R$，故误差为 $\delta(\mu_S-\mu_R)$；保留部分的变化一般也是 $O(\delta)$。例如两部分 value 分别为 1 和 -1，误差可达 $2\delta$。因此原文以 $\delta V_{max}$ 表示重归一化输出误差的界需要额外条件或修正；该问题不直接否定实测速度和质量结果。[^paper]

### 实验可信度

评测同时覆盖内核、不同稀疏率的质量和一个服务实验。质量保持是有限任务和输入下的经验结果；依赖文本长度标定与注意力分布，新的任务、模型或组合稀疏策略仍需单独测量。正文对标定平均误差的描述也应结合实际聚合方式核验。[^paper]

## 局限与后续工作

- 复核重归一化后的误差界、value 范数条件和多块累积；明确理论界与任务质量之间的关系。
- 对生产长度、并发和质量要求重新标定，分别测量 QK、value 读取、判定及请求级延迟。
- 比较训练感知方案与免训练方案时计入训练成本和模型质量变化。

## 相关

- [[HiSparse-arXiv26]]：通过稀疏选择管理 KV 驻留；BLASST 在已经计算 QK 后判断是否继续处理 value，二者节省的路径不同。
- [[FlexiCache-MLSys26]]：关注按头选择和主机/GPU 间 KV 管理；BLASST 的核心是注意力内核内动态跳块。

## 参考文献

[^paper]: Jiayi Yuan 等. 《BLASST: Dynamic BLocked Attention Sparsity via Softmax Thresholding》. MLSys，2026。[原文](https://proceedings.mlsys.org/paper_files/paper/2026/hash/c6ee784cbe46d854843e4c883a3321ef-Abstract-Conference.html)。
