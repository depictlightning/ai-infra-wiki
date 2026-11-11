---
type: theme
topic: KV-Cache-Offloading
title: LLM KV Cache Offloading
theme_kind: lens
member_tag: lens/kv-cache-offloading
candidate_tags: [kv-cache, offloading, storage-hierarchy]
paper_count: 8
first_generated: 2026-10-30
last_updated: 2026-11-11
tags: [topic-overview, llm-inference, kv-cache, offloading]
---
# LLM KV Cache Offloading

KV cache 可以从 GPU 显存扩展到 CPU 内存、SSD 或远端存储，以支持更长的上下文和更多并发请求。数据离开显存后，系统还需决定如何管理、访问和调度，才能让容量收益不被搬运和等待抵消。

本专题按五个技术问题组织。同一篇论文可以出现在多个栏目，各处只讨论它在该问题上的方法。

## 核心论文

- [[Strata-OSDI26]]（OSDI 2026）研究长上下文前缀缓存，其两项主要贡献是高效的分层 KV I/O 和缓存感知请求调度。[^strata]
- [[LMCache-arXiv25]]（2025，技术报告）提供独立的 KV 缓存层、引擎连接器和控制接口，并优化跨层搬运。[^lmcache]
- [[SolidAttention-FAST26]]（FAST 2026）围绕 SSD 卸载设计交错布局、推测预取和 I/O 调度。[^solid]
- [[Bidaw-FAST26]]（FAST 2026）结合交互行为辅助缓存替换与 I/O 感知请求调度。[^bidaw]
- [[Mooncake-FAST25]]（FAST 2025）结合分布式 KV 池、跨节点传输与缓存感知调度。[^mooncake]
- [[IMPRESS-FAST25]]（FAST 2025）结合重要 token 识别、磁盘重排和分层缓存管理。[^impress]
- [[CachedAttention-ATC24]]（ATC 2024）保存多轮会话的历史 KV，结合队列感知的分层放置与异步搬运减少重算。[^cachedattention]
- [[Tutti-arXiv26]]（2026，预印本）让 GPU 主导 SSD KV 访问，并协调数据搬运与推理计算。[^tutti]

## 分层存储：存储侧如何管理

**[[Strata-OSDI26]]**：将可复用的前缀 KV 保存在 GPU 显存、CPU 内存和可选的外部存储中；提供 write-through、selective write-through 和 write-back 三种备份策略，各层默认使用 LRU 淘汰（§4.4）。[^strata]

**[[LMCache-arXiv25]]**：在推理引擎与 CPU 内存、磁盘或远端后端之间提供**独立的 KV 缓存层**，连接器负责与引擎交换缓存位置及加载、保存请求；集中控制器汇总各实例的缓存状态，并提供查询、迁移、固定和清理等接口，供外部组件管理跨实例缓存（§4、§6–7）。[^lmcache]

**[[Bidaw-FAST26]]**：**回答长度辅助替换**根据上一轮回答长度估计下一次访问的加权复用距离下界，结合用户历史访问分布和后台模拟得到的命中潜力，选择主存中应降级到 SSD 的历史状态；该线索依赖用户交互行为。**历史张量选择**比较 KV 与中间激活的存储成本，选择更节省空间的表示；保存中间激活时，恢复 KV 需要额外计算（§3.3、§4）。[^bidaw]

**[[Mooncake-FAST25]]**：**分布式 KV 池**汇集集群节点的 CPU 内存和 SSD 保存缓存；KV 被组织为块，块键同时包含当前内容和前缀信息，用于定位和去重。**副本与淘汰**允许热点块在多个节点保留副本，以分散访问压力；容量不足时使用 LRU 淘汰未被正在运行的请求使用的块。副本可以缓解热点，但也会占用存储容量（§3.2）。[^mooncake]

**[[IMPRESS-FAST25]]**：**三层驻留与替换**在 SSD 保存完整前缀 KV 副本，GPU 和 CPU 缓存尽量互斥；系统结合块的访问频率与其中重要 KV 的比例计算分数，决定哪些块留在更快的层。这个策略依赖历史访问和重要性能反映后续请求，查询关注点变化时可能失准（§4.1、§4.4.2）。[^impress]

**[[CachedAttention-ATC24]]**：**队列感知的分层放置**将非活跃会话的 KV 保存在 CPU 内存和 SSD，参考等待队列保护即将使用的会话，并将较晚使用的会话降级到 SSD；主存预留缓冲以接收磁盘回载，迁移和淘汰以会话为单位。它利用已到达请求的队列信息管理缓存，并不能预知尚未到达的请求（§3.3）。[^cachedattention]

