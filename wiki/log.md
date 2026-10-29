# Wiki 更新记录

## [2026-10-29] 继续整理 MLSys 推理论文

- 增加 [[SHIP-MLSys26]] 正文及独立附录笔记，纳入服务专题，说明不同系统规模和实验归一化边界。

- 增加 [[PipelinedSharding-MLSys26]] 全文笔记，纳入推理专题，区分显存预算、主机内存和模型量化。

- 增加 [[CORE-MLSys26]] 正文及独立附录笔记，纳入推理专题，区分 CPU/GPU 调频和默认内存 governor。

- 增加 [[MetaInferenceConfig-MLSys26]] 全文笔记，纳入服务专题，区分算子画像模拟与生产经验。

- 增加 [[TeleRAG-MLSys26]] 正文与独立附录初稿，区分检索索引簇与 KV，记录摘要数字及预取公式待核对。

- 增加 [[PragmaticDisaggregation-MLSys26]] 正文与独立附录初稿，区分模拟和实机实验，图表倍数待复核。

- 增加 [[LayeredPrefill-MLSys26]] 正式版全文初稿，记录按层调度和能耗分母的冲突，暂不纳入专题。

- 增加 [[RaidServe-MLSys26]] 全文笔记，纳入服务与 KV 专题，区分状态恢复和完整故障处理时间。

- 增加 [[FlashAgents-MLSys26]] 全文初稿，记录流式预填充与工作流并行的边界；轮内前缀缓存的历史状态约束待复核，暂不纳入专题。

## [2026-10-28] 开始整理 FAST 2026 与 MLSys 2026

- 增加 [[TriInfer-MLSys26]] 全文笔记，纳入服务专题，标明会议页面与 PDF 的标题、摘要差异及对比基线。

- 增加 [[BEAM-MLSys26]] 全文笔记，纳入服务专题，记录能耗、实际 SLO 达成率及基线边界。

- 增加 [[LocalityAwareBeam-MLSys26]] 全文笔记，纳入推理与 KV 专题，区分分支搜索的性能收益与未测任务质量。

- 增加 [[ContextPilot-MLSys26]] 全文与独立附录笔记，纳入推理、服务和 KV 专题，区分输入改写与精确前缀复用。
- 增加 [[MoEServingTax-MLSys26]] 全文笔记，纳入 [[LLM-Inference]]，区分 prefill 长尾与 decode 权重读取成本。
- 增加 [[GhostServe-MLSys26]] 全文初稿；记录节点内故障边界，摘要与结论的数字归属相反，暂不纳入专题。
- 增加 [[SpecDecodeBench-MLSys26]] 全文笔记，纳入 [[LLM-Inference]]，区分实测吞吐与 oracle 模拟上界。
- 建立 [[FlexTrain-MLSys26]] 全文初稿，标出 DP+PP 加速倍数的图文差异，暂不纳入专题。
- [[TokenWeave-MLSys26]]、[[BreakingTheIce-MLSys26]] 和 [[HELIOS-MLSys26]] 已补读全文与附录并纳入专题；[[FaaScale-MLSys26]] 已通读但图示时间轴存在疑点，暂不纳入专题。
- 增加 [[SuperInfer-MLSys26]] 全文笔记，标出调度公式中 SLO 参数与请求状态的对应疑点，暂不纳入专题核心名单。
- 增加 [[OPKV-MLSys26]] 全文笔记，纳入 [[LLM-Inference]] 与 [[KV-Cache-Systems]]，记录选择器语义和内存预算边界。
- 按直接相关原则筛选 FAST 2026，建立 7 篇论文页并阅读正式版全文；6 篇完成复核，[[CacheSlide-FAST26]] 的算法阈值与论文解释存在矛盾，结合作者仓库审计后仍保留待复核状态。
- 开始筛选 MLSys 2026；[[MorphServe-MLSys26]]、[[FlexiCache-MLSys26]]、[[BatchLLM-MLSys26]]、[[SparseSpec-MLSys26]]、[[AXLearn-MLSys26]]、[[Dataflow-MLSys26]]、[[IntAttention-MLSys26]]、[[Stream2LLM-MLSys26]]、[[PRISM-MLSys26]]、[[ProTrain-MLSys26]]、[[ProfInfer-MLSys26]]、[[Kitty-MLSys26]] 和 [[ScaleSearch-MLSys26]] 完成全文和图表复核，[[CRAFT-MLSys26]]、[[LAPS-MLSys26]]、[[SkipKV-MLSys26]] 与 [[DistCA-MLSys26]] 是待复核初稿，会议筛选尚未结束。部分本地论文与会议正式版 PDF 不一致，另存正式版并调用 MinerU 解析；另检查并解析会议页面单独提供的补充材料。
- 将已完成且直接相关的论文同步到 [[LLM-Inference]]、[[LLM-Serving]]、[[LLM-Training]] 和 [[KV-Cache-Systems]]；通用 GPU 工作未强行列入核心专题。

