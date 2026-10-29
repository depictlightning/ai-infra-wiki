---
type: paper
name: SHIP
full_title: "SHIP: SRAM-Based Huge Inference Pipelines for Fast LLM Serving"
authors: [Andrew Bitar, Aravind Vellora Vayalapra, Baorui Zhou, Matt Boyd, Charlie Wang, Sahil Parmar]
venue: MLSys
year: 2026
tags: [llm-serving, inference-hardware, pipeline-parallelism, sram, kv-cache, area/llm-serving]
source_pdf: "[[7647966b7343c29048673252e490f736.pdf]]"
source_md: "[[7647966b7343c29048673252e490f736]]"
review_status: complete
evidence_level: full-text
last_reviewed: 2026-10-29
---

# SHIP：基于 SRAM 的大规模 LLM 推理流水线（MLSys 2026）

> **原题**：SHIP: SRAM-Based Huge Inference Pipelines for Fast LLM Serving

> **一句话总结**：Groq 将权重与活跃 KV 分布在大量 LPU 的片上 SRAM，通过同步互连、编译期调度和动态流水线管理，以较小 batch 提供 LLM 推理服务。[^paper]

**阅读状态**：已通读与会议正式版一致的 MinerU 正文及独立补充材料，并核对流水线、结果图和缓存容量表。补充材料解析见 [[ship-supplement]]。[^paper][^supplement]

## 问题与动机

decode 每生成一个 token 都需读取权重与历史 KV；HBM 容量较大，但低 batch 时可能受带宽限制。更高带宽的片上 SRAM 适合低 batch 访问，单芯片容量却有限，需要跨大量芯片组织权重、KV 和计算（§1–3）。[^paper]

SHIP 是 Groq 对第一代 LPU 系统的生产回顾。它将问题从“用更大 batch 摊薄 HBM 访问”转向“用大量低延迟、容量有限的芯片构造推理流水线”，随之需要解决跨芯片同步、流水线空洞和缓存容量。[^paper]

## 关键观察 / 隐含假设

- **带宽与容量一起考虑**：第一代 LPU 每芯片约 230 MB SRAM；权重和 KV 的容量需求决定必须使用大量芯片（§7.1）。[^paper]
- **静态执行依赖硬件可预测性**：编译器预先安排计算与片间传输；请求变化仍需运行时掩码、分页和调度（§2.1、§7.2）。[^paper]
- **低 batch 下通信暴露**：小张量通信难以依靠批量摊销，互连跳数与同步开销会限制规模（§3.2、§4.2）。[^paper]
- **缓存也有慢层**：活跃权重和 KV 以 SRAM 为主，但前缀缓存还使用节点主机 DRAM；命中 DRAM 仍需经 PCIe 回载（§5.2）。[^paper]

## 核心方法

### 同步互连与大流水线

每节点包含 8 个全连接 LPU；节点通过 QuadFour 拓扑连接邻近节点。模型层按 TP 分布在芯片组内，再将多个组按 PP 排成流水线。编译器根据确定的执行时序安排计算与传输；节点故障时可借助拓扑跳过故障节点，但跳过会降低可用带宽（§4.1）。[^paper]

通信算法按张量大小权衡跳数与带宽：补充材料区分 RedBcast、TiledRedBcast 和 PAARD。更少跳数适合较小消息，更高带宽利用率可能需要更多步骤（§4.2、附录 B）。[^paper][^supplement]

### SRAM 页面和分层前缀缓存

运行时按需分配 KV 页面，并发送压缩的页面掩码；常见页面大小为 128–512 token。小页面减少内部碎片，但会增加掩码和管理开销（§5.1）。[^paper]

系统用 BlockTrie 跟踪固定 token 块前缀；SRAM 命中可以跳过对应 prefill，DRAM 命中则先回载到 SRAM，空间耗尽时采用 LRU。论文说明不同组织的前缀缓存隔离，报告两个实例一周内观察到的 DRAM 命中率约 50–75%，这不是所有模型或用户的通用命中率（§5.2）。[^paper]

### 动态流水线

系统根据流量动态形成 prefill chunk，优先安排 decode，再用剩余容量填充 prefill；针对不同上下文位置的 decode 通过融合 context 与 batch 尺寸填补流水线空隙（§6、图 9）。这种调度可能牺牲部分首 token 延迟，长上下文仍会消耗 SRAM 容量。[^paper]

投机解码还可能减少维持一定计算效率所需的 KV 副本，但需要额外 draft 阶段和验证；论文显示 draft 模型过大或接受率不足时收益可能消失（§5.3、图 8）。[^paper]

## 设计取舍

