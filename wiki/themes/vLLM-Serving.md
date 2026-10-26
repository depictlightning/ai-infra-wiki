---
type: theme
topic: vLLM Serving
theme_kind: lens
member_tag: lens/vllm
paper_count: 0
first_generated: 2026-10-24
last_updated: 2026-10-24
tags: [topic-overview, vllm, llm-serving, pagedattention]
---

# vLLM 推理系统专题

本专题围绕 vLLM 的系统演进组织阅读：从 PagedAttention 论文中的单模型服务引擎，到 V1 的运行时重构，再到 vLLM-Omni 对多模态、多阶段生成的支持。当前先搭阅读骨架，后续再补版本时间线、组件细节和论文对照。

## 纳入范围与状态

重点收录 vLLM 项目自身的论文、技术报告和官方架构资料，以及直接集成 vLLM 的系统研究。只把 vLLM 当基线或在相关工作中提及的论文，列为邻接材料，不自动算作专题核心成员。当前分支还没有 vLLM 自身的站内论文页，因此 `paper_count` 为 0；下方外部来源是待整理入口，不代表已经完成全文笔记。

## 核心论文

待建立 vLLM 自身的站内论文页；当前成员数为 0。

## 官方论文与技术报告

- **vLLM 基础论文**：[Efficient Memory Management for Large Language Model Serving with PagedAttention](https://arxiv.org/abs/2309.06180)，SOSP 2023。作为专题起点，后续补入本站论文页。[^pagedattention]
- **vLLM-Omni 论文**：[vLLM-Omni: Fully Disaggregated Serving for Any-to-Any Multimodal Models](https://arxiv.org/abs/2602.02204)，arXiv 2026。[^omnipaper]
- **vLLM-Omni 技术报告**：[vLLM-Omni Technical Report: A Unified Serving Runtime for Omni-Modality Generation](https://arxiv.org/abs/2610.09307)，arXiv 2026。官方项目说明该报告覆盖多模态生成服务运行时。[^omnireport]

## 官方架构资料

- **V1 架构演进**：[vLLM V1: A Major Upgrade to vLLM's Core Architecture](https://vllm.ai/blog/2025-01-27-v1-alpha-release)，vLLM Team，2025。作为 V0 到 V1 的设计入口。[^v1blog]
- **V1 运行时文档**：[vLLM V1 Guide](https://docs.vllm.ai/en/stable/usage/v1_guide/)，作为组件与术语索引；具体行为需随版本回查。[^v1docs]
- **系统全景讲解**：[Inside vLLM: Anatomy of a High-Throughput LLM Inference System](https://vllm.ai/blog/2025-09-05-anatomy-of-vllm)，Aleksa Gordic，2025。作为社区长文入口，待核对并整理引用。[^anatomy]

## 专题结构

| 阅读主线 | 后续整理的问题 |
|---|---|
| 版本与运行时架构 | v0、V1 与 vLLM-Omni 的组件边界和演进关系 |
| 请求执行与调度 | continuous batching、prefill/decode、抢占与多阶段执行 |
| KV cache 与内存 | PagedAttention、prefix caching、跨设备搬运与生命周期 |
| GPU 执行 | attention kernel、CUDA Graph、编译、并行与通信 |
| 部署与多模态 | 多实例路由、弹性、硬件后端、音频/图像/视频/动作工作流 |

## 站内关联论文候选

这些论文来自其他研究组，当前只作为 vLLM 周边问题的阅读入口；它们不属于 vLLM 自身成果，也暂不计入核心成员。

- **调度与服务**：[[LMetric-OSDI26]]、[[BatchGen-OSDI26]]、[[EcoServe-OSDI26]]。
- **KV cache 与内存**：[[Strata-OSDI26]]、[[DirectKV-OSDI26]]、[[ECHO-OSDI26]]、[[Prism-OSDI26]]。
- **GPU 执行与通信**：[[MPK-OSDI26]]、[[VTC-OSDI26]]、[[Alibaba-ASI-OSDI26]]、[[UEP-OSDI26]]。
- **生产运行与弹性**：[[Janus-SOSP26]]、[[StriaTrace-OSDI26]]、[[OpenTela-OSDI26]]。

## 后续补齐

- 将官方论文与技术报告整理为站内论文页，并逐项登记版本和来源，避免遗漏 vLLM 项目自身发布的报告。
- 按上表组织系统组件和版本演进，再从站内候选中筛出直接集成、基线比较与仅背景提及三类论文。
- 对性能数字、实现细节和跨版本结论留待后续全文分析；本页当前不作这些判断。

## 参考文献

[^pagedattention]: SOSP 2023 会议论文，作者 Woosuk Kwon 等，题名：[Efficient Memory Management for Large Language Model Serving with PagedAttention](https://arxiv.org/abs/2309.06180)。arXiv:2309.06180。
[^omnipaper]: vLLM-Omni 多模态服务论文，作者 Peiqi Yin 等，题名：[vLLM-Omni: Fully Disaggregated Serving for Any-to-Any Multimodal Models](https://arxiv.org/abs/2602.02204)。arXiv:2602.02204, 2026。
[^omnireport]: vLLM-Omni 团队的多模态服务技术报告，题名：[vLLM-Omni Technical Report: A Unified Serving Runtime for Omni-Modality Generation](https://arxiv.org/abs/2610.09307)。arXiv:2610.09307, 2026。
[^v1blog]: vLLM 官方团队介绍 V1 架构的博客，题名：[vLLM V1: A Major Upgrade to vLLM's Core Architecture](https://vllm.ai/blog/2025-01-27-v1-alpha-release)。发布日期：2025-01-27。
[^v1docs]: vLLM V1 的官方使用文档：[V1 Guide](https://docs.vllm.ai/en/stable/usage/v1_guide/)。访问日期：2026-10-24。
[^anatomy]: Aleksa Gordic 对 vLLM 推理系统的长文介绍，题名：[Inside vLLM: Anatomy of a High-Throughput LLM Inference System](https://vllm.ai/blog/2025-09-05-anatomy-of-vllm)。vLLM Blog, 2025-09-05。
