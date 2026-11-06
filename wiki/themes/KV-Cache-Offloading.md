---
type: theme
topic: KV-Cache-Offloading
title: LLM KV Cache Offloading
theme_kind: lens
member_tag: lens/kv-cache-offloading
candidate_tags: [kv-cache, offloading, storage-hierarchy]
paper_count: 12
first_generated: 2026-10-30
last_updated: 2026-11-06
tags: [topic-overview, llm-inference, kv-cache, offloading]
---

# LLM KV Cache Offloading

KV cache 让模型在生成新 token 时复用历史计算，但它的容量随上下文和并发增长。把 KV 扩展到 CPU、SSD 或远端存储，可以腾出 GPU 显存；随之而来的问题是，下一次需要这些数据时，谁读取、搬多少、等待多久，以及还能否保持原来的计算语义。[^flexgen][^infinigen]

本专题沿着一条请求的数据路径比较不同方案：先看 GPU 显存不足时的基础卸载，再看活跃 KV 怎样参与计算，最后看分层缓存和集群存储怎样保存、定位与恢复 KV。**跨请求复用与活跃请求是使用场景，贯穿多个技术方案**；一个系统也可以同时处理多种场景。

## 纳入范围

核心论文直接研究 KV cache 在 GPU 与 CPU、SSD 或远端存储间的放置、卸载、访问或共享，并已有全文笔记。FlexGen 同时涉及权重和激活，作为异构卸载的基础纳入。专题区分完整注意力、近似选择和模型原生稀疏注意力；纯模型量化、永久删除 KV、仅研究 GPU 内分页的工作不作为核心成员。

名单按技术问题组织，并非互斥分类：同一个系统可以同时使用 CPU 计算、选择性回载和缓存管理。各论文的原始版本、评测条件和阅读状态以各自页面为准。

## 核心论文

- [[FlexGen-ICML23]]（2023）— GPU、CPU 和 SSD 上的张量放置、块调度与可选 CPU 注意力。
- [[InfiniGen-OSDI24]]（2024）— 以 GPU 上的部分 Key 预测后续层需要的 KV，从主机选择性回载。
- [[NEO-MLSys25]]（2025）— 将部分请求的 KV 与解码注意力放在 CPU，以非对称流水线和负载感知调度协同 GPU。
- [[Mooncake-FAST25]]（2025）— 集群级 KV 存储池、跨实例传输与缓存感知调度。
- [[LMCache-arXiv25]]（2025）— 位于推理引擎与多级存储之间的 KV 缓存、搬运与控制接口。
- [[SolidAttention-FAST26]]（2026）— 将完整历史放在 SSD，结合块级近似选择、预取及计算重叠。
- [[Bidaw-FAST26]]（2026）— 主存与 SSD 共同保存多轮对话状态，结合存储感知调度与替换。
- [[Strata-OSDI26]]（2026）— 为分层上下文缓存优化数据布局、碎片搬运与缓存感知调度。
- [[DirectKV-OSDI26]]（2026）— GPU kernel 直接访问主机内存中的 KV，减少显存暂存。
- [[ECHO-OSDI26]]（2026）— 对原生稀疏注意力模型按需回载 KV，并在计算前补齐准确选择。
- [[HiSparse-arXiv26]]（2026）— 主机保存历史、GPU 维护热工作集，用稀疏选择和可选精确预取管理驻留。
- [[Buoy-SIGMOD27]]（2026 年发表，SIGMOD 2027）— 利用前缀深度、准入和访问反馈管理分层缓存，兼顾总命中与 HBM 命中。

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
| [[Buoy-SIGMOD27]] | 前缀缓存的准入与跨 HBM、DRAM 替换；论文另测 SSD 扩展 | 按深度、近期访问和重访信号决定保留位置 | 分层命中率、回载开销、缓存准入和工作负载变化 |

