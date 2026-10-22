---
type: paper
name: LMTracer
full_title: "LMTracer: Fine-Grained and Real-Time Performance Profiling for Production LLM Systems"
authors: [Wei Liu, Yongchao He, Bohan Zhao, Hongyi Wang, Zhenhua Li, et al.]
venue: SOSP
year: 2026
tags: [llm-serving, gpu-profiling, observability, production]
source_pdf: ""
source_md: ""
review_status: abstract-only
evidence_level: abstract
last_reviewed: 2026-10-22
---

# LMTracer：生产 LLM 系统的细粒度实时剖析（SOSP 2026）

> **原题**：LMTracer: Fine-Grained and Real-Time Performance Profiling for Production LLM Systems

> **证据边界**：未取得全文。依据会议题录、公开摘要和作者项目页；两处来源对“运行时开销/部署时长”的数字不同，暂不选边，避免制造精确但不可靠的结论。

## 问题与动机

普通 CPU tracing 看不到 GPU kernel 内部与异步队列的细粒度瓶颈，而全量 GPU profiler 可能干扰生产服务。摘要和项目页介绍 LMTracer：通过 GPU 上的轻量 probe/fence kernel、图嵌入式测量与 CPU 侧缓冲流式汇总，把请求层事件关联到 GPU 执行细节，供在线瓶颈定位。它与 [[QProf-SOSP26]] 的跨服务**成本归因**不同，更接近单个 LLM serving 栈的性能剖析。

## 关键观察 / 隐含假设

- 性能定位需要连接请求级时序与 GPU 执行事件，单独的 CPU trace 或重型离线 profiler 都不足以低扰动在线覆盖。
- 轻量探针依赖采样事件能代表真实长尾；采样过稀或探针本身改变 GPU 调度时，诊断可能失真。

## 核心方法

公开材料提到 GPU probe/fence kernel 与 CPU 侧缓冲汇总。尚无全文可核对探针插入点、时间戳对齐、数据丢失处理及开销分母，因此不能进一步断言具体内核执行协议。

## 实验与结果

公开摘要称在生产部署中定位 **15 类问题**，修复后延迟下降 **21.2%**、吞吐提高 **24.5%**。项目页与另一公开摘要对探针开销分别给出约 **0.58%** 与 **0.9%**，部署时长分别约 **4 个月** 与 **8 个月**；这可能对应论文版本或统计口径差异。没有全文时只记录这个冲突，**不把其中一个当成最终值**。修复收益也不是 profiler 单独带来的即时加速，而是诊断后工程改动的效果。

## 批判性分析

### 论证链条

在线探针能提供比粗粒度日志更细的 GPU 时序，但“定位问题”到“修复后提速”之间还有工程改动；后者不能全算作探针自身收益。

### 假设压力测试

高并发时缓冲可能溢出，极短 kernel 也可能被探针扰动；采样若偏向长请求，性能画像会偏。

### 实验可信度

公开材料对开销和部署时长互相冲突，未核对全文前不把 0.58% 或 0.9% 任一数字视作最终结论。

- 需核对哪些请求/算子被采样、probe 的同步扰动、对 TP/PP/MoE 等并行场景的覆盖。
- 需分辨“15 类问题”是否都是独立生产事件，以及修复前后的 workload 是否可比。
- 多租户线上环境要看 P99、短请求开销和高负载下的数据丢失率；摘要不足以回答。

## 局限与后续工作

- 获取全文后优先统一 0.58%/0.9% 开销与 4/8 个月部署口径，再检查修复收益的对照负载和因果链。

## 相关

- **同会议**：[[SOSP-2026]]；**相邻系统**：[[QProf-SOSP26]]、[[ACDC-SOSP26]]。
- **会议记录**：[SOSP DOI](https://doi.org/10.1145/3830418.3843871)；**作者项目**：[LMTracer](https://llm-profiling.github.io/)。
