---
type: paper
name: CacheSlide
full_title: "CacheSlide: Unlocking Cross Position-Aware KV Cache Reuse for Accelerating LLM Serving"
authors: [Yang Liu, Yunfei Gu, Liqiang Zhang, Chentao Wu, Guangtao Xue, et al.]
venue: FAST
year: 2026
tags: [llm-serving, kv-cache, agent, position-encoding, cache-reuse]
source_pdf: "[[fast2026-liu-yang.pdf]]"
source_md: "[[fast2026-liu-yang]]"
review_status: needs-review
evidence_level: full-text
last_reviewed: 2026-10-28
---

# CacheSlide：跨位置复用智能体提示中的 KV cache（FAST 2026）

> **原题**：CacheSlide: Unlocking Cross Position-Aware KV Cache Reuse for Accelerating LLM Serving

> **一句话总结**：当智能体提示中间的动态内容改变长度，后续片段不能简单按原绝对位置复用；CacheSlide 联合位置编码、部分重算和分层缓存管理，在所测任务上相对 ContextCache 将 TTFT 降低 2.4–3.3 倍（§5.2、图 10）。[^paper]

**阅读状态**：已阅读会议正式版 MinerU 全文、核对系统图和实验图，并回到 PDF 检查存疑的算法描述；由于算法阈值方向尚未核实，暂保留待复核状态。[^paper]

## 问题与动机

多轮智能体提示包含会更新的片段与重复使用的片段。若中间片段改变长度，后续重复片段的绝对位置随之变化；仅复用不变前缀不能覆盖这些内容。简单地重排提示可能改变信息顺序和依赖（§2–3）。[^paper]

已有跨位置缓存方法可复用片段，但缓存时与当前上下文不同，位置编码和跨片段注意力也可能发生变化。CacheSlide 研究可复用片段相对顺序保持稳定的场景，并把重用质量与存储读写开销同时纳入设计（§3.1、§3.3）。[^paper]

## 关键观察 / 隐含假设

- **绝对位移可能导致缓存与重算结果偏离**：论文图 4 报告在所测编码和位置变化下 KV 相似度下降；不同位置编码的敏感性不同，不能把该曲线推广到任意模型（§3.2）。[^paper]
- **相对片段顺序可保持**：方法面向重复片段相对顺序稳定、动态片段位置变化的工作负载。若片段频繁重排或内容大幅变化，需要重新测量可复用性（§3.3）。[^paper]
- **加载和更新可以并行**：当缓存页不足、部分数据溢写 SSD 时，读写与计算的排布会影响等待；该收益依赖具体缓存压力和设备性能（§4.4）。[^paper]

## 核心方法

### 位置与上下文修正

Chunked Contextual Position Encoding（CCPE）区分需要重算与可复用的片段，并基于 CoPE 及针对任务的预训练分配位置编码范围，以减小重复片段在位置变化后的偏差。这里需要特定编码与训练安排，不能假定任意现成 RoPE 模型直接获得同样收益（§4.2）。[^paper]

Weighted Correction Attention 在第一层重算并比较缓存状态，选择偏差较大的 token；后续层对所选 token 重算，再融合新旧 KV，并定期调整所选集合。所选比例和阈值影响准确率与计算开销；论文所测最优附近为 0.26 和 0.12，并非对所有场景固定成立（§4.3、§5.5）。[^paper]

### 缓存加载与溢写

SLIDE 将同层 KV 读取与选中 token 的重算写入解耦；若读取尚未结束，先将新值写入新页面，之后再复用槽位。发生存储压力时优先淘汰干净页，再根据选中 token 数量处理脏页，以减少碎片化写回（§4.4、图 9）。[^paper]

## 设计取舍

- **复用速度与语义质量**：复用降低重算量，但位置编码和上下文都变动，部分重算仍不能天然保证与完整重算逐位一致。
- **跨任务适用性与准备成本**：CCPE 用到任务相关编码训练；实际部署应计入训练、适配和缓存维护成本。
- **并发与额外空间**：加载未结束时分配新页面可避免等待，但会引入临时空间及页面管理复杂度。

## 实验与结果

主要实验使用单张 A100 80 GB，70B 模型使用两张 A100；系统基于 vLLM 0.8.5，服务器配置 500 GB DRAM 和 2 TB NVMe SSD。模型为 Mistral-7B、MPT-30B 和 Llama-3 70B（§5.1）。[^paper]