## [2026-10-28] 调整站名与首页入口

- 将站名改为 Awesome AI Infra Papers Wiki；首页致谢前置，阅读入口保留五个专题。

## [2026-10-27] 按优化方法整理 LLM Inference

- 按用户提供的十类方法重排 [[LLM-Inference]]；同一论文可在多个方法下出现，核心论文仍按 16 篇去重计数。无损压缩暂留空标题，不扩大纳入范围。

## [2026-10-27] 收敛主题导航

- 按用户要求保留 [[LLM-Inference]]、[[LLM-Serving]]、[[LLM-Training]] 三条主线，分别收录 16、10、16 篇全文论文；具体方法移到页内小节，移除十个独立方法专题及其成员标签。
- 保留 [[KV-Cache-Systems]]、[[DeepSeek]] 两个专项，移除独立的 vLLM 专题并保留相关博客；五个主题的前端标题统一为英文。归属不明确的论文保留在会议与论文目录，不强行加入主线。
- 同步调整首页、博客成员和持久规则；英文主题标题仅在与显式 frontmatter title 一致时通过语言校验，其他中文正文规则继续执行。

## [2026-10-27] 建立 LLM 推理专题并收录 HiSparse

- 建立 [[LLM-Inference]]，以 29 篇已有全文笔记的论文和 3 篇既有已读博客串联模型执行、内核、KV、解码、批处理与观测；四篇摘要候选单列，不计入论文数。
- 归档 arXiv:2608.07009v1 的 20 页 PDF，经 MinerU 解析并回看图 2–8，建立 [[HiSparse-arXiv26]]；区分容量与批次收益、共享索引预取、无效输出上界及未实测的物理 P/D 分离。
- 将 HiSparse 纳入 [[KV-Cache-Systems]] 及五个有明确方法依据的优化专题，更新英文成员标签、首页计数和 [[vLLM-Hybrid-HiSparse]] 的论文阅读状态说明。

## [2026-10-27] 校对优化方法专题与机制标签

- 对照全部 73 篇论文笔记的核心方法，并回查分类争议处的原文 Markdown；逐个成员补充归类理由，明确核心名单与邻近机制的边界。
- 将 Drs.NAS 的代理评估移出稀疏化核心名单；将 LLM-42 确定性核验和 SPEX 树分支推测列为投机解码的邻近机制，并纠正资源分配率与执行利用率、活跃驻留与表示字节等标签混用。
- 补入 FlowANN 图边编码、两篇本地 MoE 的低精度内核、Strata/DSpark 调度，以及原有笔记中明确的复用、预取和分层执行机制。当前 66 篇进入至少一个方法专题，Drs.NAS 仅保留计算量机制标签，另 6 篇保留原有领域与描述标签。

## [2026-10-27] 建立按优化方法阅读的专题

- 根据用户提供的研究分类图，以十个 `lens` 方法专题交叉组织现有 73 篇论文；65 篇按笔记证据纳入至少一个方法专题，可靠性、观测与训练仿真等 8 篇保留原有分类。
- 为成员论文添加英文 `lens/` 标签，并按明确的作用机制添加英文 `mechanism/` 标签；无损压缩与量化方向目前各有 1 篇，未把准确率约束的模型合并或仅使用 FP8 的工作算入。
- 首页增加方法专题入口，在 `CLAUDE.md` 记录分类判据，保留原有领域、项目专题和所有 PDF、MinerU 原文路径。

## [2026-10-27] 整理 vLLM Hybrid HiSparse 资料

- 建立 [[vLLM-Hybrid-HiSparse]]，汇集官方博客、设计文档、实现追踪、相关论文与中文讨论入口；区分论文摘要、官方实验和待核对的讨论全文。
- 根据用户确认，加入 [[KV-Cache-Systems]] 和 `vLLM-Serving` 的相关博客，并更新首页；博客不计入专题论文数。

## [2026-10-27] 收录 HySparse2 预印本

- 归档 arXiv:2609.26368v1 PDF，经 MinerU 解析全文并回看图 2–5 与表 1–5，建立 [[HySparse2-arXiv26]]；区分跨层桥接、MQA 头配置和预填充 FLOPs 分析的贡献。
- 按用户确认将其纳入 [[KV-Cache-Systems]]，与 [[HySparse-arXiv26]] 对照，更新专题计数和首页。