例如 SGLang + HiCache 可以使用 Mooncake 作为外部后端；LMCache 也可以对接 Mooncake Store。**引擎缓存层与共享存储池处于不同职责位置**，但各自也有搬运和管理能力。配置上仍要确认实际版本、后端和数据路径。[^hicache][^lmcache-docs]

### HiCache 与 Strata 如何一起读

2025 年 HiCache 博客已经描述 HiRadixTree、GPU 辅助 I/O、GPU 按层／主机按页布局和外部存储预取。Strata 论文围绕同一 SGLang 分层缓存路线，进一步给出碎片传输、等待命中、批次平衡和使用 decode 填补 I/O 等待的分析与实验。两者有大量设计重合，专题将其作为一组资料；论文中的 SGLang-HiCache 基线是评测时的特定配置，不能泛化为 HiCache 所有版本。[^hicache][^strata]

### 缓存命中不等于立即可用

假设新请求需要已保存的 100K token 前缀：系统先查到位置，再等待可用带宽，把数据从存储、主机内存送到 GPU 并转换布局；如果实际加载比重算更慢，命中也不一定降低 TTFT。[[LMCache-arXiv25]] 图 15 的低带宽实验就出现过这样的交叉。并发请求同时等待同一前缀时，还要考虑重复加载、重复计算和排队。[[Strata-OSDI26]] 将加载时间和正在生成但尚未可复用的前缀纳入调度。[^lmcache][^strata]

[[Buoy-SIGMOD27]] 从替换策略补充这一问题：浅层前缀可能被多条请求路径共享，新块不必都长期留在显存。Buoy 用深度感知准入、访问位和仅保存块 ID 的幽灵队列过滤候选，将复用的块提升到 HBM。它的三层实验中，Qwen-B 上 S3-FIFO 总命中率为 52.45%、Buoy 为 51.85%，但 HBM 命中率分别为 0.65% 和 45.94%。这是模拟器在指定容量下的结果；实际 vLLM 测试顺序回放请求，不能直接推断高并发排队行为。[^buoy]

## SSD 卸载：容量、读取粒度与计算位置

这一节比较 **KV 数据确实放在 SSD 或其他非易失性磁盘**的推理方案。论文的使用时机可以是活跃请求的 decode，也可以是新请求的前缀恢复；下面先按解决 I/O 的机制组织，再标注场景和语义。同一工作可能同时采用多项机制。

### 1. 完整数据回载与多级缓存管理

最直接的路径是 `SSD → CPU 内存 → GPU`：完整数据保存在磁盘，所需内容先进入主机缓冲，再参与 GPU 计算。[[FlexGen-ICML23]] 将权重、激活和 KV 都纳入层次放置，通过批次复用摊薄 I/O；其基础路径可以使用完整注意力，而四位量化和稀疏选项会改变精度或计算语义。[^flexgen]

[[Bidaw-FAST26]] 侧重多轮交互：内存和 SSD 保存会话历史，主存命中请求与仍在读取的请求进入不同队列，调度时考虑 I/O 状态；替换还使用上一轮回答长度预测用户交互。它可以缓存中间激活并在 GPU 上恢复 KV，因此不应把所有存储对象都写成原始 KV。回答长度线索在作者用泊松时间戳构造的 ShareGPT 流量中未改善 miss rate，适合与真实交互轨迹一起理解。[^bidaw]

[[Strata-OSDI26]] 优化分层上下文缓存中的布局、数据搬运和调度，[[Buoy-SIGMOD27]] 研究准入和跨层替换，两者可和 HiCache 一起读。三者关注点分别是**缓存路径、替换策略和工程实现**，而不表示它们互不重叠。恢复完整前缀时，系统仍要比较加载与重算的成本；命中 SSD 不自动意味着 TTFT 更低。[^strata][^buoy][^hicache]

