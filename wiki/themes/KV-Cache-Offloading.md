---
type: theme
topic: KV-Cache-Offloading
title: LLM KV Cache Offloading
theme_kind: lens
member_tag: lens/kv-cache-offloading
candidate_tags: [kv-cache, offloading, storage-hierarchy]
paper_count: 10
first_generated: 2026-10-30
last_updated: 2026-10-30
tags: [topic-overview, llm-inference, kv-cache, offloading]
---

# LLM KV Cache Offloading

KV cache 让模型在生成新 token 时复用历史计算，但它的容量随上下文和并发增长。把 KV 扩展到 CPU、SSD 或远端存储，可以腾出 GPU 显存；随之而来的问题是，下一次需要这些数据时，谁读取、搬多少、等待多久，以及还能否保持原来的计算语义。[^flexgen][^infinigen]

本专题沿着一条请求的数据路径比较不同方案：先看 GPU 显存不足时的基础卸载，再看活跃 KV 怎样参与计算，最后看分层缓存和集群存储怎样保存、定位与恢复 KV。**跨请求复用与活跃请求是使用场景，贯穿多个技术方案**；一个系统也可以同时处理多种场景。

## 纳入范围

核心论文直接研究 KV cache 在 GPU 与 CPU、SSD 或远端存储间的放置、卸载、访问或共享，并已有全文笔记。FlexGen 同时涉及权重和激活，作为异构卸载的基础纳入。专题区分完整注意力、近似选择和模型原生稀疏注意力；纯模型量化、永久删除 KV、仅研究 GPU 内分页的工作不作为核心成员。

名单按技术问题组织，并非互斥分类：同一个系统可以同时使用 CPU 计算、选择性回载和缓存管理。各论文的原始版本、评测条件、阅读状态以各自页面为准；[[Mooncake-FAST25]] 的部分图表仍需复核。

## 核心论文

- [[FlexGen-ICML23]]（2023）— GPU、CPU 和 SSD 上的张量放置、块调度与可选 CPU 注意力。
- [[InfiniGen-OSDI24]]（2024）— 以 GPU 上的部分 Key 预测后续层需要的 KV，从主机选择性回载。
- [[NEO-MLSys25]]（2025）— 将部分请求的 KV 与解码注意力放在 CPU，以非对称流水线和负载感知调度协同 GPU。
- [[Mooncake-FAST25]]（2025）— 集群级 KV 存储池、跨实例传输与缓存感知调度；论文页仍待进一步复核。
- [[LMCache-arXiv25]]（2025）— 位于推理引擎与多级存储之间的 KV 缓存、搬运与控制接口。
- [[SolidAttention-FAST26]]（2026）— 将完整历史放在 SSD，结合块级近似选择、预取及计算重叠。
- [[Strata-OSDI26]]（2026）— 为分层上下文缓存优化数据布局、碎片搬运与缓存感知调度。
- [[DirectKV-OSDI26]]（2026）— GPU kernel 直接访问主机内存中的 KV，减少显存暂存。
- [[ECHO-OSDI26]]（2026）— 对原生稀疏注意力模型按需回载 KV，并在计算前补齐准确选择。
- [[HiSparse-arXiv26]]（2026）— 主机保存历史、GPU 维护热工作集，用稀疏选择和可选精确预取管理驻留。

## 相关博客

- [[SGLang-HiCache]] — SGLang 的多级前缀缓存和存储后端，结合 Strata 论文阅读。
- [[SGLang-HiSparse]] — SGLang 稀疏注意力卸载的工程说明；实验配置与论文不同。
- [[vLLM-Hybrid-HiSparse]] — vLLM 根据显存压力在 GPU 驻留与主机回载间调整。

## 阅读路线与判断框架

