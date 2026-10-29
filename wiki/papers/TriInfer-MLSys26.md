---
type: paper
name: TriInfer
full_title: "TriInfer: Hybrid Disaggregated Scheduling for Multimodal Large Language Model Serving"
authors: [Xianzhe Dong, Tongxuan Liu, Yuting Zeng, Weizhe Huang, Xiaoyang Zhao, Siyu Wu, Liangyu Liu, Yang Liu, Yu Wu, Hailong Yang, Ke Zhang, Jing Li]
venue: MLSys
year: 2026
tags: [lens/pd-serving, llm-serving, multimodal-inference, disaggregated-serving, scheduling, batching, area/llm-serving]
source_pdf: "[[6974ce5ac660610b44d9b9fed0ff9548.pdf]]"
source_md: "[[6974ce5ac660610b44d9b9fed0ff9548]]"
review_status: complete
evidence_level: full-text
last_reviewed: 2026-10-28
---

# TriInfer：多模态推理的混合 EPD 分离与阶段调度（MLSys 2026）

> **原题**：TriInfer: Hybrid Disaggregated Scheduling for Multimodal Large Language Model Serving

> **一句话总结**：TriInfer 将图像编码、LLM 预填充和解码视为独立阶段，通过历史流量选择阶段组合与实例比例，并在实例内采用阶段级批处理和双 CUDA stream。[^paper]

**阅读状态与版本**：已通读与会议正式 PDF 一致的 MinerU 正文及文内附录，并核对架构、实验、消融和迁移表。会议网页的标题与 PDF 标题不同，摘要写最高 3.7×，PDF 摘要写 2.4×；两者均保留在原出处，此处按 PDF 中具体实验条件解释，不把摘要数值当作统一上限。会议页面未列独立补充材料。[^paper]

## 问题与动机

图文大模型推理包含图像编码（E）、LLM 预填充（P）和逐 token 解码（D）。三个阶段的计算、显存和批处理特征不同：将它们固定组合可能让某一阶段成为瓶颈；把所有阶段拆开又会引入中间状态传输与实例分配成本（§2–3）。[^paper]

已有 LLM 服务的 token 预算可以限制 prefill 对 decode 的干扰，但仅按文本 token 计算并不能完整反映图像编码耗时。TriInfer 试图同时解决阶段放置与实例内部的执行调度。[^paper]

## 关键观察 / 隐含假设

- **阶段最佳批量不同**：论文测得图像编码、prefill、decode 随批量变化的吞吐曲线不同，不能用一个统一批量控制所有阶段（§3.2）。[^paper]
- **流量具有可预测性**：系统利用历史样本回放选择分离方式；负载急变时可能暂时采用过时配置，角色重配和加载模型需要数分钟（§4.4）。[^paper]
- **拆分有数据路径成本**：图像特征和 KV 需跨实例迁移，收益受互连和布局影响（§4.3、§5.5）。[^paper]
- **目标明确为有效吞吐**：论文将请求定义为 TTFT 达标且至少 90% 的 TBT 达标；goodput 是至少 90% 请求达标时的最高请求到达率（§2.2）。这不是无条件总吞吐。[^paper]

## 核心方法

### 阶段级批处理和双流执行

请求被拆成 encode、prefill、decode 与迁移任务。每个实例根据阶段角色和 SLO 得到图像预算、token 预算：优先纳入正在解码的请求，再安排未完成的 prefill、encode 和新请求，避免只按文本 token 估计批次时长（§4.1–4.2、算法 1）。[^paper]

同一实例上的视觉模型与语言模型分别使用 CUDA stream，可让不同请求的图像编码与语言阶段重叠。实际并发收益取决于硬件资源竞争，不能理解为同一个请求在依赖尚未满足时提前 prefill。[^paper]

### 混合阶段分离

系统支持三种候选部署：E+P+D、EP+D 和 ED+P。前者单独分配三个角色；后两者分别把编码与预填充或编码与解码放在一起。不同组合对模型驻留、图像缓存、KV 传输和各阶段排队的影响不同（§3.3、§4.4）。[^paper]

规划器先用历史轨迹估计各阶段工作量，根据画像和显存预算划分实例比例；再对三种候选部署回放历史请求，选择测得 goodput 最好的候选。它是启发式搜索，选择范围不是全部可能部署；工作负载明显变化时重新分析（算法 2）。[^paper]

### 拉取式迁移

目标实例接收请求控制信息，在有资源并调度请求时才拉取源实例的图像或 KV 缓存；传输完成后通知源实例释放资源。这种安排试图避免目标缓存溢出，也带来队列、缓存生命周期与传输协同成本（§4.3）。[^paper]