FlashGen（ASPLOS 2025）同样值得与 Bidaw 对照：作者介绍其 GPU/CPU/SSD 多级缓存和多轮请求重排。当前已核实会议录用与作者幻灯片，但没有取得论文全文，暂不将其作为已完成全文笔记的核心成员，也不把幻灯片当作论文全文。[^flashgen]

### 2. 近似选择性读取：少搬 KV，但需要衡量质量

当完整历史太大、每次回载太慢时，可以先预测重要 token 或块，再只从 SSD 读取这些数据。这会改变从完整注意力到稀疏注意力的计算，必须同时报告任务质量。**“完整 KV 在 SSD 上”只是数据保留，不代表本次 attention 使用全部历史。**

| 工作 | 如何减少 SSD I/O | 缓存语义与主要边界 |
|---|---|---|
| [[SolidAttention-FAST26]] | 将历史按块布局，基于代表向量选择块，再通过预测预取、miss 补读和任务调度重叠 SSD 与 GPU | 近似选择；补读能补齐当前**已选块**的预取 miss，不能恢复稀疏选择排除的其他块。 |
| [[IMPRESS-FAST25]] | 用少数 probe head 估计重要前缀 KV，按重要性重排磁盘块，并按频率和重要比例分层缓存 | 近似前缀恢复；未选前缀不参与后续计算。FAST 正式 PDF 已读，MinerU 及图表复核待完成。 |
| [[KVSwap-MobiSys26]] | 低秩 Key 预测重要 token 组，对齐磁盘读取，并复用相邻 decode 步的已读组 | 端侧近似选择；极紧 eMMC 预算下，RULER 个别子任务下降超过 50 个百分点。MobiSys 为 CCF B；全文复核待完成。 |
| [[KVDrive-SIGMOD26]] | 层/head 驻留、微批重叠、重要性预热及 SSD extent 联合减少活跃 KV 搬运 | 动态稀疏 decode；已读作者预印本，会议终稿及 MinerU 待核。 |
| [[Octopus-ATC26]] / Swarm | 按共同激活关系聚类，代表索引驻留 DRAM，从多个 SSD 读取选中簇 | 簇选择是近似；当前分析来自作者链接的 Swarm 预印本，Octopus 终稿待核。 |

IMPRESS 和 SolidAttention 都减少 SSD 读取，但两者选取依据不同：前者用多个 head 的选择相似性降低前缀识别成本，后者面向活跃解码的块选择和预取纠错。KVSwap 的重点是统一内存严格受限的端侧设备：低秩表示只用于预测，真实 KV 从磁盘组读取；它的滚动缓冲保存新生成数据，另一个复用缓冲保存已读历史。KVDrive 则同时考虑 GPU 驻留、主机控制和三级数据路径。[^impress][^solid][^kvswap][^kvdrive]

对比这些近似方法时，不能只固定“读取 10% KV”。块、簇和 token 的选择单位不同；同样的数据字节数可能有不同召回率和读放大。同样地，模型权重/KV 的量化、top-k 预算和任务质量也必须共同固定或分别报告。

### 3. 多 SSD：组织并发访问，而非只增加容量

Swarm 预印本的思路是将经常一起访问的条目组成簇，再把每个簇分散到多个 SSD。运行时合并被选簇、去重，并为有副本的条目挑选负载较轻的盘，最后异步批量读取到 pinned 主机缓冲，再搬到 GPU。它报告最多八盘、37.67 GB/s 的结果，但实际加速受 PCIe lane、根端口、CPU 内存带宽和队列争用约束。[^swarm]

这一方法把聚类与多盘调度结合起来，代价是离线共激活画像、重复存储和在线元数据。预印本的距离公式与一处文字解释不一致，也没有充分说明百万 token 场景中两两关系矩阵的构建成本；这些问题需要对照会议终稿和代码。作者主页将 ATC 2026 的 Octopus 链接到该 Swarm 预印本，但不能据此认定两个版本的所有结果完全一致。[^swarm][^octopus-author]

