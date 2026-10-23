# Wiki 更新记录

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
