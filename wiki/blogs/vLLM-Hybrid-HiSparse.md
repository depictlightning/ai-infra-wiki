---
type: blog
title: "GLM 5.3 Optimizations, Part 1: Hybrid HiSparse Offloading in vLLM"
author: vLLM Team
published: 2026-09-08
source_url: https://vllm.ai/blog/2026-09-08-glm53-part1-hybrid-sparse-offloading
reading_status: reviewed
tags: [vllm, llm-serving, kv-cache, offloading, sparse-attention, area/kv-cache-systems, area/llm-inference]
last_updated: 2026-10-27
---

# vLLM Hybrid HiSparse：技术解读与资料导航

这页以 vLLM 官方第一篇博客为主线，整理相关论文、实现文档、代码和后续部署文章。上方作者与日期对应**原博客**；本页是 Wiki 的转述和资料汇编，并非 vLLM 团队撰写的原文。官方系列两篇及设计文档已阅读；HiSparse 已有本站全文笔记 [[HiSparse-arXiv26]]，IndexCache 仍作为摘要级背景入口。

## 先理解它解决什么问题

长上下文稀疏注意力在每一步只读取 indexer 选出的部分历史 token，但如果完整 KV 都必须留在 GPU，显存仍随上下文和并发增长。HiSparse 原论文提出把完整历史放在主机内存，并用有限的 GPU 缓存保存被选中的 KV；其论文实验和开源实现主要针对 SGLang，不能直接把论文数字当作 vLLM 的实测结果。[^hisparse]

vLLM 的 **Hybrid HiSparse** 则在有显存余量时保留 GPU 上的 KV，显存紧张时才逐页释放驻留空间，并按 indexer 的选择从主机取回需要的 KV 行。其目标是在轻载时保留显存命中优势，在长上下文或高并发下利用主机内存扩展容量。这里的「Hybrid」是运行时驻留策略，不是对所有模型都可直接应用的通用注意力替换。[^part1]

## 它是怎么工作的

以官方 GLM 5.3 方案为例，sparse-MLA KV 的状态可随压力变化：

1. **完整驻留**：KV 留在 GPU；完成的前缀页可以提前复制到主机，为之后释放 GPU 空间做准备。
2. **混合驻留**：近期页仍在 GPU，较早的页可仅留在 CPU；被选中的行存放在 GPU hot buffer。读取时优先使用 GPU 驻留页，其次使用 hot buffer，未命中再从主机取回对应行，并更新 LRU。
3. **没有普通驻留页**：复用仅在主机上的前缀时，以占位块和 hot buffer 开始；随后按选中的 token 逐步取回所需行。

官方实现让 hot buffer 和普通驻留页共享 HMA 管理的 GPU block pool 和 KV tensor；同一页可容纳来自不同主机块的选中 token。indexer KV 与 sparse-MLA KV 是不同的缓存组：Hybrid HiSparse 的稀疏卸载主要处理后者，前者仍可能随上下文增长而占用 GPU 显存；官方也提到可通过普通 OffloadingConnector 独立处理相关缓存组。具体行为应以所用版本的文档和配置为准。[^part1][^design]

这带来一个实际取舍：**显存节省并非免费**。主机容量、CPU–GPU 带宽、hot buffer 命中率和并发都会影响结果；主机没有足够空间或取回量过大时，不能只凭显存占用推断吞吐。官方计算器本身也把结果限定为容量估算，而不是部署保证。[^part1]

## 推荐阅读顺序与资源

### 1. 首先读官方博客：机制与单机实验

