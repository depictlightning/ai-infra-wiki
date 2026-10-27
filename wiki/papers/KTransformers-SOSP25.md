---
type: paper
name: KTransformers
full_title: "KTransformers: Unleashing the Full Potential of CPU/GPU Hybrid Inference for MoE Models"
authors: [Hongtao Chen, Weiyu Xie, Boxin Zhang, Jingqi Tang, Jiahao Wang, et al.]
venue: SOSP
year: 2025
tags: [llm-inference, moe, cpu-gpu-hybrid, expert-offloading, amx, numa, mechanism/gpu-residency, mechanism/pipeline-overlap, mechanism/state-bytes, mechanism/data-movement, area/llm-inference]
source_pdf: "[[3731569.3764843.pdf]]"
source_md: "[[3731569.3764843]]"
review_status: complete
evidence_level: full-text
last_reviewed: 2026-10-27
---

# KTransformers：面向 MoE 的 CPU/GPU 混合推理（SOSP 2025）

> **原题**：KTransformers: Unleashing the Full Potential of CPU/GPU Hybrid Inference for MoE Models

> **一句话总结**：KTransformers 将常驻的路由专家留在 CPU 内存并在 CPU 上计算，将注意力和共享专家放到 GPU；无专家延迟时，单请求全精度解码相对 Fiddler 提高 2.42–4.09 倍，而可选的 Expert Deferral 再以改变模型计算顺序为代价获取最高 45% 的额外吞吐（§6.2–6.3）。[^paper]