先读 [[FlexGen-ICML23]]，理解为什么权重、KV、激活的放置和批次调度必须一起考虑。然后比较 [[InfiniGen-OSDI24]]、[[DirectKV-OSDI26]] 和 [[NEO-MLSys25]]：它们分别减少回载量、改变 GPU 访问主机内存的路径，以及让 CPU 承担部分注意力计算。接着看 [[SolidAttention-FAST26]]、[[ECHO-OSDI26]]、[[HiSparse-arXiv26]] 与 [[vLLM-Hybrid-HiSparse]]，区分近似选择和模型原生选择。最后读 [[SGLang-HiCache]] / [[Strata-OSDI26]]、[[LMCache-arXiv25]] 和 [[Mooncake-FAST25]]，理解前缀 KV 的分层保留及共享。

每次比较先回答五个问题：

1. **存什么、存在哪里？** 完整历史、所选工作集或可复用前缀分别占用 GPU、CPU、SSD 和远端多少空间？权重、索引、页表和缓冲是否计入？
2. **何时使用？** 是本请求每层 decode 的历史读取，后续请求的前缀恢复，还是 P/D 阶段之间的交接？一次系统可能覆盖其中多项。
3. **谁完成注意力？** GPU 从本地显存算、GPU 直接读主机内存算，或 CPU 对部分请求计算；所需通信路径不同。
4. **省掉哪些访问？** 搬完整历史、按预测回载、按模型本身的选择回载，或通过 CPU 计算避免回搬。
5. **正确性相对谁？** 是未改写的完整注意力、模型原有的稀疏注意力，还是已量化的模型配置？预取错误、永久淘汰与近似注意力需要分开讨论。

## 从基础卸载到不同计算路径

### FlexGen：放置、批次和计算一起决定

[[FlexGen-ICML23]] 把权重、激活与 KV 分配在 GPU、CPU 内存和 SSD 上。它让同一层权重服务多个 GPU 批次，再转入下一层，以较长批次等待换取权重 I/O 的摊薄；代价是同时驻留更多 KV 和激活。其基础调度可以保持完整计算，也提供四位量化和稀疏注意力等近似选项。它还允许 CPU 执行部分解码注意力，因此不能把 FlexGen 概括成“所有 KV 一律搬回 GPU”。[^flexgen]

论文最有名的 T4、OPT-175B 结果使用四位压缩、有效批次 144，约 4072 秒生成吞吐 1.122 token/s。这个离线批次结果不能当作在线交互速度；量化结果也不能当作完整精度的无损结果。[^flexgen]

### GPU 访问主机 KV，还是让 CPU 计算？

假设某个请求的历史已经在 CPU 内存：

```text
                  ┌─ 完整或部分 KV → GPU 暂存 → GPU 注意力
CPU 上的历史 KV ──┼─ GPU kernel 直接读取主机内存 → GPU 注意力
                  └─ CPU 就地计算注意力 → 输出送回 GPU
```

这三条路径都会受到主机内存带宽和设备互连约束，差异在于传输对象、暂存空间和计算地点。

[[DirectKV-OSDI26]] 面向 GH200 等高速 CPU–GPU 互连，使用 GPU kernel 访问主机中的 KV，减少显存暂存。直接访问不代表数据没有跨互连移动；重复读取同一主机数据仍可能放大带宽压力。论文只在其测试平台和内核配置下给出收益，不应直接外推到普通 PCIe 服务器。[^directkv]

[[NEO-MLSys25]] 则让部分请求的 KV 和解码注意力留在 CPU，GPU 同时执行其他请求的预填充、注意力和线性层。CPU 请求仍需要与 GPU 交换 Q/K/V 和注意力输出；收益依赖 CPU 带宽、可并发请求和两侧流水线平衡。作者在 A10G 在线测试的指定延迟点，相对 vLLM 的吞吐提高 6.4%；不同实验中收益甚至可能很小或为负。[^neo]

两者不是同一硬件下的直接对照：DirectKV 测试高速一致性互连上的 GPU 访问，NEO 测试将部分计算放在本机 CPU 的在线调度。比较需要固定模型、序列、主机内存带宽、互连、GPU 和延迟目标。

## 选择性回载：减少数据访问需要付出什么

### 近似选择：相对完整注意力可能改变计算

