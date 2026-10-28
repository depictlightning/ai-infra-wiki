---
type: paper
name: SuperInfer
full_title: "SuperInfer: SLO-Aware Rotary Scheduling and Memory Management for LLM Inference on Superchips"
authors: [Jiahuan Yu, Mingtao Hu, Zichao Lin, Minjia Zhang]
venue: MLSys
year: 2026
tags: [llm-serving, kv-cache, scheduling, offloading, nvlink]
source_pdf: "[[8f14e45fceea167a5a36dedd4bea2543.pdf]]"
source_md: "[[8f14e45fceea167a5a36dedd4bea2543]]"
review_status: needs-review
evidence_level: full-text
last_reviewed: 2026-10-28
---

# SuperInfer：面向 CPU–GPU 高带宽互连的推理请求轮转（MLSys 2026）

> **原题**：SuperInfer: SLO-Aware Rotary Scheduling and Memory Management for LLM Inference on Superchips

> **一句话总结**：SuperInfer 在 GH200 上按延迟风险主动暂停和恢复请求，同时提前把已写满的 KV 块复制到 CPU 内存，以降低内存压力下的请求等待；其调度公式与变量文字说明的对应关系仍需核对。[^paper]

**阅读状态**：已阅读与会议正式 PDF 一致的 MinerU 全文，核对调度公式、主结果图与传输实验。核心参数的正文定义与公式存在不一致，暂保留待复核状态。[^paper]

## 问题与动机

并发请求的 KV cache 占满 GPU 内存后，新请求可能等待正在运行的长请求完成，导致首 token 延迟（TTFT）违反 SLO。把 KV 换到 CPU 内存可以腾出显存，但换出和换回太慢又会延长已开始请求的 token 间隔（TBT）（§3）。[^paper]

GH200 的 CPU 与 GPU 由 NVLink-C2C 连接，提供比常见 PCIe 路径更高的带宽。论文观察到，已有服务框架的 KV 小块搬运未充分利用该链路，于是同时改造调度与搬运路径（§3–4）。[^paper]

## 关键观察 / 隐含假设

- **硬件带宽是前提**：高频换入换出依赖 CPU–GPU 链路和主机内存；实验所用 GH200 结论不能直接推广到 PCIe 机器（§3.2–3.3）。[^paper]
- **已写满 KV 块可视为不再更新**：系统据此提前复制到 CPU 并在抢占时复用副本；维护块状态和副本一致性是正确性条件（§4.3.2）。[^paper]
- **两类延迟目标会相互竞争**：优先新请求可能延缓已有请求生成，较高 TBT 优先级又可能增加新请求等待（§4.2、§5.3.2）。[^paper]

## 核心方法

### 按延迟风险轮转

RotaSched 将请求分为等待、运行和暂时换出三类。它用虚拟滞后时间（VLT）衡量等待或已暂停请求的延迟风险，优先安排滞后较大的请求，并在显存受限时选择运行时间较长的请求换出；每轮还受可用显存块和传输预算限制。若显存可容纳所有请求，则退回先到先服务（§4.2）。[^paper]

**公式疑点**：论文将 $S_B$、$S_F$ 分别定义为 TTFT 和 TBT 的 SLO，但式中用 $\beta_B S_B$ 衡量换出请求自上次生成后的等待，用 $\beta_F S_F$ 衡量新请求自到达后的等待；这与变量对应的延迟阶段不直观一致。后文对 $\beta_B$、$\beta_F$ 的实验描述也需结合实现确认，不能在未核实前自行交换符号（§4.2.2、§5.3.2）。[^paper]

### 提前复制与双向搬运

DuplexKV 将已写满、之后不再变化的 KV 块提前复制到 CPU DRAM；抢占时可释放 GPU 中已有可靠副本的块，只需继续处理尚未同步的块。这样换回的目标块和换出的源块更容易分离，减少双向并发传输中的数据竞争（§4.3.2、图 13）。[^paper]

系统将同一 KV 块的不同层数据按块连续排列，并批量提交搬运。论文举例中 Qwen2.5-32B 的单层片段为 64 KB，合并后一个跨层块为 4 MB；调度和传输还与相邻迭代的模型计算重叠（§4.3、图 14–15）。[^paper]

## 设计取舍