### 4. 计算型 SSD：移动计算，减少历史 KV 出盘

[[InstAttention-HPCA25]] 的作者预印本 InstInfer 采用另一条路径：将完整 KV 留在计算型 SSD 内，由设备内 attention 引擎读取 flash 计算，再把 attention 输出交给 GPU。GPU 保留投影、FFN 和 prefill，设备之间仍要交换 query、新 KV 和结果。稠密配置用于考察计算位置，SparF 进一步以近似选择减少内部读取。[^instinfer]

计算型 SSD 并非普通 NVMe 软件栈的自然功能：它依赖设备内算力、控制器、固件及特定通信路径。预印本用 OpenSSD 原型测量部分成本，并借助 NVMeVirt 建模更大配置；主实验的 1K 输入、1K 输出和大 batch 也不能直接代表今天的 128K+ 在线服务。预印本与 HPCA 会议终稿的差异尚待核验。[^instinfer]

### 5. 如何判断“无损”和实验是否可比

| 对照维度 | 必须明确的问题 |
|---|---|
| 存储完整性 | 是否将所有原始 KV 保存在 SSD？是否丢弃、量化或只保留中间激活？ |
| 计算语义 | 当前 attention 是否使用完整历史，还是只用预测 token/块/簇？如模型本来是稀疏注意力，是否保证补齐模型真正选中的所有条目？ |
| 一次 miss 的路径 | 读取是否经过 OS 页缓存、pinned host 内存和 PCIe？是否由 GPU 直达，还是由计算型 SSD 在设备内完成运算？ |
| SSD 成本 | 统计实际读取字节、I/O 大小与队列深度、后台重排/写回、设备数、拓扑、冷/热缓存与其他负载干扰。 |
| 效果与质量 | 活跃 decode 报告 token 间延迟和吞吐；前缀恢复报告 TTFT 和加载/重算交叉点；近似方案同时报告逐任务质量和最差案例。 |

例如，KVSwap 在极紧 eMMC 内存预算下 LongBench 平均只下降 2.7 个百分点，但同一设置的 RULER 个别子任务下降超过 50 个百分点；平均质量无法替代目标任务的最低质量要求。IMPRESS 把不重要前缀排除在计算外；SolidAttention 的补读只保证当前稀疏选择中的块就绪；这三种都不能归入完整注意力无损回载。[^kvswap][^impress][^solid]

## 横向对照

表中的“语义”描述论文所讨论的主要配置；它不保证不同硬件内核逐位一致。各论文有不同模型、硬件和工作负载，此表用于理解机制，不能用论文摘要的加速倍数直接排序。

