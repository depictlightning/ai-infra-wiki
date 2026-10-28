---
type: paper
name: ProTrain
full_title: "ProTrain: Efficient LLM Training via Automatic Memory Management"
authors: [Hanmei Yang, Jin Zhou, Yao Fu, Xiaoqun Wang, Ramine Roane, Hui Guan, Tongping Liu]
venue: MLSys
year: 2026
tags: [llm-training, memory-management, activation-checkpointing, offloading, data-parallelism, area/llm-training]
source_pdf: "[[56280ad5fb53967fd55e4ba1b1cfc418.pdf]]"
source_md: "[[56280ad5fb53967fd55e4ba1b1cfc418]]"
review_status: complete
evidence_level: full-text
last_reviewed: 2026-10-28
---

# ProTrain：自动协调大模型训练内存策略（MLSys 2026）

> **原题**：ProTrain: Efficient LLM Training via Automatic Memory Management

> **一句话总结**：ProTrain 用 profiling 和成本模型联合选择参数驻留、缓冲、激活卸载与重计算配置，尽量在固定显存下缩短训练迭代；其优化范围是给定数据并行方案下的内存策略。[^paper]

**阅读状态**：已阅读会议正式版与独立补充材料的 MinerU 全文，并回看吞吐与模型容量表。旧版本与会议版不同，本页以正式版为准。[^paper][^supplement]

## 问题与动机

训练中的参数、梯度、优化器状态与激活共同占用显存。ZeRO 切分、CPU 卸载和梯度检查点都能节省显存，但它们之间存在相互作用：激活卸载与参数预取可能竞争同一 CPU–GPU 带宽；检查点降低显存却增加反向计算。逐个旋钮独立调优很难找到协调后的较优方案（§1–2）。[^paper]

ProTrain 将状态与激活的管理组织为可建模的配置空间，通过一次 profiling 建立延迟及峰值显存估计，再搜索满足显存约束的配置（§3）。[^paper]

## 关键观察 / 隐含假设

- **执行轨迹具有可预测性**：系统依赖模型执行次序、块结构和模型状态大小进行规划；动态控制流或运行期间负载变化可能削弱模型准确性（§3.1–3.3）。[^paper]
- **资源可以重叠但不会无限重叠**：参数搬运、激活交换、通信和计算互相争用带宽；成本模型按 profiling 与结构假设估计，并不保证所有实际环境中完全隐藏（§3.3、补充材料 A.1）。[^paper][^supplement]
- **搜索范围有限**：系统使用给定数据并行策略，搜索的是自身定义的块驻留、缓冲和激活策略，不是所有 DP/TP/PP 组合的全局最优（§3.3、§6）。[^paper]

## 核心方法

### 分层模型状态管理

将参数、梯度与优化器状态组织成 chunk。一部分持久驻留 GPU，其他部分切分并在 CPU 或其他 rank 保留，按需收集和预取；非驻留部分的 CPU 参数更新与 GPU 反向计算重叠。持久 chunk 从模型前部开始选取，原因是前几层缺少足够时间隐藏前向预取及反向后的 CPU 更新（§3.1.1）。[^paper]

chunk 内按实际执行顺序组织参数，而非仅按模型定义顺序，以减少来回访问；预分配缓冲并按可预测顺序装入、驱逐，也让成本模型更容易模拟（§3.1.1）。[^paper]

### 交错管理激活

按 Transformer block 决定激活交换、梯度检查点或保留；交换块与检查点块交错排列，控制激活累积，并使部分交换能够与计算重叠。显存压力和传输带宽改变时，自动搜索不同策略数量（§3.1.2、图 2）。[^paper]

### Profiling 与约束搜索

Profiler 记录算子内部瞬时峰值以及两个可挂钩模块之间未被 hook 捕获的内存变化；单 GPU 容量不足时按需分配和释放张量，再依据执行轨迹重建目标配置下的状态与激活驻留。论文在 10B GPT-2、batch 16 的例子中，指出被忽略的临时和不可挂钩操作占峰值约 17.2%（§3.2）。[^paper]

搜索变量包括持久 chunk 数、缓冲数、交换 block 数与检查点 block 数；在峰值显存约束下，通过枚举和剪枝选择预测迭代时间最短的配置。补充材料把单步时间建模为前向时间加上 GPU 反向及更新与 CPU 更新两条路径中较长者；具体重叠和带宽争用按 profiling 估计（§3.3、附录 A）。[^paper][^supplement]

## 设计取舍

- **驻留换容量**：更多常驻状态减少数据搬运，却压缩激活和缓存空间。
- **重算与交换换取不同资源**：重算消耗算力，交换占用主机内存和互连；最优选择随 batch、模型与硬件变化。
- **可预测的搜索空间换通用性**：固定策略结构便于建模，但可能遗漏不在候选空间内的更优调度。

## 实验与结果

