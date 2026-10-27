---
type: theme
topic: Operator-and-Compiler-Optimization
theme_kind: lens
member_tag: lens/operator-and-compiler-optimization
paper_count: 19
first_generated: 2026-10-27
last_updated: 2026-10-27
tags: [topic-overview, optimization-method]
---

# 算子、编译与执行加速

本专题按优化方法组织站内论文，可与 [[KV-Cache-Systems]]、[[LLM-Serving]] 等系统领域专题交叉。以下是本站依据论文页「核心方法」作的归类；明确的副机制也可纳入，理由在条目中说明。

## 纳入范围

收录算子实现、编译映射、融合、执行图及设备内数据布局的具体优化。跨请求、阶段或作业的资源分配另见调度专题；只有使用某个现成内核，不足以成为本专题成员。核心名单由人工确认，标签不自动决定成员；「仅公开摘要」表示尚未取得全文。

## 核心论文

- [[HiSparse-arXiv26]] — 将哈希、槽位扫描、LRU 更新与 GPU 辅助 I/O 融合进可捕获的召回内核。
- [[ADAngel-OSDI26]] — 构造多种低精度矩阵乘内核，并按形状与设备选择映射。
- [[Ave-SOSP26]] — 利用编译期数据流不变量辅助智能体生成和组合 GPU 内核优化。
- [[CoPilotIO-OSDI26]] — 将 completion 轮询交给 CPU，GPU warp 通过硬件 barrier 等待。
- [[DirectKV-OSDI26]] — 设计远端 KV 感知 tiling、warp pipeline 和投影—Attention 融合。
- [[FlowANN-OSDI26]] — 在单个搜索内核中组织跨步 discovery、异步复制与工作量平衡。
- [[GraCE-OSDI26]] — 改写不可捕获图并间接寻址参数，提高 CUDA Graph 的适用范围。
- [[KTransformers-SOSP25]] — 按算术强度选择 CPU 内核，结合缓存分块、投影融合与 NUMA 任务划分。
- [[MeldKernel-SOSP26]] — 跨 SM 协作、流水分派与即时归约减少跨阶段写回。
- [[MoonBright-OSDI26]] — 用 GPU 并行生成页表项，并以新虚拟地址减少 TLB shootdown。
- [[MorphX-SOSP26]] — 摘要提出编译变换生成可变形内核，配合画像和调度支持 GPU 多任务。（仅公开摘要）
- [[MPK-OSDI26]] — 把算子图编译成 SM 级任务，由 persistent mega-kernel 执行。
- [[SANI-OSDI26]] — 为 big/LITTLE 核分别选择内核，并转换跨簇迁移任务的布局。
- [[Strata-OSDI26]] — 以 GPU I/O kernel 并行搬运小页，并完成 page-first/layer-first 转换。
- [[Syncopate-OSDI26]] — 自动推导 tile/chunk 依赖、重排执行并选择通信实现。
- [[TileLoom-OSDI26]] — 把 tile DSL 映射到空间核阵列，规划布局、广播和循环顺序。
- [[Twill-OSDI26]] — 联合求解软件流水与 warp specialization。
- [[VTC-OSDI26]] — 用索引映射表示虚拟张量，避免数据搬运算子的中间物化。
- [[Wang-LocalMoEInference-OSDI26]] — 以专用 FP8 GEMV、量化转换融合及细粒度同步优化 CPU MoE 后端。

## 主题导读

[[GraCE-OSDI26]]、[[MPK-OSDI26]] 与 [[Twill-OSDI26]] 分别处理执行图、巨型内核和流水调度。[[VTC-OSDI26]] 省去中间张量物化，[[Strata-OSDI26]] 把小页搬运与布局转换合入 GPU I/O 内核。它们改变的开销不同，不能从总体加速直接推断硬件利用率。
