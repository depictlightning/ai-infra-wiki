---
type: paper
name: SANDHI
full_title: "SANDHI: Fine-grained Merging for Memory Efficient Multi-Model Serving"
authors: [Vima Gupta, Oytun Kuday Duran, Nandan Suresh Meda, Ikhyun An, Ganesh Ananthanarayanan, et al.]
venue: SOSP
year: 2026
tags: [multi-model-serving, gpu-memory, model-merging]
source_pdf: ""
source_md: ""
review_status: abstract-only
evidence_level: abstract
last_reviewed: 2026-10-22
---

# SANDHI：多模型服务的细粒度合并（SOSP 2026）

> **原题**：SANDHI: Fine-grained Merging for Memory Efficient Multi-Model Serving

> **证据边界**：目前没有取得论文全文。以下只依据会议元数据与公开摘要；没有核验设计章节、图表、评测配置和代码。数字仅按摘要原意记录。

## 问题与动机

多个同族模型共同服务时，逐个完整驻留会占用大量 GPU 高带宽内存（HBM）。摘要提出“细粒度组件合并”：在满足各模型准确率要求的约束下，让模型共享可兼容的部分，而不是把每个完整权重副本独立放进显存。它与 [[Janus-SOSP26]] 的副本放置不同，直接改变多模型驻留时的权重表示；与 [[TensorDex-SOSP26]] 的仓库无损压缩也不同，目标是在线服务显存。

## 关键观察 / 隐含假设

- 同族模型存在可共享组件，是细粒度合并能节省 HBM 的前提；模型间差异越大，候选共享部分可能越少。
- 合并受任务准确率约束，而非位级无损约束；部署方必须接受并测量这类语义差异。

## 核心方法

公开摘要只说明“在准确率要求下做细粒度组件合并”，没有足够信息还原组件划分、搜索算法或运行时装载协议。可以确认它改变模型驻留表示；具体合并步骤等待全文核对。

## 实验与结果

摘要称在 **3 个模型家族、12 个模型、9 个 benchmark** 上，HBM 占用最多减少 **49.8%**、吞吐最多提升 **2.93 倍**、服务成本约降低 **2 倍**。这些都是摘要中的最佳结果，缺少全文时无法判断它们是否来自同一工作负载、相同准确率门槛和相同基线。

## 批判性分析

### 论证链条

摘要把多模型 HBM 冗余与细粒度合并直接连接，但没有说明准确率约束如何进入选择过程。空间、吞吐与成本三个最高值需要分场景核对。

### 假设压力测试

若模型不共享基座或架构，共享组件可能很少；若准确率阈值极严，合并空间也可能收缩。

### 实验可信度

12 模型、3 家族和 9 benchmark 提供一定覆盖，但摘要未给出逐模型质量、尾延迟和同场景成本明细。

- 共享是否要求同架构、同基座或特定微调方式，摘要不足以确定；跨家族与严格隔离场景的适用性尚不清楚。
- “满足准确率要求”比“位级完全等价”弱。未来拿到全文后应核验每个任务的精度损失、长尾延迟、在线模型更新及合并计算成本。
- 目前不能断言所用合并算法、显存碎片处理方式或推理内核实现。

## 局限与后续工作

- 拿到全文后，在统一准确率门槛下核对 HBM、吞吐和成本三项收益是否同时成立，并检查新模型上线时的合并延迟。

## 相关

- **同会议**：[[SOSP-2026]]；**相邻系统**：[[Janus-SOSP26]]、[[TensorDex-SOSP26]]。
- **会议记录**：[SOSP DOI](https://doi.org/10.1145/3830418.3843911)。