实验使用 4 张 RTX 3090（每张 24 GB、PCIe 3、无 NVLink）或 4 张 A100（每张 80 GB、PCIe 4、GPU 间 NVLink），模型包括 GPT-2、OPT、Mistral、LLaMA；主要吞吐对照序列长度固定为 1024。各系统在不同 batch 下寻找自身最高吞吐，因此不应把结果误解为完全相同 batch 的逐步对比（§5.1–5.2）。[^paper]

| 结果 | 条件与边界 |
|---|---|
| 4 张 A100、Mistral-7B | ProTrain 11060.92 tokens/s；DeepSpeed 卸载 7708.30、不卸载 9748.03。相对不卸载基线约 1.13 倍，而不是把 1.43 倍当作相对该框架所有配置（表 3）。 |
| 4 张 A100、GPT-2-10B | ProTrain 8266.40 tokens/s；Colossal-AI 不卸载 7855.46，约 1.05 倍；基线选项影响结论（表 3）。 |
| 可训练模型容量 | 单 A100 上报告 ProTrain 可训练 75B GPT-2，DeepSpeed 为 34B、Colossal-AI 为 53B、FSDP 为 10B；该表衡量能否放入，不衡量训练时间、总内存或成本（表 2）。 |
| 单步模型误差 | 10B GPT-2 的所测配置中，运行时间和峰值显存预测偏差均在 4% 内；不能当作任意动态模型的误差保证（§5.3.2、图 6）。 |
| 多节点示例 | 4 节点 16 张 A100、175B GPT、batch 256：2218.70 tokens/s；对比 DeepSpeed 约 2.58 倍、Colossal-AI 约 2.09 倍，FSDP OOM（§5.5）。 |
| Profiling 和搜索开销 | RTX 3090、batch 4 的 7B Mistral profiling 3.09 秒，20B GPT-2 为 5.38 秒；所测搜索平均 0.06 秒，分别是不同阶段的开销（§5.3.4）。 |

论文的 1.43–2.71 倍是指定硬件和基线的平均吞吐比较；也报告单个配置对 FSDP 的更高倍数，二者不是同一统计口径。论文还推算 GH200 的激活交换时间，该推算不能当作 GH200 真机测量（§5.2–5.3）。[^paper]

## 论断—证据表

| 论断 | 证据 | 评测边界 | 置信度 |
|---|---|---|---|
| 联合管理可提高所测固定硬件下的最大训练吞吐 | §5.2、表 3 | 最优 batch 可能不同，基线版本与调优影响结果 | 中高 |
| 轨迹与成本模型可指导候选配置选择 | §3.2–3.3、§5.3.2、补充材料 A | 所测结构和配置；未证明全局最优 | 中 |
| 模型容量增加意味着训练成本更低 | 表 2 | 容量表不提供完成训练的时间和总成本 | 低 |

## 批判性分析

### 论证链条

论文将状态、激活和互连竞争统一为有限配置空间，再用预测与实测的对应关系支撑搜索。消融显示不同优化贡献不一样：3090 的 PCIe 易被参数预取占满，激活交换收益有限，换硬件不能沿用同一结论（§5.3）。[^paper]

### 假设压力测试

若训练期间模型路径、序列长度或其他作业造成的 I/O 竞争变化，单次 profiling 得到的预测可能过期。按论文假设进行重叠估计时，CPU 更新或传输没有被完全隐藏也会增加真实迭代时间；这是对模型适用条件的分析。[^paper][^supplement]

### 实验可信度

论文报告强基线的卸载和非卸载结果，但测试序列为 1024，规模主要为单机四卡，基线版本、CPU 数量、PCIe/NVLink 以及最大 batch 都影响结果。是否适合现代长上下文和更复杂并行组合仍需直接测量。[^paper]

## 局限与后续工作

- 对可变序列长度、动态模型与共享主机带宽测试预测误差和 OOM 风险。
- 比较联合并行策略搜索和本方案的工程成本、收敛效率与端到端训练成本，而不是只比较 tokens/s。

## 相关

- [[LLM-Training]]：训练执行与内存管理。
- [[AXLearn-MLSys26]]：跨硬件训练框架与恢复，研究边界不同。

## 参考文献

[^paper]: Hanmei Yang 等. 《ProTrain: Efficient LLM Training via Automatic Memory Management》. MLSys，2026。[原文](https://proceedings.mlsys.org/paper_files/paper/2026/hash/56280ad5fb53967fd55e4ba1b1cfc418-Abstract-Conference.html)。
[^supplement]: Hanmei Yang 等. 《ProTrain: Efficient LLM Training via Automatic Memory Management》补充材料. MLSys，2026。[原文](https://proceedings.mlsys.org/paper_files/paper/2026/file/56280ad5fb53967fd55e4ba1b1cfc418-Supplemental-Conference.pdf)。
