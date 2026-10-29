---
type: paper
name: Pipelined Sharding
full_title: "Efficient, VRAM-Constrained xLM Inference on Clients"
authors: [Aditya Ukarande, Deep Shekhar, Marc Blackstein, Ram Rangan]
venue: MLSys
year: 2026
tags: [llm-inference, cpu-gpu-inference, offloading, scheduling, multimodal-inference, area/llm-inference]
source_pdf: "[[eb160de1de89d9058fcb0b968dbbbd68.pdf]]"
source_md: "[[eb160de1de89d9058fcb0b968dbbbd68]]"
review_status: complete
evidence_level: full-text
last_reviewed: 2026-10-29
---

# Pipelined Sharding：显存受限客户端的 CPU/GPU 推理调度（MLSys 2026）

> **原题**：Efficient, VRAM-Constrained xLM Inference on Clients

> **一句话总结**：系统以算子画像和显存预算为依据，按 token 数选择 CPU/GPU 子层放置及权重搬运方案；对视觉语言模型另配合视觉张量卸载与 attention 内存优化。[^paper]

**阅读状态**：已通读与会议正式 PDF 一致的 MinerU 正文及文内 artifact 附录，并核对系统图、设备配置和主要实验表；会议页面未列独立补充材料。[^paper]

## 问题与动机

客户端的独立 GPU 可能放不下模型与 KV，或需要给游戏等应用保留一部分显存。纯静态按层放置不一定适合所有情况：prefill 与多请求批量的计算量较大，GPU 可能更有利；逐 token decode 的数据搬运和主机内存带宽可能更关键（§1–3）。[^paper]

论文以显存预算为输入，让系统自动选择子层在 CPU/GPU 的驻留和执行方式。它所称的“无损”是相对选定模型执行调度不额外做剪枝等近似；实验模型已有不同量化格式，不能把它等同于“原始全精度模型完全无损”。[^paper]

## 关键观察 / 隐含假设

- **主机内存足够**：论文要求系统 RAM 能容纳目标模型与 KV；显存用得少不意味着模型只占相同数量的总内存（§2）。[^paper]
- **搬运与计算会竞争**：CPU 计算和 PCIe 传输共享主机内存控制器，重叠时未必各自保持单独执行的带宽（§4）。[^paper]
- **候选计划受画像限制**：选择器只比较预先设计的三个计划，最优结果限于候选集；新算子、驱动和硬件可能需要重新画像（§4）。[^paper]
- **输入和平台有边界**：实验在 Windows 客户端和 NVIDIA GPU 上执行，算法构想不等于已在所有 GPU、系统或移动设备验证（§2、§6）。[^paper]

## 核心方法

### 画像、规划与运行

安装时对 CPU/GPU 算子进行微基准，包括 CPU 与 PCIe 同时工作时的资源竞争；论文称一次画像约 15 分钟。规划时按 attention、KV、FFN 和输出等优先级分配显存常驻与 scratch 空间，并按每轮新 token 数建立计划表（§4）。[^paper]

运行时根据 batch 中本轮待处理的新 token 数选择 tier。prefill、decode 与混合请求的 token 数不同，因而可能选择不同策略；该数字不是历史上下文总长度。[^paper]

### 三种候选计划

1. **GPU-only**：未常驻权重仍在主机，使用双缓冲按需拷到 GPU 执行；较大 token 批次有机会隐藏传输。
2. **Static**：未常驻子层分给 CPU 或 GPU，尽量只在设备间传输中间结果；有利于避免每轮重复搬运大权重。
3. **Dynamic**：一部分子层由 CPU 执行，另一部分权重流式送 GPU，并尝试将复制与计算重叠。

规划器用画像估算计算与传输的重叠时间，选择该 tier 的最低估计耗时。它不是在所有可能算子切分与映射中求全局最优（§4）。[^paper]

### 视觉路径的额外优化

对图像 VLM，论文将视觉权重放到主机、给视觉编码启用 FlashAttention，并在视觉阶段结束后释放其显存，再初始化语言阶段。查询分块可以降低峰值显存，但也可能降低 attention 执行效率；这些 VLMOpt 方法与本文的语言路径调度组合使用（§5）。[^paper]

## 设计取舍