| 工作 | 主要低层 | 数据与计算方式 | 语义及主要代价 |
|---|---|---|---|
| [[FlexGen-ICML23]] | CPU、SSD | 权重、激活与 KV 分层；支持 GPU 计算及部分 CPU 注意力 | 基础计算可完整；量化、稀疏选项为近似；大批次等待长 |
| [[InfiniGen-OSDI24]] | CPU | GPU 预测，再回载所选完整 KV | 相对完整注意力近似；索引占显存，主机池可能淘汰 |
| [[DirectKV-OSDI26]] | CPU | GPU kernel 直接读取主机 KV | 完整访问路径；依赖高速互连及访存布局 |
| [[NEO-MLSys25]] | CPU | 以请求为单位将 KV 和部分解码注意力留在 CPU | 不靠稀疏裁剪；CPU 带宽、调度和 Q/K/V 通信是代价 |
| [[SolidAttention-FAST26]] | SSD，经过主机缓冲 | 选择块并预取，GPU 对所选历史计算 | 相对完整注意力近似；SSD 竞争和局部性影响性能 |
| [[Bidaw-FAST26]] | CPU、SSD | 多轮状态缓存、I/O 感知调度与重要张量选择 | 恢复时可能计算中间张量；效果依赖用户交互与存储位置 |
| [[ECHO-OSDI26]] | CPU | 模型选择后召回；提前预取并在计算前补齐 | 预取保持原生稀疏模型选择；主机池与索引仍占资源 |
| [[HiSparse-arXiv26]] | CPU | 完整历史与 GPU 热缓存，未命中时召回 | 保持所测稀疏选择语义；索引和元数据另计 |
| [[Strata-OSDI26]] | CPU、外部存储 | 分层前缀匹配、GPU 辅助搬运与调度 | 精确前缀复用；加载、复制及调度等待需要核算 |
| [[LMCache-arXiv25]] | CPU、磁盘、远端 | 引擎连接器和可配置缓存后端 | 普通复用可保存完整 KV；若启用压缩需另核对精度 |
| [[Mooncake-FAST25]] | 集群内主机存储与远端 | KV 存储池、跨节点共享和传输 | 缓存位置、网络、调度及多副本占用影响收益 |
| [[Buoy-SIGMOD27]] | CPU；另评估 SSD | 按前缀结构和复用反馈准入、提升及淘汰 | 缺失部分正确重算；命中层次、写入与元数据影响性能 |

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
[^buoy]: Liang Wang 等. 《Buoy: Efficient and Effective Cache Replacement for Prefix Caching》. Proceedings of the ACM on Management of Data，2026；SIGMOD 2027 录用。[论文](https://doi.org/10.1145/3837101)。
[^bidaw]: Shipeng Hu 等. 《Bidaw: Enhancing Key-Value Caching for Interactive LLM Serving via Bidirectional Computation–Storage Awareness》. FAST，2026。[论文](https://www.usenix.org/conference/fast26/presentation/hu-shipeng)。
[^flashgen]: Jinwoo Jeong、Jeongseob Ahn. 《Accelerating LLM Serving for Multi-turn Dialogues with Efficient Resource Management》. ASPLOS，2025。[论文记录](https://doi.org/10.1145/3676641.3716245)；[作者报告幻灯片](https://jeongseob.github.io/assets/talks/jeong_asplos2025_talk.pdf)。
[^impress]: Weijian Chen 等. 《IMPRESS: An Importance-Informed Multi-Tier Prefix KV Storage System for Large Language Model Inference》. FAST，2025。[论文](https://www.usenix.org/conference/fast25/presentation/chen-weijian-impress)。
[^kvswap]: Huawei Zhang、Chunwei Xia、Zheng Wang. 《KVSwap: Disk-aware KV Cache Offloading for Long-Context On-device Inference》. MobiSys，2026。[论文](https://summerspringwei.github.io/chunweixia.github.io/publication/mobisys26/mobisys26.pdf)。
[^kvdrive]: Jian Lin 等. 《KVDrive: A Holistic Multi-Tier KV Cache Management System for Long-Context LLM Inference》. arXiv:2605.18071v1；SIGMOD 2026 录用。[预印本](https://arxiv.org/abs/2605.18071v1)；[会议录用名单](https://2026.sigmod.org/sigmod_papers.shtml)。
[^swarm]: Tuowei Wang 等. 《Swarm: Co-Activation Aware KVCache Offloading Across Multiple SSDs》. arXiv:2603.17803v1，2026。[预印本](https://arxiv.org/abs/2603.17803v1)。
[^octopus-author]: Tuowei Wang. [作者主页，Octopus 条目链接 Swarm 预印本](https://pairshoe.github.io/)；ATC 2026 [会议日程](https://atc.sigops.org/2026/schedule.html)。
[^instinfer]: Xiurui Pan 等. 《InstInfer: In-Storage Attention Offloading for Cost-Effective Long-Context LLM Inference》. arXiv:2409.04992v1；会议题名 InstAttention，HPCA 2025。[预印本](https://arxiv.org/abs/2409.04992v1)；[会议记录](https://dblp.org/rec/conf/hpca/PanLLLSZLWZ25.html)。
