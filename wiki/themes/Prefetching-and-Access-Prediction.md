---
type: theme
topic: Prefetching-and-Access-Prediction
theme_kind: lens
member_tag: lens/prefetching-and-access-prediction
paper_count: 6
first_generated: 2026-10-27
last_updated: 2026-10-27
tags: [topic-overview, optimization-method]
---

# 预取与访问预测

本专题按优化方法组织站内论文。它可与 [[KV-Cache-Systems]]、[[LLM-Serving]] 等按研究对象组织的专题交叉；一篇论文可属于多个方法专题。

## 纳入范围

利用未来访问的可预测性提前备好数据或推测访问范围，减少等待；需区分预取命中、额外流量和过期风险。核心名单按现有论文笔记人工确认，标签用于导航，不自动决定成员。只有摘要的论文标为「仅公开摘要」。

## 核心论文

- [[Chen-LLMDataPipelines-OSDI26]]
- [[ECHO-OSDI26]]
- [[FlowANN-OSDI26]]
- [[Helmsman-OSDI26]]
- [[KAIROX-OSDI26]]
- [[Strata-OSDI26]]

## 主题导读

[[ECHO-OSDI26]] 预取稀疏注意力将读取的 KV，[[Strata-OSDI26]] 在排队阶段把外部存储命中的 KV 预取到主机，[[KAIROX-OSDI26]] 预测下一层活跃神经元。[[Chen-LLMDataPipelines-OSDI26]] 利用已知训练计划复制热点文件；[[Helmsman-OSDI26]] 预测查询扫描范围。预测对象和允许误判的代价各不相同。
