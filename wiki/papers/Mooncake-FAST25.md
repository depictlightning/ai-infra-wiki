---
type: paper
name: Mooncake
full_title: "Mooncake: Trading More Storage for Less Computation — A KVCache-centric Architecture for Serving LLM Chatbot"
authors: [Ruoyu Qin, Zheming Li, Weiran He, Jialei Cui, Feng Ren, Mingxing Zhang, Yongwei Wu, Weimin Zheng, Xinran Xu]
venue: FAST
year: 2025
tags: [area/kv-cache-systems, lens/kv-cache-offloading, lens/pd-serving, llm-serving, disaggregated-serving, kv-cache, scheduling, storage]
source_pdf: "[[fast2025-qin.pdf]]"
source_md: "[[fast2025-qin]]"
review_status: needs-review
evidence_level: full-text
last_reviewed: 2026-10-29
---

# Mooncake：以共享 KV cache 支撑分离式推理服务（FAST 2025）

> **原题**：Mooncake: Trading More Storage for Less Computation — A KVCache-centric Architecture for Serving LLM Chatbot

> **一句话总结**：Mooncake 将 P/D 分离与跨节点 KV cache 池结合，利用缓存感知调度减少重复计算；在工具与智能体轨迹、200 ms TBT 门槛下，有效请求占比相对启用前缀缓存的 vLLM 提高 42%（§5.2.2）。[^paper]

**阅读状态**：MinerU Flash 输出全文文本但未导出图像；已回看 PDF 的图 6、9–11，其余图表和实验细节尚未复核。[^paper]

## 问题与动机

长上下文和多轮交互可能反复使用共同前缀。只用单机缓存会受容量和请求路由影响；但跨机读取旧 KV 也消耗网络。Mooncake 同时安排计算阶段和缓存位置，以满足 TTFT、TBT 目标。

## 关键观察 / 隐含假设

- 在作者的工作负载分析中，单节点约 300 万 token 的缓存容量无法达到多数轨迹理论命中率的一半；该分析只模拟请求顺序，不计计算时间和热点复制（§5.3.1、图 9）。
- 缓存复用收益需要足够高的命中率和网络带宽；§2.2 的阈值分析依赖模型、硬件和传输与计算的重叠。

## 核心方法

全局调度器为请求选择 prefill 与 decode 实例。Mooncake Store 汇集节点的 CPU 内存与 SSD 存储 KV，并以拓扑感知传输引擎跨节点取回；调度器兼顾缓存位置和负载，复制热点块（§3–4）。prefill 按块和层发送新 KV 给 decode，长输入还可利用分块流水线并行。旧 KV 的外部读取与 P/D 交接是两类不同数据流，[[DualPath-SIGCOMM26]] 进一步讨论前者的网卡瓶颈。

## 设计取舍

### 一条请求经过哪些位置

Conductor 在请求进入后选择 prefill 和 decode 节点：先根据块键寻找可复用前缀，将命中的旧 KV 装入 prefill 侧；然后只对新增 token 执行 prefill。新生成的 KV 按层异步发送到 decode 节点的 CPU 内存，与后续 prefill 计算重叠；完整 KV 到达后，请求加入 decode 的连续批处理（§3.1、图 3）。没有命中时会跳过旧 KV 加载，但仍要完成新 KV 的交接。[^paper]

这里有两个不同的网络动作：旧 KV 从共享缓存池读回 prefill，新的请求状态从 prefill 发往 decode。前者能省计算，后者服务于阶段交接；测量时应分别统计字节、排队和可见等待。[^paper]

### 缓存如何组织和淘汰

Store 将 KV 分成分页块，键同时包含当前块内容及其前缀信息，以便识别可复用前缀；热点块可有多个副本。缓存池满时采用 LRU，正在被请求访问的块不会作为淘汰对象（§3.2.1）。块大小与模型及传输粒度有关，实验设置为 256 token；块过小会增加元数据和传输操作，过大则可能减少部分前缀的复用机会。最后一句是基于设计的推断，需要实测。[^paper]

传输引擎根据源、目的内存的 NUMA 和 PCIe 拓扑选择本地或远端网卡，把一次传输切成 16 KB 的片以使用多条路径。连接按需建立并受连接池大小约束；检测到临时网卡或链路问题时，系统尝试其他可达路径（§3.2.3）。这些机制减少单路径瓶颈，但网络重试不能代替完整节点故障恢复。[^paper]

### 路由为何不能只追求最长前缀

