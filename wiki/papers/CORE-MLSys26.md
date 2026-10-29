---
type: paper
name: CORE
full_title: "Rethinking DVFS for Mobile LLMs: Unified Energy-Aware Scheduling with CORE"
authors: [Zongpu Zhang, Pranab Dash, Qiang Xu, Y. Charlie Hu, Jian Li, Haibing Guan]
venue: MLSys
year: 2026
tags: [llm-inference, mobile-inference, energy-efficiency, dvfs, scheduling, area/llm-inference]
source_pdf: "[[aab3238922bcc25a6f606eb525ffdc56.pdf]]"
source_md: "[[aab3238922bcc25a6f606eb525ffdc56]]"
review_status: complete
evidence_level: full-text
last_reviewed: 2026-10-29
---

# CORE：协同 CPU 与 GPU 调频的端侧 LLM 推理（MLSys 2026）

> **原题**：Rethinking DVFS for Mobile LLMs: Unified Energy-Aware Scheduling with CORE

> **一句话总结**：CORE 为设备与模型离线搜索 CPU/GPU 频率，在 prefill 和 decode 阶段应用对应配置，减轻独立调频策略相互压低频率造成的推理延迟。[^paper]

**阅读状态**：已通读与会议正式 PDF 一致的 MinerU 正文及独立补充材料，并核对频率轨迹和实验图。补充材料解析见 [[core-supplement]]。[^paper][^supplement]

## 问题与动机

移动端 LLM 即使主要在 GPU 上运行，也依赖 CPU 向 GPU 提交任务，并受到内存带宽影响。Android 的 CPU、GPU 和内存调频策略分别观察各自利用率：GPU 等待 CPU 时可能降低 GPU 频率，CPU 等待 GPU 时也可能降低 CPU 频率，双方进一步等待，导致系统整体延迟增加（§2、§5）。[^paper]

论文希望在能耗和时延之间选择更好的频率组合；较高瞬时功率不必然意味着每次推理能耗更高，因为更短的运行时间也会影响总能耗。[^paper]

## 关键观察 / 隐含假设

- **CPU 仍是 GPU 路径的一部分**：所测 Mali GPU 与 OpenCL 栈需要 CPU 持续提交任务；不同 GPU、驱动或执行后端可能有不同表现（§2.2）。[^paper]
- **频率曲线适合搜索**：CORE 用性能画像和近似 U 形能耗曲线缩减搜索空间；温度、后台工作和未采样形状会影响结果（§6.1）。[^paper]
- **配置可按设备与模型复用**：离线画像按设备—模型建立，论文按 prefill 长度分档；换设备、模型或推理后端需要重新确认（§6）。[^paper]
- **内存调频没有被直接联合搜索**：摘要称协调 CPU、GPU 和内存频率，但 §6.1 明确将内存 governor 保持默认，实际搜索与设置 CPU/GPU；应按实现描述理解。[^paper]

## 核心方法

### 分析独立 governor 的相互影响

论文固定两个频率，观察第三个默认 governor：在其设备上，decode 阶段 GPU 利用率偏低会触发 GPU 降频，而 CPU 等待 GPU 也可能让 CPU 利用率下降。控制实验分别调整 CPU 或 GPU 频率，并观察另一侧 governor 的选择，支持两者存在相互影响（§5.1–5.4）。[^paper]

图 8 展示从固定 GPU 到放开默认 governor 后 CPU/GPU 频率逐步下降的轨迹；补充材料还展示从固定 CPU 开始的对应实验。这解释的是所测设备、调频表与 OpenCL 执行路径，不能认为所有移动平台都会形成相同循环。[^paper][^supplement]

### 离线搜索和运行时切换

CORE 支持两个目标：G1 在给定能耗预算下尽量降低时延；G2 在给定时延目标下降低能耗。开发者为每个设备—模型组合对 prefill 长度分档，并分别为 prefill/decode 寻找配置（§6）。[^paper]

为减少搜索次数，CORE 先试 GPU 频率，选出最多两个候选，再对候选搜索 CPU 频率；内存频率交给原有 governor。运行时由推理框架通知阶段起止并应用事先选择的配置，不是在每个 token 上重新遍历全部频率。[^paper]

## 设计取舍

- **离线成本与运行时开销**：设备—模型画像可复用，但型号、量化、驱动或负载变化会降低适配性。
- **延迟与能耗目标**：更快的频率设置要按总能耗验证；“预算内”不等于功率峰值不变或设备不会升温。
- **特定平台可控性**：原型使用具备频率控制能力的 Android 设备；普通应用未必拥有设置系统频率的权限。

