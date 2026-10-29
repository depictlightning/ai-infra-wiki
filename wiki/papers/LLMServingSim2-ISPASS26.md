---
type: paper
name: LLMServingSim2
full_title: "LLMServingSim 2.0: A Unified Simulator for Heterogeneous and Disaggregated LLM Serving Infrastructure"
authors: [Jaehong Cho, Hyunmin Choi, Guseul Heo, Jongse Park]
venue: ISPASS
year: 2026
tags: [lens/inference-simulation, llm-serving, simulation, disaggregated-serving, kv-cache]
source_pdf: "[[ispass26-cho-llmservingsim2.pdf]]"
source_md: "[[ispass26-cho-llmservingsim2]]"
review_status: needs-review
evidence_level: full-text
last_reviewed: 2026-10-29
---

# LLMServingSim 2.0：异构与分离式推理服务模拟器（ISPASS 2026）

> **原题**：LLMServingSim 2.0: A Unified Simulator for Heterogeneous and Disaggregated LLM Serving Infrastructure

> **一句话总结**：LLMServingSim 2.0 在请求调度、算子执行图、网络与多级内存之间建立反馈，用同一套模型研究 P/D 分离、异构设备及能耗；其低平均误差来自指定真实系统与工作负载，不是所有异构部署的精度保证。[^paper]

**版本与阅读状态**：根据作者 arXiv v2 全文与 MinerU 正文，回看图 5–9 和 PIM 案例；作者公开的 artifact 页面标注 ISPASS 2026 接收。未逐项比对会议出版文件，未复现实验。[^paper][^artifact]

## 问题与动机

在 P/D 分离服务中，单独估算一轮 prefill 和 decode 的执行时间并不够。路由会改变各实例队列，prefill 结束后 KV 必须放到 decode 能访问的位置；网络与缓存竞争又会改变后续完成时间。更复杂的部署还可能引入不同 GPU、CPU 缓存、CXL 内存或专家卸载。[^paper]

LLMServingSim 2.0 将服务决策与硬件执行放进同一运行循环。它是作者 2024 年 LLMServingSim 的后续版本，新增与扩展的功能不能倒推为早期版本已有能力；当前仓库代码也可能继续演化，需要固定版本评测。[^paper]

## 关键观察 / 隐含假设

- **硬件与服务决策互相影响。** 批次、放置、KV 迁移和网络拥塞共同决定完成时间，而完成时间又影响队列和下一批请求。只对固定批次做一次静态测量无法直接重现这个反馈（§III–IV）。[^paper]
- **跨层状态是收益与代价的来源。** prefix cache 命中可以减少 prefill，但把 KV 放在 CPU 或远端内存需要读取和传输；研究分离部署时，缓存位置和交接链路必须一起建模（§IV–V）。[^paper]
- **新硬件需要可用性能模型。** 框架支持从实机画像或外部硬件模拟器取得算子耗时，并不意味着没有性能数据就能准确预测任何未来设备（§IV、§VII-C）。[^paper]

## 核心方法

### 请求、模型服务组与执行图

输入包括请求轨迹、集群配置、算子性能画像和策略。执行规划器创建一个或多个模型服务组（MSG），为其配置设备池、批处理、并行和内存策略；请求路由器将请求交给相应服务组。每个服务组保存请求队列、KV 状态和可用设备，批调度器在显存和批量限制内选择请求（§IV-A–C、图 1–2）。[^paper]

服务组将当轮运算映射到设备并生成带依赖的执行图；节点表示算子、数据搬运或同步，系统模拟器按拓扑、设备耗时和资源竞争推进，再把完成时间与状态反馈给服务组。重复这个循环直到请求完成，输出 TTFT、每输出 token 时间（TPOT）、排队、吞吐、显存和能耗等指标。[^paper]

### P/D 分离如何进入模型

规划器建立多个 prefill 与 decode 服务组，路由器给请求选择 prefill 及目标 decode 组。prefill 完成后，系统把 KV 传输和依赖加入执行图，依据配置的网络带宽、拓扑和内存模型估算交接；decode 随后在目标组排队与执行（§V-B、图 4）。[^paper]

这使它适合研究固定配置下不同 P:D 组数、硬件组合、链路和请求分布，但框架支持表达配置不等于论文已经证明某个动态配比算法最优。角色切换、冷启动、在线预测与控制策略若未被对应实验实现和校验，需要使用者另行建模。[^paper]

### 多级缓存、MoE 与异构设备

模拟器根据策略追踪设备、主机及 CXL 等内存层中的前缀缓存，命中时减少有效 prefill，未命中时计入读取和移动。多实例可配置共享的 CPU 缓存池。MoE 场景则按照模拟路由策略安排 token 与专家，向执行图加入专家加载和通信（§IV-C、§V）。这些策略的抽象是否贴近目标系统，必须与具体实现核对。[^paper]

设备池允许把算子映射到 GPU 或其他类型的设备，底层使用扩展的 ASTRA-sim 与 Chakra 处理执行图、通信和内存。论文的 PIM 案例使用模拟硬件模型；这与 TPU 案例中用真实 TPU 运行 vLLM 验证的证据等级不同（§IV-D、§VII-C）。[^paper]

### 功率模型

系统将加速器表示为 idle、active、standby 三种状态，并估算 CPU、DRAM、链路、网卡、存储等组成的功率；部分设备按传输量计能耗，其余部分使用常量。这个模型能分析不同阶段下的能耗构成，但状态切换和常量设置决定预测可信范围（§IV-C、图 6）。[^paper]

## 设计取舍