**版本与证据**：本页依据 SOSP 2025 正式论文，沿用仓库已有的 PDF 和 MinerU Markdown，并核对图 3、图 7、图 9–14 及表 1–2。当前 [KTransformers 项目](https://github.com/kvcache-ai/ktransformers)仍在演进；下述机制与实验特指 2025 年论文版本。[^paper]

## 问题与动机

MoE 模型每个 token 只激活少量专家，但全部专家权重仍要有地方存放。对于低并发本地推理，把被选专家的完整权重从 CPU 内存反复搬到 GPU，容易受 PCIe 带宽限制。KTransformers 沿用 Fiddler 的计算卸载思路：**路由专家权重常驻 CPU DRAM，并在 CPU 上计算；GPU 负责注意力、门控与共享专家**。设备间主要传递激活和结果，而不是每步搬运整个专家权重（§1、§2.1）。[^paper]

这一分工把问题转移到 CPU 算力与内存带宽、CPU/GPU 同步，以及多路 CPU 的 NUMA 访问。论文在 DeepSeek-V3 的混合部署中观察到 GPU 利用率低于 30%；对 Fiddler 的解码剖析显示每 token 超过 7,000 次 CUDA kernel 启动，启动开销占其 GPU 执行时间的 73%。这些是特定实现和硬件下的观察，不代表所有推理引擎都有相同瓶颈（§1、§2.3、图 4）。[^paper]

## 关键观察 / 隐含假设

- **预填充与解码需要不同 CPU 内核**：每个专家同时处理较多 token 时，矩阵乘法更能利用 AMX；token 数很少时，AMX 填充和调度成本可能超过收益。图 7 的测试中，分给每个专家不超过 4 个 token 时，AVX-512 优于 AMX。阈值依赖具体模型、Xeon 和内核实现，不能视为通用常数（§3.2）。[^paper]
- **专家自然分工依赖模型结构**：共享专家每次都运行，放在 GPU 上便于与 CPU 路由专家并行；路由专家分散存放在 CPU。没有共享专家的模型需要另行选择热点专家，论文提到可参考 Fiddler 的离线热度统计，但没有证明此策略对所有路由分布有效（§1）。[^paper]
- **跨层并行有语义代价**：标准 Transformer 的第 `k+1` 层注意力依赖第 `k` 层全部专家结果。若要让 CPU 延迟专家与下一层 GPU 注意力重叠，就必须暂时跳过这条依赖。Expert Deferral 因此是近似执行，残差连接提供了之后合入结果的路径，但不保证输出逐位相同（§4.1、图 9）。[^paper]

## 核心方法

**模型放置**：注意力、门控和共享专家放在 GPU，路由专家权重留在 CPU；GPU 算出路由后，把激活及专家任务交给 CPU 线程池，GPU 可同时执行共享专家。CPU 结果返回后才参与该层结果合并。不同于按需把专家权重搬到 GPU，计算卸载把数据传输压力换成 CPU 算力、DRAM 带宽和线程调度压力（§2.1、§3.1、§3.3）。[^paper]

**算术强度感知的 CPU 内核**：加载权重时将矩阵重排为适合 AMX tile 的子块，按 64 字节对齐，并配合 L2/L3 缓存分块；Int4/Int8 权重采用分组量化，scale 单独存放。预填充优先 AMX，低算术强度解码使用与同一权重布局兼容的 AVX-512。Gate、Up、Down 等投影被融合为较少的任务批次，线程从轻量队列动态领取子任务，以缓解专家负载不均（§3.2、图 6–7）。[^paper]

**NUMA 与跨设备协调**：双路机器不把整位专家固定到某一个 socket，而是将每个专家的权重矩阵切分到两个 socket，让各 socket 读取本地分片，再合并局部结果。GPU 侧将 CPU 任务提交和等待放入 CUDA Graph 的主机回调节点，后台线程执行路由专家，使一个 token 的解码路径能够被一个 CUDA Graph 组织，减少逐层启动和同步开销（§3.3、图 8）。[^paper]

**Expert Deferral：延迟一部分专家结果**。先按路由分数区分立即执行和延迟合入的专家。第 `k` 层的部分专家仍在 CPU 计算，但下一层注意力先使用立即专家的结果开始；延迟专家的输出之后才加入残差路径，供更后的计算使用。最终层不做延迟。以 DeepSeek-V3 BF16 的分析为例，5 个立即专家、3 个延迟专家使 CPU 与下一层 GPU 工作更好重叠；量化版本评测可延迟 6 个，不能把两个配置混为一谈（§4.1–4.2、§6.3、图 9–10）。[^paper]

## 设计取舍

- **单 GPU 换大 CPU 内存和带宽**：论文的双路机器为每个 Xeon 8452Y 配 1 TB DDR5，总计 2 TB。表 1 的 DeepSeek-V3 把约 654B 参数放 CPU；若按 BF16 每参数 2 字节粗算，权重本身约 1.31 TB，尚未计入框架与运行开销。这是容量估算，不是论文实测峰值；“单卡运行”仍依赖服务器级主机资源（§6.1、表 1）。[^paper]
- **AMX 与 AVX-512 共用布局换适应性**：一个布局支持不同 token 数下切换内核，但加载时需要预处理；图 7 的切换收益与线程、缓存及指令集相关。
- **CUDA Graph 降启动成本，同时引入跨设备协调**：回调、任务队列、CPU 工作线程和结果合并仍需要正确同步。论文测了吞吐，没有系统评估进程故障、任务取消或多租户隔离。
- **延迟专家换重叠**：专家没有被直接丢弃，但它们的输出绕过了原本应该接收该结果的下一层注意力；后续合入不等价于原始模型。预填充没有启用该策略，因为不同 token 的专家覆盖增加，分成两类后可能接近重复读取权重（§4.1）。[^paper]

## 实验与结果

**环境**：单台双路 Xeon 8452Y，每路 36 核、1 TB DDR5；GPU 分别使用 A100 40 GB 和 RTX 4080 16 GB，CPU/GPU 通过 PCIe 4.0 连接。模型为 DeepSeek-V3-0324、DeepSeek-V2.5-1210 和 Qwen2-57B-A14B。A100 上比较 BF16/FP16；4080 上 DeepSeek-V3 使用 Int4，其余两个模型使用 Int8。所有吞吐实验 batch 为 1；预填充输入 32–8192 token，解码使用 32-token 提示和最多 512 个输出 token（§6.1、表 1）。[^paper]

- **CPU 内核**：单路 DeepSeek-V3 MoE 微基准中，AMX 内核峰值 21.3 TFLOPS，为 PyTorch/oneDNN 实现的 3.98 倍；这不是整机端到端的加速倍数（图 3、§6.2）。[^paper]
- **保持原计算顺序的解码**：全精度模型上，KTransformers 相对 Fiddler 为 2.42–4.09 倍、相对 Llama.cpp 为 1.25–1.76 倍。论文为 Llama.cpp 增加了专家级卸载支持；它的全精度对照使用 FP16，而 KTransformers 和 Fiddler 使用 BF16（§6.1–6.2、图 12）。[^paper]
- **额外的 Expert Deferral**：开启后解码吞吐最高再提升 45%；所测配置相对 Llama.cpp 的整体加速为 1.66–2.56 倍。DeepSeek-V3 BF16 的单层分析中，延迟 3 个专家使层执行时间减少 26%，端到端解码吞吐增加 33%；继续延迟到 4 个没有进一步改善该单层时间（图 10、图 12、§4.2、§6.3）。[^paper]
- **质量代价**：量化 DeepSeek-V3 的 LiveBench 测试中，延迟 6 个专家的平均准确率相对下降 0.5%，直接跳过同样数量专家则下降 13.3%。图 13 中数学和推理子项各下降 1.6%，平均值不能代表每个任务；该测试对 LiveBench 各子项做 10 次采样、温度 0.3（图 13、§6.1、§6.3）。[^paper]
- **消融**：NUMA 感知张量并行对解码最多带来 1.63 倍加速，CUDA Graph 最多再带来 1.23 倍；预填充中动态调度最多带来 1.83 倍。它们来自逐项加入优化的性能分解，不能与不同基线、模型或精度下的最大值直接相乘（图 14、§6.4）。[^paper]

## 论断—证据表

| 论断 | 证据 | 评测边界 | 置信度 |
|---|---|---|---|
| 预填充 CPU 专家计算受内核和布局影响 | 图 3：单路 AMX 内核 21.3 TFLOPS、相对 PyTorch 3.98 倍 | Xeon 8452Y、DeepSeek-V3 MoE 微基准 | 中 |
| 保持计算顺序的混合推理可超过已有实现 | 图 12：全精度解码相对 Fiddler 2.42–4.09 倍 | batch 1；Llama.cpp 对照精度及卸载实现不同 | 中 |
| 延迟专家可提高 CPU/GPU 重叠 | 图 10、12：DS-3 BF16 吞吐提高 33%，所有测试最高增加 45% | 低并发，延迟数量随模型和精度变化 | 中 |
| 延迟专家比直接跳过更能保留质量 | 图 13：DS-3 延迟 6 个平均下降 0.5%，跳过下降 13.3% | LiveBench 2024-11-25、量化模型、10 次采样；不保证逐 token 等价 | 中 |

## 批判性分析

### 论证链条

**加速来源必须分层看**：AMX、NUMA 和 CUDA Graph 主要减少执行开销；Expert Deferral 改变了计算依赖。图 12 将不开启与开启延迟的结果分开，使用者应先确认质量要求，再决定能否接受后者。论文摘要把多种模型和阶段的加速范围集中陈述，不能把最高倍数套到所有 prompt 或硬件上。

### 假设压力测试

**延迟不是简单的“晚一点算”**：被延迟专家仍对旧层输入计算，但其结果没有参加下一层注意力以及随后的门控决策；后续残差合入无法恢复这些中间计算。路由变化会在后续层和生成步骤累积。LiveBench 的平均下降 0.5% 是有限数据和采样条件下的经验结果，长链推理、代码执行、逐 token 一致性与不同随机种子的风险仍需单独评估。

**主机成本常被单卡叙事遮住**：A100 只装注意力和共享专家，671B 模型的大部分权重放在 2 TB DRAM 的双路服务器上。低并发时每 token 只读被路由的专家，内存带宽有机会支撑推理；当并发、专家覆盖率或上下文增长时，CPU 带宽和线程争用可能改变瓶颈。论文的 batch 1 与短解码输入不足以证明其在高并发在线服务中仍优于全 GPU 部署。

### 实验可信度

**基线和外推**：Llama.cpp 为实验增加专家级卸载，全精度对比还存在 BF16 与 FP16 的差异；因此数字是论文实现与配置下的系统结果，不是所有版本 Llama.cpp 的固有差距。两个 Xeon socket 的结果也不能直接推广到单路桌面 CPU、ARM 或不同 PCIe/NUMA 拓扑。

### 系统性缺陷

论文没有评估 CPU 工作线程失败、请求取消时的跨设备同步和多租户资源隔离；这些问题在长期运行的服务中需要独立验证。

## 局限与后续工作

- 在相同模型、精度、CPU 内存预算和上下文长度下测量完整请求时延、吞吐和能耗；增加 batch 与并发，找出 CPU/GPU 混合方案与全 GPU 方案的交叉点。
- 对不同延迟专家数量报告每类任务质量、逐 token 输出差异与长生成误差；根据质量阈值探索自适应延迟，并测量选择策略本身的开销。
- 比较单路和双路 NUMA、不同专家访问分布及权重量化方案，分别测量 DRAM 带宽、跨 socket 流量和 CUDA Graph 回调等待。

## 相关

- **会议与专题**：[[SOSP-2025]]、[[LLM-Serving]]。
- **站内对照**：[[Janus-SOSP26]] 研究生产多模型的资源放置和弹性；[[Prism-OSDI26]] 管理模型权重与 KV cache 的显存占用。KTransformers 的重点是单机 CPU/GPU 分工及专家执行重叠，工作负载与前两者不同。
- **原始材料**：[[3731569.3764843.pdf]]、[[3731569.3764843]]。

## 参考文献

[^paper]: Hongtao Chen 等. 《KTransformers: Unleashing the Full Potential of CPU/GPU Hybrid Inference for MoE Models》. SOSP 操作系统会议，2025-10-12。[原文](https://doi.org/10.1145/3731569.3764843)。
