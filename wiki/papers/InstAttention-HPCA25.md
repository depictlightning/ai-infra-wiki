---
type: paper
name: InstAttention
full_title: "InstAttention: In-Storage Attention Offloading for Cost-Effective Long-Context LLM Inference"
authors: [Xiurui Pan, Endian Li, Qiao Li, Shengwen Liang, Yizhou Shan, Ke Zhou, Yingwei Luo, Xiaolin Wang, Jie Zhang]
venue: HPCA
year: 2025
tags: [llm-inference, kv-cache, ssd-offloading, computational-storage, sparse-attention]
source_pdf: "[[arxiv24-pan-instinfer.pdf]]"
source_md: ""
review_status: needs-review
evidence_level: full-text
last_reviewed: 2026-11-06
---

# InstAttention：在计算型 SSD 内执行解码注意力（HPCA 2025）

> **会议论文**：InstAttention: In-Storage Attention Offloading for Cost-Effective Long-Context LLM Inference

> **一句话总结**：该工作把 KV 和 decode 的注意力算子一起放进计算型 SSD，用设备内部 flash 通道读取数据并把结果返回 GPU；预印本的稠密和 SparF 稀疏配置分别说明数据路径的收益与近似选择的额外收益，评测还包含软件定义 CSD 建模。[^preprint]

**版本与阅读状态**：HPCA 2025 正式录用题名为 InstAttention；当前读到的作者预印本题名为 **InstInfer**，作者与副标题对应，但尚未取得会议终稿，因此本页机制、图表和数字均明确指预印本，不能断言与终稿逐项一致。PDF 正文已读取、部分原图已核对；MinerU 转换和会议版本差异仍待核验。[^venue][^preprint]

## 问题与动机

显存装不下 KV 时，把数据放在 CPU 或 SSD 能扩大容量，但逐层 decode 若反复将历史 KV 经主机和 PCIe 搬到 GPU，带宽可能成为主瓶颈。预印本的 FlexGen 测试中，KV 位于较慢层时，KV 访问最多占 decode 延迟的 98.94%（§III-A、图 5）；这是特定模型和批次下的延迟分解。[^preprint]

计算型存储设备（CSD）在 SSD 控制器附近带有计算资源，可利用多条 flash 内部通道。作者提出将低算术强度、依赖大量 KV 的 decode attention 放到 CSD，GPU 保留 prefill、投影及 FFN；这样跨设备传递的是 query、输出和新生成 KV，不必每步把全部历史搬回 GPU。普通商用 NVMe SSD 不自动具备这个计算路径。[^preprint]

## 关键观察 / 隐含假设

- **观察 1：GPU–host/SSD 链路比 GPU 本地内存慢，历史 KV 传输随 batch 增大成为主要成本。** FlexGen 的预印本实验显示高 batch 时 KV 访问最多占 decode 延迟 98.94%（§III-A、图 4–5）。[^preprint]
  - **依赖假设**：被卸载的 KV 足够大且需要反复读取，而内部计算节约的外部传输能覆盖 CSD 运算、通信与调度开销。KV 本来能常驻内存、小 batch 或较快互连下，收益可能变小。
- **观察 2：decode attention 的计算强度低于投影和 FFN。** 作者用 A6000 与 Zynq7045 型 CSD 的 roofline 分析，认为应将依赖 KV 的 Logit/Attend 放到存储侧，而不是把整个 decode 阶段交给较弱的 CSD（§III-B、图 6）。[^preprint]
  - **依赖假设**：设备内可用 flash 带宽和 attention 计算资源能同时得到利用。若算力、片上缓冲、功耗或单通道带宽不足，移动计算后瓶颈可能从 PCIe 转为 CSD。
- **观察 3：稀疏 token 或 hidden dimension 的读取粒度远小于 flash 页。** 作者以 head dimension 128、FP16 为例，一个向量 256 B，而 flash 页为 4 KB；朴素随机访问可能造成最多 16 倍粒度浪费（§IV-C）。[^preprint]
  - **依赖假设**：可以控制 SSD 内部布局与过滤硬件。按页分组仍会读入同页未选元素，只有实际选择局部性足够高时，页级节省才接近 token 级稀疏度。
- **观察 4：近似 Key 分数可以指导较小的 KV 工作集。** 预印本在 OPT-13B 和 LLaMA-2-7B 的质量测试中，SparF 与 SparQ 结果接近，作者据此采用 1/8 的默认配置（§IV-B、§VI-B、图 11）。[^preprint]
  - **依赖假设**：top-r query 维度和 top-k token 对目标任务足够准确。硬件加速的选择是近似的，所测数据集上的结果不代表任意长上下文任务无损。