## 实验与结果

作者在 Pixel 7 / 7 Pro 上使用 Android 13、llama.cpp 的 OpenCL CLBlast 后端及 4-bit 模型；设备已 root、屏幕关闭，使用外接 Monsoon 电源监测仪并绕过电池。200 条 ShareGPT 请求的输入不超过 512 token、输出不超过 384 token（§3、§6.2）。[^paper]

| 结果 | 实验条件与边界 |
|---|---|
| 默认策略观察 | 32 输入 / 32 输出 token 的固定频率比较中，部分组合可在相近能耗下降低 TTFT 或 TPOT；这是离线固定频率基线，并非运行时 CORE 自适应结果。 |
| G1 实际轨迹 | TinyLlama 的平均 TTFT 从 10.8 降至 9.2 秒、TPOT 从 212.1 降至 153.1 ms；报告的总电量为 865.5 与 859.4 mAh（§6.3）。 |
| G1 不同模型 | DeepSeek-R1-Distill-Qwen 的 TTFT / TPOT 分别下降 17.7% / 39.6%；StableLM 为 8.5% / 38.3%，均相对默认 governor。 |
| G2 | TinyLlama 在相近目标时延下，总电量由 865.5 降至 800.6 mAh，约下降 7.5%；其他模型结果见 §6.3。 |
| 离线搜索 | G1 下 TinyLlama、StableLM、Llama-2 所有分档的搜索分别约需 17.7、43.1、78.5 分钟；成本由开发者承担，不能描述为无需画像。 |

论文以“相同”能耗或时延概括目标约束，但图 11 中 StableLM 的 G1 能耗约为基线的 1.005，G2 TTFT 约为 1.024；另一个固定长度示例的能耗也有小幅差别。因此应理解为实验中近似满足预算，而非严格逐请求保证。[^paper]

## 论断—证据表

| 论断 | 证据 | 评测边界 | 置信度 |
|---|---|---|---|
| 独立 CPU/GPU 调频在所测栈中可能相互影响 | §5、图 7–8、附录 F | Pixel Tensor G2、Mali、OpenCL | 中高 |
| 阶段化 CPU/GPU 配置改善所测时延—能耗权衡 | §6、图 10–11 | 离线画像、短上下文和特定模型 | 中高 |
| CORE 直接联合搜索并锁定内存频率 | 摘要与 §6.1 | 正文明确保留默认内存 governor | 低 |

## 批判性分析

### 论证链条

论文先比较默认和固定频率，再分别固定资源做控制实验，最后通过离线搜索与轨迹测试验证。它证明所测平台上存在可改进空间，但实际实现只调整 CPU/GPU，摘要对三类频率的概括比实现更宽。[^paper]

### 假设压力测试

持续高频可能受到热管理、电池和后台任务限制；若 GPU 驱动队列、频率权限或调度器不同，原有画像和 CPU/GPU 相互作用也可能失效。论文将考虑热量的配置选择列为未来工作。[^paper]

### 实验可信度

外接电源和关闭屏幕便于隔离测量，却不等于真实手机在日常交互、系统节电模式与长期热约束下的表现。样本长度和模型规模也受设备内存限制。[^paper]

## 局限与后续工作

- 在可授权的多种设备和推理后端上测量长时间运行、温度、电池及后台干扰。
- 在多种长度和输入分布下比较画像迁移、目标约束达成率与搜索开销。

## 相关

- [[LLM-Inference]]：端侧模型执行与能耗。
- [[BEAM-MLSys26]]：数据中心 LLM 服务联合调整批处理和 GPU 频率；硬件、控制目标及能耗口径不同。
- [[Sereno-OSDI26]]：端侧推测解码的执行调度，不是同一调频机制。

## 参考文献

[^paper]: Zongpu Zhang 等. 《Rethinking DVFS for Mobile LLMs: Unified Energy-Aware Scheduling with CORE》. MLSys，2026。[原文](https://proceedings.mlsys.org/paper_files/paper/2026/hash/136b9a13861308c8948cd308ccd02658-Abstract-Conference.html)。
[^supplement]: Zongpu Zhang 等. 《Rethinking DVFS for Mobile LLMs: Unified Energy-Aware Scheduling with CORE》补充材料. MLSys，2026。[原文](https://proceedings.mlsys.org/paper_files/paper/2026/file/136b9a13861308c8948cd308ccd02658-Supplemental-Conference.pdf)。
