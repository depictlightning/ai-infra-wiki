---
type: paper
name: ContextPilot
full_title: "ContextPilot: Fast Long-Context Inference via Context Reuse"
authors: [Yinsicheng Jiang, Yeqi Huang, Liang Cheng, Cheng Deng, Xuan Sun, Luo Mai]
venue: MLSys
year: 2026
tags: [llm-inference, llm-serving, kv-cache, prefix-caching, context-reuse, rag, area/llm-inference, area/llm-serving, area/kv-cache-systems]
source_pdf: "[[38b3eff8baf56627478ec76a704e9b52.pdf]]"
source_md: "[[38b3eff8baf56627478ec76a704e9b52]]"
review_status: complete
evidence_level: full-text
last_reviewed: 2026-10-28
---

# ContextPilot：重排与去重长上下文以复用前缀（MLSys 2026）

> **原题**：ContextPilot: Fast Long-Context Inference via Context Reuse

> **一句话总结**：ContextPilot 在推理引擎之前索引检索上下文，把共享内容排列成可命中的前缀，并对多轮重复内容去重；它用文本注释提示原始顺序和被删除内容的位置，以缓解质量变化。[^paper]

**版本与阅读状态**：已通读与会议正式 PDF 一致的 MinerU 正文及论文集单独发布的四页附录，并回看架构、实验表和正文公式。附录解析保存在 [[b0131b6ee02a00b03fc3320176fec8f5-supplement]]。[^paper][^supplement]

## 问题与动机

RAG 和多轮智能体请求可能反复包含相同文档，但检索顺序和历史上下文不同。常见前缀缓存要求 token 前缀一致：即使两个请求包含相同文档，顺序不同也可能无法复用其 KV cache（§1–2）。[^paper]

ContextPilot 试图改变输入的组织方式，使共享文档更可能成为相同前缀；同时删除已经出现在对话历史中的重复块。它在应用上下文层处理提示词，依赖推理引擎现有前缀缓存，而不直接拼接不同上下文产生的 KV（§3）。[^paper]

## 关键观察 / 隐含假设

- **工作负载有重叠**：多会话命中依赖不同请求访问相同上下文，多轮去重依赖历史内容仍然可供模型访问（§3.1、附录 C）。[^paper][^supplement]
- **改变顺序会改变模型输入**：缓存只对重排后实际相同的前缀做精确复用；重排和注释不保证与原提示词的输出分布或答案完全一致，必须按任务评估（§5.3）。[^paper]
- **索引必须反映缓存状态**：如果引擎已淘汰相关前缀，索引仍指向它会高估可复用内容；论文使用请求 ID 和淘汰通知同步索引（§4.1）。[^paper]
- **批量调度有前提**：离线多会话实验可预先获取上下文并统一建索引；在线到达不一定有同样的重排窗口（§7.1）。[^paper]

## 核心方法

### 索引与上下文对齐

索引树以上下文块 ID 表示文档组合，节点记录共享块、搜索路径和访问频率。初始化时根据两个上下文的重叠数量与位置差异做层次聚类，内部节点表示共享部分；新请求沿树查找匹配，将匹配文档移到前面，其余文档按原顺序追加（§4–5）。[^paper]

例如，请求原始文档顺序为 `[B, A, C]`，缓存中已有 `[A, B]`，系统可将输入改为 `[A, B, C]`，让引擎复用已有前缀。因为这改变了检索排序，系统在问题附近附加原始相关性顺序的注释，而不是声称输入语义没有变化（§5.1、§5.3）。[^paper]

正文距离公式在 MinerU 中缺失，已回看正式 PDF 第 6 页：它以共享块占比和共享块位置差计算距离。该公式对零重叠未给出分母处理；当前开源 CPU 实现将零重叠距离设为 1，这一代码行为不能自动视作论文实验版本的证明。[^paper][^code]

### 调度、去重和注释

系统根据索引搜索路径将相近请求分组，并优先执行较长共享前缀，使它们在缓存淘汰之前相继运行。该策略提高复用机会，但其收益取决于排队窗口和缓存容量（§5.2）。[^paper]

多轮去重先识别此前轮次已出现的完整块，再对新块进行内容定义切分，识别跨块重复片段；重复内容由指向先前位置的注释替换。若历史已被截断、文件发生变化或引用超出授权范围，不能假定注释仍能恢复原有信息（§6）。[^paper]

## 设计取舍

- **复用与质量**：对齐和删除重复文本会改变提示词，注释在所测任务中有帮助，但并非语义等价证明。
- **聚类与到达延迟**：离线批处理可全局组织请求；实时服务需权衡索引、等待及调度带来的 TTFT。
- **缓存命中与状态同步**：更大缓存提高复用机会，索引需要跟踪引擎淘汰；跨会话使用还需保持原有数据访问边界。

## 实验与结果