- **主动换出换更快准入**：可降低等待队列压力，但会增加搬运并可能打断已运行请求。
- **提前复制换主机容量和带宽**：完整块在 CPU 和 GPU 暂时同时驻留；块表必须准确跟踪写入及副本状态。
- **批量搬运换布局适配**：较大连续传输减少启动开销，但需要修改分页注意力的布局与访问步长。

## 实验与结果

实验使用 GH200 NVL2，每个 CPU–GPU 对配置 144 GB HBM 和 480 GB DRAM，其中 400 GB DRAM 用于 KV 卸载。测试 LLaMA-3-8B、Qwen2.5-32B、Mixtral-8x7B，以及按泊松到达重放的 ShareGPT 和 LMSYS 数据；TTFT 和 TBT 阈值分别为 5 秒和 100 毫秒（§5.1–5.2）。[^paper]

| 结果 | 条件与边界 |
|---|---|
| TTFT 达标率改善 | 主结果比较 vLLM、TensorRT-LLM、LightLLM、LTR 和 NEO；增益随模型、轨迹和请求速率变化。图 16 中 TensorRT-LLM 在部分配置的 TBT 达标率明显高于 SuperInfer，不能概括为对所有基线都更好。 |
| KV 双向搬运 | Qwen2.5-32B、两方向各传输 8 GB：DuplexKV 的两个方向分别约 181/179 GB/s，总耗时 46.80 ms；论文 DRAM 理想对照为 41.66 ms（表 1）。 |
| 高传输预算需要高效搬运 | 在 Qwen2.5-32B 和 ShareGPT 消融中，低效搬运引擎强行使用较大传输预算会损害 TBT 达标率；调度收益和搬运能力需要一起看（§5.3.1、图 17）。 |
| 双 GPU 测试 | TP=2 的 Qwen2.5-32B 和 Mixtral 测试优于所比较的 vLLM，但两张 GPU 的实验不能直接证明更大集群扩展性（§5.3.5、图 22）。 |

论文报告部分高负载条件下吞吐最高比 vLLM 高 29.2%；这是特定对照和负载的 token 吞吐结果，不是 TTFT 达标率的增幅（§5.3.6）。[^paper]

## 论断—证据表

| 论断 | 证据 | 评测边界 | 置信度 |
|---|---|---|---|
| 主动轮转能改善所测内存压力下的请求等待 | §5.2–5.3、图 16–17 | 依赖 GH200、工作负载与 SLO；参数定义仍有疑点 | 中 |
| 合并搬运并预同步可提高所测双向带宽 | §4.3、表 1 | 设备及 8 GB/方向微基准条件 | 中高 |
| 所有基线的两项 SLO 都被同时超过 | 图 16 | TensorRT-LLM 在部分 TBT 配置更高 | 低 |

## 批判性分析

### 论证链条

论文先把新请求等待与缓存压力联系起来，再用调度消融和传输微基准区分两个组件的作用。传输实验支持减少小块与启动开销，但不能单独证明所有在线负载的端到端收益。[^paper]

### 假设压力测试

如果主机内存或链路被其他任务占用，提前复制可能与计算或其他流量竞争；缓存块可修改、请求中止或故障恢复也要求状态管理正确。这些是由设计推导的部署问题，需要额外验证。[^paper]

### 实验可信度

论文给出不同基线和调度消融，也展示 TTFT 与 TBT 的权衡；但主要部署为单个 GH200 CPU–GPU 对，双卡仅测 TP=2，且公式符号与文字定义的对应关系未能从正文消除疑问。[^paper]

## 局限与后续工作

- 对照作者实现或勘误核实 $S_B$、$S_F$ 与两个请求状态的公式对应关系。
- 在不同带宽、主机内存压力和真实到达轨迹下同时测量 TTFT、TBT、吞吐及副本占用。

## 相关

- [[LLM-Serving]]、[[KV-Cache-Systems]]：请求调度和 KV 分层；本篇暂不计入核心成员，待公式核对。
- [[DirectKV-OSDI26]]：利用 CPU–GPU 高带宽互连访问主机 KV，数据路径与调度目标不同。

## 参考文献

[^paper]: Jiahuan Yu 等. 《SuperInfer: SLO-Aware Rotary Scheduling and Memory Management for LLM Inference on Superchips》. MLSys，2026。[原文](https://proceedings.mlsys.org/paper_files/paper/2026/hash/07fd64f9316f40193c6a4d87d8afa011-Abstract-Conference.html)。
