---
type: paper
name: EnerTune
full_title: "Beyond Utilization: Energy-Conscious GPU Sharing for Inference Serving"
authors: [Prasoon Sinha, Dimitrios Liakopoulos, Nathan Lemma, Neeraja J. Yadwadkar]
venue: SOSP
year: 2026
tags: [gpu-sharing, energy-efficiency, inference-serving, scheduling]
source_pdf: ""
source_md: ""
review_status: abstract-only
evidence_level: abstract
last_reviewed: 2026-10-22
---

# EnerTune：以能耗约束 GPU 推理共享（SOSP 2026）

> **原题**：Beyond Utilization: Energy-Conscious GPU Sharing for Inference Serving

> **证据边界**：未取得全文；只依据会议题录与公开摘要。系统名 EnerTune 来自公开实现，论文标题本身没有这一名称。

## 问题与动机

高 GPU 利用率不保证低能耗：共置可能提高频率/功耗，也可能触发干扰，导致在同一 SLO 下每瓦性能更差。摘要介绍能耗感知 GPU 共享：用性能与功耗模型预测不同任务共置、GPU 分配、频率和批处理配置的成本，再把能耗纳入 placement/bin-packing 决策。相对 [[MorphX-SOSP26]] 的 kernel 级资源伸缩，它主要在集群放置和配置层优化。

## 关键观察 / 隐含假设

- GPU 利用率和每请求能耗不是同一指标；任务共置可能改变频率、排队和功率曲线。
- 能耗感知放置依赖预测模型在新任务和动态 batch 下仍足够准确，并且调频/迁移响应速度赶得上负载变化。

## 核心方法

摘要说明用性能与功耗模型选 GPU 放置、频率、批次等配置，并考虑共置时的功率。没有全文，不能确认具体优化目标函数、模型校准方法或是否有在线反馈闭环。

## 实验与结果

公开摘要给出能耗降低 **1.4–2.3 倍**、SLO 条件下功耗改善 **1.3–2.6 倍**的范围。没有全文，无法确认每个区间的基线、模型类别、硬件、工作负载到达分布以及是否计入迁移或重新调频成本。用户原先提及 thermal throttling，但公开摘要不足以确认这是论文实验里的主瓶颈，故不当作已证明事实。

## 批判性分析

### 论证链条

把功耗纳入放置目标可解释为何同利用率下有不同能耗；摘要的倍数还需结合 SLO、基线和能耗计量口径判断。

### 假设压力测试

变长生成、突发到达或热管理变化可能使离线功率模型失准，调频控制周期也可能跟不上负载。

### 实验可信度

没有全文无法核对模型类型、GPU 代际、P99 保证和功率仪表；摘要区间仅支持后续阅读优先级判断。

- 预测模型在新 GPU 代际、变长 LLM 请求和突发负载下可能失准；尤其应核验 P99 SLO 而非只看平均功耗。
- 频率调节与任务搬迁有时间尺度，短请求是否能及时受益要看控制周期。
- 多租户公平性与能耗账单归属可能影响实践部署，摘要没有足够信息。

## 局限与后续工作

- 拿到全文后固定 P99 SLO，在同一混合负载上对照焦耳/请求、功率峰值与共置干扰，核对各倍数分母。

## 相关

- **同会议**：[[SOSP-2026]]；**相邻系统**：[[MorphX-SOSP26]]、[[Janus-SOSP26]]。
- **会议记录**：[SOSP DOI](https://doi.org/10.1145/3830418.3843913)；**代码**：[EnerTune](https://github.com/UT-SysML/EnerTune)。
