---
type: theme
topic: Compute-and-State-Reuse
theme_kind: lens
member_tag: lens/compute-and-state-reuse
paper_count: 21
first_generated: 2026-10-27
last_updated: 2026-10-27
tags: [topic-overview, optimization-method]
---

# 计算与状态复用

本专题按优化方法组织站内论文，可与 [[KV-Cache-Systems]]、[[LLM-Serving]] 等系统领域专题交叉。以下是本站依据论文页「核心方法」作的归类；明确的副机制也可纳入，理由在条目中说明。

## 纳入范围

收录明确复用已有数据、计算结果或有效状态以减少额外计算、装载或副本的工作。复用对象可位于寄存器、设备、模型仓库或故障恢复路径；提前执行一次未来计算，本身不等于复用。核心名单由人工确认，标签不自动决定成员；「仅公开摘要」表示尚未取得全文。

## 核心论文

- [[HiSparse-arXiv26]] — 逐请求、逐层保留近期选中的 KV，热缓存命中时避免重复从主机取回。
- [[ACDC-SOSP26]] — 按已知任务前缀排序，并复用重复图片的编码结果。
- [[AEGIS-OSDI26]] — 利用训练原有的确定性重计算作自等价校验，避免另做一份检查计算。
- [[Anchor-SOSP26]] — 独立进程保留健康显存中的对象，worker 重启后重新连接。（仅公开摘要）
- [[Cocoon-OSDI26]] — 按 tile 复用驻留噪声历史，并把两次访问间的噪声合并。
- [[DiFlow-SOSP26]] — 跨工作流复用已加载模型；另支持近似中间缓存，其语义需单独评估。
- [[ECHO-OSDI26]] — 缓存已选 token 的 KV，命中时避免重复从主机取回。
- [[HySparse-arXiv26]] — 后续稀疏层复用前一完整层的 KV 和重要块索引。
- [[HySparse2-arXiv26]] — 共享跨解码器的投影输入，并在层组内复用完整层 KV。
- [[LMetric-OSDI26]] — 路由时利用目标实例已有的前缀 KV，减少新增预填充。
- [[llmovoice-SOSP26]] — 检索并复用跨轮会话状态；对象是应用上下文。
- [[RobustRL-OSDI26]] — 保留已完成 trajectory、工具状态和有效角色，恢复时避免重做 rollout。
- [[SANDHI-SOSP26]] — 在准确率约束下共享多模型的兼容组件；不保证位级无损。（仅公开摘要）
- [[Seer-OSDI26]] — 共享 prompt group 的后缀草稿上下文，并在迁移时复用历史 KV。
- [[Sereno-OSDI26]] — 从提示、已验证输出及被拒草稿中复用 n-gram 候选，仍由目标模型验证。
- [[Strata-OSDI26]] — 复用多层前缀缓存及同批请求的共同加载结果。
- [[TensorDex-SOSP26]] — 精确去重 tensor，并以参考 tensor 加残差表示其他权重。
- [[TensorHub-SOSP26]] — 把已有 worker 上的权重副本作为后续传输源。
- [[TileLoom-OSDI26]] — 分析仿射访问的时空复用，选择循环外提、片上保留或广播。
- [[UnlimitedOCR2-TechReport26]] — 相邻文档块复用边界页视觉 KV 与前块末尾的文本 KV。
- [[Wang-LocalMoEInference-OSDI26]] — 节点内 P/D 共享同一份 DRAM 权重，避免复制整模型。

## 主题导读

[[HySparse-arXiv26]] 共享跨层 KV，[[TileLoom-OSDI26]] 分析片上时空复用，[[TensorDex-SOSP26]] 复用参考 tensor。可靠性工作也按机制判断：[[AEGIS-OSDI26]] 利用已有重计算结果作校验参考，[[RobustRL-OSDI26]] 和 [[Anchor-SOSP26]] 保留仍有效的状态，分别避免重做 rollout 或冷加载。它们的任务目标不同，复用收益也不能互换。
