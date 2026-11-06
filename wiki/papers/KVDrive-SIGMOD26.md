---
type: paper
name: KVDrive
full_title: "KVDrive: A Holistic Multi-Tier KV Cache Management System for Long-Context LLM Inference"
authors: [Jian Lin, Jiazhi Mi, Zicong Hong, Haodong Wang, Qianli Liu, Haoyue Zhang, Peng Li, Song Guo]
venue: SIGMOD
year: 2026
tags: [llm-inference, kv-cache, ssd-offloading, sparse-attention, tiered-storage, scheduling]
source_pdf: "[[arxiv26-lin-kvdrive.pdf]]"
source_md: ""
review_status: needs-review
evidence_level: full-text
last_reviewed: 2026-11-06
---

# KVDrive：协同 GPU、内存与 SSD 的活跃 KV 管理（SIGMOD 2026）

> **原题**：KVDrive: A Holistic Multi-Tier KV Cache Management System for Long-Context LLM Inference

> **一句话总结**：KVDrive 将活跃请求的完整 KV 放入 GPU、CPU 和 SSD 分层，联合优化稀疏选择、显存驻留、微批流水线和 SSD 布局；预印本在所测长上下文系统上报告比既有方案最多高 1.74 倍吞吐，但其动态稀疏注意力与完整注意力并非等价。[^paper]

**版本与阅读状态**：SIGMOD 2026 官方录用清单列有同题论文及作者；本页阅读的是 arXiv:2605.18071v1 预印本，而非已核对的会议终稿。全文和部分原图已核查，MinerU 转换及与终稿的差异仍待复核。[^venue][^paper]

## 问题与动机

长上下文或大批次让活跃请求的 KV 同时超过显存和主机内存时，系统只能把完整历史再下沉到 SSD。每次解码既要判断重要 KV，又要处理 GPU 缓存未命中、CPU 控制、磁盘读取以及 GPU 计算；仅提高稀疏率可能损害质量，仅增加一层存储也可能把瓶颈从显存转移到 I/O。KVDrive 把这些问题放在同一个数据路径里分析（§1–3）。[^paper]

它研究的是**活跃请求**每个 decode step 的历史访问；同一 SSD 也能被用于保存可复用前缀，但那不是本论文主问题。其默认路径使用动态稀疏注意力，必须与保存完整历史这一存储属性分开：磁盘上有全量 KV，不意味着每次 attention 都使用它们。[^paper]

## 关键观察 / 隐含假设

- **观察 1：相邻生成步的重要 KV 有重叠。** 图 7 比较当前 top-M 与下一步 top-K，相交数量先随 M 快速上升再趋于饱和；因此当前高分项具有再次使用的可能（§5.1）。[^paper]
  - **依赖假设**：访问局部性在新的上下文、模型、层和请求阶段仍足够稳定。若关键位置突然转移，按当前分数驱逐可能提高 miss 与等待。
- **观察 2：不同 layer/head 扩大驻留窗口的收益不同。** 图 8 的层、head 对照显示，给部分 head 更多 GPU 空间可比均匀分配节省更多传输（§5.2）。[^paper]
  - **依赖假设**：离线画像能代表在线输入及当前 GPU 预算，且精细分配、查表和重新画像的成本没有抵消收益。
- **观察 3：选择、搬运与计算占用不同资源。** GPU 索引扫描、CPU 命中检查、主机到 GPU 传输及 FFN 计算若顺序执行会相互等待，论文据此拆成不同阶段（§6.1、图 10–11）。[^paper]
  - **依赖假设**：存在足够 micro-batch 供重叠，且 CPU、GPU、PCIe 和磁盘之间的竞争不会抵消重叠。低并发或短请求下启动、同步和流水线气泡的比例可能更高。
- **观察 4：SSD 读局部性与 KV 物理布局有关。** 预印本将语义接近或共同访问的 KV 打包为 extent，并按 layer/head 分区，以较大、可合并的请求替代散乱小读（§7.2–7.3、图 12）。[^paper]
  - **依赖假设**：共同访问关系在当前 workload 有效，布局构建和读入同 extent 中无关 KV 的开销能摊销。顺序读带宽不代表每个动态稀疏请求都能达到该带宽。
- **观察 5：当前高注意力不总是更好的驱逐依据。** 表 3 中，lookahead 相比 LRU 在多数 Llama/Qwen 配置提升命中，但 Phi-4 上 Quest 和 ShadowKV 分别下降 1.8 和 1.5 个百分点（§9.3）。[^paper]
  - **依赖假设**：当前重要性对下一步有预测力，且提升的命中足以抵扣分数维护与查找开销。不同模型可能出现反例，不能无条件称其优于 LRU。

