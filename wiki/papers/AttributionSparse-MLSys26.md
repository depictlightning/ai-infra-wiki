---
type: paper
name: Attribution-based Sparse Activation
full_title: "Attribution-based Sparse Activation in Large Language Models"
authors: [Jifeng Song, Xiangyu Yin, Boyuan Yang, Kai Huang, Weichen Liu, Wei Gao]
venue: MLSys
year: 2026
tags: [llm-inference, sparse-inference, neuron-sparsity, model-compression]
source_pdf: "[[c9e1074f5b3f9fc8ea15d152add07294.pdf]]"
source_md: "[[c9e1074f5b3f9fc8ea15d152add07294]]"
review_status: needs-review
evidence_level: full-text
last_reviewed: 2026-10-29
---

# 基于归因的 LLM 稀疏激活

> **原题**：Attribution-based Sparse Activation in Large Language Models

> **一句话总结**：根据神经元对输出的归因分数动态选择要执行的部分，并尝试修正层间依赖导致的排序误差，以质量损失换取稀疏计算收益。[^paper]

**阅读状态**：已阅读与会议版一致的 MinerU 正文、独立补充材料，并回看关键公式与实验表；理论证明所需条件和实际端到端开销仍需进一步核验。[^paper][^supplement]

## 问题与动机

LLM 的每次前向不一定需要所有 MLP 神经元和注意力头。只依据激活幅度关闭神经元可能忽略其对最终输出的影响；对输入计算归因分数可更有针对性地决定保留哪些部分，但前一层的关闭操作又可能改变后一层的分数和排序（§3–4）。[^paper]

该方法是**有损**稀疏化：改变了实际计算路径及模型输出，并不承诺逐 token 或输出分布等价。它不同于仅跳过原本数值为零的激活。[^paper]

## 关键观察 / 隐含假设

- **归因是局部近似**：Gradient × Output（GxO）将梯度与神经元输出相乘，近似描述关闭该神经元对模型输出的影响；多个神经元同时关闭时，梯度和排序可能变化。[^paper]
- **层间依赖会改变排序**：第一层的掩码可能改变下一层应保留的神经元，因此作者设计校正项；校正公式的严格保证依赖补充材料中的条件。[^paper][^supplement]
- **产生零值不等于节省时间**：实际收益要求使用稀疏存储和计算，并覆盖掩码预测、格式转换及索引开销。
- **质量约束取决于任务和指标**：BLEU 或 ROUGE 的某个平均分不能保证所有问题的语义、真实性和安全性均不变。补充材料的例子中，较低激活率可出现语义变化或无法理解的输出。[^supplement]

## 核心方法

### 归因与校正

作者对 attention 与 MLP 层安装前向 hook，以生成 token 的对数概率对中间输出求梯度，计算 GxO；再根据梯度范数和激活幅度加入校正项，试图减轻层间归因分数变化对排序的影响（§5–6）。注意力按头聚合，MLP 按神经元选择；各层分别使用阈值，以避免不同层的分数尺度直接比较。[^paper]

### 实用预测与稀疏执行

论文指出每生成一个 token 都执行完整反向传播代价过高，因此提出先在代表性数据上离线计算掩码，再训练轻量 MLP 根据隐藏状态预测运行时掩码。其稀疏实现将关闭的相关权重置零，转为 `torch.sparse` 格式并用稀疏矩阵乘法；动态掩码的预测和转换是否完整计入最终性能，需要结合实现及计时路径核验（§6）。[^paper]

## 实验与结果

作者使用 H100 80 GB、batch=1，测试 Phi-2、Gemma-2B、MobiLlama 和 Llama-3-8B 等模型。激活比例 AR 表示**保留**的比例；AR=30% 不是 30% 稀疏，而是约 70% 关闭。[^paper]