- **观察 5：真实 FPGA 原型与更大配置的评测能力不同。** Daisyplus OpenSSD 原型只有 64 GB、4 flash 通道且使用昂贵 FPGA；作者据实测统计建立 NVMeVirt 软件定义设备，并按 Zynq7045 及 8 通道配置估计后续系统（§V）。[^preprint]
  - **代价假设**：测得的局部开销、仿真设备及目标 FPGA 配置能较好代表真实多 CSD 系统；这需要对固件、争用、功耗和控制器实现做实机验证。

## 核心方法

### 1. GPU–CSD 分工与一次解码

prefill 在 GPU 上执行，系统按层发送 KV 到 CSD。进入 decode 后，GPU 完成 QKV 投影；CSD 用 query 和设备上保存的历史 KV 执行 attention，再将结果送回 GPU，后者完成输出投影和 FFN。新生成的 KV 也要写入设备；主机仍承担控制、调度和数据搬运协调（§IV-A、§IV-D）。[^preprint]

作者设计 GPU 与 CSD 的 PCIe peer-to-peer DMA 和按层传输，旨在减少经过主机的软件栈和额外复制。P2P 是否可用取决于拓扑、设备和驱动支持；即便跨 PCIe 的历史 KV 流量减少，query、输出及写入仍需要通信。[^preprint]

### 2. SparF：先按维度估计，再按 token 精算

SparF 基于 SparQ 的稀疏思路。首先从 query 中挑选最大的 `r` 个 hidden 维度，读取对应的 K 列，估计所有历史 token 的注意力分数；然后根据估计分数选 top-k token，读取它们完整的 K/V，计算近似输出。其公式还用被选 token 的概率质量和未选部分的统计量组合结果，不能简单描述成“只算 top-k、其余值恒为零”（§IV-B、算法 1）。[^preprint]

flash 以页读取，因此两个阶段都分两步：先跳过整页都不需要的数据，把需要的页读进 NAND flash controller 缓冲；再在控制器过滤页内未选元素，只把需要的数据送给 attention 引擎。页级实际 I/O 稀疏度通常低于算法级稀疏度。[^preprint]

### 3. 两种 Key 排列和设备内算子

按 token 选取时需要完整 K/V 向量；按 hidden 维度估分时则需要跨 token 的 K 列。为支持两种访问方向，系统在 flash 中保存两份不同布局的 Key。示例中每个 token 的 head 为 256 B，16 个连续 token 组成 4 KB 页；按维度访问时则将多个 token 的同一维度连续存放。设备内有 `argtopk`、attention kernel 和靠近 flash controller 的过滤器，分别承担选择、向量计算和页内过滤（§IV-B–C、图 8–9）。[^preprint]

重复存储 Key 换取两种访问方向的效率；磁盘容量、初次写入和更新流量都增加。新 token 先在 CSD DRAM 组缓冲中累积，达到写入粒度后后台刷入 flash，这也意味着缓冲容量与一致性需要管理。[^preprint]

### 4. 原型与软件定义评测

作者先基于 Daisyplus OpenSSD 实现设备并收集时间和开销；该板卡受制于容量、通道数、FPGA 价格及旧主板支持。为了研究更现实的配置，作者使用 NVMeVirt 建立软件定义 CSD，将设备计算资源按 Zynq7045 配置，并将 flash 参数扩为 8 通道、每通道 1.4 GB/s。资源表报告约 80.27% LUT 和 85.33% DSP 利用率（§V、表 I）。[^preprint]

这提供了有根据的设计空间研究，但不能将所有主实验结果表述为“在同一批量产计算型 SSD 上端到端实测”。必须分别说明哪些开销来自原型，哪些系统结果依赖软件定义设备或建模。[^preprint]

## 设计取舍

- **内部带宽换硬件约束**：CSD 计算减少历史 KV 跨主机传输，却需要定制控制器、计算资源、固件和 P2P 路径。
- **稀疏化换精度风险**：预印本同时有 dense 和 SparF 两条配置；后者可减少内部 I/O 和计算，也可能改变生成结果。
- **双布局换容量和写入**：Key 的两种排列减少维度与 token 两种读取方式的随机访问，却增加存储占用、刷写和一致性负担。
- **大 batch 换等待**：论文强调离线吞吐，大 batch 能摊薄开销；实际交互 TTFT、单请求 token 间延迟和排队未由离线峰值直接证明。

## 实验设置

预印本使用单 CPU/GPU 平台，A6000 48 GB、Xeon 5320、96 GB DDR4，SSD 参数参考 2 TB Samsung 980 Pro，GPU 通过 PCIe Gen4 ×16 连接 CPU。主模型 OPT-13B、FP16，输入和输出各 1024 token；质量还测试 LLaMA-2-7B。基线包括 host 内存卸载的 DeepSpeed、SSD 卸载的 FlexGen、FlexGen-SparQ、CSD dense 和 CSD SparF（§VI-A）。[^preprint]

