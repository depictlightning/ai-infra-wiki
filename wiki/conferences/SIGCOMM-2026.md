---
type: conference
venue: SIGCOMM
year: 2026
paper_count: 1
first_generated: 2026-10-23
last_updated: 2026-10-23
---

# SIGCOMM 2026：本 Wiki 已收录论文

目前按用户指定只收录 [[DualPath-SIGCOMM26]] 一篇。`paper_count` 是本站已收录数，不表示会议相关论文总数，也不代表已筛选 SIGCOMM 2026 全会。

DualPath 研究预填充与解码分离后的 KV cache 读取：原路径让 PE 的存储网卡承受主要 I/O，系统把 DE 侧空闲的存储网卡加入读取路径，再通过计算网络把旧 KV 送回 PE。笔记基于 [arXiv v2 全文](https://arxiv.org/abs/2602.21548)；会议终版 PDF 尚未取得，版本差异见论文页。相关的单侧分层缓存问题见 [[Strata-OSDI26]]。
