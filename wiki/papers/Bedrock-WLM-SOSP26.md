---
type: paper
name: Bedrock-WLM
full_title: "Taming Inference Workloads at Global Scale: Foundation Model Serving in Amazon Bedrock"
authors: [Pratik Pankaj Raichura, Somu Perianayagam, Rama Krishna Sandeep Pokkunuri, Divya Jain]
venue: SOSP
year: 2026
tags: [foundation-model-serving, global-routing, admission-control, production]
source_pdf: ""
source_md: ""
review_status: abstract-only
evidence_level: abstract
last_reviewed: 2026-10-22
---

# Amazon Bedrock：全球规模的基础模型推理负载管理（SOSP 2026）

> **原题**：Taming Inference Workloads at Global Scale: Foundation Model Serving in Amazon Bedrock

> **证据边界**：未取得全文；只依据会议题录和公开摘要。以下是生产系统的公开描述，不是我对实现细节的复现。

## 问题与动机

全球推理服务需要在用户配额、模型容量、跨区域流量与延迟之间作决策。摘要描述 Amazon Bedrock 的 workload management（WLM）：覆盖准入控制、流量分类、跨区域路由、请求调度和编排，并把入口路径与执行路径拆开。入口使用局部近似配额等机制，避免每次请求都等待全球强同步；模型/权重及提示缓存相关信息影响路由选择。它与 [[Janus-SOSP26]] 的单集群多模型容量规划同属推理控制面，但部署范围和产品约束更广。

## 关键观察 / 隐含假设

- 跨区域强同步会进入请求关键路径，因此入口需要局部决策；其代价是配额视图可能短时间陈旧。
- 模型部署和提示缓存具有区域局部性，路由不能仅按网络距离选择。

## 核心方法

摘要列出 WLM 的准入、分类、跨区路由、调度和编排层，但未给出每层算法、控制周期或故障策略。因此当前只能把它视作生产架构线索，不能视作可复现的调度方案。

## 实验与结果

摘要给出 **30 多个区域、每分钟数百万请求** 的服务规模，说明问题不是实验室负载。它没有提供可直接用于对照的吞吐提升、P99 降低或成本节约倍数；不能把生产规模写成算法收益。

## 批判性分析

### 论证链条

30 多区域和每分钟数百万请求证明全球调度问题真实存在；它们本身不证明任何一个 WLM 子策略提升了 SLO 或降低了成本。

### 假设压力测试

局部陈旧配额在全区突发或跨区故障时可能累计超配；提示缓存局部性也可能与最低网络时延路由冲突。

### 实验可信度

摘要只有部署规模，没有统一基线、开关组件实验、分位延迟或成本数字，当前不能做定量效果判断。

- 局部配额与跨区域调度可能面对短时超配、区域故障和缓存局部性冲突；这些是基于摘要的研究问题，未由全文核实。
- 需要全文才能区分具体准入策略、路由算法、模型异构性、区域间一致性和故障降级机制。
- 如用于自身架构设计，先关注 WLM 的接口与 SLO/成本权衡，不能从摘要推断可直接复制的配置参数。

## 局限与后续工作

- 拿到全文后核对不同区域失效、突发超配、提示缓存命中率与 P99 延迟，区分系统规模与优化收益。

## 相关

- **同会议**：[[SOSP-2026]]；**相邻系统**：[[Janus-SOSP26]]、[[ACDC-SOSP26]]、[[QProf-SOSP26]]。
- **会议记录**：[SOSP DOI](https://doi.org/10.1145/3830418.3843855)。