这里论文称之为长上下文，但 1K 输入和 1K 输出远短于当前 128K 或百万 token 工作负载；结果尤其体现大 batch 的 KV 容量压力。模型时代、软件栈及 CSD 建模应随数字一同保留。[^preprint]

## 实验与结果

| 结果 | 条件与边界 |
|---|---|
| 单 CSD、dense 在 batch 64 时比 FlexGen 快 6.85 倍 | 图 12、§VI-C；评测路径含软件定义 CSD 配置。 |
| 单 CSD、SparF 最多比 FlexGen 快 11.1 倍 | 同图；稀疏方案改变注意力，不能解释成 dense 路径独立贡献。 |
| dense CSD 的最高吞吐仅比 DeepSpeed 的最高点高 4.6% | CSD batch 256 与 DeepSpeed batch 16 的各自最佳点；设备内部带宽仍低于 host–GPU 路径，不能只报最大对 SSD 基线的倍数。 |
| 两 CSD 的 SparF 对 FlexGen-SparQ 达到 3.11 倍 | 分别取 batch 256 和 batch 32，比较点不同；并非相同 batch 的直接加速（图 13）。 |
| KV 访问延迟占比下降 | dense、batch 64 时，FlexGen 为 98.9%，单/双 CSD 为 80.7%/76.4%；占比变化不等于总体延迟按同一比例变化（图 14–15）。 |
| 扩展到 20 CSD | 相比单 CSD，dense / sparse 吞吐分别为 8.99 / 7.29 倍；属于配置扩展评估，不应称为 20 台量产设备的独立实测（图 17）。 |

## 论断—证据表

| 论断 | 证据 | 限制 | 判断 |
|---|---|---|---|
| 将 decode attention 移至存储侧可减少跨层 KV 流量 | 数据路径与 §VI-D 延迟分解 | 依赖可编程 CSD、P2P、控制器和内部资源 | 机制成立，实际收益依赖部署 |
| 页感知稀疏选择可进一步提速 | dense 与 SparF 对照、图 11–12 | 相对完整注意力近似，且存在页级读放大 | 质量与速度需要一起评估 |
| 多设备可扩展 | 图 17 的通道和设备扩展 | 软件定义设备及建模不能覆盖所有实机争用 | 有设计空间证据，实机证据有限 |
| 方案适合现代长上下文服务 | 1K 输入、1K 输出的离线测试 | 未覆盖 128K+、在线尾延迟或多租户 | 证据不足以直接外推 |

## 批判性分析

与 [[SolidAttention-FAST26]] 和 [[KVSwap-MobiSys26]] 的“选择 KV 并回载到计算设备”相比，InstInfer 的关键变化是计算地点：CSD 读取本地 flash 并执行注意力，GPU 不需要为该算子取回全部历史。两类系统仍共同面对选择准确度、布局、页粒度和后台写入问题；不能把普通 SSD 上的软件预取等同于设备内 attention。[^preprint]

预印本主结果包含稠密 CSD 与稀疏 CSD。dense 路径的意义是隔离计算位置的影响；11.1 倍还叠加了 SparF 的近似，因此不能把它单独归因于近存储计算。单 CSD dense 最佳点只比 host 内存 DeepSpeed 的最佳点高 4.6%，也提醒读者：当主机内容量足够且带宽较高时，CSD 不必然更快。[^preprint]

最重要的外推边界是实现真实性：Daisyplus 的原型测量为模型提供依据，但更高通道和更多设备的结果依赖软件定义设备与 FPGA 资源假设。应在真实 CSD 上核查端到端带宽、页级 I/O、P2P 可达性、功耗、写放大和不同 batch 的单请求延迟，并与同精度、同容量的 CPU/GPU 卸载基线比较。

## 可继续研究的问题

1. 对原生稀疏模型，设备内能否保留模型准确选择，并只把真正被选择的 KV 用于注意力？
2. 在 PCIe 交换机和多 CSD 拓扑下，P2P、head 划分和主机控制开销能否随设备数扩展？
3. 双 Key 布局长期更新产生多少写入和磨损？如何控制写放大与故障恢复？
4. 实测 128K+ 上下文、batch 1、在线延迟和真实设备功耗时，计算位置的交叉点在哪里？

## 参考资料

[^preprint]: Xiurui Pan 等. “InstInfer: In-Storage Attention Offloading for Cost-Effective Long-Context LLM Inference.” arXiv:2409.04992v1，2024。[作者预印本](https://arxiv.org/abs/2409.04992v1)。
[^venue]: Xiurui Pan 等. “InstAttention: In-Storage Attention Offloading for Cost-Effective Long-Context LLM Inference.” HPCA 2025。[会议记录](https://dblp.org/rec/conf/hpca/PanLLLSZLWZ25.html)。
