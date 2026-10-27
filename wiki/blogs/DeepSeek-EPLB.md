---
type: blog
title: "DeepSeek EPLB：专家复制与分层负载均衡"
author: kaiyuan
published: 2025-03-15
source_url: https://zhuanlan.zhihu.com/p/29963005584
reading_status: reviewed
tags: [llm-serving, moe, expert-parallelism, load-balancing, area/llm-serving, lens/deepseek, area/llm-inference]
last_updated: 2026-10-27
---

# DeepSeek EPLB：专家复制与分层负载均衡

> **原文**：[kaiyuan《MoE并行负载均衡：EPLB的深度解析与可视化》](https://zhuanlan.zhihu.com/p/29963005584) · 2025-03-15 · 作者：kaiyuan

> **一句话总结**：DeepSeek 的 EPLB 根据各逻辑专家的估计负载，决定哪些热门专家需要副本、各副本放到哪张 GPU；kaiyuan 的博客用可视化解释了分层与全局放置的区别，也提醒读者把负载预测、方案计算和权重搬迁分开看。

**阅读范围与引用**：已用本地知乎抓取工具阅读博客正文，并对照 [DeepSeek 官方 EPLB 说明](https://github.com/deepseek-ai/EPLB/blob/main/README.md)与 [算法源码](https://github.com/deepseek-ai/EPLB/blob/main/eplb.py)。原文图片未逐张核对；下文的数字例子是作者的示意，不是吞吐或延迟实验。本文只概括机制，不转载原文图像或代码。

## 热门专家为什么需要副本

MoE 的路由会让不同专家接到不同数量的 token。EP 把专家分配到多张 GPU 后，最慢的 GPU 会拖住一次专家计算。kaiyuan 用四个专家、两张 GPU 的玩具例子说明：如果两卡负载是 75% 和 25%，只调整放置顺序可以变成约 45% 和 55%；给热门专家增加一个副本并分流，则可构造约 50% 和 50% 的分配。前者要重排现有专家，后者多占显存。示意比例只说明取舍，不代表 DeepSeek 生产负载的测量结果（[原文 §1](https://zhuanlan.zhihu.com/p/29963005584)）。

EPLB 把两步接起来：先为高负载的逻辑专家分配更多物理副本，再把副本打包到设备上。官方 README 明确把它描述为**依据估计专家负载生成复制与放置方案**的算法，同时说明负载如何预测不在开源仓库范围内（[DeepSeek README](https://github.com/deepseek-ai/EPLB/blob/main/README.md)）。

## 一次重平衡实际算什么

`rebalance_experts(weight, num_replicas, num_groups, num_nodes, num_gpus)` 的 `weight` 是每层、每个**逻辑专家**的负载估计。这里容易看错的是 `num_replicas`：它表示**复制后的物理专家总数**，不是额外副本数。官方例子每层有 12 个逻辑专家，`num_replicas=16`，所以新增 4 个副本，放到两台机器共 8 张 GPU 上。输出包含物理到逻辑、逻辑到物理的映射，以及各逻辑专家的副本数（[官方 README 示例](https://github.com/deepseek-ai/EPLB/blob/main/README.md)、[源码入口](https://github.com/deepseek-ai/EPLB/blob/main/eplb.py)）。

源码分配冗余名额时，反复找当前 `weight / 副本数` 最大的专家并增加一个副本；随后按估计负载给物理副本打包，限制每个设备的专家槽位数。这是基于当前输入统计的启发式放置。博客把逻辑专家、副本和实际部署的物理专家画成映射图，有助于分清每个逻辑专家的副本数与副本的设备位置（[原文 §2](https://zhuanlan.zhihu.com/p/29963005584)、[DeepSeek 源码](https://github.com/deepseek-ai/EPLB/blob/main/eplb.py)）。

## 分层放置与全局放置

当专家组数能被节点数整除时，EPLB 可以先把**组**分配到节点，再在每个节点内复制热门专家，最后把物理专家放到该节点的 GPU。这样利用 DeepSeek-V3 的组内路由，尽量少让 token 跨节点传输。官方说明将这种分层方案用于较小 EP 规模的预填充场景；原文通过 `num_groups=4`、`num_nodes=2` 的图解释了组与节点的关系（[DeepSeek README，Hierarchical Load Balancing](https://github.com/deepseek-ai/EPLB/blob/main/README.md)、[原文 §2–3](https://zhuanlan.zhihu.com/p/29963005584)）。

不满足上述分组条件时，代码退化为**全局放置**：复制和打包不再受原来的专家组约束。官方说明它可用于 EP 规模较大的解码场景；博客用 `num_groups=1` 展示原专家与副本可能跨节点分布。预填充与解码是这两种策略的典型使用场景，真正选择哪条分支由 `num_groups % num_nodes` 决定，不能仅凭阶段名称判断（[官方源码](https://github.com/deepseek-ai/EPLB/blob/main/eplb.py)、[原文 §3](https://zhuanlan.zhihu.com/p/29963005584)）。

## 动态系统还缺哪些环节

kaiyuan 把完整运行过程拆成统计或预测专家热度、计算新放置、执行权重搬迁三个环节。这是博客提出的系统视角；DeepSeek 开源的 `eplb.py` 主要覆盖中间的方案计算。官方 README 也明确说负载预测不在仓库内，历史统计的移动平均只是可能的输入方式。源码输出的是映射，没有包含搬运权重、切换请求路由和故障恢复的执行器（[原文 §1.2 与问题讨论](https://zhuanlan.zhihu.com/p/29963005584)、[DeepSeek README](https://github.com/deepseek-ai/EPLB/blob/main/README.md)）。

博客判断频繁调整会带来权重重载成本，因此不宜默认每轮迭代都重排；这属于作者的工程分析，文章没有测量最优调整周期。若要把方案接进推理引擎，还需测专家热度漂移、复制占用的 HBM、跨节点 token 流量、搬迁暂停时间与 P99 延迟。只看最终每卡的估计 token 数，无法判断实际服务收益。

## 引用与关联

- **主要讲解**：[kaiyuan，2025，《MoE并行负载均衡：EPLB的深度解析与可视化》](https://zhuanlan.zhihu.com/p/29963005584)。本文的 75%/25% 玩具例子、动态闭环问题和分组可视化取自该文。
- **官方实现**：[DeepSeek EPLB](https://github.com/deepseek-ai/EPLB)；用于核对算法覆盖范围、分层条件、`num_replicas` 含义和返回映射。
- **作者的辅助工具**：[EPLB_visualization](https://github.com/CalvinXKY/EPLB_visualization)，在原算法上增加输入热图和部署图；这些图帮助理解映射，不构成性能评测。
- **Wiki 关联**：[[LLM-Serving]]、[[DeepSeek]]、[[DeepSeek-V4.1-Flash-Serving]]。后一篇讨论更完整的生产推理部署，不能据此认定本博客中的可视化例子就是其线上 EPLB 配置。
