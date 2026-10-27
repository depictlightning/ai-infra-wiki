---
type: theme
topic: Speculative-Decoding-and-Multi-Token
theme_kind: lens
member_tag: lens/speculative-decoding-and-multi-token
paper_count: 4
first_generated: 2026-10-27
last_updated: 2026-10-27
tags: [topic-overview, optimization-method]
---

# 投机解码与多 Token 生成

本专题按优化方法组织站内论文，可与 [[KV-Cache-Systems]]、[[LLM-Serving]] 等系统领域专题交叉。以下是本站依据论文页「核心方法」作的归类；明确的副机制也可纳入，理由在条目中说明。

## 纳入范围

核心成员以草稿、多 token 候选及目标模型验证减少目标模型的串行生成步骤。树搜索分支推测和确定性核验列为邻近机制，分别说明目标，不当作逐 token 投机解码的实测结果。核心名单由人工确认，标签不自动决定成员；「仅公开摘要」表示尚未取得全文。

## 核心论文

- [[ACDC-SOSP26]] — 依据任务长度、batch 和接受率决定是否启用推测解码及候选长度。
- [[DSpark-arXiv26]] — 以半自回归草稿和置信度调度每个请求的验证预算。
- [[Seer-OSDI26]] — 利用同 prompt group 的后缀上下文生成自适应草稿。
- [[Sereno-OSDI26]] — 允许提前结束草稿层，并用 n-gram 填充和可变验证批次控制带宽。

## 邻近机制

- [[LLM-42-SOSP26]] — 以同模型快路径生成候选，再按固定形状核验确定性；不等同于小模型草稿或减少自回归生成轮次。
- [[SPEX-OSDI26]] — 预测树搜索下一批分支，隐藏 reward 等待；每条分支内部仍按原生成器执行。

## 主题导读

[[DSpark-arXiv26]]、[[Seer-OSDI26]] 和 [[ACDC-SOSP26]] 分别调整草稿、共享上下文或验证长度；[[Sereno-OSDI26]] 用弹性投机解码控制带宽争用。验证、错误草稿和回滚会增加工作量，是否加速取决于接受率、batch 和负载。