[[InfiniGen-OSDI24]] 在 GPU 保留每个历史 token 的部分 Key 通道，用前一层输入预测下一层哪些 token 重要；完整或大部分历史 KV 留在主机，只为选中 token 取回完整 KV。它的预测和稀疏计算相对原模型完整注意力是近似的。主机池若有容量上限，也可能永久淘汰部分历史；“可以回载”不能无条件理解成所有历史永远保留。[^infinigen]

[[SolidAttention-FAST26]] 面向内存受限个人电脑，将完整历史放入 SSD，按块选择所需上下文；数据布局、基于历史选择的预取和细粒度任务调度减少等待。块选择本身是近似注意力：系统对预测预取漏掉的**已选块**会补读，但它不会因此恢复被稀疏选择排除的全部块。论文在 RTX 4070 Laptop、128K 输入、Llama-3.1-8B 上，相对同样使用 SSD 和稀疏注意力的基线达到 3.1 倍吞吐；后台 SSD 读取也会明显降低收益。[^solid]

### 保持原生稀疏选择：补齐模型真正选中的 KV

原生稀疏注意力模型本身就规定某一层只访问索引器选出的部分位置。卸载系统如果确保模型所选 KV 在计算前全部可用，就可以保持**该模型原有的稀疏选择语义**；这并不说明模型与完整注意力等价，浮点逐位一致也需要另测。[^echo][^hisparse]

[[ECHO-OSDI26]] 将完整历史放在主机、GPU 管理所选工作集。它的预取可以猜测，但最终会补齐真实 top-k 的未命中项。预测偏差因此主要影响多搬或等待，而不由预测结果替代模型的最终选择。论文所测系统最大约 1000 GB 主机池，并报告单项预取对端到端吞吐的贡献有限；容量扩展带来的更大并发不能全部归因于预取。[^echo]

[[HiSparse-arXiv26]] 和 [[SGLang-HiSparse]] 以 GPU 热缓冲保留近期常用项，未命中时从主机召回。只有模型提供跨层共享索引等可提前知道的选择时，系统才能做相应的精确跨层预取；论文中对普通相邻层猜测的实验没有端到端收益。[[vLLM-Hybrid-HiSparse]] 则根据显存压力调整 GPU 普通驻留页和主机侧历史的比例；其索引器状态和其他缓存组仍要单独计入。[^hisparse][^hisparseblog][^vllm]

## 分层缓存与存储池

这一组方案更侧重把可复用 KV 在请求之外保存下来，以及让引擎或其他实例找到它。它们也可能支持 P/D 交接或其他路径；以下按主要职责比较，不将三者视作互不兼容的产品。[^hicache][^lmcache][^mooncake]

| 系统 | 主要位置与职责 | 下沉后如何使用 | 特别需要核对 |
|---|---|---|---|
| [[SGLang-HiCache]] / [[Strata-OSDI26]] | SGLang 内的 GPU、CPU 与外部存储分层；HiRadixTree 和控制器管理位置 | 匹配前缀后加载，支持分层预取与计算重叠 | 存储布局、写入策略、缓存命中后等待与请求调度 |
| [[LMCache-arXiv25]] | 推理引擎与不同存储／网络后端之间的独立缓存层 | 连接器报告命中并发起读取、保存或传输 | 引擎版本、chunk 与页的转换、管理器和后端配置 |
| [[Mooncake-FAST25]] | 面向服务集群的 KV 存储池及传输、调度 | 在不同实例之间定位、共享或交接 KV | 节点位置、网络、热点复制、负载及 P/D 分配 |

例如 SGLang + HiCache 可以使用 Mooncake 作为外部后端；LMCache 也可以对接 Mooncake Store。**引擎缓存层与共享存储池处于不同职责位置**，但各自也有搬运和管理能力。配置上仍要确认实际版本、后端和数据路径。[^hicache][^lmcache-docs]

### HiCache 与 Strata 如何一起读

2025 年 HiCache 博客已经描述 HiRadixTree、GPU 辅助 I/O、GPU 按层／主机按页布局和外部存储预取。Strata 论文围绕同一 SGLang 分层缓存路线，进一步给出碎片传输、等待命中、批次平衡和使用 decode 填补 I/O 等待的分析与实验。两者有大量设计重合，专题将其作为一组资料；论文中的 SGLang-HiCache 基线是评测时的特定配置，不能泛化为 HiCache 所有版本。[^hicache][^strata]

