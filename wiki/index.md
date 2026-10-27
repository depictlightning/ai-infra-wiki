# AI 基础设施 Wiki

本分支目前整理 OSDI 2026 的 49 篇 AI 基础设施论文、用户人工选定的 SOSP 2026 的 18 篇、SOSP 2025 论文 [[KTransformers-SOSP25]]、SIGCOMM 2026 论文 [[DualPath-SIGCOMM26]]、arXiv 预印本 [[DSpark-arXiv26]]、[[HiSparse-arXiv26]]、[[HySparse-arXiv26]] 与 [[HySparse2-arXiv26]]，以及技术报告 [[UnlimitedOCR2-TechReport26]]。SOSP 2026 中 11 篇已按全文阅读，7 篇只有公开摘要；每篇页面和会议综述都标明证据等级。

## 论文来源

- [[OSDI-2026]] — 49 篇：推理服务、KV 缓存、训练与 RL、GPU 软件栈、可靠性、检索与数据路径。
- [[SOSP-2025]] — 指定收录 1 篇：KTransformers 的 CPU/GPU 混合 MoE 推理。
- [[SOSP-2026]] — 人工选定 18 篇：推理与服务、训练与工作流、GPU 内核与调度、模型存储、可观测性。
- [[SIGCOMM-2026]] — 指定收录 1 篇：分离式 KV cache 存储 I/O 与智能体推理。
- arXiv 2026 — [[DSpark-arXiv26]]：按置信度调度验证的半自回归推测解码；[[HiSparse-arXiv26]]：稀疏注意力解码的分层 KV 管理；[[HySparse-arXiv26]]、[[HySparse2-arXiv26]]：跨层共享 KV 的混合稀疏注意力。
- 技术报告 2026 — [[UnlimitedOCR2-TechReport26]]：稀疏视觉参考与分块长文档解析。

## 主题

### 主线

- [[LLM-Inference]] — 16 篇论文 | 注意力与 KV、低精度内核、异构执行、解码和输出语义。
- [[LLM-Serving]] — 10 篇论文 | 批处理、阶段调度、路由、模型放置、数据路径与生产观测。
- [[LLM-Training]] — 16 篇论文 | 并行训练、RL 后训练、权重同步、数据供给、容错和调试。

具体方法在这三页内部组织。按论文的主要问题归类，核心成员均已有全文笔记；不符合范围的内容仍可从论文和会议目录阅读。

### 专项

- [[KV-Cache-Systems]] — 9 篇论文 | KV cache 的共享、驻留、搬运与复用。
- [[DeepSeek]] — 2 篇核心论文 | DeepSeek 模型与推理基础设施资料。

专项可与主线交叉；博客另列，不计入论文数。

## 博客

- [[vLLM-Hybrid-HiSparse]] — vLLM 稀疏 KV 动态卸载的机制与资料导航；所属专题：[[LLM-Inference]]、[[KV-Cache-Systems]]。
- [[DeepSeek-V4.1-Flash-Serving]] — DeepSeek V4.1 Flash 大规模推理部署实践；所属专题：[[LLM-Serving]]、[[KV-Cache-Systems]]、[[DeepSeek]]。
- [[DeepSeek-EPLB]] — kaiyuan 解读 DeepSeek 的专家复制与分层负载均衡；所属专题：[[LLM-Inference]]、[[DeepSeek]]。
- [[Zartbot-DualPath]] — zartbot 的 DualPath 讲解；目前仅记录原文入口，尚未核对正文。所属专题：[[LLM-Serving]]、[[DeepSeek]]。

## 阅读入口

- 主线入口：[[LLM-Inference]]、[[LLM-Serving]]、[[LLM-Training]]
- 长上下文缓存：[[HiSparse-arXiv26]]、[[HySparse-arXiv26]]、[[HySparse2-arXiv26]]、[[Strata-OSDI26]]、[[ECHO-OSDI26]]、[[DirectKV-OSDI26]]
- 多页文档解析：[[UnlimitedOCR2-TechReport26]]
- 推理服务：[[KTransformers-SOSP25]]、[[BatchGen-OSDI26]]、[[EcoServe-OSDI26]]、[[Prism-OSDI26]]、[[DSpark-arXiv26]]
- 训练与 RL：[[Tessera-OSDI26]]、[[RollArt-OSDI26]]、[[Weave-OSDI26]]
- GPU 软件栈：[[MPK-OSDI26]]、[[Syncopate-OSDI26]]、[[UCCL-Tran-OSDI26]]
- 可靠性：[[AEGIS-OSDI26]]、[[SDCHunter-OSDI26]]、[[OpGuard-OSDI26]]
- 检索与数据：[[FlowANN-OSDI26]]、[[Helmsman-OSDI26]]、[[Umap-OSDI26]]

### SOSP 2026 阅读入口

- 推理服务：[[Janus-SOSP26]]、[[ACDC-SOSP26]]、[[LLM-42-SOSP26]]；仅摘要：[[SANDHI-SOSP26]]、[[StreamInfer-SOSP26]]、[[Bedrock-WLM-SOSP26]]。
- 训练与工作流：[[TensorHub-SOSP26]]、[[CrystalLLM-SOSP26]]、[[DiFlow-SOSP26]]。
- GPU 软件栈：[[Ave-SOSP26]]、[[MeldKernel-SOSP26]]；仅摘要：[[MorphX-SOSP26]]、[[EnerTune-SOSP26]]。
- 状态与存储：[[TensorDex-SOSP26]]、[[llmovoice-SOSP26]]；仅摘要：[[Anchor-SOSP26]]。
- 可观测性：[[QProf-SOSP26]]；仅摘要：[[LMTracer-SOSP26]]。

完整名单、证据边界和跨论文比较见 [[OSDI-2026]]、[[SOSP-2026]]。

### SIGCOMM 2026 阅读入口

- [[DualPath-SIGCOMM26]]：利用解码节点的存储网卡读取旧 KV cache；已按 SIGCOMM 正式版全文核对。

## 致谢

感谢 [lqhl/awesome-system-papers](https://github.com/lqhl/awesome-system-papers/) 开放系统论文收藏与整理成果。本 Wiki 在该仓库基础上整理 AI 基础设施方向的论文与阅读笔记。