## 核心方法

### 1. 稀疏索引与驻留窗口

prefill 完成后系统构造分层索引：先对相邻 KV 做 chunk，再以代表 Key 构成更高层索引，以降低筛选时扫描的数据量。每一步先查找可能重要的历史，确定 GPU 内哪些命中，未命中部分再从 DRAM/SSD 回载并参与 attention（§4、图 6）。[^paper]

GPU 缓存保留近期重要 KV 的滑动工作集；lookahead 驱逐倾向淘汰当前注意力分数较低的项。它预测的是后续**缓存复用**，不是能保证完整注意力结果的预知。与 [[ECHO-OSDI26]] 的最终 exact top-k 未命中补齐不同，本论文使用的动态稀疏选择本身相对 dense attention 是近似的。[^paper]

### 2. 层与 head 的二维窗口分配

对每个 layer/head 和候选窗口大小，离线画像其内存成本与减少的传输量；给定总显存预算 `M`，最大化所有层和 head 的传输收益之和，并约束窗口成本总和不超过 `M`。作者将其视为多重选择背包问题：小模型可穷举，较大模型贪心地按增量收益/成本扩窗（§5.2）。[^paper]

这一设计比均匀窗口更细，但画像对应特定模型和 workload，预印本的“接近最优”是其算法和实验结论，不能据此推断在线全局最优。实际 GPU 空间还需扣除模型、激活、索引及缓冲。[^paper]

### 3. SFC：选择、获取、计算拆分

Selection（选择）和 Fetching（获取）按 micro-batch 交错，GPU 扫描当前 micro-batch 索引时，CPU 可检查前一批缓存命中，并为更早一批发起搬运；整个 batch 的所需 KV 就绪后执行计算。缓存元数据更新与计算重叠（§6.1）。[^paper]

索引 centroid 数影响选择质量和扫描成本；GPU 缓存越大，回载可能变少，但 CPU 命中检查可能更慢；micro-batch 太大会产生流水线空洞，太小又增加 launch 和同步。作者在运行前做短时间校准，选择适合当前配置的参数（§6.2）。这不是严格承诺所有阶段始终没有等待。[^paper]

### 4. SSD 层：预热、布局与分级搬运

prefill 末尾用最后约 16–64 个 token 的注意力估计重要性：完整 KV 先保存在 SSD，高分项装入 HBM，后续高分项放 DRAM。这个 warm-up 是初始**放置**；后续动态稀疏选择仍可能丢掉完整注意力本会使用的 KV（§7.1）。[^paper]

磁盘组织采用语义连续的 extent 与 layer/head 分区。读取时只取被选择的块，合并为较大的 extent 访问；系统用 `numpy.memmap` 访问磁盘/页缓存，再用预分配 pinned buffer 暂存并异步送往 GPU，同时在 memmap 缓存和 pinned 空间之间分配主机内存（§7.2–8、图 12）。[^paper]

因此这不是“SSD 直接作为显存”的零成本扩展：至少有 SSD→DRAM、DRAM 页缓存或 pinned buffer→GPU 两段路径，缺页、队列深度、额外复制和其他进程争用都会影响效果。[^paper]

## 设计取舍

- **模型质量换 I/O**：稀疏选取降低回载量，但保留全部磁盘副本不保证本次计算访问完整历史。
- **驻留空间换控制成本**：较大 GPU 工作集可减少传输，却增加 GPU 元数据、CPU 命中检查与潜在同步。
- **画像换适配性**：二维分配和 warm-up 利用过去或前缀尾部的信号；任务转移时可能需要重新调优。
- **布局换写入与读放大**：语义 extent 提高顺序性，构建和更新布局需空间与写入，跨范围访问可能读入无关数据。
- **流水线吞吐换尾部风险**：微批重叠依赖并发和足够空闲资源，SSD 干扰、磁盘缺页及长尾批次可能让收益变化。

## 实验设置

预印本测试 Llama-3-8B-1048K、Qwen3-8B/14B 和 Phi-4-mini，任务使用 RULER 与 LongBench。三种单机环境分别为 L20 48 GB/100 GB 主存、H20 96 GB/200 GB 主存、RTX 4090 24 GB/120 GB 主存，磁盘为 NVMe U.2 SSD（§9.1）。[^paper]