### 缓存命中不等于立即可用

假设新请求需要已保存的 100K token 前缀：系统先查到位置，再等待可用带宽，把数据从存储、主机内存送到 GPU 并转换布局；如果实际加载比重算更慢，命中也不一定降低 TTFT。[[LMCache-arXiv25]] 图 15 的低带宽实验就出现过这样的交叉。并发请求同时等待同一前缀时，还要考虑重复加载、重复计算和排队。[[Strata-OSDI26]] 将加载时间和正在生成但尚未可复用的前缀纳入调度。[^lmcache][^strata]

## 横向对照

表中的“语义”描述论文所讨论的主要配置；它不保证不同硬件内核逐位一致。各论文有不同模型、硬件和工作负载，此表用于理解机制，不能用论文摘要的加速倍数直接排序。

| 工作 | 主要低层 | 数据与计算方式 | 语义及主要代价 |
|---|---|---|---|
| [[FlexGen-ICML23]] | CPU、SSD | 权重、激活与 KV 分层；支持 GPU 计算及部分 CPU 注意力 | 基础计算可完整；量化、稀疏选项为近似；大批次等待长 |
| [[InfiniGen-OSDI24]] | CPU | GPU 预测，再回载所选完整 KV | 相对完整注意力近似；索引占显存，主机池可能淘汰 |
| [[DirectKV-OSDI26]] | CPU | GPU kernel 直接读取主机 KV | 完整访问路径；依赖高速互连及访存布局 |
| [[NEO-MLSys25]] | CPU | 以请求为单位将 KV 和部分解码注意力留在 CPU | 不靠稀疏裁剪；CPU 带宽、调度和 Q/K/V 通信是代价 |
| [[SolidAttention-FAST26]] | SSD，经过主机缓冲 | 选择块并预取，GPU 对所选历史计算 | 相对完整注意力近似；SSD 竞争和局部性影响性能 |
| [[ECHO-OSDI26]] | CPU | 模型选择后召回；提前预取并在计算前补齐 | 预取保持原生稀疏模型选择；主机池与索引仍占资源 |
| [[HiSparse-arXiv26]] | CPU | 完整历史与 GPU 热缓存，未命中时召回 | 保持所测稀疏选择语义；索引和元数据另计 |
| [[Strata-OSDI26]] | CPU、外部存储 | 分层前缀匹配、GPU 辅助搬运与调度 | 精确前缀复用；加载、复制及调度等待需要核算 |
| [[LMCache-arXiv25]] | CPU、磁盘、远端 | 引擎连接器和可配置缓存后端 | 普通复用可保存完整 KV；若启用压缩需另核对精度 |
| [[Mooncake-FAST25]] | 集群内主机存储与远端 | KV 存储池、跨节点共享和传输 | 缓存位置、网络、调度及多副本占用影响收益 |

## 如何设计可比较的实验

- **固定模型和计算语义**：分开测完整注意力、原生稀疏注意力与近似裁剪；量化权重和 KV 的精度单独列出。比较输出误差或任务质量时，不能用同一模型名称掩盖配置差异。
- **统计全部存储**：逐层记录 GPU 上的 KV、预测索引、元数据、暂存区，CPU 上的完整历史和 pinned 内存，以及 SSD、远端副本。GPU 省出的容量不等于系统总成本减少。
- **分清每条链路**：分别记录 CPU–GPU、SSD–CPU、网络与可能的 GPU 直达路径；标注 PCIe、NVLink-C2C、NUMA、RDMA 和竞争任务。DirectKV 的高带宽互连不能直接代表普通 PCIe。
- **区分使用阶段和计量单位**：活跃 decode 关注每 token 延迟与召回；前缀复用关注命中、读取与 TTFT；P/D 交接还需计入两端排队和传输。平均值、尾延迟、请求吞吐与输出 token 吞吐不能互换。
- **比较加载和重算**：同一前缀长度、链路负载和缓存预算下测量冷／热命中及重算，包含排队和布局转换；单纯提高命中率不保证降低总延迟。
- **覆盖失败与压力**：测量未命中、预测错误、容量不足、后台 I/O、CPU 争用和突发请求。区分论文实际评估过的事实与还没有实验覆盖的风险。