正文使用 SGLang 0.4.6 和 vLLM 0.10.0，并在 H100、A6000 等设备上评测 RAG、多轮对话、智能体和端侧场景。多会话 RAG 使用预取上下文的离线模式；多轮和 Mem0 使用冷启动在线模式（§7）。[^paper]

| 结果 | 条件与边界 |
|---|---|
| 多会话 prefill | MultihopRAG、Qwen3-32B、top-k=15 时，表 2 的 prefill 吞吐为 36,296.1，相比 RadixCache 的 17,682.6 约 2.05×；对应 F1 从 60.4 变为 64.4，不能推断所有任务都提高质量（§7.1）。 |
| DeepSeek-R1 | 附录表 6 的 16×H20 实验中，MultihopRAG 的 prefill 吞吐从 9,636.69 增至 17,498.75，约 1.81×；NarrativeQA 约 1.52×，均相对附录 Vanilla 基线。[^supplement] |
| 多轮请求 | MT-RAG、Qwen3-4B 的 TTFT 从 RadixCache 的 0.44 秒降至 0.22 秒；准确率使用 LLM 评判，不等同于逐 token 一致（§7.1、表 3）。 |
| 智能体任务 | RTX 5090、OpenClaw 文档分析任务中，平均 prefill 从 7.2 秒降至 2.6 秒，平均总耗时从 26.1 秒降至 20.7 秒；编码任务的总耗时下降较小（§7.2、表 4）。 |
| 组件开销 | 附录报告 A6000、2,000 个请求、k=15 的搜索、对齐和去重合计约 0.7 ms / 请求；不包含任何场景下均固定不变的全系统成本。[^supplement] |

质量并非总是提升：正文表 2 中 QASPER 的 Qwen3-32B F1 从 36.0 降到 34.9；Mem0 的 k=100 场景也报告从 0.437 降到 0.420。论文的“保持质量”应按具体模型和任务理解（§7.1–7.2）。[^paper]

## 论断—证据表

| 论断 | 证据 | 评测边界 | 置信度 |
|---|---|---|---|
| 重排与调度可增加所测工作负载的前缀复用 | §7.4、图 7；附录 D | 取决于文档重叠、缓存容量和调度模式 | 中高 |
| 注释在部分任务可弥补顺序变化的质量损失 | §7.1、表 2；附录表 7 | 存在质量下降的任务，指标不证明语义等价 | 中 |
| 可以无条件删除多轮重复上下文 | §6–7 | 需要历史仍可见且注释被模型正确使用 | 低 |

## 批判性分析

### 论证链条

论文先测文档重叠，再构造可共享前缀，通过任务结果和组件消融检查收益。DeepSeek-R1 的详细数值位于独立附录；不同实验的吞吐、TTFT 和端到端耗时口径不能直接相加。[^paper][^supplement]

### 假设压力测试

若文档高度个性化、跨用户不可共享，或对话历史超出窗口，索引和去重可能缺少收益或无法引用先前内容。排序本身携带相关性，模型未遵循注释时可能遗漏证据；内容更新也要求可靠失效。[^paper]

### 实验可信度

论文给出任务质量与性能对照，但离线多会话预先构建索引；线上动态到达、租户隔离及模型对注释的敏感性需要单独测量。与 CacheBlend 的结果只代表文中所测配置，不能推断其所有实现和质量设置。[^paper]

## 局限与后续工作

- 在动态到达、有限缓存和上下文截断条件下，同时测量排队、TTFT、命中率与任务质量。
- 验证文档更新、租户权限和引用注释的正确性，并与不改写提示词的前缀缓存基线比较。

## 相关

- [[LLM-Inference]]、[[LLM-Serving]]、[[KV-Cache-Systems]]：前缀复用和请求组织。
- [[BatchLLM-MLSys26]]：离线分组共享前缀；ContextPilot 额外调整文档顺序和多轮重复内容。
- [[Stream2LLM-MLSys26]]：上下文流式到达时的最长公共前缀和缓存管理。

## 参考文献

[^paper]: Yinsicheng Jiang 等. 《ContextPilot: Fast Long-Context Inference via Context Reuse》. MLSys，2026。[原文](https://proceedings.mlsys.org/paper_files/paper/2026/hash/b0131b6ee02a00b03fc3320176fec8f5-Abstract-Conference.html)。
[^supplement]: Yinsicheng Jiang 等. 《ContextPilot: Fast Long-Context Inference via Context Reuse》补充材料. MLSys，2026。[原文](https://proceedings.mlsys.org/paper_files/paper/2026/file/b0131b6ee02a00b03fc3320176fec8f5-Supplemental-Conference.pdf)。
[^code]: EfficientContext. 《ContextPilot: CPU distance computation》. GitHub。[原文](https://github.com/EfficientContext/ContextPilot/blob/main/contextpilot/context_index/compute_distance_cpu.py)。