对照有完整 GPU 驻留、FlexGen、Quest、ShadowKV、PQCache、MagicPIG 以及两种 RetroInfer 配置。作者称所有基线重新实现并纳入统一框架；除完整计算和 FlexGen 外，基线采用精确 prefill 加动态稀疏 decode，并固定 sink 与 local token。统一实现有利于控制配置，也可能与原项目最新实现、调优方式不同。[^paper]

## 实验与结果

| 结果 | 条件与限制 |
|---|---|
| 预印本摘要报告最高 1.74 倍吞吐 | 这是不同场景中的最高值；图 13 的 L20 场景相对 ShadowKV 最高约提高 70%，模型、上下文和 batch 需逐项查看。 |
| H20 与 RTX 4090 的吞吐为最佳对照的 1.23–1.53 倍 | 图 14；不能将不同论文、不同硬件的倍数直接排序。 |
| 动态稀疏质量有下降 | 表 2 的 Qwen3-8B：RULER 平均 Full-KV 为 73.94、KVDrive 为 68.07；Llama-3-8B 为 77.74 与 76.15。平均接近不等于逐任务不变。 |
| lookahead 不是所有模型都提升命中 | 表 3：Phi-4 的 Quest、ShadowKV 分别较 LRU 低 1.8、1.5 个百分点；KVDrive 自身从 78.0% 升到 81.0%。 |
| 不同窗口存在拐点 | 图 16 扫描窗口 2、3、4；较大窗口可减少 I/O，但可能增加 lookup 时间。 |
| centroid 数随上下文变化 | 图 18 在固定 1.56% 稀疏预算下比较 2048、4096、8192；论文报告 60K 更适合 4096、120K 更适合 8192。 |
| 容量限制影响基线 | MagicPIG 在部分配置初始化 OOM；Full-KV 也会因显存不足 OOM，FlexGen 的全量搬运在所测场景低于 1 token/s。 |

## 论断—证据表

| 论断 | 证据 | 边界 | 判断 |
|---|---|---|---|
| 显存驻留与流水线能减少等待 | 图 15–18、表 3 的单项实验 | 不同优化相互影响，主吞吐不能全部归因某一项 | 所测配置有支持 |
| SSD 可扩展总 KV 容量 | §7 的完整后备层和图 12 的搬运设计 | 实际 I/O、写入、页缓存及 cold start 成本随配置变化 | 机制成立，成本需另测 |
| 保持模型准确率 | 表 2 同时给出多个模型和任务 | Qwen RULER 平均较 Full-KV 下降，个别任务差异更大 | 只能称测试中部分指标接近 |
| lookahead 普遍优于 LRU | 表 3 的多数正收益 | Phi-4 存在负收益 | 不是普遍结论 |

## 批判性分析

KVDrive 有三条相互耦合的贡献链：驻留减少 CPU–GPU 搬运，SFC 尝试隐藏剩余等待，SSD 布局和分级暂存减少更慢层的 I/O。读图时需确认某个消融到底测试 DRAM–GPU 还是确实触发 SSD；仅凭完整系统较快，不能推定 SSD 布局单项贡献与总加速相同。[^paper]

预印本一方面把完整历史保存在 SSD，另一方面对 decode 用动态稀疏 attention。存储完整性与注意力完整性是两个问题。表 2 的 Qwen3 RULER 平均下降约 5.87 个百分点，足以说明“保持准确率”不能理解为无损或每个任务均无变化。比较无损系统时，应固定语义、精度和需要访问的 token 集合。[^paper]

另一个边界是基线均由作者统一框架重实现。公平性取决于具体实现与参数是否得到各自适当调优，而不只取决于模型和硬件相同。需要额外测量冷缓存、稳态、后台 SSD I/O、CPU 争用、预热时间、页缓存大小和在线 P95/P99，以判断容量收益能否在真实部署中保留。

## 可继续研究的问题

1. 在相同精度预算下，SSD 的选择性读取、全量读取和直接重算在哪些上下文长度交叉？
2. 离线的层/head 分配遇到任务突变时，多久重画像才值得其成本？
3. 是否可以把 SSD、主机内存和 GPU 的带宽争用纳入同一调度器，并对尾延迟设约束？
4. 当模型原生提供确定的稀疏选择时，能否将最终选中项补齐并分别评估预测误差与缓存 miss？

## 参考资料

[^paper]: Jian Lin 等. “KVDrive: A Holistic Multi-Tier KV Cache Management System for Long-Context LLM Inference.” arXiv:2605.18071v1，2026。[作者预印本](https://arxiv.org/abs/2605.18071v1)。
[^venue]: SIGMOD 2026. [正式录用论文名单](https://2026.sigmod.org/sigmod_papers.shtml)。