更广的模型结构、GPU 驻留和请求路由主题见 [[KV-Cache-Systems]]；P/D 配置与资源划分见 [[PD-Serving]]。

## 参考文献

[^flexgen]: Ying Sheng 等. 《FlexGen: High-Throughput Generative Inference of Large Language Models with a Single GPU》. ICML，2023。[论文](https://proceedings.mlr.press/v202/sheng23a.html)。
[^infinigen]: Wonbeom Lee 等. 《InfiniGen: Efficient Generative Inference of Large Language Models with Dynamic KV Cache Management》. OSDI，2024。[论文](https://www.usenix.org/conference/osdi24/presentation/lee)。
[^neo]: Xuanlin Jiang 等. 《NEO: Saving GPU Memory Crisis with CPU Offloading for Online LLM Inference》. MLSys，2025。[论文](https://proceedings.mlsys.org/paper_files/paper/2025/hash/66a026c0d17040889b50f0dfa650e5e0-Abstract-Conference.html)。
[^directkv]: Shutian Luo、Haiying Shen. 《No Buffer, No Bottleneck: Efficient Zero-Copy KV Cache Offloading for Long-Context LLMs》. OSDI，2026。[论文](https://www.usenix.org/conference/osdi26/presentation/luo)。
[^solid]: Xinrui Zheng 等. 《SolidAttention: Low-Latency SSD-based Serving on Memory-Constrained PCs》. FAST，2026。[论文](https://www.usenix.org/conference/fast26/presentation/zheng)。
[^echo]: Guangda Liu 等. 《ECHO: Efficient KV Cache Offloading with Lossless Prefetching for Serving Native Sparse Attention LLMs》. OSDI，2026。[论文](https://www.usenix.org/conference/osdi26/presentation/liu-guangda)。
[^hisparse]: Zhiqiang Xie 等. 《HiSparse: Scaling Sparse-Attention Decoding with Hierarchical KV Cache Management》. arXiv:2608.07009v1，2026。[论文](https://arxiv.org/abs/2608.07009v1)。
[^hisparseblog]: Zhiqiang Xie、Zhangheng Huang、Tingwei Huang. 《HiSparse: Turbocharging Sparse Attention with Hierarchical Memory》. LMSYS，2026-04-10。[博客](https://www.lmsys.org/blog/2026-04-10-sglang-hisparse/)。
[^vllm]: vLLM Team. 《GLM 5.3 Optimizations, Part 1: Hybrid HiSparse Offloading in vLLM》. 2026-09-08。[博客](https://vllm.ai/blog/2026-09-08-glm53-part1-hybrid-sparse-offloading)。
[^hicache]: Zhiqiang Xie. 《SGLang HiCache: Fast Hierarchical KV Caching with Your Favorite Storage Backends》. LMSYS，2025-09-10。[博客](https://www.lmsys.org/blog/2025-09-10-sglang-hicache/)。
[^strata]: Zhiqiang Xie 等. 《Strata: Hierarchical Context Caching for Long Context Language Model Serving》. OSDI，2026。[论文](https://www.usenix.org/conference/osdi26/presentation/xie-zhiqiang)。
[^lmcache]: Yuhan Liu、Jiayi Yao、Yihua Cheng 等. 《LMCache: An Efficient KV Cache Layer for Enterprise-Scale LLM Inference》. arXiv:2510.09665v2，2025。[报告](https://arxiv.org/abs/2510.09665v2)。
[^lmcache-docs]: LMCache. 《Architecture Overview》. [项目文档](https://docs.lmcache.ai/developer_guide/architecture.html)，访问于 2026-10-30。
[^mooncake]: Ruoyu Qin 等. 《Mooncake: Trading More Storage for Less Computation — A KVCache-centric Architecture for Serving LLM Chatbot》. FAST，2025。[论文](https://www.usenix.org/conference/fast25/presentation/qin)。