- **系统覆盖与运行成本**：显式建模请求、设备、内存和通信能够探索更多交互，但执行图和状态更新比简单平均速率模型更重。
- **异构外推与画像成本**：同一框架能接入多个硬件后端，但要收集与目标框架、模型和设备相匹配的性能数据。
- **能耗细节与假设**：功率模型覆盖多个组成部分，但其他组件的常量假设可能遮蔽负载相关变化；耗能结论应与测量范围一起给出。

## 实验与结果

主要对照使用 vLLM，设备包括 RTX A6000、H100，以及 TPU-v6e-1；除特别说明外，请求从 ShareGPT 采样 300 条，并以每秒 10 个请求的泊松过程生成到达。模型包括 Llama 3.1 与 Mixtral；这些是指定短轨迹和所测配置，不是大规模多租户生产流量（§VI）。[^paper]

- **动态与聚合误差不同**：图 5 对多模型、prefix cache、P/D 分离和 MoE 的吞吐时间序列比较中，RTX A6000、H100 的平均误差分别为 5.14%、3.29%；将吞吐和延迟聚合到整个执行后，平均误差分别为 0.99%、1.54%（§VII-A）。不能把较低聚合误差当作每个时间点或尾延迟的误差。[^paper]
- **功率**：图 6 使用 A6000，10 条 ShareGPT 请求在 60 秒间隔下重复三次；TP1/TP2 的总能耗平均误差为 1.34%。这种主动构造的忙闲周期有助检查状态切换，但不是长时生产能耗分布。[^paper]
- **前缀缓存**：图 7 对 GPU 本地缓存及两个实例共享 CPU 缓存做对照；作者报告单实例内存与命中率平均误差 0.93%，多实例对应平均误差 0.41%。实验使用 LMCache 和指定缓存块大小，不能推广到所有缓存实现。[^paper]
- **与其他模拟器比较**：图 8 中单实例 dense 场景，LLMServingSim 2.0 耗时 302.3 秒，作者配置的 Vidur 为 485.3 秒；多实例 dense 分别为 633.2 秒与 495.1 秒。正文笼统称新方法相对 Vidur 等轻量模拟器开销更高，但图中相对顺序随配置改变，应按具体实验读取。X 表示基线不支持或无结果，不是零误差。[^paper]
- **新设备案例**：真实 TPU-v6e-1 的单实例 dense 测试中，吞吐时间序列平均误差为 3.95%；GPU+PIM 的 1.43 倍 decode 吞吐和每 token 能耗下降 14.8% 来自模拟案例，输入/输出分别为 128/512 token、256 个请求，不是物理 PIM 生产实测（§VII-C、图 9–10）。[^paper]

## 论断—证据表

| 论断 | 证据 | 评测边界 | 置信度 |
|---|---|---|---|
| 模拟能跟踪所测系统的阶段变化和汇总性能 | 图 5、§VII-A | 300 条 ShareGPT、泊松到达、特定设备与 vLLM 配置 | 中 |
| 多级前缀缓存和功率模型可近似所测曲线 | 图 6–7 | 特定 LMCache 配置、A6000 和构造的忙闲阶段 | 中 |
| 模拟器可用于异构设备探索 | §VII-C、图 9–10 | TPU 有实测对照；PIM 案例由模拟模型给出 | 中 |

## 批判性分析

### 论证链条

论文将执行层、内存层与服务调度的状态互相连接，再以真实 vLLM 的吞吐曲线、功率和缓存曲线作对照，这比只核对一个静态算子更接近服务系统。验证仍集中于有限平台和短轨迹；平均值掩盖个别请求与时间点的误差，论文中的聚合误差不能替代 P99、超时比例或配比排序验证。[^paper]

### 与 Vidur 的对比

[[Vidur-MLSys24]] 面向模型部署配置搜索，以实测算子和事件调度重现工作负载；本论文强调异构设备、分离和多级内存，并在图 8 按作者所用版本比较多个模拟器。支持的配置不同，图中的缺失项和不一致运行时间要求逐场景解释，不宜宣称一者在所有设置上既更准又更快。[^paper]

### 功能支持与验证覆盖

论文展示 P/D 映射、prefix cache 和 MoE 的模型接口，但真实平台覆盖不等于每一种硬件组合、拓扑和资源切换都经过对照。CPU 画像、网络拥塞、缓存 eviction、实际的角色迁移和新模型 kernel 的误差可能叠加；这些结论需要针对目标部署验证。[^paper]

## 局限与后续工作

- 使用同一请求到达轨迹与 P/D 候选配比，比较真实系统和模拟的 TTFT、逐 token 延迟、队列、链路流量及选择出的最优配比。
- 增加长上下文、前缀命中突变和突发流量，验证高负载下的误差传播。
- 分别测量画像准备、单次模拟和多配置搜索的耗时，固定模拟器代码与后端版本。
- 针对角色切换和模型加载显式测量时间、KV 清理及过渡状态，再加入动态资源分配实验。

## 相关

- **专题**：[[Inference-Simulation]]、[[PD-Serving]]。
- **模拟器对照**：[[Vidur-MLSys24]]、[[Charon-MLSys26]]。
- **阶段部署**：[[DistServe-OSDI24]]、[[Splitwise-ISCA24]]、[[SplitScaling-IJCAI26]]。
- **代码与实验数据**：[LLMServingSim 仓库](https://github.com/casys-kaist/LLMServingSim)、[论文 artifact](https://zenodo.org/records/18879965)。

[^paper]: Cho et al., [LLMServingSim 2.0, arXiv v2](https://arxiv.org/abs/2602.23036v2), 2026；本地来源见 frontmatter。
[^artifact]: [ISPASS 2026 artifact 页面](https://zenodo.org/records/18879965)。
