---
type: paper
name: ProfInfer
full_title: "ProfInfer: An eBPF-based Fine-Grained LLM Inference Profiler"
authors: [Bohua Zou, Debayan Roy, Dhimankumar Yogesh Airao, Weihao Xu, Binqi Sun, Yutao Liu, Haibo Chen]
venue: MLSys
year: 2026
tags: [llm-inference, profiling, observability, ebpf, edge-inference, llama-cpp, area/llm-inference]
source_pdf: "[[03dbc11a22e79cd38bea53cf518c2371.pdf]]"
source_md: "[[03dbc11a22e79cd38bea53cf518c2371]]"
review_status: complete
evidence_level: full-text
last_reviewed: 2026-10-28
---

# ProfInfer：面向端侧 LLM 的细粒度推理剖析（MLSys 2026）

> **原题**：ProfInfer: An eBPF-based Fine-Grained LLM Inference Profiler

> **一句话总结**：ProfInfer 在 llama.cpp 等端侧运行路径上使用 eBPF 探针，将 token、计算图、算子和 CPU 调度事件关联到性能计数器，帮助定位执行瓶颈；观察粒度和开销受后端、探针选项及丢样影响。[^paper]

**阅读状态**：已阅读会议正式版 MinerU 全文，核对剖析开销表、异构后端说明及 MoE 案例。[^paper]

## 问题与动机

端侧 LLM 推理受 CPU 算力、带宽、内存容量、线程调度和能耗共同限制。仅看整体 token/s 很难判断时间花在哪个算子，或者 CPU/GPU/NPU 分工、线程抢占、缓存和页面换入如何影响性能（§1）。[^paper]

论文以 llama.cpp/GGML 为主要对象，在不修改或重新编译目标运行时源码的情况下挂接探针，重建与推理语义相联系的性能视图。这是观测系统，不是直接提高推理吞吐的调度器（§1、§3–4）。[^paper]

## 关键观察 / 隐含假设

- **探针依赖运行时内部结构**：要识别函数及张量结构并随版本维护；无需修改源码不意味着无需适配或调试符号与权限支持（§3、§7）。[^paper]
- **测量会扰动被测程序**：更细粒度、更多计数器和事件会增加开销与丢样；结果需要同时报告覆盖度和测量代价（§5.1–5.2）。[^paper]
- **异步设备提交不等于执行结束**：部分 OpenCL 内核的设备时间需要结合内建 OpenCL profiling，不能仅以主机函数进出时间替代（§5.3.2、图 7）。[^paper]

## 核心方法

### 多层探针

用户态程序加载并配置 BPF 程序，向 llama.cpp/GGML 相关函数挂接入口和返回探针，也读取 Linux 线程调度事件。处理器记录时间、线程、CPU，以及按需读取算子类别、张量形状和来源；通过缓冲区交给用户态分析。缓冲策略存在权衡：论文指出 ring buffer 开销较低，但不能报告缺失事件；perf buffer 可获知丢失事件（§3.1–3.2）。[^paper]

token 层追踪一次 prefill 或 decode 调用；图层记录后端划分；算子层通过 GGML 数据结构恢复操作类型与依赖；调度事件帮助判断线程是否被抢占。硬件计数器进一步提供缓存、内存访问和停滞等信息（§3.3）。[^paper]

### 三种视图

ProfDAG 以张量依赖重建计算图并将时间或计数器标注到节点；ProfTime 在时间线上关联线程、算子与后端；ProfStat 跨 token、算子形状或 MoE 专家聚合统计。按论文实现，只有特定后端可直接追踪算子层；OpenCL 异步路径还需额外内建 profiling 才能观察对应设备内核（§4、§5.3）。[^paper]

系统可在解码速度低于设定阈值时关闭部分观测以减轻干扰，但缺失的观测粒度与性能开销之间需要权衡（§3.1）。[^paper]

## 设计取舍

- **多粒度观测换探针维护**：关联模型结构与硬件事件很有用，但函数、结构体与后端随运行时版本变化。
- **详细结构换事件量**：打开张量信息和计数器更利于诊断，也可能丢样或增加开销。
- **无源码改动换平台依赖**：eBPF 能动态挂载，但受内核、权限、可探测函数与设备后端限制。