- **显存与主机资源**：较低显存预算会增加系统 RAM、CPU 和 PCIe 需求；需要一起检查容量与带宽。
- **预取与竞争**：双缓冲和异步复制可以隐藏部分等待，但 CPU 与传输争抢内存带宽，不能假定完全重叠。
- **显存峰值与并发**：视觉阶段和语言阶段串行释放可降低峰值，却可能限制阶段之间的并行。

## 实验与结果

实验使用 Windows 11 的一台笔记本和两台桌面机；高端桌面 `cli3` 配 RTX 5090、32 GB VRAM、16 核 EPYC 和 **256 GB 主机内存**。论文比较经过手工搜索 GPU 层数的 llama.cpp 基线，视觉模型另有 vLLM 比较；模型、预算和上下文各不相同（§6）。[^paper]

| 结果 | 实验条件与边界 |
|---|---|
| 低显存运行 | 表 4 中，`cli3` 的 Qwen-235B 量化模型磁盘大小约 77 GB，在 2G 显存预算、1K 上下文时约 7.7 TPS，16K 时约 5.2 TPS；这依赖大容量主机内存。 |
| 交互性能 | 图 2 在多模型、多预算和上下文上报告平均 TTFT 加速 2×、TPS 加速 3.7×，最大值分别为 6.7× 和 30×；最高值不能理解为所有配置的收益。 |
| 计划选择 | 在 105 个所测配置中，选择器在三种候选计划中均选中实测最快者；76 / 19 / 10 次分别为 Static / Dynamic / GPU-only。这不是对未测试配置或其他计划的最优保证。 |
| 批量 | `cli3` 的多请求实验平均总 TPS 加速约 2.3×，最高 8.2×；批量增大后 KV 超出预算时，部分配置 TPS 反而下降。 |
| VLM 显存 | 对 CR1 图像推理，方案与特定 vLLM 基线比较将显存需求从约 20 GB 降到 2 GB；论文指出两个系统处理多模态输入也有差异。 |

文中按 `TTFT + 100 / TPS` 估计标准输出长度的端到端延迟；它不是所有请求输出长度与逐 token 波动下的直接实测（§6）。文内 artifact 说明全套模型约需 175 GB 磁盘，本文未下载模型或运行复现实验。[^paper]

## 论断—证据表

| 论断 | 证据 | 评测边界 | 置信度 |
|---|---|---|---|
| 按条件选择 CPU/GPU 计划可改善所测客户端推理 | §7、图 2–5 | NVIDIA GPU、主机带宽及已有三个候选 | 中高 |
| 严格显存预算可借助主机 RAM 运行较大模型 | 表 3–4 | 模型量化和主机容量仍是关键成本 | 高 |
| 画像选择器在任意平台都找到全局最优计划 | §7 | 仅比较 105 个配置中的三种方案 | 低 |

## 批判性分析

### 论证链条

论文从 CPU、GPU、PCIe 的资源三角出发，用竞争条件下的画像估算三种候选计划，再用多显存预算与上下文长度实验检验。消融表明三种计划均有适用条件，但效果仍取决于基线的手工卸载选项和被测硬件。[^paper]

### 假设压力测试

若系统 RAM 不足、主机带宽被其他应用占用或 PCIe 低速运行，大模型虽然可在低显存预算下放置，响应速度也可能不实用。论文没有实现温度和动态功耗状态的自适应画像（§4）。[^paper]

### 实验可信度

论文报告多模型、多设备和游戏共存，但使用受控输入，并将标准 100 token 端到端时间由 TTFT 和 TPS 推算；同时，某些 vLLM 与视觉路径差异让单一加速比不能完全归因于所提调度。[^paper]

## 局限与后续工作

- 在真实交互输入、不同主机带宽及并发应用下同时测量 TTFT、逐 token 延迟、总内存和能耗。
- 验证新算子和驱动的画像失配、温度变化，以及显存预算对其他应用的影响。

## 相关

- [[LLM-Inference]]：CPU/GPU 执行与分层存储。
- [[KTransformers-SOSP25]]：面向 CPU/GPU 的 MoE 推理优化；两者的模型和调度策略不同。
- [[LocalityAwareBeam-MLSys26]]：面向多分支生成的 KV 组调度，目标同样受显存和主机搬运约束。

## 参考文献

[^paper]: Aditya Ukarande 等. 《Efficient, VRAM-Constrained xLM Inference on Clients》. MLSys，2026。[原文](https://proceedings.mlsys.org/paper_files/paper/2026/hash/7cd265ae802235b8d5778a4a96ff22dd-Abstract-Conference.html)。