- **高带宽与芯片数量**：片上 SRAM 降低低 batch 读取等待，但较低容量要求更多芯片和跨芯片通信。
- **静态安排与动态负载**：确定性执行有利于流水线和通信重叠，变量长度、缓存回载和专家路由仍需要动态处理。
- **decode 优先与 TTFT**：优先输出 token 可稳定交互，却可能让新请求等待；容量填充 prefill 只能部分缓解。

## 实验与结果

论文比较 Qwen3-235B-A22B 的 SHIP 配置（TP=16、PP=95、最大并发 380）与 8×B200、SGLang v0.5.8 配置；后者比较不同混合批次 token 预算。**图 10 每个系统分别归一化到自己的峰值**，不能据此直接计算两类芯片的单位成本或能效倍数。[^paper]

| 结果 | 实验条件与边界 |
|---|---|
| 动态流量 | 图 10 比较不同 prefill:decode 比例和上下文长度；SHIP 在部分组合下维持较稳定吞吐，但长上下文、低 prefill 比例仍受 SRAM 容量影响。 |
| 生产轨迹 | 补充材料从两个生产模型 30 天流量中各抽 2,000 个长度样本；出于隐私使用随机 token 模拟，不是完整用户 prompt 回放。[^supplement] |
| 比较设置 | 补充材料说明对比实验关闭前缀缓存和投机解码，并从 TTFT 和输入速率中移除排队时间；这与真实在线请求从到达到响应的时延不同。[^supplement] |
| 缓存容量 | 表 2 的某个 gpt-oss-120B 实例由 72 个 LPU 节点组成；扣除权重后列出约 69.7 GB SRAM 和 72,000 GB 主机 DDR4 可用于相关缓存，具体取决于该部署。 |
| 通信 | 论文报告 LPU 节点间单跳约 300 ns；这是互连属性，不能直接替代整次 collective 的端到端延迟。 |

作者比较不同系统规模和运行时，强调延迟优势；由于设备数量、精度、功率和归一化方式不同，图中的相对曲线不足以单独证明相同预算下的总拥有成本或每瓦吞吐优势。[^paper]

## 论断—证据表

| 论断 | 证据 | 评测边界 | 置信度 |
|---|---|---|---|
| SRAM 与同步互连可支撑大规模低 batch 推理 | §2–4、§6 | 第一代 LPU 与其软件栈 | 中高 |
| 动态流水线可适应所测 P:D 和上下文变化 | 图 9–10 | 长上下文仍受容量约束，数据分别归一化 | 中 |
| 图 10 已证明同成本同功耗下全面优于 B200 | §6、附录 D | 芯片规模不同，未提供同预算对照 | 低 |

## 批判性分析

### 论证链条

论文先用带宽与容量的差异解释硬件设计，再讨论跨芯片互连、KV 页面、前缀缓存和流水线调度，最后用受控流量与生产长度样本展示系统表现。前缀缓存的生产命中率和关闭缓存的比较实验应分开阅读。[^paper][^supplement]

### 假设压力测试

更长上下文、热点流量、频繁故障或更多动态专家执行会增加容量、流水线和网络压力；节点跳过虽可恢复计算，也会损失链路带宽。跨架构比较还需计入完整集群的功率、成本与请求排队。

### 实验可信度

文章有生产系统经验与公开基线，但对比使用很不相同的系统规模；实验关闭了部分线上优化，并排除排队时间。作者讨论 GPU/LPU 混合和 attention/FFN 分离为未来方向，正文没有完整评测，不能当作已验证收益。[^paper][^supplement]

## 局限与后续工作

- 在统一模型、服务目标、总设备与功率预算下测量成本、P95/P99 和缓存命中对端到端吞吐的影响。
- 测量跨芯片故障、长上下文和动态专家负载对流水线恢复与利用率的影响。

## 相关

- [[LLM-Serving]]：推理服务架构、阶段调度与部署。
- [[MetaInferenceConfig-MLSys26]]：研究 GPU 部署配置搜索；SHIP 则围绕 SRAM 硬件与流水线共设计。
- [[PragmaticDisaggregation-MLSys26]]：讨论 P/D 分离与速率匹配；其动态 Planner 图文倍数待复核。

## 参考文献

[^paper]: Andrew Bitar 等. 《SHIP: SRAM-Based Huge Inference Pipelines for Fast LLM Serving》. MLSys，2026。[原文](https://proceedings.mlsys.org/paper_files/paper/2026/hash/9c20f16b05f5e5e70fa07e2a4364b80e-Abstract-Conference.html)。
[^supplement]: Andrew Bitar 等. 《SHIP: SRAM-based Huge Inference Pipelines for Fast LLM Serving》补充材料. MLSys，2026。[原文](https://proceedings.mlsys.org/paper_files/paper/2026/file/9c20f16b05f5e5e70fa07e2a4364b80e-Supplemental-Conference.pdf)。
