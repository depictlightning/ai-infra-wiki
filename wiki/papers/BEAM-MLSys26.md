---
type: paper
name: BEAM
full_title: "BEAM: Joint Resource–Power Optimization for Energy-Efficient LLM Inference under SLO Constraints"
authors: [Hyunjae Lee, Sangjin Choi, Seungjae Lim, Youngjin Kwon]
venue: MLSys
year: 2026
tags: [llm-serving, energy-efficiency, scheduling, dvfs, batching, area/llm-serving]
source_pdf: "[[43ec517d68b6edd3015b3edc9a11367b.pdf]]"
source_md: "[[43ec517d68b6edd3015b3edc9a11367b]]"
review_status: complete
evidence_level: full-text
last_reviewed: 2026-10-28
---

# BEAM：联合批处理与 GPU 频率的推理节能调度（MLSys 2026）

> **原题**：BEAM: Joint Resource–Power Optimization for Energy-Efficient LLM Inference under SLO Constraints

> **一句话总结**：BEAM 在流水线并行的 LLM 服务中，根据请求状态联合选择 prefill 分块、decode 微批数量和 GPU 频率，尝试利用时延预算降低 GPU 能耗。[^paper]

**阅读状态**：已通读与会议正式版一致的 MinerU 正文与正文内 artifact 附录，回看公式、系统图、实验图和消融；会议页面未列出独立补充材料。[^paper]

## 问题与动机

LLM 推理服务需要同时处理首 token 延迟（TTFT）和 token 间延迟（TBT）。如果当前请求距离 SLO 截止时间仍有余量，服务可以用一部分余量换取更低的能耗：例如形成更高效的批次，或降低 GPU 频率。[^paper]

这两个选择会相互影响。批次大小改变算子利用率和内存访问特征，因而改变最低能耗对应的频率；等待更大批次和降低频率也会共同消耗延迟预算。分别固定其中一个维度，未必找到联合空间中的更好配置（§2）。[^paper]

## 关键观察 / 隐含假设

- **存在可用余量**：在接近饱和或请求已超时的情况下，可用于节能的余量可能消失（§5.4）。[^paper]
- **频率与批次耦合**：最低能耗点随 token 批次变化，降低频率不必然使总能耗最低，因为执行时间也会改变（§2.2）。[^paper]
- **预测可迁移到当前状态**：控制器依赖按模型、设备和并行配置离线采样的延迟—能耗表；实际工作负载变化可能引入预测误差（§3.3、§5.6）。[^paper]
- **SLO 是约束目标，不是硬保证**：算法筛掉模型预测超时的配置，但真实实验中仍存在违反 SLO 的请求。[^paper]

## 核心方法

### 事件触发和离线画像

BEAM 在 vLLM 调度器中观察新 prefill 到达及请求完成等事件，根据当前状态调用 prefill 或 decode 调度器。离线画像以单次前向处理的 token 数和频率为输入，记录每个模型分片的延迟与能耗，再用解析模型估计整个流水线。论文报告一次模型和 GPU 并行配置的画像约需 30 分钟（§3.1–3.3）。[^paper]

这样的二维表避免枚举所有完整工作负载组合，但也将运行时未直接建模的因素压缩到近似结果中；预测误差、排队变化和 GPU 状态会影响约束判断。[^paper]

### S1：prefill 分块与频率

新 prefill 到达时，S1 枚举候选 chunk 大小与 GPU 频率，预测该请求的 TTFT、已有 decode 的 TBT 和对应能耗；先排除预测违反任一 SLO 的配置，再从剩余候选中选择能耗最低者（§3.4、算法 1）。[^paper]

较大 chunk 通常提高计算效率，但可能扩大流水线气泡和对 decode 的阻塞；较小 chunk 更容易保护交互延迟，却可能使计算效率变差。原文图 5 示例展示了一个较大 chunk、较高频率的可行配置比另一个较小 chunk 配置更省能，说明不能只按最低频率选点。[^paper]

### S2：decode 微批与频率

仅有 decode 工作时，S2 按活跃请求数枚举微批数量与频率。假设活跃请求数为 D、微批数为 m，则每个微批约有 ⌈D/m⌉ 个 token；系统查表估算延迟和能耗，筛去预测 TBT 超标的配置。更多微批可改善流水线利用率，但更小的单次计算也可能降低资源效率（§3.5、算法 2）。[^paper]

正文公式在 MinerU 中缺失，已回看正式 PDF 第 6–7 页：模型用单分片延迟乘流水线阶段数估计 TBT，并用查表结果估计能耗；这些是调度近似，不是逐请求 SLO 的形式化证明。[^paper]

### 异步执行

