---
type: paper
name: MeldKernel
full_title: "Taming Dynamism on GPUs: Cross-SM Kernel Fusion via SM Cooperation and Just-in-Time Reduction"
authors: [Jingkai He, Guangda Sun, Tianjian Li, Dong Du, Yubin Xia, Haibo Chen]
venue: SOSP
year: 2026
tags: [gpu-kernels, kernel-fusion, moe-inference, dynamic-shapes, compiler, mechanism/execution-utilization, mechanism/pipeline-overlap, mechanism/data-movement, area/llm-inference]
source_pdf: "[[3830418.3843857.pdf]]"
source_md: "[[3830418.3843857]]"
review_status: complete
evidence_level: full-text
last_reviewed: 2026-10-22
---

# MeldKernel：动态负载下跨 SM 融合内核（SOSP 2026）

> **原题**：Taming Dynamism on GPUs: Cross-SM Kernel Fusion via SM Cooperation and Just-in-Time Reduction

> **一句话总结**：变长 attention 和 MoE 路由让不同 SM 的工作量失衡，跨阶段依赖又常迫使中间值写回 HBM。MeldKernel 在单个内核中组织跨 SM 子任务协作与即时归约，按输入形状在运行时决定映射；解码 attention 在所测配置中平均比最佳对照快 1.3 倍（摘要、§6）。

## 问题与动机

传统 GPU 融合通常发生在一个线程块或 SM 内。动态输入使不同 SM 处理的序列长度、专家 token 数不均；若把每个阶段拆成独立内核来建立全局依赖，会付出启动、HBM 中间结果和阶段末尾等待长尾 SM 的成本（§1–§2、图 1–3）。

## 关键观察 / 隐含假设

- **跨 SM 依赖可细化为子任务依赖**：变长 split-KV attention 中，不同分片完成时间不同；下游归约无需等所有 SM 走到全局内核边界，可在相关分片就绪时推进（§2–§3、图 2–3）。
- **片上合作有局部性边界**：现代 GPU 的 SM cluster 内可用 TMA 和 mbarrier 做高速传输与同步，但跨 cluster 往往仍走 HBM；硬件 cluster 的大小也不均匀，固定 CUDA cluster 大小会碎片化 SM（§2.2、§4.2）。因此“跨 SM”不是任意两 SM 都有低成本片上直连。
- **工作映射应由真实输入决定**：静态编译时不知道变长序列和专家路由，编译器保留计算定义，运行时决定子任务队列与硬件分布（§5）。

## 核心方法

MeldKernel（论文简称 Meld）把跨 SM 合作提升为编程抽象：程序描述 split-KV 计算、迭代范围、亲和性与归约语义，编译器保留逻辑任务图，运行时才依据输入序列长度和硬件 cluster 形态决定任务并行度与映射（§3–§4.1）。这将形状相关决策推迟到知道请求实际规模后，避免为平均形状提前固定 SM 分配。

首先，Meld 对硬件 cluster 做**混合大小分派**：选几种 cluster 大小填满不规则 SM 拓扑，而不是只用一个 CUDA cluster size；cluster 内使用 TMA 异步复制和 mbarrier，跨 cluster 则经全局显存交换（§4.2、图 7）。其次，两级调度在 cluster 间按数据流图把通信强的任务尽量放近，在 cluster 内把相邻阶段流水化，缩短中间结果生命周期（§4.3）。最后，即时归约在部分输出可用时触发后续工作，不必把所有部分结果先写回 HBM，再启一个独立归约内核（§4.4–§5）。关键优化是减少“跨阶段写回 + 等待最慢分片”，而非取消所有跨 cluster 传输。

论文在 GQA/MLA 解码 attention、MoE、稀疏 attention indexer 等内核上实例化这一方式（§6、表 4）。它优化 GPU 算子内部的执行边界，与 [[StreamInfer-SOSP26]] 的跨 GPU MoE 解码 barrier 属于不同层级。

## 设计取舍

- **细粒度流水换同步协议**：子任务之间需要正确的 mbarrier 就绪信号、内存序与死锁防护；动态调度还需在内核运行时付出开销。
- **少写 HBM 换片上竞争**：片上通信和归约可能占用 shared memory、寄存器、SM 计算槽。
- **编译器抽象换硬件专用路径**：在不同 GPU 上需要适配 cluster 特性；论文说明 RTX 5090 某些 cluster 功能可能由软件模拟（§6 注释）。

## 实验与结果

- 主要评测在 H100、GB200 和 RTX 5090 上，使用 Qwen、DeepSeek、GPT-OSS 等真实负载形态（§6、表 4）。GQA 解码相对 FlashInfer 为 1.02–1.53 倍，相对 FlashDecoding 为 1.24–3.68 倍；MLA 相对 FlashMLA 为 0.98–1.65 倍，说明并非每点都胜过最强手工内核（§6.1）。
- 摘要汇总解码 attention 平均相对最佳方案约 1.3 倍。图 11 的共享前缀实验报告 1.04–1.71 倍范围，但属于特定前缀形态，不能与前述加速直接相乘。
- §6.2 的消融分解子任务流水、即时归约和中间拷贝吸收的贡献；端到端 §6.4 则检验内核加速能否穿透整个推理栈。

## 论断—证据表

| 论断 | 证据 | 评测边界 | 置信度 |
|---|---|---|---|
| 跨 SM 融合提高动态 attention 效率 | 最佳对照平均约 1.3 倍（摘要、§6.1） | 所测 GPU、模型与输入形状 | 中到强 |
| 不同基线下收益差异大 | GQA 对 FlashInfer 1.02–1.53 倍，对 FlashDecoding 1.24–3.68 倍（§6.1） | 两基线设计不同 | 强（该实验） |
| 并非所有配置都获益 | MLA 对 FlashMLA 最低 0.98 倍（§6.1） | 某些形状和平台 | 强（负结果） |

## 批判性分析

### 论证链条

论文把“不规则工作造成阶段长尾”与“内核边界导致 HBM 往返”拆开，分别用混合 cluster 分派、跨 SM 阶段流水与即时归约处理。片上 cluster 局部性使调度器必须兼顾负载均衡和跨 cluster 流量；相对多个强基线的范围值及最低 0.98 倍，表明融合并非无条件最优。

### 假设压力测试

静态、形状一致的算子可能没有值得消除的长尾，动态调度反而增加开销。跨 GPU 张量并行造成的通信不由单 GPU 跨 SM 融合直接解决；若模型主瓶颈在网络，内核收益难变成端到端收益。

### 实验可信度

覆盖 Hopper、Blackwell 与消费级 GPU，也包括 GQA、MLA、MoE 等多类算子；但真实线上批次形态和数值精度约束仍会影响可移植性。应区分单内核加速、算子图加速和服务吞吐。

### 系统性缺陷

编译器和运行时共同维护子任务依赖，调试死锁或边缘数值错误可能比独立内核困难。硬件 cluster 实现差异提高后续适配成本。

## 局限与后续工作

- 在持续变化的在线批次上测 P99 算子时长与子任务队列开销，并报告端到端 TTFT/TPOT。
- 对不同 cluster 实现、片上带宽和 shared memory 容量测量收益退化点。

## 相关

- **同会议**：[[SOSP-2026]]；**相邻系统**：[[Ave-SOSP26]] 的内核优化约束、[[MorphX-SOSP26]] 的 GPU 多任务资源调整。
- **全文**：[作者机构 PDF](https://ipads.sjtu.edu.cn/zh/publications/sosp26-he.pdf)。
