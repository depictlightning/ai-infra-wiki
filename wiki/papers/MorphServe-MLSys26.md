---
type: paper
name: MorphServe
full_title: "MorphServe: Efficient and Workload-Aware LLM Serving via Runtime Quantized Layer Swapping and KV Cache Resizing"
authors: [Zhaoyuan Su, Zeyu Zhang, Tingfeng Lan, Zirui Wang, Haiying Shen, Juncheng Yang, Yue Cheng]
venue: MLSys
year: 2026
tags: [llm-serving, quantization, kv-cache, scheduling, elastic-serving, area/llm-serving]
source_pdf: "[[fc490ca45c00b1249bbe3554a4fdf6fb.pdf]]"
source_md: "[[fc490ca45c00b1249bbe3554a4fdf6fb]]"
review_status: complete
evidence_level: full-text
last_reviewed: 2026-10-28
---

# MorphServe：运行时调整量化层与 KV 容量的 LLM 服务（MLSys 2026）

> **原题**：MorphServe: Efficient and Workload-Aware LLM Serving via Runtime Quantized Layer Swapping and KV Cache Resizing

> **一句话总结**：负载高峰时，全精度模型与 KV cache 争用 GPU 内存；MorphServe 在推理期间替换部分模型层，并把腾出的空间交给 KV cache，在所测精度优先模式下，相对全精度服务将 P95 TTFT 降低 2.2–3.9 倍，同时报告了有限的质量下降（§5.1、图 4）。[^paper]

**版本与阅读状态**：本地 PDF 与会议正式版逐字节一致；已阅读已有 MinerU 全文和附录，并核对系统图、主要结果、消融表及解析缺失的公式。[^paper]

## 问题与动机

服务负载在时间上波动：高峰时，全精度权重与增长中的 KV cache 可能挤满显存，造成排队和抢占；静态量化虽能降低权重占用，低负载时也持续承担量化带来的质量损失。论文研究能否把模型精度和 KV 预算作为可随压力调整的运行状态（§3）。[^paper]

其目标是服务时的质量—延迟权衡，而不是提出新的量化算法。和只调整 KV 放置的方案相比，该系统同时改变模型层精度，必须将生成质量纳入比较。[^paper]

## 关键观察 / 隐含假设

- **不同层对量化的敏感度不同**：作者离线比较层输入输出、量化前后层输出，以及逐层替换前后的模型输出，形成替换顺序。跨数据集是否保持同样排序取决于模型和校准数据（§4.2、表 2–3）。[^paper]
- **模型与 KV 竞争同一内存预算**：量化部分层后腾出的空间可交给 KV cache；只有当压力确实由显存容量主导时，这种交换才可能缓解排队（§4.4）。[^paper]
- **切层的传输可以重叠**：论文测得 Llama 2 7B 的 INT4 层替换约 6 ms，并在其设置中与解码重叠；不同层大小、PCIe 争用和执行时长会影响该假设（§4.3）。[^paper]

## 核心方法

控制环由服务监视器、形变控制器和执行器组成。请求压力升高时，LayerSwapper 按预先分析的层顺序，把部分 FP16 层替换为量化版本；压力下降后可恢复高精度。不同精度副本预先保存在固定内存的主机区域，对应内核提前编译（§4.1–4.3）。[^paper]

离线层排序综合局部变换、替换失真与当前已量化层条件下的全模型输出变化，使用贪心算法逐层选择。附录说明这一画像是可选的；没有画像时，系统默认从前向后替换，32 层模型的一次离线画像在所测单 GPU 上不到 15 分钟（附录 A）。[^paper]

KVResizer 使用层替换释放的显存动态扩展 KV block；负载减小时再回收空间供模型层恢复。执行中仍需保证已有请求状态及模型层切换的同步正确性（§4.4、图 3）。[^paper]

## 设计取舍

