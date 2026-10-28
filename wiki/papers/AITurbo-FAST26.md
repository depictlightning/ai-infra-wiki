---
type: paper
name: AITurbo
full_title: "Fast Cloud Storage for AI Jobs via Grouped I/O API with Transparent Read/Write Optimizations"
authors: [Yingyi Hao, Ting Yao, Xingda Wei, Dingyan Zhang, Tianle Sun, et al.]
venue: FAST
year: 2026
tags: [ai-storage, llm-training, llm-serving, checkpointing, kv-cache, rdma, area/llm-training]
source_pdf: "[[fast2026-hao.pdf]]"
source_md: "[[fast2026-hao]]"
review_status: complete
evidence_level: full-text
last_reviewed: 2026-10-28
---

# AITurbo：面向 AI 训练和推理的分组云存储 I/O（FAST 2026）

> **原题**：Fast Cloud Storage for AI Jobs via Grouped I/O API with Transparent Read/Write Optimizations

> **一句话总结**：训练检查点存在跨设备重复写入，模型扩容也可能让多个实例读取同一权重；AITurbo 让存储层利用分组信息安排去重和传输，在一组有重复写入的训练配置下，检查点耗时最高比作者复现的 GEMINI 低 5.9 倍（§5.1、图 13）。[^paper]

**阅读状态**：已阅读会议正式版的 MinerU 全文，并核对系统结构、检查点、模型读取、KV 读取和消融图；不同场景的缓存及接口条件分别记录。[^paper]

## 问题与动机

训练检查点、扩容时的模型权重加载和推理 KV cache 读取都可能形成大体量 I/O。云环境的存储节点与计算节点分离，瓶颈可能来自存储后端带宽，也可能来自计算节点的存储网卡；单纯提升其中一侧不一定有效（§2）。[^paper]

作者观察到多个设备常在相近时间访问相同或部分相同的数据。若应用逐设备调用普通 I/O，存储层难以识别跨设备重复和整体传输机会；因此提出 grouped I/O 接口（§2–3）。[^paper]

## 关键观察 / 隐含假设

- **重复取决于并行策略**：数据并行副本可以重复保存参数，但 ZeRO 等切分策略会改变重复关系；因此不能假设所有检查点都可按同一比例去重（§2.3、表 1）。[^paper]
- **计算互连与主机内存有可用余量**：系统用它们暂存和转发数据；若计算通信或其他租户占满资源，额外 I/O 可能造成争用（§3–4）。[^paper]
- **分组信息需要应用提供**：参与节点和操作需要能被协调。论文明确指出 Mooncake 的 KV cache 实验没有使用 grouped API，因为协调运行中的推理实例具有挑战（§5.4）。[^paper]

## 核心方法

应用使用分组读写接口描述参与者。写入返回两个完成事件：数据到达主机 DRAM 和数据持久化到后端存储。训练是否可以在前者之后继续，取决于应用所需的故障与持久性保证；二者不能视作同一个完成时刻（§3）。[^paper]

写路径根据分块校验和识别重复数据，生成负载均衡计划，经计算互连写入主机内存，必要时再写至存储。读取时先从存储获取数据，再使用计算互连广播到多个参与者；数据规模较大时将各阶段流水化（§4.1–4.2）。[^paper]

控制器对传输时间设定边界，将固定时间下的带宽分配转为线性可行性问题，再逐步收紧时间范围。重复检查点可复用计划；训练初期优化器状态可能为零，重复关系尚未稳定，因此不能无条件复用早期计划（§4.1）。[^paper]

论文的 Mooncake 实验复用 AITurbo 的存储和计算互连路径，但没有调用 grouped API；推理端收益不能作为该接口已解决在线推理分组协调的证据（§5.3–5.4）。[^paper]

## 设计取舍

