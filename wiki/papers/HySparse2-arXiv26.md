---
type: paper
name: HySparse2
full_title: "HySparse2: Hybrid Sparse Attention with Two-Level KV Sharing"
authors: [Jianyu Wei, Yizhao Gao, Qihao Zhang, Shimao Chen, Zhengju Tang, et al.]
venue: arXiv
year: 2026
tags: [llm-inference, sparse-attention, kv-cache, cross-layer-sharing, long-context, agentic-inference, area/kv-cache-systems]
source_pdf: "[[arxiv26-wei-hysparse2.pdf]]"
source_md: "[[arxiv26-wei-hysparse2]]"
review_status: complete
evidence_level: full-text
last_reviewed: 2026-10-27
---

# HySparse2：两级 KV 共享与提前结束预填充（arXiv 2026）

> **原题**：HySparse2: Hybrid Sparse Attention with Two-Level KV Sharing

> **一句话总结**：HySparse2 把模型分为 self-decoder 与 cross-decoder，前者的隐藏状态经各层独立投影构造后者的 KV，再让稀疏层复用完整注意力层的 KV；在 80B-A3B 配置下，论文估算 1M token 时预填充 FLOPs 为 HySparse 的约 1/2.92，但两代比较同时改变了 KV 头数和维度（§3、§4.2、图 4）。[^paper]