- **延迟与生成质量**：量化层减少内存占用，但运行中的输出仍可能受低精度影响；恢复高精度不意味着之前生成的 token 没有受到影响。
- **显存与主机资源**：预加载多个精度副本和异步传输需要主机内存、PCIe 带宽及预编译内核。
- **离线校准与适应性**：预先选择层顺序降低在线决策开销，但可能不适应任务分布改变。

## 实验与结果

作者使用 Vicuna-7B、Llama 2 7B、Llama 3 8B 和 CodeLlama 34B；前三者在 24 GB L4 上测试，34B 使用 80 GB A100。Azure 和 BurstGPT 轨迹分别缩放到原到达率的 1/4.75 和 1/1.75，论文展示 72 秒片段；生成质量使用 F1 和 ROUGE-L（§5）。[^paper]

系统构建在 SwiftLLM 上，所有对照方案使用同一服务引擎；这不等同于已经在原生 vLLM 生产环境验证（§5）。[^paper]

| 结果 | 条件与边界 |
|---|---|
| P95 TTFT 降低 2.2–3.9 倍 | 精度优先模式对比全精度服务；作者报告该模式 F1 或 ROUGE-L 下降 0.11%–2.18%，不是完全无质量损失（§5.1、图 4）。 |
| 吞吐提高 1.6–1.83 倍 | DuReader 上不同请求率与模型，对比全精度服务的饱和点（§5.1、图 6）。 |
| P95 TTFT 平均降低 1.73 倍 | 精度优先模式对比 PyramidKV，最高为 2.4 倍；这是另一组基线和口径（§5.1）。 |
| KV 容量最高超出全精度上限 32.97% | 由权重层量化释放显存，不能把它理解为全系统显存增加（§5.1、图 5）。 |

## 论断—证据表

| 论断 | 证据 | 评测边界 | 置信度 |
|---|---|---|---|
| 按负载交换部分层可缓解显存压力 | §4.3–4.4、§5.1，图 5–6 | 所测 L4/A100、模型和缩放后请求轨迹 | 中高 |
| 动态调整改善质量—延迟权衡 | §5.1–5.2、图 4、表 5 | F1/ROUGE-L 等所测任务，质量仍有下降 | 中 |
| 替换开销可被计算掩盖 | §4.3、§5.1 | 所测层大小和 PCIe 环境 | 中 |

## 批判性分析

### 论证链条

负载和显存测量支撑在线释放空间的动机，静态混合精度对照支持动态调整的价值。摘要中强调保持质量，但正文仍报告可测的 F1 和 ROUGE-L 下降，阅读时应保留这一差异。[^paper]

### 假设压力测试

长时间高负载可能使更多 token 处于量化路径；模型层变大或链路繁忙时，层替换也可能暴露在 TPOT 关键路径。动态精度变化是否影响推理可复现性尚需单独测量。[^paper]

### 实验可信度

论文包含多模型、两类轨迹和不同静态基线，但流量经过缩放并展示短片段；短期准确率与延迟不足以证明长期生产成本、恢复稳定性和所有任务的质量表现。[^paper]

### 系统性缺陷

多个精度副本、控制循环和动态块管理增加实现复杂度；同时发生权重替换与 KV 访问时需要保证同步、资源隔离及故障恢复。[^paper]

## 局限与后续工作

- 在长输出和持续高负载下记录逐 token 精度、TPOT、输出质量和显存使用，定位质量与拥塞共同变化的区间。
- 加入 PCIe 后台流量并改变模型层大小，测量替换是否仍能与解码重叠，以及 P99 延迟变化。

## 相关

- 同会议：[[MLSys-2026]]。
- 相邻研究：[[Prism-OSDI26]]、[[LLM-42-SOSP26]]。

## 参考文献

[^paper]: Zhaoyuan Su 等. 《MorphServe: Efficient and Workload-Aware LLM Serving via Runtime Quantized Layer Swapping and KV Cache Resizing》. MLSys，2026。[原文](https://proceedings.mlsys.org/paper_files/paper/2026/hash/8144a9d62e506af0fcdeac0e456b2710-Abstract-Conference.html)。