## 选择性读取

**[[IMPRESS-FAST25]]**：**少量 head 探测**先读取少数 probe head 的 Key，分别计算重要 token，再检查它们的选择是否相似；相似时用探测结果近似其他 head 的选择，否则回退到常规识别。系统只加载被选中的前缀 KV，未选中的前缀不参与后续计算，因此相对于完整前缀注意力是近似的（§4.2–4.3）。[^impress]

## 改变访问和计算位置

**[[Tutti-arXiv26]]**：**GPU 主导 SSD 访问**将前缀 KV 保存在 NVMe SSD，GPU 利用预先建立的地址映射直接发起数据传输，减少逐次 I/O 对 CPU 的依赖。CPU 仍负责缓存索引、映射管理和提前提交 I/O 内核，因此 GPU 主导指关键 I/O 路径，并非整个系统完全脱离 CPU（§3.1–3.2）。[^tutti]

**[[LMCache-Phoenix]]**：**SSD 直达 GPU**将 Phoenix 接入 LMCache 的 GDS L1 后端，使 SSD 通过 DMA 直接访问 GPU 显存，避免 KV 数据先进入主机中转缓冲。CPU 仍负责发起 I/O；GPU 上的小型暂存区与前缀缓存之间仍有一次设备内复制（原文 §3.1）。[^phoenix]

## I/O 效率与延迟隐藏

**[[Strata-OSDI26]]**：针对前缀匹配中碎片化小页难以高效传输的问题，使用 **GPU 内核并行搬运**，并转换 GPU （layer first)、CPU(page first) 和外部存储按页的布局，兼顾细粒度匹配与传输效率；系统还将主存回载与逐层计算重叠，并在请求排队期间预取外部存储数据(best_effort、wait complete、timeout)。（§4.2–4.2.1）。[^strata]

**[[LMCache-arXiv25]]**：将分散的 GPU 缓存页聚合成较大的连续传输块，加载时再拆回引擎页，以减少小粒度 I/O 的开销；按层流水线在计算当前层时加载下一层，也可利用排队时间预取慢层缓存。批量搬运需要暂存与布局转换，异步传输不保证等待都能被隐藏（§5）。[^lmcache]

**[[SolidAttention-FAST26]]**：**交错 KV 布局**将同一 token 的 K、V 交错存放，并预先拼接投影权重，使新 KV 直接按该布局生成，在不增大块选择粒度的情况下扩大 SSD 传输单元。**推测预取**根据同一层上一生成步骤的块选择提前读取；当前选择确定后补读遗漏块，用错误预取的空间容纳新块。**SSD 感知调度**将投影、块选择、读取、注意力和写回组织成有依赖关系的 DAG，依据任务的最晚开始时间识别关键路径，优先执行关键任务并尽早发起 I/O；非关键任务复用必要的同步点，减少额外同步。预取纠错只保证当前稀疏选择所需的块就绪，块选择相对于完整注意力仍是近似的（§4–6）。[^solid]

**[[Mooncake-FAST25]]**：**拓扑感知传输**根据内存和网卡的位置选择传输路径，并将传输任务分散到多张网卡；**分层流水线**在 prefill 过程中按块、按层发送 KV，让阶段间传输与后续计算重叠。跨节点读取仍受网络带宽和拥塞影响（§3.1–3.2）。[^mooncake]

**[[IMPRESS-FAST25]]**：**重要性重排**根据历史访问中的 token 重要性，将重要 KV 尽量集中到磁盘块中，提高读取块内有用数据的比例，减少读放大。重排限制在同一个前缀树节点内，并维护位置映射，以保留逻辑前缀关系；代价是后台重排、磁盘写入和映射维护（§4.4.1）。[^impress]

**[[CachedAttention-ATC24]]**：**队列预取**在请求等待期间，将其命中 SSD 的会话 KV 提前读入主存；**逐层预加载**在 GPU 计算当前层时加载后续层 KV，并使用显存读缓冲提前发起搬运；**异步保存**按层将新 KV 写回主存，显存写缓冲容纳本轮结束时尚未完成的写入。预取不改变本次需要加载的历史 KV，但能否隐藏等待取决于队列提前量、搬运速度和缓冲空间（§3.2–3.3）。[^cachedattention]

**[[Tutti-arXiv26]]**：**对象组织与空闲窗口调度**将引擎的 KV 块映射为 SSD 对象，按层批量组织传输；**GPU io_uring** 将 I/O 命令提交与完成处理交给 GPU，并划分 GPU 计算和 I/O 资源；**空闲窗口感知调度**根据离线画像优先安排前缀读取，将写入推迟到合适时机，减少 SSD 读写争用和 I/O 内核对推理计算的干扰。重叠效果依赖画像与实际负载相符，且仍受 GPU 资源和 SSD 带宽限制（§3.1–3.3）。[^tutti]