控制器调整 vLLM 的调度参数，并通过 NVML 设置 GPU 时钟；频率调整交由后台线程，以免其调用直接阻塞调度循环。论文报告调度搜索低于 1 ms，而频率设置约需 7–10 ms，因此“决策快”不等于硬件频率瞬时生效（§4、§5.8）。[^paper]

## 设计取舍

- **能耗与延迟余量**：把 P90 延迟推近 SLO 可以降低能耗，但会缩小应对突发和预测误差的安全余地。
- **响应速度与稳定性**：事件触发比固定窗口更快感知变化；频繁事件和硬件调频延迟仍可能导致动作滞后。
- **画像成本与预测精度**：二维表容易查询，但新模型、硬件或并行配置需要重新画像，单次查表不等于对完整请求队列的准确建模。

## 实验与结果

主要端到端实验使用 8×A100-80GB、Llama 3.3-70B、TP=2 和 PP=4；另在 4×A100 上评测 Qwen2.5-32B。工作负载是根据 ServeGen 的生产轨迹缩放并重新生成的一小时片段，不是直接部署在生产环境的实验（§5.1–5.2）。GPU 能耗由 NVML 计数器记录，不包含整个数据中心能源（§4）。[^paper]

| 结果 | 实验条件与比较范围 |
|---|---|
| GPU 能耗 | Llama 场景中 BEAM 为原始 vLLM 的 **49%**，相对于 Window-DVFS 约下降 30%；Qwen 场景为原始 vLLM 的 **53%**（图 6）。 |
| SLO 达成 | Llama 场景 BEAM 的 TTFT、TBT 达成率为 **94.9% / 94.5%**；Window-DVFS 为 95.0% / 95.4%，因此不能描述为零违约。 |
| 延迟代价 | Llama 场景 P90 TTFT 从 vLLM 的 0.65 秒变为 0.80 秒，P90 TBT 从 0.15 秒变为 0.18 秒（图 6）。 |
| 组件消融 | 合成轨迹下，DVFS Only、S1 Only 和完整 BEAM 分别消耗 vLLM 的 73%、71% 和 52%；该 52% 属于消融场景，不能与另一轨迹的 49% 混为同一实验（图 10）。 |
| 模型误差 | 表 1 中 TTFT MAPE 为 11.0%–27.0%，TBT MAPE 为 9.2%–18.8%；平均相对误差不能替代尾部误差保证。 |

Window-DVFS 是作者实现的单节点、每 5 秒调整频率基线，借鉴 DynamoLLM 的周期性控制，但没有实现其全部集群级模型重分片和扩缩容机制；不能把结果直接写成对完整 DynamoLLM 的比较（§5.1）。[^paper]

## 论断—证据表

| 论断 | 证据 | 评测边界 | 置信度 |
|---|---|---|---|
| 联合批处理和频率调节能降低所测 GPU 能耗 | 图 6、图 10 | A100 与所测模型、轨迹；节能伴随延迟变化 | 中高 |
| 事件触发可更快响应突发 | 图 8 | 合成泊松负载；需要考虑 7–10 ms 调频执行延迟 | 中 |
| 每个请求均满足 SLO | 图 6 | 实际达成率低于 100%，预测也存在误差 | 低 |

## 批判性分析

### 论证链条

论文先证明频率与批次存在耦合，再用轻量画像搜索可行配置，通过端到端比较、突发测试和组件消融支持节能。消融显示主要收益并非只来自联合调度：单独 DVFS 已从 100% 降到 73%，S1 分块选择在该实验中进一步降至 71%，S2 才降至 52%。[^paper]

### 假设压力测试

SLO 很紧、预测偏差大、硬件调频受限或负载已饱和时，候选空间可能不存在满足约束的点。论文的高负载实验中所有方法均出现 TTFT 违约；算法也未因此自动创造额外服务容量。[^paper]

### 实验可信度

论文给出了基线、硬件配置和达到率，但主要场景采用固定的流水线并行拓扑；模型、流量、功率边界以及监控口径变化可能改变结果。作者提到可以适配 prefill/decode 分离系统，正文未据此给出分离部署的端到端评测，不能当作已验证结果。[^paper]

## 局限与后续工作

- 在更严格的尾延迟约束下评估预测保守量、请求违约率和实际调频生效时间。
- 在新 GPU、不同并行拓扑及真实生产流量下比较画像成本、系统级能耗和稳定性。

## 相关

- [[LLM-Serving]]：请求调度、资源与服务目标。
- [[MoEServingTax-MLSys26]]：分析 MoE 推理成本；BEAM 调节服务配置和频率，关注点不同。

## 参考文献

[^paper]: Hyunjae Lee 等. 《BEAM: Joint Resource–Power Optimization for Energy-Efficient LLM Inference under SLO Constraints》. MLSys，2026。[原文](https://proceedings.mlsys.org/paper_files/paper/2026/hash/eb3c42ddfa16d8421fdba13528107cc1-Abstract-Conference.html)。
