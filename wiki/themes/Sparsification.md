---
type: theme
topic: Sparsification
theme_kind: lens
member_tag: lens/sparsification
paper_count: 6
first_generated: 2026-10-27
last_updated: 2026-10-27
tags: [topic-overview, optimization-method]
---

# 稀疏化与加速

本专题按优化方法组织站内论文，可与 [[KV-Cache-Systems]]、[[LLM-Serving]] 等系统领域专题交叉。以下是本站依据论文页「核心方法」作的归类；明确的副机制也可纳入，理由在条目中说明。

## 纳入范围

收录显式选择、裁剪注意力位置、活跃神经元或检索候选来减少实际处理集合的工作，也包含利用原生稀疏选择设计的执行系统。单纯用代理评估代替训练、把计算移到别处或延后执行，不据此算作稀疏化。核心名单由人工确认，标签不自动决定成员；「仅公开摘要」表示尚未取得全文。

## 核心论文

- [[ECHO-OSDI26]] — 利用原生稀疏注意力的 token 选择按需回载 KV；未提出新的注意力稀疏化算法。
- [[Helmsman-OSDI26]] — 以学习式 nprobe 决定每个查询扫描的簇，裁剪检索候选。
- [[HySparse-arXiv26]] — 少数完整层选择重要块，后续层只关注选中块与局部窗口。
- [[HySparse2-arXiv26]] — 使用 token 级稀疏选择与强制局部窗口，并跨层共享 KV。
- [[KAIROX-OSDI26]] — 预测下一层活跃神经元，只对所需组进行计算和搬运。
- [[UnlimitedOCR2-TechReport26]] — 以 Page Gate 和 Token Sieve 限制当前输出可访问的视觉块。

## 邻近机制

- [[Drs-NAS-OSDI26]] — 主要通过代理指标省去候选模型训练，不能把这部分加速解释为稀疏注意力或激活剪枝。

## 主题导读

[[HySparse-arXiv26]] 与 [[HySparse2-arXiv26]] 改变注意力结构，[[ECHO-OSDI26]] 则利用已有稀疏注意力的选择结果设计缓存；两者在模型与运行时中的位置不同。[[Helmsman-OSDI26]] 的学习式候选裁剪属于检索侧选择。[[Drs-NAS-OSDI26]] 的主要加速来自零成本代理评估，放在本页邻近机制中阅读。