## [2026-10-27] KTransformers 加入 LLM Serving 专题

- 按用户确认，将 [[KTransformers-SOSP25]] 纳入 [[LLM-Serving]]，补充单机 CPU/GPU 专家放置与专家延迟的范围，并更新成员标签和首页计数。

## [2026-10-27] 收录 SOSP 2025 指定论文 KTransformers

- 依据仓库已有的 SOSP 正式版 PDF 与 MinerU Markdown，建立 [[KTransformers-SOSP25]]，回看内核、执行时间线、实验图与质量消融，区分原计算顺序和专家延迟的收益。
- 建立 [[SOSP-2025]] 指定论文入口并更新首页；本次未筛选会议其他论文。

## [2026-10-27] Unlimited OCR 2 加入 KV cache 专题

- 按用户确认，将 [[UnlimitedOCR2-TechReport26]] 纳入 [[KV-Cache-Systems]]，补充多页 OCR 的视觉 KV 驻留与跨块复用边界，并更新成员标签和首页计数。

## [2026-10-27] 收录 Unlimited OCR 2 技术报告

- 归档百度文心团队的技术报告，经 MinerU 解析全文并核对原图、表 1–4；建立 [[UnlimitedOCR2-TechReport26]]，记录稀疏视觉参考、跨块 KV 复用、实验结果及全局上下文限制。
- 更新首页论文来源与多页文档解析入口。

## [2026-10-26] 收录 HySparse 预印本

- 归档 arXiv:2602.03560v1 PDF，经 MinerU 生成 Markdown 后建立 [[HySparse-arXiv26]]；用原文表 2–4 核对能力与消融结果，并明确服务吞吐和延迟尚无实验。
- 将其纳入 [[KV-Cache-Systems]] 的跨层共享方向，更新首页入口和专题成员数。

## [2026-10-26] KV cache 专题补充知乎阅读入口

- 在 [[KV-Cache-Systems]] 增加用户提供的知乎问题页链接；官方问题回答接口返回 7 条与该问题 ID 对应的回答摘要，全文尚未阅读。

## [2026-10-24] 建立 vLLM 阅读框架

- 新建 `vLLM-Serving`，先按版本架构、调度、KV cache、GPU 执行和多模态部署搭建阅读结构；登记目前确认的官方论文、vLLM-Omni 技术报告与 V1 架构资料。
- 本分支尚无 vLLM 自身的站内论文页，因此暂不把作为基线的 OSDI/SOSP 等论文列为核心成员；性能和实现细节留待后续分析。

## [2026-10-24] 统一 DeepSeek 专题的脚注与参考文献格式

