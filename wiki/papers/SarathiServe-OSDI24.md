---
type: paper
name: Sarathi-Serve
full_title: "Taming Throughput-Latency Tradeoff in LLM Inference with Sarathi-Serve"
authors: [Amey Agrawal, Nitin Kedia, Ashish Panwar, Jayashree Mohan, Nipun Kwatra, Bhargav Gulavani, Alexey Tumanov, Ramachandran Ramjee]
venue: OSDI
year: 2024
tags: [lens/pd-serving, llm-serving, scheduling, chunked-prefill, latency-slo]
source_pdf: "[[osdi2024-agrawal.pdf]]"
source_md: "[[osdi2024-agrawal]]"
review_status: needs-review
evidence_level: full-text
last_reviewed: 2026-10-29
---

# Sarathi-Serve：用分块预填充平衡吞吐和延迟（OSDI 2024）

> **原题**：Taming Throughput-Latency Tradeoff in LLM Inference with Sarathi-Serve

> **一句话总结**：长 prefill 会阻塞正在生成的请求，Sarathi-Serve 将 prefill 切块并优先安排已有 decode；Yi-34B、ShareGPT 工作负载下，严格延迟目标内的容量相对 vLLM 最高提高 3.7 倍（§5.1）。[^paper]

**阅读状态**：MinerU Flash 提取全文文本但没有导出图像；已回看 PDF 的模型配置和图 10–11，其余图表和全文细节尚未复核。[^paper]

## 问题与动机

同一实例同时服务新请求和正在生成的请求时，完整长 prompt 可能占据一次迭代，使其他请求的 token 间隔突然增大。流水线并行还会因为相邻微批耗时不同出现气泡（§3）。论文希望在不分离实例的情况下减少这些停顿，同时保留批处理带来的吞吐。

## 关键观察 / 隐含假设

- 论文在所测模型上观察到，中等长度的 prefill 已能较充分使用 GPU 计算能力；长输入可切成仍有计算效率的块（§3.1、§4.1）。块大小的收益依赖模型和硬件。
- 相邻微批的输入长度、阶段及上下文长度不同会造成流水线气泡（§3.3）。若请求长度或服务配置改变，固定预算未必仍合适。

## 核心方法

每轮先放入正在运行的 decode，再安排已经开始的 prefill，最后在 token 预算允许时接纳新请求；长 prefill 只取本轮剩余预算容许的部分（§4.1–4.2）。这使 decode 每轮得到执行，但混合批次仍会增加单轮耗时，“无停顿”不等于零干扰。预算由延迟目标与性能测量确定（§4.3）。[[DCP-OSDI26]] 则研究随负载和 SLO 动态调整块大小。

## 设计取舍

### 一轮调度具体发生什么

设一轮预算为 512 token，已有 32 个请求处于 decode，则先为它们各安排 1 个 token，剩余预算最多可容纳 480 个 prefill token。正在处理的长 prompt 优先消费剩余预算，新请求只有在显存可分配、预算仍有余量时才进入。下一轮重新计算预算，而不是等待当前 prompt 整体完成。这是根据算法 3 构造的例子，实际批次还受请求结束、显存与最大批量约束。[^paper]

“不暂停 decode”指已运行的 decode 每轮都能被安排，而非混入 prefill 后这轮执行时间与纯 decode 相同。论文图 9 显示完整 prefill 混入会显著拉长迭代，分块主要限制这部分增量；这个区别对理解 TBT 很重要。[^paper]

### 块大小为何不是越小越好

每个新块的注意力要读此前块的 KV；将 prompt 分成 N 块后，第一个块的 KV 最多会被后续 N−1 块重复读取。更小块还会使矩阵规模下降、增加内核启动开销，并受 GPU tile 对齐影响。论文报告某些情况下 257 token 块比 256 token 块的 prefill 耗时高 32%，因此不能假定延迟随 token 预算平滑变化（§4.3）。[^paper]

论文利用 [[Vidur-MLSys24]] 做离线画像和模拟，选择目标硬件与 SLO 下的预算。严格配置通常使用 512 token，宽松配置通常使用 2048；LLaMA2-70B 的宽松配置例外为 1536，以减少流水线气泡。它并未实现按在线流量持续调整预算，这也是与 [[DCP-OSDI26]] 的区别（§5.1）。[^paper]