- [GLM 5.3 Optimizations, Part 1: Hybrid HiSparse Offloading in vLLM](https://vllm.ai/blog/2026-09-08-glm53-part1-hybrid-sparse-offloading)：最直接的入门材料，包含三种驻留状态示意、HMA、hot buffer、容量计算器和复现实验命令。[^part1]
- [官方并发容量计算器](https://vllm.ai/blog-assets/interactive_pages/hisparse_concurrency_calculator.html)：调整模型、上下文、设备和主机池等条件，观察容量取舍；计算结果不等同于真实性能。[^part1]

第一篇的 OpenHands 曲线使用 **单机 8×H200、TP8、FP8 KV、MTP3、142K admission limit**；对照组和 Hybrid 组共用 512 GiB 主机预算，后者分为 384 GiB HiSparse 池与 128 GiB 普通卸载池。其吞吐统计包含前缀缓存命中的逻辑输入 token，比较时需沿用原文口径。原文还讨论 1M 上下文可运行性，但不要把上述 142K 曲线当成 1M 上下文的测量。[^part1]

### 2. 看跨节点部署：Prefill、Decode 与通信

- [GLM-5.3 Optimizations, Part 2: PCP, DCP, and Hybrid HiSparse on 16 H200s](https://vllm.ai/blog/2026-10-23-glm53-part2-pcp-dcp-hisparse)：两节点 P/D 分离案例，prefill 侧结合 PCP 与 DCP，decode 侧使用 Hybrid HiSparse，并讨论 NIXL 和 KV 布局转换。[^part2]

该文在指定 AgentX 测试下报告：同吞吐时，PCP8 × DCP8 prefill 加 Hybrid HiSparse decode 的每用户速度为所比较数据并行部署的 **1.7–3.2 倍**。这是整套部署相对基线的结果，不能单独归因于 Hybrid HiSparse。文章使用 16 张 H200，给出 vLLM 版本、提交和复现入口；跨硬件、负载或版本比较前应重新测量。[^part2]

### 3. 看论文：原始方法和 indexer 背景

- [HiSparse: Scaling Sparse-Attention Decoding with Hierarchical KV Cache Management](https://arxiv.org/abs/2608.07009v1)：Zhiqiang Xie 等的原始研究，适合看主机分层缓存、GPU 命中与替换、按层预取的设计；已整理本站全文笔记 [[HiSparse-arXiv26]]，其中的 SGLang 实验须与 vLLM 官方实验分开看。[^hisparse]
- [IndexCache: Accelerating Sparse Attention via Cross-Layer Index Reuse](https://arxiv.org/abs/2603.12201v1)：Yushi Bai 等研究跨层复用 indexer 的 token 选择，用来理解官方博客提及的 GLM 5.3 IndexShare 背景；它处理索引计算复用，与 HiSparse 管理 KV 驻留是不同问题。[^indexcache][^part1]

### 4. 看实现和版本状态

- [vLLM HiSparse 设计文档](https://docs.vllm.ai/en/latest/design/hisparse/)：从 `HiSparseCoordinator`、`HiSparseConnector`、`HiSparseWorker` 和 `HiSparseRuntime` 的职责入手，理解 GPU/主机块管理、迁移和与其他 connector 的组合；文档标为 experimental，`latest` 会随项目演进。[^design]
- [vLLM HiSparse 主实现 PR #53781](https://github.com/vllm-project/vllm/pull/53781)：跟踪上游实现及讨论；阅读源码时请固定具体版本。[^pr]
- [HiSparse 问题与优化追踪 #60072](https://github.com/vllm-project/vllm/issues/60072)：查看正确性、P/D、性能和兼容性工作的当前状态，勾选项也应以相应提交是否进入目标版本为准。[^tracking]
- [vLLM-Ascend 适配 RFC #19293](https://github.com/vllm-project/vllm-ascend/issues/19293)：介绍已有 Ascend 稀疏 KV 卸载与计划适配上游 HiSparse 的区别；这是 RFC，不能据此认定所有功能已经落地。[^ascend]

### 5. 中文讨论与站内延伸

- [知乎关于 2026 年 KV cache 优化的回答](https://www.zhihu.com/question/2081079592870212699/answer/2081080025613391343)：此前核对的摘要提到 Hybrid HiSparse，可作为中文讨论入口；全文尚未核对，暂不据此归纳作者观点。[^zhihu]
- [[ECHO-OSDI26]]：站内已有的原生稀疏注意力 KV 卸载论文笔记，可以对照看主机内存扩容和按需搬运。
- [[HySparse-arXiv26]] 与 [[HySparse2-arXiv26]]：名称相近，但它们是另一条改变模型注意力结构、跨层共享 KV 的研究路线；不要把其中的模型结构或 FLOPs 结果当成 vLLM Hybrid HiSparse 的运行时指标。

## 参考文献

[^part1]: vLLM Team. 《GLM 5.3 Optimizations, Part 1: Hybrid HiSparse Offloading in vLLM》. vLLM Blog，2026-09-08。[原文](https://vllm.ai/blog/2026-09-08-glm53-part1-hybrid-sparse-offloading)。
[^part2]: vLLM Team。《GLM-5.3 Optimizations, Part 2: PCP, DCP, and Hybrid HiSparse on 16 H200s》。vLLM Blog，2026-10-23。[原文](https://vllm.ai/blog/2026-10-23-glm53-part2-pcp-dcp-hisparse)。
[^hisparse]: Zhiqiang Xie 等. 《HiSparse: Scaling Sparse-Attention Decoding with Hierarchical KV Cache Management》. arXiv，2026-08-07。[原文](https://arxiv.org/abs/2608.07009v1)。
[^indexcache]: Yushi Bai 等. 《IndexCache: Accelerating Sparse Attention via Cross-Layer Index Reuse》. arXiv，2026-03-12。[原文](https://arxiv.org/abs/2603.12201v1)。
[^design]: vLLM Project. 《HiSparse local KV offload architecture》. vLLM Documentation。[原文](https://docs.vllm.ai/en/latest/design/hisparse/)。
[^pr]: vLLM Project. 《[3/N] HiSparse: host-resident sparse-MLA decode hot-buffering》. GitHub，2026-08-25。[原文](https://github.com/vllm-project/vllm/pull/53781)。
[^tracking]: vLLM Project. 《[Tracking] HiSparse bugfixes and performance improvements》. GitHub，2026-10-05。[原文](https://github.com/vllm-project/vllm/issues/60072)。
[^ascend]: vLLM-Ascend Project. 《[RFC]: Adapt to vLLM HiSparse》. GitHub，2026-10-20。[原文](https://github.com/vllm-project/vllm-ascend/issues/19293)。
[^zhihu]: 对《2026年了，关于大模型kv cache的优化有什么新的方法吗？》的回答. 知乎。[原文](https://www.zhihu.com/question/2081079592870212699/answer/2081080025613391343)。