| 实验 | 结果与边界 |
|---|---|
| 质量与稀疏度 | 表 1 的 TruthfulQA / Phi-2：完整模型 BLEU 33.9，校正 GxO 的 AR=30% 为 26.8，绝对下降 7.1 分、相对约下降 21%；不能把该配置笼统描述为低于 5% 的质量损失。 |
| 不同模型 | 同表 Llama-3-8B 在 AR=40% 时 BLEU 为 24.48，完整模型为 26.52；阈值、模型和误差口径都会影响可接受的稀疏度。 |
| 性能 | §7.6 对 Phi-2、AR=30% 报告约 35% 延迟和 43% 显存节省；该配置对应上面 33.9→26.8 的 BLEU 变化，不能直接与其他稀疏度的质量结论拼接。 |
| 计时边界 | 作者将完整请求开始时的权重冷加载计入延迟，但不是每个生成 token 都重新加载模型；文中又称测量一次前向，因此不应直接等同于稳定驻留模型的整段生成延迟。 |
| 分层消融 | 表 2 分别关闭 Phi-2 的 MLP 神经元或 attention 头，显示两类结构对稀疏化敏感度不同；结果不能直接推广到其他模型或任务。 |

上述质量数值为作者报告的 BLEU，而非事实正确率；实验观察与理论界分别评估。[^paper]

## 论断—证据表

| 论断 | 证据 | 边界 | 置信度 |
|---|---|---|---|
| 所测条件下校正归因比若干基线有更好的质量—稀疏度表现 | 表 1–4 | 模型、数据集、指标和激活率限定结果 | 中 |
| 稀疏权重可降低所测冷启动延迟和显存 | §7.6、表 5 | 热驻留生成与预测、转换开销需进一步核验 | 中 |
| 理论界无条件适用于一般 Transformer | 补充材料 A–C | 使用同号变化和梯度不变等假设，证明步骤需复核 | 低 |

## 批判性分析

### 论证链条

论文先展示现有归因方法在多神经元关闭时可能受依赖影响，再构造校正分数并通过质量、消融和冷启动实验比较。实测表明某些配置有收益，但不能单靠实验验证所有理论假设，也不能将不同激活率的最好质量与最好速度拼成同一配置。[^paper][^supplement]

### 假设压力测试

补充材料 A 对引理 5.1 假设下一层所有神经元的归因变化同号；其表 1 只报告约 94%–95% 的神经元分数下降，并未证明所有变化同号。补充材料 B 的推导将 $g(X)$ 替换为 $J_g(X)X$；对于一般非线性函数或带偏置的变换，这个等式并不自动成立，需要额外条件。补充材料 C 又假设掩码前后相关梯度相同。上述限制意味着不能直接将其证明当作无条件保证；不据此断言实现一定错误。[^supplement]

### 实验可信度

实测覆盖多个模型和任务，但单卡单请求、冷启动与生成指标限制了外推。正文同时描述离线预测器及每次前向计算归因；应核对具体实验使用哪条路径、预测器精度和完整系统开销，才能判断部署收益。[^paper]

## 局限与后续工作

- 复核理论假设与公式适用范围，比较真实 Transformer 中掩码前后的梯度和归因排序。
- 复现时分别测量离线标签生成、预测器、动态权重与格式转换、冷启动和热驻留 decode。
- 在相同稀疏度下同时报告延迟、显存与多维任务质量，检查个体样本的语义变化。

## 相关

- [[KAIROX-OSDI26]]：同样利用输入相关稀疏性，但系统重点是活跃神经元组预测与 CPU/GPU 分工。

## 参考文献

[^paper]: Jifeng Song 等. 《Attribution-based Sparse Activation in Large Language Models》. MLSys，2026。[原文](https://proceedings.mlsys.org/paper_files/paper/2026/hash/29591f355702c3f4436991335784b503-Abstract-Conference.html)。
[^supplement]: Jifeng Song 等. 《Attribution-based Sparse Activation in Large Language Models - Appendix》. MLSys，2026。[原文](https://proceedings.mlsys.org/paper_files/paper/2026/file/29591f355702c3f4436991335784b503-Supplemental-Conference.pdf)。