- **少写重复数据换协调与校验成本**：去重和计划可被缓存，但配置变化需要重新评估。
- **内存暂存换持久性边界**：DRAM 中的检查点可用于某些恢复场景，节点和故障域同时失效时仍需后端持久副本。
- **复用计算互连换共享资源干扰**：模型通信、检查点和 KV 传输可能竞争同一链路与主机内存。

## 实验与结果

评测使用 Ascend 910B 或 NVIDIA A800 集群，最多 64 个 XPU；每节点 8 个 XPU、1.5 TB 主机 DRAM，后端存储带宽最多 30 GB/s。检查点比较中的 GEMINI 是作者复现的主要技术，而非原始闭源实现（§5、§5.1）。[^paper]

| 结果 | 具体条件 |
|---|---|
| 检查点写入最高比 GEMINI 快 5.9 倍 | 具有跨节点重复写入的配置；比较作者复现的 GEMINI，非全部并行策略（§5.1、图 13）。 |
| XPU 浪费时间最多减少 6% | 对照 SFSTURBO，按设定故障率和最佳检查点频率计算；不是训练吞吐提高 58 倍（§5.1、图 16）。 |
| 平均 TTFT 降低 23% | Qwen-14B、8 XPU、30 分钟 Qwen-Bailian 轨迹，Mooncake+AITurbo 对比 Mooncake；未使用 grouped API（§5.3–5.4、图 18）。 |
| 分组协调开销最多 45 ms | 论文在最多 64 XPU 的检查点测试中测量；可复用计划的情况能跳过部分协调（§5.4、图 20）。 |

协调开销不能等同首次求解计划的完整成本：论文另报一个 38B、64 XPU 示例中，未经优化的单线程 Python 求解器用约 4 秒生成计划，之后可在重复模式稳定时缓存复用（§4.1）。[^paper]

## 论断—证据表

| 论断 | 证据 | 评测边界 | 置信度 |
|---|---|---|---|
| 跨设备分组和去重可改善训练检查点 I/O | §4.1、§5.1、图 13 | 取决于副本重复、互连与主存余量；基线含复现 | 中高 |
| 计算互连可改善所测推理 KV 读取 | §5.3–5.4、图 18 | Qwen-14B 和单条 30 分钟轨迹；不使用 grouped API | 中高 |
| 提前完成内存暂存不等于持久化完成 | §3 | 持久性由应用是否等待第二个事件决定 | 强 |

## 批判性分析

### 论证链条

分组 API 使存储系统看见跨设备冗余，测试再区分检查点写、权重读和 KV 读。论文中的三个场景采用的接口和缓存条件不同，不能把任一最佳数字解释为整体加速。[^paper]

### 假设压力测试

如果训练通信和 I/O 同时占满计算互连，额外传输可能挤占计算；短请求和在线推理实例难以对齐分组，论文的 Mooncake 适配也反映了这一限制。[^paper]

### 实验可信度

实验覆盖两种加速器和多种并行配置，但 GEMINI 基线是复现版本，故障造成的浪费由模型计算；推理结果来自特定轨迹和部署规模。[^paper]

### 系统性缺陷

接口需要应用配合，计划缓存和分组成员变化需要管理。并发租户的隔离、共享互连拥塞和 DRAM 机会成本应继续测量。[^paper]

## 局限与后续工作

- 在模型通信饱和时加入检查点与 KV 读，分别测量训练步时、网络排队与请求 P99。
- 注入计算节点、暂存节点及后端存储故障，区分等待两个完成事件时的可恢复状态。

## 相关

- 同会议：[[FAST-2026]]、[[AdaCheck-FAST26]]。
- 相邻研究：[[DualPath-SIGCOMM26]]、[[TensorHub-SOSP26]]。

## 参考文献

[^paper]: Yingyi Hao 等. 《Fast Cloud Storage for AI Jobs via Grouped I/O API with Transparent Read/Write Optimizations》. FAST，2026。[原文](https://www.usenix.org/conference/fast26/presentation/hao)。
