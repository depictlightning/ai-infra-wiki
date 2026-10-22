---
type: paper
name: QProf
full_title: "QProf: Fleetwide Transitive Cost Profiling of Warehouse-Scale Services"
authors: [Sam (Likun) Xi, Alexey Alexandrov, Ali Sheikh, Ning Wang, Vance Lankhaar, et al.]
venue: SOSP
year: 2026
tags: [distributed-tracing, cost-attribution, observability, warehouse-scale]
source_pdf: "[[3830418.3843854.pdf]]"
source_md: "[[3830418.3843854]]"
review_status: complete
evidence_level: full-text
last_reviewed: 2026-10-22
---

# QProf：全集群服务的传递性成本剖析（SOSP 2026）

> **原题**：QProf: Fleetwide Transitive Cost Profiling of Warehouse-Scale Services

> **一句话总结**：单个 RPC 的本地 CPU 成本看不到下游微服务的连锁开销，普通 tracing 又有采样偏差和观察者效应。Google 的 QProf 用独立随机的轻量 tracing、常驻 CPU 采样器与批处理归因，把跨服务请求转成可聚合的成本调用栈；对约一万种 RPC profile，QPS 误差在 10% 内的比例由普通 tracing 的 65% 提高到 73%（§4–§5）。

## 问题与动机

大型服务的一次用户请求可能触发多层 RPC。某个上游服务只消耗很少 CPU，却使下游树上的总 CPU 大幅增加；只看每个程序自己的 profile 会错过这种传递性成本。传统分布式 tracing 主要服务于单次故障排查，有大量注释、动态开启逻辑和人为选样；拿它来估算全集群成本，会因为被跟踪请求本身更贵、用户选择性抽样等原因失真（§1–§3）。

**论文是通用 warehouse-scale 服务可观测性工作，并没有专门评测 LLM 推理或 GPU 算力归因。**它对 AI infra 的价值是提供跨服务成本视角，尤其可用于推理网关、特征服务、缓存和存储组成的请求链；不能把用户原先提到的“精确 AI 算力剖析”当作论文已验证结果。

## 关键观察 / 隐含假设

- **成本采样需要与原有 tracing 决策独立**：应用只对慢请求打开 tracing 时，样本天然偏贵；普通 tracing 自身还增加 CPU 和延迟。QProf 在请求树根部独立均匀抽样，且对应用隐藏 skeletal 状态（§3.1、§4.1、图 6）。这依赖抽样概率和 root 传播在所有 RPC 框架中正确实现。
- **单次 10 ms CPU 采样很粗，但全集群可聚合**：大多数短 span 会得到 0 个 CPU tick；在数十亿 trace 和数万亿 span 的聚合量级，按 RPC profile 汇总才能估计平均成本（§4.2）。低 QPS、小成本服务的误差仍较高。
- **批处理破坏树形 trace 的单亲节点假设**：一个 batch 由多个上游请求合成，不能随意把完整 batch 费用挂给任意一个父请求（§3、§4.3、图 8）。论文的统计归因还假设批内请求平均成本近似相同。

## 核心方法

### Skeletal tracing：为统计而设计的最小 trace

QProf 从请求根节点做与普通 tracing 无关的随机采样，并沿 RPC 链传播选样。对于应用，`is_traced` 仍返回 false，普通注释 API 不执行；只有 RPC 框架等获授权的基础组件能写 CPU、TPU、内存等关键成本注释（§4.1）。这样避免应用因“正在被跟踪”而走不同代码，也减少 trace 本身的开销。代价是它只保留结构和少量成本字段，不能替代详细故障诊断 trace。

### Ambient CPU profiler：低侵入覆盖自成本

Google 的后台 per-thread CPU profiler 每消耗约 10 ms CPU 触发一次中断采栈。QProf 在每个请求的 `TraceContext` 中加原子计数器；中断处理器递增当前线程上的上下文计数，等上下文引用计数归零时把 CPU tick 写入 trace（§4.2）。因此在响应返回后继续执行的异步工作也能归到该请求。服务框架负责跨 RPC 和任务队列传播上下文；论文明确说 Go/Python 的后台 CPU profiler 支持有限，相应服务 CPU 成本可能报零，但在其 fleet 中占比小。QProf 再沿 trace 树把自成本累积成上游可见的传递性成本。

### 批处理归因与离线聚合

`BatchedTraceContext` 给 batch 建一个独立 root，并记录它与多个原始请求的连接。后处理时按采样概率校正，从被抽中的父请求中随机选择一个，把 batch trace “缝合”进去；多次聚合后统计上把 batch 总成本分配给父请求流量（§4.3、图 8）。这种做法要求 batch 成本与请求数大致线性且单项平均成本近似相等；大型后台 compaction 等没有明确小批次父请求的操作不适用。