- 参照用户提供的[引用范例](https://keyu-tian-sabotage.github.io/)，在 [[DeepSeek]] 的转述和事实判断旁加入 Markdown 脚注，文末集中列原作者、标题、出处与原文链接；保留现有 frontmatter 与 wiki 内链。
- 将该写法写入 `CLAUDE.md`，供后续 theme 和博客使用；待读书签继续标明未核对，不纳入正文证据。

## [2026-10-24] 标明 DeepSeek 专题的外部引用归属

- 在 [[DeepSeek]] 中明确标注官方报告、代码和第三方博客的原作者来源，区分原文转述、本站综合判断与尚未核对的外部讲解书签。

## [2026-10-24] 建立 DeepSeek 策展专题

- 根据用户本地《DeepSeek优质资料》清单、DeepSeek 官方技术报告与代码入口，以及本站已有的两篇全文论文笔记和三篇博客，建立 [[DeepSeek]] 阅读路线。
- 官方 V2、V3、V3.2、V4、V4.1 报告目前只核对入口与摘要；外部知乎讲解保留为待读书签，不当作已核对的技术证据。同步论文与博客的专题成员标签和首页入口。

## [2026-10-23] 首页增加原仓库致谢

- 在 [[index]] 首页致谢 [lqhl/awesome-system-papers](https://github.com/lqhl/awesome-system-papers/) 的系统论文收藏与整理成果。

## [2026-10-23] 收录 DeepSeek EPLB 博客解读

- 阅读 kaiyuan 的[知乎原文](https://zhuanlan.zhihu.com/p/29963005584)，对照 DeepSeek 官方 EPLB README 与源码，建立 [[DeepSeek-EPLB]]；区分博客的示意和工程判断、官方开源的复制放置算法，以及尚未开源的负载预测与权重搬迁环节。
- 人工加入 [[LLM-Serving]] 的「相关博客」，更新首页和 [[DeepSeek-V4.1-Flash-Serving]] 的关联入口；博客不计入专题 `paper_count`。

## [2026-10-23] 收录 DSpark 推测解码预印本

- 归档 [arXiv:2607.05147v1](https://arxiv.org/abs/2607.05147) 的 33 页 PDF，用 MinerU 分段提取并合并全文 Markdown；建立 [[DSpark-arXiv26]]，标明预印本身份和 DeepSeek-V4 预览版的生产实验边界。
- 人工加入 [[LLM-Serving]] 核心论文，关联 [[DeepSeek-V4.1-Flash-Serving]]，更新首页与专题篇数；严格 SLA 下的 661%/406% 吞吐比单列解释，避免误当常规加速倍数。

## [2026-10-23] 收录 DeepSeek V4.1 Flash 生产推理博客

- 根据 DeepSeek 于 2026-10-20 发布的知乎原文，建立 [[DeepSeek-V4.1-Flash-Serving]]，梳理阶段分离、负载均衡、弹性、两种 KVCache 生命周期与解码路径优化；图表未逐一核对，生产数字标为作者自报。
- 人工加入 [[LLM-Serving]] 与 [[KV-Cache-Systems]] 的「相关博客」，首页增加入口；两篇专题的 `paper_count` 保持论文数量。

## [2026-10-23] 增加博客条目与专题关联

- 建立 `wiki/blogs/`，将此前引用的 zartbot DualPath 博客整理为 [[Zartbot-DualPath]]，并加入 [[LLM-Serving]] 的「相关博客」。当前仅确认原文链接，正文尚未核对，未归纳博客观点。
- 首页增加博客入口；专题的 `paper_count` 仍只统计「核心论文」。

## [2026-10-23] 建立 LLM Serving 专题

- 从服务与推理标签的候选论文中选定 9 篇有全文笔记的核心论文，建立 [[LLM-Serving]]；按请求执行、模型与资源、离线任务与输出约束组织导读。
- 首页增加专题入口；仅摘要的候选未纳入核心成员，完整实验对照留待后续整理。

## [2026-10-23] 建立 KV cache 系统专题入口

- 参照 `main` 分支的 theme 格式建立 [[KV-Cache-Systems]]，从现有论文的 `kv-cache` 标签发现候选，并人工选定 5 篇核心论文；首页增加主题专栏入口。
- 先写纳入范围与阅读路线，后续再补充完整的跨论文实验矩阵。

## [2026-10-23] DualPath 核对 SIGCOMM 正式版

- 用户提供的正式版 PDF 已归档到 `papers/sigcomm-2026/`，用 MinerU 生成对应 Markdown；[[DualPath-SIGCOMM26]] 的来源和状态改为正式版全文、`complete`。
- 重新核对主实验数字；修正虚拟通道高优先级保留份额为 99.9%，补充正式版附录 A.5 的流量隔离微基准及附录 A.2–A.3 的实验条件。同步更新 [[SIGCOMM-2026]] 和首页。

## [2026-10-23] SIGCOMM 2026 指定论文 DualPath

- 加入 [[DualPath-SIGCOMM26]] 与 [[SIGCOMM-2026]]，只收录用户指定的这一篇；首页增加入口。
- 下载 arXiv v2 PDF 并用 MinerU 解析全文，回看路径图、主结果和消融图。会议题名和作者据公开会议记录，技术论断据预印本；ACM 终版 PDF 尚未核对，论文页标为 `needs-review`。

## [2026-10-22] SOSP 2026 用户选定的 AI 基础设施论文

- 按用户给定的 18 篇清单建立 [[SOSP-2026]]：11 篇有全文的论文页与 7 篇仅摘要页；摘要页明确标记未取得 PDF。
- 全文 PDF 存于 `papers/sosp-2026/`，页码标记的文本存于 `markdowns/sosp-2026/`。本次使用 `pdftotext -raw`，没有使用 MinerU；图表和复杂公式需回 PDF 核对。
- 重读并补强已有短页，明确 [[CrystalLLM-SOSP26]] 的 2048 GPU 精度边界、[[LLM-42-SOSP26]] 的全确定性负结果、[[llmovoice-SOSP26]] 与 KV cache 的区别、[[QProf-SOSP26]] 与 GPU 专用 profiler 的区别。

## [2026-10-22] OSDI 2026 AI 基础设施重建

- 重筛 136 篇会议论文，纳入 49 篇；[[OSDI-2026]] 记录筛选口径和边界案例。
- 论文页采用 `main` 已有的全文研读版本，并逐一核对当前原文 Markdown 路径；此前的 12 篇短页被替换。
- 重写会议综述与首页，只呈现本次纳入的 OSDI 2026 论文。
