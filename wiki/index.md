# AI 基础设施 Wiki

## 致谢

感谢 [lqhl/awesome-system-papers](https://github.com/lqhl/awesome-system-papers/) 开放系统论文收藏与整理成果。本 Wiki 在该仓库基础上整理 AI 基础设施方向的论文与阅读笔记。

## 关于本站

本分支目前整理 OSDI 2026 的 49 篇 AI 基础设施论文、用户人工选定的 SOSP 2026 的 18 篇、SOSP 2025 论文 [[KTransformers-SOSP25]]、SIGCOMM 2026 论文 [[DualPath-SIGCOMM26]]、arXiv 预印本 [[DSpark-arXiv26]]、[[HiSparse-arXiv26]]、[[HySparse-arXiv26]] 与 [[HySparse2-arXiv26]]，以及技术报告 [[UnlimitedOCR2-TechReport26]]。SOSP 2026 中 11 篇已按全文阅读，7 篇只有公开摘要；每篇页面和会议综述都标明证据等级。

[[FAST-2026]] 和 [[MLSys-2026]] 正在筛选与整理，当前结果与待复核状态见会议页。

## 论文来源

- [[OSDI-2026]] — 49 篇：推理服务、KV 缓存、训练与 RL、GPU 软件栈、可靠性、检索与数据路径。
- [[SOSP-2025]] — 指定收录 1 篇：KTransformers 的 CPU/GPU 混合 MoE 推理。
- [[SOSP-2026]] — 人工选定 18 篇：推理与服务、训练与工作流、GPU 内核与调度、模型存储、可观测性。
- [[SIGCOMM-2026]] — 指定收录 1 篇：分离式 KV cache 存储 I/O 与智能体推理。
- [[FAST-2026]] — 7 篇已建页，正在逐篇复核；包括 KV 存储、模型加载与训练检查点。
- [[MLSys-2026]] — 正在筛选；已建 39 篇笔记，其中 27 篇完成复核，尚非完整名单。
- arXiv 2026 — [[DSpark-arXiv26]]：按置信度调度验证的半自回归推测解码；[[HiSparse-arXiv26]]：稀疏注意力解码的分层 KV 管理；[[HySparse-arXiv26]]、[[HySparse2-arXiv26]]：跨层共享 KV 的混合稀疏注意力。
- 技术报告 2026 — [[UnlimitedOCR2-TechReport26]]：稀疏视觉参考与分块长文档解析。

## 主题

### 主线

- [[LLM-Inference]] — 33 篇论文 | 按稀疏化、量化、内核、复用、投机解码等十类方法查找。
- [[LLM-Serving]] — 23 篇论文 | 批处理、阶段调度、路由、模型放置、数据路径与生产观测。
- [[LLM-Training]] — 20 篇论文 | 并行训练、RL 后训练、权重同步、数据供给、容错和调试。

具体方法在这三页内部组织。按论文的主要问题归类，核心成员均已有全文笔记；不符合范围的内容仍可从论文和会议目录阅读。

### 专项

- [[KV-Cache-Systems]] — 20 篇论文 | KV cache 的共享、驻留、搬运与复用。
- [[DeepSeek]] — 2 篇核心论文 | DeepSeek 模型与推理基础设施资料。

专项可与主线交叉；博客另列，不计入论文数。

## 博客

- [[vLLM-Hybrid-HiSparse]] — vLLM 稀疏 KV 动态卸载的机制与资料导航；所属专题：[[LLM-Inference]]、[[KV-Cache-Systems]]。
- [[DeepSeek-V4.1-Flash-Serving]] — DeepSeek V4.1 Flash 大规模推理部署实践；所属专题：[[LLM-Serving]]、[[KV-Cache-Systems]]、[[DeepSeek]]。
- [[DeepSeek-EPLB]] — kaiyuan 解读 DeepSeek 的专家复制与分层负载均衡；所属专题：[[LLM-Inference]]、[[DeepSeek]]。
- [[Zartbot-DualPath]] — zartbot 的 DualPath 讲解；目前仅记录原文入口，尚未核对正文。所属专题：[[LLM-Serving]]、[[DeepSeek]]。

## 阅读入口

- [[LLM-Inference]]、[[LLM-Serving]]、[[LLM-Training]]、[[KV-Cache-Systems]]、[[DeepSeek]]