### 流水线里的三种失衡

论文将气泡来源分为相邻 prefill 长度不同、prefill 与 decode 交替、以及 decode 的上下文长度或批量不同。以 Falcon-180B 为例，4K prefill 约 1150 ms，而 32 请求的纯 decode 迭代约 200 ms；两者交替会使流水线阶段等待。分块让批次工作量更接近，但不能消除所有由上下文和 decode 变化造成的不均衡（§3.3）。[^paper]

- 小块减轻单轮干扰并使流水线更均匀，但重复处理历史上下文等成本使 prefill 更慢。
- 大 token 预算提高 prefill 效率，却可能使 decode 的尾部 token 间隔超标。

## 实验与结果

- 作者使用 Mistral-7B、Yi-34B、LLaMA2-70B、Falcon-180B，对 ShareGPT 和论文摘要数据的请求长度采样，以泊松过程产生到达；过长请求被过滤（§5）。
- Yi-34B、ShareGPT、严格 SLO 下，相比 vLLM 容量最高提高 3.7 倍；LLaMA2-70B 的流水线配置最高提高 4.3 倍（§5.1、图 10–11）。
- Yi-34B 两张 A100 的分块开销实验中，512 token 块的 prefill 总耗时额外增加最多约 25%；2048 token 块的开销接近可忽略（§5.4.1、图 14）。表 4 的组件消融显示单独分块或单独混合各有延迟代价。

### 消融揭示了什么

表 4 在 Yi-34B、两张 A100、128 个请求、1024 token 预算下比较组件。ShareGPT 负载中，仅混合批处理的 P50 TTFT 为 0.53 秒、P99 TBT 为 0.68 秒；仅分块为 1.04 秒和 0.17 秒；结合后为 0.76 秒和 0.14 秒。组合改善的是该实验中同时兼顾两项指标的折中，并非每个指标均优于两个单组件：单独混合的 TTFT 仍更低。[^paper]

实验中的严格和宽松 TBT 目标分别取无 prefill 干扰的单次 decode 耗时的 5 倍和 25 倍；最大负载还要求中位调度等待不超过 2 秒。不同模型绝对门槛不同，例如 Yi-34B 的严格目标为 0.2 秒，LLaMA2-70B 为 1 秒（表 3）。因此这些容量值不是所有模型采用统一交互体验阈值后的排序。[^paper]

## 论断—证据表

| 论断 | 证据 | 边界 | 置信度 |
|---|---|---|---|
| 分块混合可增加严格延迟下的容量 | §5.1、图 10–11 | 特定 A100/A40、模型和合成到达 | 中 |
| 小块会带来 prefill 开销 | §5.4.1、图 14 | Yi-34B、两张 A100 | 中 |

## 批判性分析

### 论证链条

论文从迭代干扰和流水线失衡出发，设计分块与优先级，并用端到端容量和组件消融支持机制；但图表仍需从 PDF 视觉复核。

### 假设压力测试

固定 token 预算对突发流量、不同上下文长度及注意力内核的延迟变化未必稳健。预算无法保证每个请求都满足严格时限。

### 实验可信度

使用真实数据集的长度分布，但到达过程是合成的；旧版本基线结果不应直接推及当前引擎。论文的严格目标是 P99 token 间隔，并非所有请求同时满足 TTFT 与 TPOT。

### 系统性缺陷

论文未充分量化在线预算重调、故障恢复和多租户隔离的开销。

## 局限与后续工作

- 以相同硬件、请求轨迹和 SLO 比较静态预算与 [[DCP-OSDI26]] 的动态预算，记录 P99 延迟和排队。
- 在长上下文与不同前缀命中率下测量分块额外计算及最优预算变化。

## 相关

- [[PD-Serving]]、[[EcoServe-OSDI26]]、[[BatchLLM-MLSys26]]

[^paper]: Agrawal 等. “Taming Throughput-Latency Tradeoff in LLM Inference with Sarathi-Serve.” OSDI 2024。[论文](https://www.usenix.org/conference/osdi24/presentation/agrawal)；本地全文 [[osdi2024-agrawal]]。