## 设计取舍

- **分离与传输**：更多独立阶段可减少资源干扰，但需要迁移中间状态；更快网络与更小状态通常有利于分离。
- **历史拟合与动态变化**：回放可按工作负载选择部署，数分钟的重新规划和重载不适合假定瞬时应对所有突发。
- **预算与端到端 SLO**：单批执行时间受控并不自动保证完整请求的排队、迁移和生成均达标。

## 实验与结果

论文使用最多 4 台服务器、每台 8×H20；比较 LLaVA-1.5-7B、LLaVA-NeXT-7B 和 Qwen2-VL-7B，使用 TextCaps 等五类图文任务。历史样本取数据集的十分之一，其余用于测试；请求到达时间采用生产轨迹并按请求率缩放。为控制负载，实验先记录输出长度，后续固定生成长度并忽略 EOS（§5.1）。[^paper]

| 结果 | 实验条件与解释 |
|---|---|
| goodput | PDF §5.2 报告 LLaVA-1.5-7B 相对 vLLM 为 1.3–3.7×，LLaVA-NeXT-7B 为 1.2–2.1×；不同数据集和 SLO 对应不同结果。 |
| 组件消融 | LLaVA-NeXT-7B、TextCaps 中，完整方案 goodput 约 3.9 请求/秒；改为平均分配的 E+P+D 约 2.6，再关闭阶段级调度约 2.2（图 10）。这是依次移除组件的比较，不是各组件独立贡献。 |
| 迁移 | 表 4 中 4E12P16D 在 64 / 320 请求每秒下，EP 迁移 P95 为 2.6 / 4.2 ms，PD 为 8.4 / 14.4 ms；迁移相关 TBT 违约率为 0.4% / 0.9%，不能概括为始终没有影响。 |
| 启发式 | 8 实例实验枚举 35 个方案，比较启发式选择的平均排名；这不构成大规模全局最优保证（§5.4）。 |

比较设置影响解读：vLLM 和 SGLang 使用非分离配置，关闭前缀缓存与 CUDA Graph，vLLM 使用 eager 模式；[[EPDServe-ICML25]] 和 DistServe 的核心分离方式由作者在 TriInfer 内重实现，并非直接运行两者原版系统（§5.1）。[^paper]

## 论断—证据表

| 论断 | 证据 | 评测边界 | 置信度 |
|---|---|---|---|
| 混合分离与阶段调度可提高所测 goodput | 图 9–10 | 结果依赖 SLO、模型、数据集和基线配置 | 中高 |
| 历史轨迹启发式能找到较好的部署 | §5.4、表 3 | 仅在 8 实例可枚举空间中检查排名 | 中 |
| 方法已验证适用于任意多模态管线 | §4.4–5 | 实验聚焦图文模型，视频与音频属于设计扩展 | 低 |

## 批判性分析

### 论证链条

阶段负载差异说明静态部署可能失衡；阶段预算、角色选择和迁移机制构成系统方案。论文分别给出整体结果与消融，但两项被连续关闭，无法由图 10 单独推导每个组件在所有拓扑下的收益。[^paper]

### 假设压力测试

图像尺寸、生成长度或请求模式突变会使历史划分过时；长上下文 KV、网络拥塞和跨节点通信也可能使迁移成为瓶颈。需要将重新规划期间的排队与权重加载纳入端到端成本。[^paper]

### 实验可信度

比较覆盖三个图文模型，但关闭前缀缓存与 CUDA Graph，以及用固定输出长度控制请求，限制了对真实生产部署的直接外推。作者对其他多模态方式的泛化是架构判断，并非对应实验。[^paper]

## 局限与后续工作

- 在开启现代推理引擎常用优化、真实输出长度和跨节点拥塞下重新比较 goodput 与尾延迟。
- 评估负载突变、阶段重配与模型加载成本，并扩展至视频或音频任务。

## 相关

- [[LLM-Serving]]：阶段分离、请求批处理和服务调度。
- [[DualPath-SIGCOMM26]]：针对分离式 LLM 服务的 KV 存储读取；TriInfer 额外组织视觉编码和阶段角色。
- [[BEAM-MLSys26]]：同样按推理阶段调度，但优化对象是 GPU 能耗与 SLO。

## 参考文献

[^paper]: Xianzhe Dong 等. 《TriInfer: Hybrid EPD Disaggregation for Efficient Multimodal Large Language Model Inference》. MLSys，2026。[原文](https://proceedings.mlsys.org/paper_files/paper/2026/hash/f068c65585985c25c17f221390774ec7-Abstract-Conference.html)。
