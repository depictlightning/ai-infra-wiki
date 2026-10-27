# Wiki 更新记录

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

- 新建 [[vLLM-Serving]]，先按版本架构、调度、KV cache、GPU 执行和多模态部署搭建阅读结构；登记目前确认的官方论文、vLLM-Omni 技术报告与 V1 架构资料。
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