| 结果 | 条件和边界 |
|---|---|
| TTFT 降低 2.4–3.3 倍 | 与 ContextCache 对比；图 10 覆盖不同模型和数据集，准确率按论文指标比较（§5.2）。 |
| 层内并行等待降低 26.7%–51.5% | SLIDE 加载与重算解耦，batch 从 2 到 6 的所测配置（§5.4、图 12）。 |
| SSD 写放大降低 3.11–3.62 倍 | 图 12 的所测工作负载和模型；仅当缓存溢写并出现写回时才有相应优化空间。 |
| 正确完成任务的 QPS 峰值靠近 top-k 0.26、阈值 0.12 | Mistral-7B / Reflexion 与 Llama-3 70B / SWE-Agent 的参数扫描；不等于预设参数在所有模型下最优（§5.5、图 14）。 |

论文摘要同时报告延迟降低 3.11–4.3 倍、吞吐提高 3.5–5.8 倍；这些范围的具体工作负载和基线仍需逐图核对，不能直接与图 10 的 TTFT 数字混用。[^paper]

## 论断—证据表

| 论断 | 证据 | 评测边界 | 置信度 |
|---|---|---|---|
| 相对顺序稳定的提示存在跨位置复用空间 | §2–3、图 2–4 | 论文所选 agent 提示及编码 | 中 |
| 位置编码与部分重算能改善效率—质量权衡 | §4.2–4.3、§5.2 | 需要任务相关训练；不同数据集指标不完全相同 | 中 |
| 读写解耦和脏页策略改善有缓存压力时的执行 | §4.4、§5.4、图 12 | 所测批次和 SSD 溢写配置 | 中高 |

## 批判性分析

### 论证链条

论文把位置变化造成的状态偏移、部分重算和缓存层读写串在一起，并对 SLIDE 做了分组件分析。TTFT 改善与输出质量要联合看；仅引用速度倍数会忽略近似复用的代价。[^paper]

### 假设压力测试

任务类型、提示模板或片段相对顺序改变时，训练得到的编码模式可能失效。较低缓存压力时 SSD 淘汰优化的收益也可能缩小；这些是根据设计作出的推断，需要重新评测。[^paper]

### 实验可信度

论文提供质量、TTFT、并发及参数扫描，但摘要的最大倍数不能代替每组模型与基线的详细结果。CacheSlide 使用为 CoPE 继续预训练的 LoRA 适配器，基线则保留原生 RoPE 或 ALiBi；因此质量和性能比较也包含位置编码及适配器差异，不是纯粹的缓存管理控制实验（§5.1）。[^paper]

论文将 CKSim 定义为余弦相似度，但 §4.3 和算法 2 又写在 CKSim 低于阈值时认为已充分收敛并移出重算集合；这与通常较高相似度表示更接近的含义存在张力。PDF 原文也有同样表述。作者仓库的论文一致性审计确认这一矛盾：新版实验性实现默认保留论文写出的比较方向，另将 `1 - cosine < threshold` 作为显式变体；该仓库说明新版尚未复现论文的质量、GPU 性能和完整 SSD 路径，因此不能据此推定会议实验实际使用了哪一种条件。[^paper][^code-audit]

### 系统性缺陷

新编码、任务训练、缓存页状态和异步读写增加实现复杂度；模型兼容范围、额外训练费用及运行时隔离需要结合具体部署核对。[^paper]

## 局限与后续工作

- 在提示片段重排、不同更新长度和跨任务迁移下，测量答案质量、KV 命中率、重算比例及 TTFT。
- 改变 SSD 带宽和缓存容量，分别测量读取等待、溢写字节及重算开销，验证 SLIDE 的收益来源。

## 相关

- 同会议：[[FAST-2026]]。
- 相关系统：[[Bidaw-FAST26]]、[[SolidAttention-FAST26]]。

## 参考文献

[^paper]: Yang Liu 等. 《CacheSlide: Unlocking Cross Position-Aware KV Cache Reuse for Accelerating LLM Serving》. FAST，2026。[原文](https://www.usenix.org/conference/fast26/presentation/liu-yang)。
[^code-audit]: CacheSlide 项目. 《Paper-conformance audit》. GitHub，2026-09-15。[原文](https://github.com/SJTU-Storage-Lab/CacheSlide/blob/main/docs/paper_conformance.md)。
