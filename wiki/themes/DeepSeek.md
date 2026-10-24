---
type: theme
topic: DeepSeek
theme_kind: lens
member_tag: lens/deepseek
paper_count: 2
first_generated: 2026-10-24
last_updated: 2026-10-24
tags: [topic-overview, deepseek, llm-serving, moe, kv-cache]
---

# DeepSeek：从模型结构到推理基础设施

根据 [DeepSeek-V2 报告](https://arxiv.org/abs/2405.04434)、[DeepSeek-V3.2 报告](https://arxiv.org/abs/2512.02556)和 [[DualPath-SIGCOMM26]]，MLA、MoE、稀疏注意力及多轮智能体请求分别改变 KV 占用、专家通信、长上下文访存和旧 KV 取回的成本。本专题按这些问题组织阅读材料，并把模型报告、开源实现与生产部署说明放在一起看。

## 纳入范围与证据

收录 DeepSeek 自身的模型或基础设施、直接研究其生产推理路径的工作，以及已有的相关博客。`## 核心论文` 只列本站已完成全文笔记的论文；官方技术报告虽有原文链接，目前尚无本站全文笔记，列在后面的阅读路线中。其他论文即使使用 DeepSeek 模型做实验，也不因此成为本专题成员。博客的阅读状态以各自页面为准；外部讲解仅作为阅读线索，不据其标题推断技术结论。

**引用说明**：下方论文、官方报告、代码仓库和博客链接均指向原作者的作品。技术机制是据所列来源写的转述，生产数据只代表原作者的报告；「从已有笔记能看到什么」中的跨材料判断与待测问题是本站的综合分析。本站不转载他人的原文段落、图表或代码。文末第三方讲解尚未核对正文和作者，只标为引用入口，不据此撰写技术结论。

## 核心论文

- [[DSpark-arXiv26]] — 草稿生成与验证长度调度；已有全文笔记，论文是 arXiv 预印本，包含 DeepSeek-V4 预览版的线上实验。
- [[DualPath-SIGCOMM26]] — 在预填充与解码节点之间分担旧 KV 的存储读取；已有 SIGCOMM 2026 正式版全文笔记，实验使用 DeepSeek 的 3FS 与分离式推理部署。

## 相关博客

- [[DeepSeek-V4.1-Flash-Serving]] — 引用 [DeepSeek 官方生产部署文章](https://zhuanlan.zhihu.com/p/2095560295930508953)；已阅读正文，架构图和性能图尚未逐一核对。
- [[DeepSeek-EPLB]] — 引用 [kaiyuan 的 EPLB 讲解](https://zhuanlan.zhihu.com/p/29963005584)；已阅读原文并核对官方 EPLB 代码。
- [[Zartbot-DualPath]] — 引用 [zartbot 的 DualPath 讲解](https://mp.weixin.qq.com/s/vrUw-a_6QiQkbiBaC3rC6Q)；目前仅确认链接，尚未核对正文。

## 按问题阅读

| 想弄清的问题 | 先读的官方材料 | 接着看本站内容 |
|---|---|---|
| 为什么长上下文首先压迫 KV cache？ | [DeepSeek-V2 报告](https://arxiv.org/abs/2405.04434)介绍 MLA；[DeepSeek-V3 报告](https://arxiv.org/abs/2412.19437)继续讨论 MLA、MoE 与多 token 预测。 | [[KV-Cache-Systems]]、[[LLM-Serving]]；先区分模型降低每 token 状态与系统搬运既有状态。 |
| 专家分布不均时，通信和放置分别怎样处理？ | [DeepEP](https://github.com/deepseek-ai/DeepEP)提供 MoE 专家并行通信实现；[EPLB](https://github.com/deepseek-ai/EPLB)给出依据估计负载生成专家副本及放置方案的代码。 | [[DeepSeek-EPLB]]；开源 EPLB 不包含完整的线上负载预测、权重搬迁与切换流程。 |
| 稀疏注意力怎样改变缓存与索引的成本？ | [DeepSeek-V3.2 报告](https://arxiv.org/abs/2512.02556)介绍 DeepSeek Sparse Attention；[DeepSeek-V4 报告](https://arxiv.org/abs/2606.19348)介绍 CSA 与 HCA 混合注意力；[DeepSeek-V4.1-Flash 报告](https://arxiv.org/abs/2609.19969)介绍 CSA2、跨层复用与 FP4 KV。 | [[DeepSeek-V4.1-Flash-Serving]]；相关模型报告目前仅核对官方入口与摘要，细节请回原文。 |
| 多轮智能体请求如何少算旧上下文、少等旧 KV？ | [3FS](https://github.com/deepseek-ai/3FS)是 DeepSeek 开源的分布式文件系统；[DeepSpec](https://github.com/deepseek-ai/DeepSpec)提供推测解码算法的训练和评估代码。 | [[DualPath-SIGCOMM26]]看旧 KV 读路径，[[DSpark-arXiv26]]看目标模型验证预算，[[DeepSeek-V4.1-Flash-Serving]]看生产系统如何安排缓存寿命与阶段分离。 |

## 本站的综合判断与待验证问题

据 [[DualPath-SIGCOMM26]] 和 [[DSpark-arXiv26]] 两篇论文，一次请求可能分别在旧 KV 读取和目标模型验证上等待：前者在高前缀复用时借用解码节点空闲的存储网卡，后者按候选 token 被接受的概率和引擎负载分配验证长度。前者依赖分离的预填充、解码节点及两侧可用的网络带宽；后者依赖接受率估计和吞吐曲线。本站判断：两篇论文的收益不能直接相加，因为它们没有在相同工作负载上给出联合实验。

据 [DeepSeek 的生产部署文章](https://zhuanlan.zhihu.com/p/2095560295930508953)，其系统把可长期复用的 Block KV 放到共享 3FS，把短期复用的 SWA KV 留在本地，并在缺失时做有限重算。这是原作者对其生产部署的陈述；文章没有提供与 DualPath 同条件的直接对照。据 [kaiyuan 的 EPLB 讲解](https://zhuanlan.zhihu.com/p/29963005584)及 [DeepSeek 官方 EPLB 说明](https://github.com/deepseek-ai/EPLB)，开源放置算法之外还需考虑线上负载预测、权重搬迁与切换。本站将这些环节视作部署时应测量的成本，不把它们归为开源代码已经实现的功能。

这几份材料留下一个具体的系统问题：当模型结构继续压缩每 token 的 KV、同时请求历史变长且专家热度变化时，瓶颈会在 HBM 容量、存储读取、专家通信和验证算力之间移动。要判断哪种优化优先，应在同一请求轨迹上同时记录缓存命中后的读取量、各节点网卡利用率、专家负载偏斜、验证接受长度与尾延迟；现有两篇本站论文没有覆盖这样一组联合测量。

## 外部讲解书签

以下每个链接都是对第三方文章的**引用入口**，来自用户本地的《DeepSeek优质资料》清单；本站尚未逐篇读取，也未核对这些文章的作者和正文，因此暂不署未经确认的作者名，不转述其观点或内容。官方报告及代码应作为技术结论的首要依据。

- **V2 / V3**：[MLA 讲解](https://zhuanlan.zhihu.com/p/16730036197)、[MoE 演进](https://zhuanlan.zhihu.com/p/18565423596)、[MTP 讲解](https://zhuanlan.zhihu.com/p/18056041194)、[V3 报告中文解读](https://zhuanlan.zhihu.com/p/14890557782)。
- **V3.2**：[V3.2-Exp 源码逻辑](https://zhuanlan.zhihu.com/p/1956798179065443580)、[DSA 与 NSA 讲解](https://zhuanlan.zhihu.com/p/1957032283270812718)。
- **V4**：[技术报告解读](https://zhuanlan.zhihu.com/p/2030970565092177542)、[CSA 与 HCA](https://zhuanlan.zhihu.com/p/2031146581542580917)、[SGLang 部署讲解](https://zhuanlan.zhihu.com/p/2040898154392965328)。
- **V4.1**：[报告逐章解读](https://zhuanlan.zhihu.com/p/2081404436598018735)、[CED 与后训练](https://zhuanlan.zhihu.com/p/2081395729059395473)、[CSA2 源码讲解](https://zhuanlan.zhihu.com/p/2084567376754954336)。
- **推测解码与存储**：[DSpark 代码讲解](https://zhuanlan.zhihu.com/p/2054690012256641161)、[3FS 架构分析上篇](https://zhuanlan.zhihu.com/p/1912450944957935863)、[3FS 架构分析下篇](https://zhuanlan.zhihu.com/p/1888175461802172890)。

本页先建立可追溯的阅读路线。V2、V3、V3.2、V4、V4.1 和 3FS 的逐篇全文笔记、跨版本实验矩阵，以及外部讲解的正文核对，仍待后续补充。