## 实验与结果

论文使用 Orange Pi 和 RUBIK Pi 设备，以及不同版本的 llama.cpp 后端；主要 FP16 开销测试为 LLaMA-3.2-1B、Qwen2.5-1.5B 和 Gemma2-2B，生成 50、100 或 200 token，每种配置重复五次（§5、表 4–6）。[^paper]

| 结果 | 条件与边界 |
|---|---|
| BCC 实现解码速度平均下降 1.63%–3.67% | 不同结构信息、PMC 和缓冲选项；表 6 中各项有波动，不是所有设备和工作负载的硬性上界。 |
| libbpf 最低报告 1.19% | 特定配置，且论文指出该实现部分功能尚不支持；不能脱离功能覆盖比较（§5.1、表 6）。 |
| 开启结构信息可能丢失约 0.4%–0.6% 事件 | 作者建议用 perf buffer、较少线程并在空闲环境重复采样；ring buffer 无法报告缺失事件（§3.2、§5.2）。 |
| KV 增长带来解码算子时间增长 | LLaMA-3.2-1B 两线程、生成 450 token 的案例中，KQ 与 KQV 是时间变化的主要来源（§5.3.3、图 9）。 |
| MoE 案例发现换页瓶颈 | Qwen1.5-MoE-A2.7B 的 4-bit 权重约 8.9 GB，在所测内存受限设备上通过 mmap 运行；结合 major page fault 等观察，作者判断磁盘 I/O 成为瓶颈（§5.3.3、图 13）。 |

表 6 标题将 BCC 平台称为 Orange Pi 5 Pro，而设备表列的是 Orange Pi 5 Ultra；正文没有明确解释名称差异，因此这里不把该项强行归到某个具体板型。[^paper]

## 论断—证据表

| 论断 | 证据 | 评测边界 | 置信度 |
|---|---|---|---|
| 探针可关联 LLM 执行语义与系统事件 | §3–4、图 5–8 | llama.cpp/GGML 和指定后端，移植需适配 | 中高 |
| 所测端侧推理剖析开销较低 | §5.1、表 6 | 不同功能与平台不完全相同，短算子更易受扰动 | 中 |
| 多后端算子都可仅凭主机探针精确计时 | §5.3.2、图 7 | 异步 OpenCL 设备内核需要额外 profiling | 低 |

## 批判性分析

### 论证链条

系统通过探针构建图和时间线，再用模型案例说明如何区分线程、算子、KV 增长及缺页造成的等待；它能提出优化线索，但不能仅凭可视化证明某个新调度策略的端到端收益。[^paper]

### 假设压力测试

当运行时升级、符号不可见、设备执行异步或事件频率很高时，探针位置、时间语义和丢样率都可能发生变化。计数器的可用性也依赖处理器，移植不能只复制分析脚本。[^paper]

### 实验可信度

论文同时报告速度损失和少量事件丢失，且公开不同模型和端侧设备；但实验规模偏小，时间戳开销是微秒级，未说明所有短算子的误差上界。表 7 的功能列标注 ProfInfer 支持结构与 PMC，而正文称对照实验关闭这两项；功能支持和实验开关应分别理解（§5.1–5.2）。[^paper]

## 局限与后续工作

- 针对不同运行时版本、异步后端和更短的算子量化计时误差、遗漏率与适配成本。
- 在多请求、热节流和真实应用下验证观测是否能指导稳定的资源调整。

## 相关

- [[LLM-Inference]]：推理执行与性能分析。
- [[StriaTrace-OSDI26]]：面向分布式在线服务关键路径，观测层次和部署范围不同。

## 参考文献

[^paper]: Bohua Zou 等. 《ProfInfer: An eBPF-based Fine-Grained LLM Inference Profiler》. MLSys，2026。[原文](https://proceedings.mlsys.org/paper_files/paper/2026/hash/03dbc11a22e79cd38bea53cf518c2371-Abstract-Conference.html)。