**[[LMCache-Phoenix]]**：**Linux I/O 栈与批量提交**让 GPU 内存作为文件 I/O 缓冲，复用文件系统、块层和 `io_uring`；**流内顺序**将 CPU 发起的 DMA 与 GPU 内核放入同一 stream 的依赖顺序，协调 KV 的加载和保存。直达路径能否发挥作用仍取决于硬件、拓扑和实际 I/O 负载（原文 §3）。[^phoenix]

## 缓存感知请求调度

**[[Mooncake-FAST25]]**：**全局缓存感知调度**选择 prefill 实例时，同时估计已有前缀命中、远端加载、剩余计算和排队成本，而非只选择本地前缀最长的节点；必要时通过热点迁移和副本扩散缓解访问拥塞。调度器还需结合延迟目标选择 decode 实例，跨节点读取是否划算取决于实际传输成本（§4）。[^mooncake]

**[[Bidaw-FAST26]]**：**双队列调度**按历史状态位置区分主存中已就绪的请求和等待 SSD 加载的请求；准备队列结合数据大小与等待时间安排磁盘读取，加载完成后按原始到达时间进入就绪队列，减少慢 I/O 对已就绪请求的阻塞（§3.2）。[^bidaw]

**[[LMCache-arXiv25]]**：其控制器向外部路由器提供缓存位置和命中长度，以支持缓存感知路由；连接器也向引擎调度器报告可复用前缀，并协调异步加载。这些接口提供调度依据，具体路由与请求调度仍由外部组件或引擎负责（§6–7）。[^lmcache]

**[[Strata-OSDI26]]**：当共享前缀仍在排队或计算时，推迟后续匹配请求，减少延迟命中造成的重复 prefill；依据请求的加载与计算量平衡组批，并在批次仍受加载限制时调度 decode，填充 I/O 空档。推迟请求也可能增加个体等待时间（§4.3）。[^strata]

## 相关博客

- [[SGLang-HiCache]]
- [[SGLang-HiSparse]]
- [[vLLM-Hybrid-HiSparse]]
- [[LMCache-Phoenix]]

## 参考文献

[^strata]: Zhiqiang Xie 等. “Strata: Hierarchical Context Caching for Long Context Language Model Serving.” OSDI 2026. [论文](https://www.usenix.org/conference/osdi26/presentation/xie-zhiqiang)。
    
[^lmcache]: Yuhan Liu 等. “LMCache: An Efficient KV Cache Layer for Enterprise-Scale LLM Inference.” arXiv:2510.09665v2，2025。[论文](https://arxiv.org/abs/2510.09665v2)。
    
[^solid]: Xinrui Zheng 等. “SolidAttention: Low-Latency SSD-based Serving on Memory-Constrained PCs.” FAST 2026。[论文](https://www.usenix.org/conference/fast26/presentation/zheng)。
[^bidaw]: Shipeng Hu 等. “Bidaw: Enhancing Key-Value Caching for Interactive LLM Serving via Bidirectional Computation–Storage Awareness.” FAST 2026。[论文](https://www.usenix.org/conference/fast26/presentation/hu-shipeng)。
[^mooncake]: Ruoyu Qin 等. “Mooncake: Trading More Storage for Less Computation — A KVCache-centric Architecture for Serving LLM Chatbot.” FAST 2025。[论文](https://www.usenix.org/conference/fast25/presentation/qin)。
[^impress]: Weijian Chen 等. “IMPRESS: An Importance-Informed Multi-Tier Prefix KV Storage System for Large Language Model Inference.” FAST 2025。[论文](https://www.usenix.org/conference/fast25/presentation/chen-weijian-impress)。
[^cachedattention]: Bin Gao 等. “Cost-Efficient Large Language Model Serving for Multi-turn Conversations with CachedAttention.” USENIX ATC 2024。[论文](https://www.usenix.org/conference/atc24/presentation/gao-bin-cost)。
[^tutti]: Shi Qiu 等. “Tutti: Making SSD-Backed KV Cache Practical for Long-Context LLM Serving.” arXiv:2605.03375v1，2026。[论文](https://arxiv.org/abs/2605.03375)。
[^phoenix]: Shi Qiu. “LMCache & xPU-IO: LMCache Meets GPU Direct Storage.” LMCache Blog，2026-11-09。[原文](https://blog.lmcache.ai/en/2026/11/09/lmcache-xpu-io-lmcache-meets-gpu-direct-storage/)。
