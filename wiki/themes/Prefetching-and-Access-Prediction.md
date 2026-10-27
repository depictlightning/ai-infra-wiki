---
type: theme
topic: Prefetching-and-Access-Prediction
theme_kind: lens
member_tag: lens/prefetching-and-access-prediction
paper_count: 10
first_generated: 2026-10-27
last_updated: 2026-10-27
tags: [topic-overview, optimization-method]
---

# 预取与访问预测

本专题按优化方法组织站内论文，可与 [[KV-Cache-Systems]]、[[LLM-Serving]] 等系统领域专题交叉。以下是本站依据论文页「核心方法」作的归类；明确的副机制也可纳入，理由在条目中说明。

## 纳入范围

收录在实际使用前准备数据，或根据预测/已知计划决定访问集合及就绪时机的工作。精确已知的未来访问也可用于预取；需区分传输量下降和传输被隐藏。单纯调节运行资源，不据此算作访问预测。核心名单由人工确认，标签不自动决定成员；「仅公开摘要」表示尚未取得全文。

## 核心论文

- [[HiSparse-arXiv26]] — 在共享索引层组中提前回载各层所需 KV；普通相邻层猜测未获端到端收益。
- [[Chen-LLMDataPipelines-OSDI26]] — 利用 evaluation 计划和恢复 rank 映射，提前复制 checkpoint 与热点文件。
- [[Cocoon-OSDI26]] — 由 sampler seed 确定 embedding 下一次访问，安排合并噪声写入。
- [[DiFlow-SOSP26]] — 编译时前移 LoRA 加载，使其与早期去噪步骤重叠。
- [[ECHO-OSDI26]] — 利用跨层稀疏选择相关性，提前取回后续层需要的 KV。
- [[FlowANN-OSDI26]] — 预测 discovery 到 expansion 的窗口，安排异步图边访问及同步。
- [[Helmsman-OSDI26]] — 预测查询需要的 nprobe，一次决定 posting-list 读取范围。
- [[KAIROX-OSDI26]] — 提前一层预测活跃神经元组并预取下一层权重。
- [[Strata-OSDI26]] — 在排队期间把 SSD 命中的 KV 预取到 DRAM。
- [[Wang-LocalMoEInference-OSDI26]] — loader 提前准备下一专家权重，以事件和环形缓冲衔接计算。

## 主题导读

[[ECHO-OSDI26]] 和 [[Strata-OSDI26]] 处理不同层级的 KV 预取，[[KAIROX-OSDI26]] 预测活跃组，[[Wang-LocalMoEInference-OSDI26]] 提前加载专家。[[Cocoon-OSDI26]] 根据采样序列知道下一次访问；[[FlowANN-OSDI26]] 估计 discovery 的可延后窗口。预测集合、安排读取时机和提前做计算是不同机制，需看具体条目。