离线管线把 span 展开成 root→leaf RPC 调用栈，按抽样概率逆权重估 QPS 与资源使用率，再合并同结构栈。零成本 span 可以再降采样、按结构签名聚合，文中报告存储量减少约 10–20 倍，同时保留 QPS 估计能力（§4.4、图 1）。工程师由此可在一个调用栈里看见上游功能引出的下游成本。

## 设计取舍

- **低观察者效应换单 trace 诊断细节**：skeletal trace 对应用隐藏，注释被限制；适合总体成本估计，不适合代替正常 debug trace。
- **统计归因换单请求精确账单**：批处理“随机缝合”在大样本下有效，不保证某次用户请求的精确 CPU 分摊。
- **全集群部署换框架依赖**：上下文传播、RPC span、后台 CPU profiler 必须覆盖主要服务；漏掉的语言或异步路径会系统性低估成本。

## 实验与结果

- **采样准确性**：在约一万类 RPC profile 上，普通 tracing 的 65% 与 skeletal tracing 的 73% 可以把 QPS 估到真实值 ±10%；论文用后台 CPU profiler 和服务监控作对照，skeletal tracing 即使采样率 1/10000、低于普通 tracing 的 1/4000，准确性仍更好（§5、图 9–10）。这支持“减少偏差比单纯多采样重要”，不是每类 RPC 都精确。
- **验证限制**：作者明确说全集群传递性 CPU 真值无法直接测得；评测验证自成本和 QPS，再**推断**若 trace 结构无偏，传递性成本也可靠（§5）。这一步是重要假设，不应写成直接 ground truth 验证。
- **真实用例**：缓存成本案例区分 cache hit、miss 和所避开的下游调用树；批处理案例发现 44% 的 batch 大小为 1，其余 batch 平均大小为 23；某项返回空集的功能接近服务 CPU 足迹的 30%，使团队能找出浪费；还识别低优先级批任务占用高优先级磁盘/CPU 的优先级倒置（§6.1–§6.3、图 14）。这些案例证明可操作性，但不是统一的性能加速倍数。

## 论断—证据表

| 论断 | 证据 | 评测边界 | 置信度 |
|---|---|---|---|
| 轻量独立抽样减小成本画像偏差 | ±10% QPS 命中率 65%→73%（§5、图 9） | Google fleet 约一万 RPC profile | 强（QPS），中（传递成本） |
| 批处理需要显式多父关系 | 44% batch 大小为 1，其余平均 23；fan-in API 做统计缝合（§4.3、§6.1） | 批项平均成本近似相同 | 中 |
| 跨服务成本画像能找浪费 | 空结果功能约占服务 CPU 30%（§6.2） | 单项生产案例 | 强（案例），弱（普遍性） |

## 批判性分析

### 论证链条

普通 tracing 的观察者效应与选择偏差是统计成本分析的直接障碍；skeletal tracing 的独立根部抽样和应用不可见性对准这两个原因。QPS/自 CPU 的准确度改进支持方法。把其扩展到**传递**成本时，仍需依赖 RPC 图完整、成本字段覆盖、fan-in 假设和抽样无偏。论文没有逐请求的全集群成本真值，这是清楚写出的边界。

### 假设压力测试

AI 服务的不同请求可能有相差巨大的输入/输出 token 长度；同一批里的 GPU 成本未必能按“每项均等”分摊。若训练/推理服务还通过消息队列、异步作业或跨数据中心触发后续工作，根上下文传播也可能断掉。大规模共享缓存和后台 compaction 的收益归因需要另建因果基线。

### 实验可信度

覆盖多年生产 fleet、数十亿 trace/天、约一万类 RPC，规模非常强；低抽样率下准确性改善有说服力。但部分资源指标依赖服务自报，Go/Python CPU 支持弱；GPU 计算成本与 LLM 请求级归因并未单独验证。相对于“所有成本都能精确归到每次请求”，证据更支持“多数高流量服务的聚合成本画像”。

### 系统性缺陷

长期保存跨服务调用栈及成本元数据涉及访问权限和高基数字段治理。作者通过丢弃 build hash 等字段做聚合压缩，但某些稀有部署版本的成本异常可能被掩盖。低 QPS RPC 的抽样方差仍需人工判断。

## 局限与后续工作

- 对变长 LLM batch 建立按 token、实际 GPU 时间或算子份额的成本归因，与均摊假设直接比较。
- 为异步队列与后台压缩等无清晰父请求工作建立可审计的成本归属，并量化未覆盖比例。

## 相关

- **同会议**：[[SOSP-2026]]；**相邻论文**：[[LMTracer-SOSP26]] 关注 LLM 推理过程的底层实时性能剖析，[[ACDC-SOSP26]] 关注离线推理的资源效率。QProf 提供跨服务成本层视角。
- **全文**：[Google Research PDF](https://storage.googleapis.com/gweb-research2023-media/pubtools/1044643.pdf)。