**版本与证据**：本页依据小米 LLM-Core 的 [arXiv:2609.26368v1](https://arxiv.org/abs/2609.26368v1) 全文，经 MinerU 解析并回看图 2–5 和表 1–5。该文是 [[HySparse-arXiv26]] 的后续研究；正文中的比较来自本文重新训练的对照模型，不能直接与前一篇的另一组实验表格拼接。[^paper][^hysparse]

## 问题与动机

多轮智能体常输出较短的工具调用，却接收较长的搜索结果、日志或文档。新观察进入上下文后需要预填充；随着历史变长，注意力计算和 KV cache 也随之增加。[[HySparse-arXiv26]] 在完整注意力层后安排稀疏层，后者复用完整层的 KV 与重要块索引，但预填充仍要逐层计算模型，独立的滑窗分支也需要由各稀疏层自己的隐藏状态构建局部 KV（§1–2）。[^paper]

HySparse2 同时处理三件事：用跨解码器桥接缩短**历史 token 的缓存构建路径**，将块级选择改成 token 级选择以改善细粒度检索，并去掉稀疏层的独立滑窗 KV，让跨层缓存能够提前构建。方法改变了模型架构和训练，而非给已有 HySparse 模型直接打开一个服务开关（§3、§4.1）。[^paper]

## 关键观察 / 隐含假设

- **模型后半段的 KV 可以从前半段隐藏状态投影得到**：cross-decoder 完整注意力层保留各自的 K/V 投影，但投影输入来自对应的 self-decoder 完整层。预填充历史 KV 因而无需运行 cross-decoder 的所有注意力与前馈层。该设计假设前半段隐藏状态提供了足够的信息；表 5 在 290B-A8B 上显示多数指标接近，但 DROP 从 71.37 降到 68.17（§3.2、§4.5）。[^paper]
- **固定预算下，块选择会把邻居一起纳入**：表 3 在相同骨干、1,024 个全局 token 和独立 128-token 滑窗的条件下比较 64-token 块与逐 token 选择；RULER-v2 从 49.56 升到 56.13，但 NoLiMa 从 40.27 降到 38.43。收益依赖任务和选择粒度，不能概括成 token 级总是更准（§4.3）。[^paper]
- **局部信息仍然重要**：第二代去掉的是稀疏层的独立 SWA 分支，而非局部窗口本身。强制选取最近 128 个 token 可以消除该分支的独立 KV 依赖；但表 4 中强制窗口相对门控 SWA 的 GSM8K 低 5.08 分、MRCR-v2 低 4.99 分（§3.3、§4.4）。[^paper]

## 核心方法

**模型分成两半**。论文的 49 层 80B-A3B 示例中，前 25 层是 self-decoder：1 层完整注意力与 24 层滑动窗口注意力；后 24 层是 cross-decoder：4 层完整注意力与 20 层稀疏注意力。两部分都用于生成；预填充中历史 token 的 KV 构建可以在 self-decoder 后退出，但仍要执行桥接 K/V 投影，不能说后半模型及其投影从此无需计算（图 2、§3.5）。[^paper]

**外层：KV Bridging**。对 self-decoder 完整层 `i` 与 cross-decoder 完整层 `j`，后者的 `K_j`、`V_j` 是用 `j` 自己的投影矩阵处理 `i` 的输入隐藏状态；`Q_j` 仍来自 cross-decoder 当前隐藏状态。同一个源隐藏状态可供多个目标完整层使用，但每个目标层的 K/V 投影不同，**不能把它们误认为指向同一份 KV cache**。预填充节点先计算这些投影并准备缓存；论文建议在预填充/解码分离时传输投影后的 KV（式 1、图 2、§3.2、§3.5）。[^paper]

**内层：KV Reuse**。cross-decoder 的完整注意力层计算注意力分数，选出 1,024 个全局 token，并强制纳入最近 128 个 token；随后的稀疏层沿用该组索引和完整层的 KV，使用各自的 query 计算注意力。完整层是它自身 query 的精确评分来源，但这些索引对后续层仍是跨层近似；论文没有逐层报告所选 token 对后续层注意力质量的覆盖率（§3.3–3.4、表 1）。[^paper]

**为什么去掉独立 SWA 分支**。第一代在稀疏层另算局部 SWA，并保存由该层自身隐藏状态生成的局部 KV。若第二代还保留它，历史 token 到达 cross-decoder 后才可构造这份缓存，而且局部状态会跨层级联，无法直接跳过所有 cross-decoder 的历史计算。强制局部窗口改为读取完整层共享的 KV，消除了 cross-decoder 稀疏层的这项独立缓存依赖；self-decoder 自身仍有 SWA 层（§3.3、图 2）。[^paper]

## 与 HySparse 的区别

| 问题 | [[HySparse-arXiv26]] | HySparse2 |
|---|---|---|
| 缓存共享 | 一个完整层与随后稀疏层共享该层 KV 和索引 | 在块内复用之外，再用 self-decoder 隐藏状态为 cross-decoder 完整层生成各自 KV |
| 全局选择 | 以 64 token 为块选择，共 1,024 个全局 token | 逐 token 选择 1,024 个全局 token |
| 稀疏层局部信息 | 独立的 128-token SWA 分支与 KV | 强制选入最近 128 token，读取共享 KV |
| 预填充缓存构建 | 仍需运行各层以构造其所需状态 | 运行 self-decoder 和桥接投影后构造 cross-decoder 缓存 |

第一代 7B 消融中独立 SWA 优于两分支共享 KV；第二代采用不同架构、强制窗口和重新训练，表 4 显示强制窗口在部分任务上仍低于门控 SWA。两篇的结果并不构成“所有模型都必须保留独立分支”或“强制窗口始终等价”的证明。[^hysparse][^paper]

## 设计取舍

- **缩短预填充路径，增加模型结构约束**：cross-decoder 的历史 KV 依赖 self-decoder 隐藏状态及目标层投影。推理时每个生成 token 仍需执行用于生成的后半模型；论文没有给出预填充首个输出 token 的端到端耗时拆分。
- **token 粒度换检索精度**：同样 1,024 个全局 token 可以分散在更多位置，避免整块邻居占预算，但离散索引和非连续读取可能影响实际 GPU 内核效率。论文没有给出同硬件下 token/块稀疏内核的延迟对照。
- **局部缓存换局部表达**：强制窗口复用完整层的 KV，省去稀疏层独立的 K/V 投影和局部缓存，却牺牲了一部分独立局部表示；收益与损失随任务变化（表 4）。[^paper]

## 实验与结果

- **训练和基线**：80B-A3B、49 层模型分别使用 Hybrid SWA、HySparse 和 HySparse2 三种注意力设计；预训练约 500B token、32K 上下文，轻量后训练另用约 100B token 并扩展到 256K。各模型在每阶段共享数据和训练计划，但 HySparse2 使用 MQA，另两者使用 GQA，头维度也不同（§4.1、表 1）。[^paper]
- **检索质量**：预训练后 RULER 分别为 HySparse2 90.77、HySparse 84.89、Hybrid SWA 88.71；NoLiMa 分别为 49.76、40.27、30.13。通用任务并非全面上升，例如 DROP 为 58.99、63.78、60.90（表 2）。[^paper]
- **后训练后的长上下文**：在 256K RULER-v2 上，HySparse2 为 58.45，HySparse 为 32.61，Hybrid SWA 为 35.74；跨已报告长度等权平均后，HySparse2 相对 HySparse 的 MRCR-v2 和 RULER-v2 分别高 11.30 和 19.81 分。AgentPPL 的内部数据有 1,000 条轨迹，LongPPL 使用 349 个样例；两者是似然指标，不等同于在线智能体任务成功率（§4.1–4.2、图 3）。[^paper]
- **计算量与缓存量**：论文对 80B-A3B 模型的 FP8 KV cache 做 1K–1M token 成本分析。在 1M 处，HySparse2 的预填充 FLOPs 分别为 HySparse 和 Hybrid SWA 的 1/2.92、1/5.02；KV cache 分别为 2.69、6.72、12.09 GB。图 4 报告的是 FLOPs 和缓存容量，不能把这些倍数直接当成实测时延或服务吞吐（§4.2、图 4）。[^paper]
- **局部窗口消融**：相同骨干的预训练结果中，强制窗口相对门控 SWA 的 RULER 为 89.84 对 88.19，但 GSM8K 为 59.44 对 64.52、MRCR-v2 为 22.67 对 27.66。比较都在 32K 训练上下文内，且门控分支改变了参数与缓存依赖（表 4）。[^paper]
- **桥接消融**：290B-A8B、约 1.8T 训练 token 的实验里，带/不带桥接的 RULER 为 96.01/96.32，DROP 为 68.17/71.37；图 5 的 80B 对照中，桥接与 KV Mirror 的 RULER 分别为 87.65 和 81.28。后者的表示质量解释是作者假设，不是被单独验证的因果结论（表 5、图 5、§4.5）。[^paper]

### 1M 缓存缩减来自哪些变化

表 1 的 HySparse 与 HySparse2 都有 5 层完整注意力，但 KV 头配置不同。以下是**本站按表 1 的估算**：只计算完整层每 token 的 K/V 元素，忽略窗口缓存、索引、scale 与内存对齐；FP8 下每个元素按 1 字节计算。[^paper]

| 配置 | KV 头数 | 单头 K/V 维度 | 每完整层、每 token 的 KV 元素 |
|---|---:|---:|---:|
| 本文的 HySparse 对照 | 4 | 192 / 128 | `4×(192+128) = 1,280` |
| HySparse2 | 1 | 256 / 256 | `1×(256+256) = 512` |

两者比值为 `1,280/512 = 2.5`；图 4 报告的 1M 缓存容量比 `6.72/2.69` 也约为 2.5。由此推测，这组长上下文缓存差距的主要部分可以由 MQA/GQA 及头维度变化解释，不能把 2.5 倍全部归给 KV Bridging。桥接的主要作用是让后半模型的 KV **可由前半隐藏状态提前构建**；各 cross-decoder 完整层仍有独立投影和独立缓存。要精确拆分贡献，需要固定头配置的缓存与预填充消融。[^paper]

## 论断—证据表

| 论断 | 论文证据 | 评测边界 | 置信度 |
|---|---|---|---|
| 历史 KV 构建可绕过 cross-decoder 层计算 | 式 1、图 2：目标层 KV 由 self-decoder 隐藏状态经独立投影产生 | 需要新架构和预训练；投影及生成阶段仍有开销 | 中 |
| token 级选择改善部分长上下文检索 | 表 3：RULER-v2 56.13 对 49.56，MRCR-v2 21.08 对 12.94 | 相同预算与独立窗口，测试不超过 32K；NoLiMa 下降 | 中 |
| 强制窗口让提前退出可行 | §3.3、表 4：移除 cross-decoder 独立 SWA KV，RULER 89.84 对 88.19 | GSM8K、MRCR-v2 低于门控分支，非无损等价 | 中 |
| 1M 时预填充和缓存成本更低 | 图 4：相对 HySparse 为 2.92 倍 FLOPs 缩减、2.69 GB 对 6.72 GB | 成本分析而非服务时延；MQA/GQA 和头维度不同 | 中 |
| 长上下文智能体服务效果已得到全面证明 | 图 3 有检索与轨迹 PPL | 未报告端到端工具调用成功率、生产请求吞吐或 SLO | 无直接证据 |

## 批判性分析

### 论证链条

桥接解释了为什么构造历史 KV 时可以绕过 cross-decoder；强制局部窗口移除了该路径的独立 SWA KV 障碍。表 3–5 对选择粒度、窗口和桥接分别提供了模型质量对照，但**图 4 的总体成本差异同时包含层结构、KV 头数、维度与局部分支变化**。2.92 倍是整套设计相对特定基线的 FLOPs 分析，不能直接当成某一组件的独立贡献。

### 假设压力测试

完整注意力层给自身 query 算出 token 分数，再把索引交给后续稀疏层使用。若后续 query 的信息需求发生漂移，精确评分并不保证这些索引对所有后续层仍是最佳选择。可离线测量每个层距上被选 token 覆盖的注意力质量，并按工具调用、长代码和远距离引用分组；这是一项待验证的研究问题。

### 实验可信度

检索测试最长 256K，而 1M 的图 4 是 FLOPs 和缓存容量分析；不能据此推断 1M 的检索准确率。AgentPPL 与 LongPPL 使用轨迹文本及部分关键 token 的困惑度，并未测工具环境中的真实成功率。桥接消融使用 290B-A8B，粒度与窗口消融则使用另一组配置；结果应各自保留规模和训练阶段。

### 系统性缺陷

论文没有报告真实 GPU 的预填充延迟、top-k 选择和非连续访存开销、缓存传输带宽、首 token 延迟或并发 SLO。预填充/解码分离及 DFlash 异步草稿在 §3.5 是部署建议或未来工作，不能当作已经完成的端到端实验。MQA 的缓存收益与 token 粒度索引的访存代价可能同时改变瓶颈。

## 局限与后续工作

- 固定 KV 头数、K/V 维度、精度和完整层数量，单独比较是否桥接、是否强制窗口；分别记录缓存字节、投影计算、实际 prefill 时间和首 token 延迟。
- 在相同 GPU、上下文和 token 预算下比较块级与 token 级内核，计入索引生成、访存和缓存传输；再随并发变化测量吞吐与尾延迟。
- 扩展到超过 256K 的实际检索与多轮工具任务，按层距测量继承索引的覆盖率，并报告真实任务完成率与失败模式。

## 相关

- **专题与前作**：[[KV-Cache-Systems]]、[[HySparse-arXiv26]]。
- **系统对照**：[[ECHO-OSDI26]] 处理已有稀疏模型在 CPU/GPU 间的 KV 卸载与召回；HySparse2 改变模型结构与预填充缓存构建，二者的实验条件和收益不能直接相乘。
- **原始材料**：[[arxiv26-wei-hysparse2.pdf]]、[[arxiv26-wei-hysparse2]]。

## 参考文献

[^paper]: Jianyu Wei 等. 《HySparse2: Hybrid Sparse Attention with Two-Level KV Sharing》. arXiv，2026-09-22。[原文](https://arxiv.org/abs/2609.26368v1)。
[^hysparse]: Yizhao Gao 等. 《HySparse: A Hybrid Sparse Attention Architecture with Oracle Token Selection and KV Cache Sharing》. arXiv 计算机科学预印本，2026-02-03。[原文](https://arxiv.org/abs/2602.03560v1)。