对于每个 prefill 候选，调度器估计排队、剩余 prefill 和可能的缓存传输时间，选择预测 TTFT 最短的实例；若估计无法达到 SLO，则提前返回 HTTP 429（§4.1–4.2）。短队列可能比最长本地命中更有价值。热门前缀被其他实例取回后会形成副本，进而分散后续请求，但预测误差与链路拥塞会改变实际收益。[^paper]

### 长输入的流水线

Mooncake 将多个 prefill 节点组织为流水线组，长输入切成块，由不同节点处理不同层组；节点间主要在流水线边界传递状态。论文选择此方式以避免跨节点 TP 在每层进行频繁集合通信（§3.3）。多个块在依赖允许时重叠，收益取决于块数、流水线填充及网络竞争，不能简单视作节点数倍加速。[^paper]

更大的共享缓存用内存、网络与副本维护换取较少重复计算；热点复制缓解读取拥塞，也占用容量。P/D 分离允许单独配置资源，但需要匹配阶段比例。

## 实验与结果

- 重现实验使用 LLaMA3-70B 架构的 dummy 模型、16 个八卡 A800 节点；对比 vLLM 0.5.1 及单独启用前缀缓存或分块预填充的配置（§5.1–5.2）。两项 vLLM 功能未同时启用。
- 工具与智能体轨迹在 200 ms TBT 门槛下，图 6 的有效请求占比相对前缀缓存 vLLM 提高 42%；TBT 定义为最长 10% token 间隔的平均值（§5.1、§5.2.2）。这不是吞吐量提高 42%。
- 10 个 prefill 节点的隔离实验中，全局缓存相对本地缓存命中率最高增加 136%，prefill GPU 时间最高减少 48%；输出固定为 1 token（§5.3.2、图 10）。
- 16 节点合成负载的比例实验中，约 1:1 的 P/D 比例得到最高有效请求容量；该比例不应推广到其他负载（§5.5、图 15）。

### 工作负载和指标

论文表 2 的对话、工具与智能体、合成工作负载平均输入分别为 12035、8596、15325 token，平均输出分别为 343、182、149 token；对应前缀缓存比例为 40%、59%、66%。前两者来自 Kimi 各一小时轨迹并使用原始到达时间，合成负载组合公开数据集并使用泊松到达（§5.2.1）。缓存比例与最终缓存命中率不是同一个量：位置、容量和淘汰策略都会影响实际命中。[^paper]

§5.1 将有效请求定义为同时满足 TTFT 与 TBT 门槛的请求，并把其占比称为有效请求容量；相关图的纵轴是请求比例。论文也讨论服务最大吞吐，阅读时应结合图轴，不能把某个阈值下占比增长直接说成同等幅度的最大请求速率增长。[^paper]

## 论断—证据表

| 论断 | 证据 | 边界 | 置信度 |
|---|---|---|---|
| 共享缓存可以减少所测负载的 prefill 工作 | §5.3.2、图 10 | 10 个 prefill 节点，输出限制为 1 | 中 |
| 端到端服务受益于缓存及阶段组织 | §5.2、图 6 | A800、dummy 模型和特定基线 | 中 |

## 批判性分析

### 论证链条

容量分析、缓存隔离实验和端到端比较分别支持共享缓存的动机与作用；端到端差距同时包含调度、缓存和阶段分离，不能全部归因于单个组件。

### 假设压力测试

低前缀复用、共享链路拥塞、缓存热点变化或更小 KV 表示都可能改变收益；带宽门槛是特定实验条件的结果。

### 实验可信度

作者使用生产轨迹和合成负载；历史生产统计与 dummy 模型回放应分别看待。基线版本较旧且缓存和分块没有联合启用。

### 系统性缺陷

论文未充分量化大规模节点故障时缓存恢复、调度元数据一致性与拥塞尾部的成本。

## 局限与后续工作

- 固定模型与 GPU 预算，扫描前缀复用率、网络带宽与缓存容量，记录 TTFT 和网络排队。
- 对比不同动态 P/D 配比在突发流量下的恢复时间。

## 相关

- [[PD-Serving]]、[[KV-Cache-Systems]]、[[DualPath-SIGCOMM26]]

[^paper]: Qin 等. “Mooncake: Trading More Storage for Less Computation — A KVCache-centric Architecture for Serving LLM Chatbot.” FAST 2025。[论文](https://www.usenix.org/conference/fast25/presentation/qin)；本地全文 [[fast2025-qin]]。
