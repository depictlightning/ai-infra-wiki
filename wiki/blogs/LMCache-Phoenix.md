---
type: blog
title: "LMCache & xPU-IO: LMCache Meets GPU Direct Storage"
author: Shi Qiu
published: 2026-11-09
source_url: https://blog.lmcache.ai/en/2026/11/09/lmcache-xpu-io-lmcache-meets-gpu-direct-storage/
reading_status: reviewed
tags: [llm-serving, kv-cache, gpu-storage, ssd, lmcache, lens/kv-cache-offloading]
last_updated: 2026-11-11
---

# LMCache 与 Phoenix：SSD 到 GPU 的直达存储

> **原文**：[LMCache & xPU-IO: LMCache Meets GPU Direct Storage](https://blog.lmcache.ai/en/2026/11/09/lmcache-xpu-io-lmcache-meets-gpu-direct-storage/) · Shi Qiu · 2026-11-09

本文整理 LMCache 博客介绍的 Phoenix 直达存储后端。文中的性能结果由博客作者报告，未在本工作区复现。[^blog]

## 它解决什么问题

多轮智能体任务往往有较长且反复复用的前缀。将 KV 保存在 SSD 可以扩大容量，但 LMCache 文中介绍的 `fs_native` 路径需要先将数据从 SSD 读入主机缓冲，再复制到 GPU。即使命中率高，慢速恢复也可能抵消减少 prefill 的收益。[^blog]

作者在单节点、8 GPU、TP=8、仅 prefill 的轨迹回放中，以 54 个请求样本说明开销：`fs_native` 路径的平均 TTFT 为 1101 ms，其中 SSD 到 GPU 的加载约 600 ms，GPU prefill 约 313 ms。这个分解来自特定样本和配置，不是所有 LMCache 后端的固定比例（原文 §2）。[^blog]

## Phoenix 如何接入 LMCache

Phoenix 是 xPU-IO 提供的直达存储层。SSD 使用 DMA 直接访问 GPU 显存，数据不再经过主机内存中的中转缓冲；CPU 仍参与发起和管理 I/O。它接入 LMCache 的 GDS L1 后端接口，`GDSContext` 协调 GPU 与 NVMe slab 文件之间的传输。DMA 先使用较小的 GPU 暂存区，再由 GPU 内核将数据复制到前缀缓存，因此绕过主机缓冲并不等于取消所有显存内复制（原文 §3.1）。[^blog]

Phoenix 通过 `libphxfile` 实现后端接口。为保持 GPU 计算和传输顺序，它使用 `cudaLaunchHostFunc` 一类运行时接口将主机发起的 DMA 封装为 stream 中的回调：读取完成后，后续内核才使用数据；写入前，先前的聚合内核必须完成。具体行为依赖对应平台实现与 stream 依赖，而不是 GPU 自主提交每条 SSD 命令（原文 §3.1）。[^blog]

## 标准 I/O 栈与硬件支持

Phoenix 使用 Linux `ZONE_DEVICE` 为 GPU 内存建立 `struct page`，让标准文件系统和块层可以处理对应的 I/O 缓冲。它提供 `pread`、`pwrite` 及批量 `io_uring` 等接口；文章介绍了 ext4、xfs 和若干网络或并行文件系统，并称 NVIDIA、MetaX、AMD 路径已得到验证。不同设备仍需要对应驱动与 P2P 映射支持，不能把“使用标准 I/O 栈”理解为无需硬件适配（原文 §3.2）。[^blog]

## 与 Tutti 的关系

Phoenix 和 [[Tutti-arXiv26]] 同属 xPU-IO，但本文介绍的控制路径不同：Phoenix 接入 LMCache 后由主机侧发起直达 DMA；Tutti 将 SSD 命令的逐条提交和完成处理交给 GPU，并使用资源空档调度 I/O。两者都可以减少主机中转，但不能据此认为实现机制、适用硬件或实验结果相同。[^blog][^tutti]

## 结果与边界

博客报告，在生产轨迹回放的高并发条件下，相对 DRAM-only 缓存平均 TTFT 降低至约 1/5.6；在 5 秒 TTFT 目标下，相对 DRAM+SSD 分层配置可服务约 2.9 倍并发请求。这两项比较的基线不同，不能将两个倍数直接相乘。博客同时介绍四块 PCIe 4.0 NVMe 的带宽测试；硬件拓扑、缓存容量、请求分布和缓存命中率都会影响实际收益。[^blog]

文章主要是工程方案和作者测量的介绍；未经相同硬件与固定软件版本的独立对照，不能把结果推及所有模型和部署。Phoenix 的可用性还取决于 GPU、SSD、驱动、内核和 PCIe P2P 的组合。[^blog]

## 相关

- [[KV-Cache-Offloading]]、[[LMCache-arXiv25]]、[[Tutti-arXiv26]]

## 参考文献

[^blog]: Shi Qiu. “LMCache & xPU-IO: LMCache Meets GPU Direct Storage.” LMCache Blog，2026-11-09。[原文](https://blog.lmcache.ai/en/2026/11/09/lmcache-xpu-io-lmcache-meets-gpu-direct-storage/)。
[^tutti]: Shi Qiu 等. “Tutti: Making SSD-Backed KV Cache Practical for Long-Context LLM Serving.” arXiv:2605.03375v1，2026。[论文](https://arxiv.org/abs/2605.03375)。
