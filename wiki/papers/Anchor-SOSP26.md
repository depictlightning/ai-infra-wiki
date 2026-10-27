---
type: paper
name: Anchor
full_title: "Anchor: Mitigating GPU Shallow Disruptions with Decoupled Memory"
authors: [Haoyi Ma, Shiwei Gao, Youmin Chen, Junrong Huang, Youyou Lu, Jiwu Shu]
venue: SOSP
year: 2026
tags: [gpu-memory, fault-recovery, llm-serving, training]
source_pdf: ""
source_md: ""
review_status: abstract-only
evidence_level: abstract
last_reviewed: 2026-10-22
---

# Anchor：解耦显存以缓解 GPU 浅层中断（SOSP 2026）

> **原题**：Anchor: Mitigating GPU Shallow Disruptions with Decoupled Memory

> **证据边界**：未取得全文，只依据 SOSP 题录与公开摘要。题录标题含 “GPU”，某些预印本列表省略此词；这里以会议题录为准。

## 问题与动机

一些故障只杀死 GPU worker 进程，设备和显存仍健康。常规重启会随进程一起释放大型模型权重和运行态显存，再从存储重新加载，恢复延迟很高。Anchor 据摘要让独立 daemon 持有 GPU 内存，worker 进程通过 IPC 与重新映射访问；worker 故障后，新进程重连存活的显存对象。摘要还提到访问路径身份解析和一致性处理，用于避免重连后对象别名或陈旧状态引起错误。

## 关键观察 / 隐含假设

- Worker 失败不必等于 GPU 内存失效。只有设备和显存保持健康，独立进程持有的对象才可重连。
- 进程重启后旧显存状态仍可能含未完成写入，因此“重用”必须配合身份解析和一致性规则。

## 核心方法

摘要公开的核心是独立 daemon 持有 GPU 显存，通过 IPC 让 worker 映射与重连，并处理访问路径身份与一致性。未拿到全文，无法描述具体 CUDA IPC、指针重定位、事务边界或状态回滚算法。

## 实验与结果

摘要称在 DeepSpeed 与 vLLM 案例中，推理平均恢复时间降低 **60.7%**、训练降低 **26.6%**；Qwen3-235B 案例从 **143.78 秒缩短至 27.85 秒**。这些数字只适用于设备显存仍可保留的“浅层中断”，不能代表 GPU 驱动重置、显存损坏或机器失联时的恢复。

## 批判性分析

### 论证链条

若进程失败而显存仍在，独立 daemon 保留对象确实能省去重装载时间；Qwen3-235B 的恢复案例支持该窄故障模型。

### 假设压力测试

GPU reset、显存 ECC 错误和节点失联都破坏“显存仍可信”的前提，不能沿用同一个恢复路径。

### 实验可信度

摘要给了平均降幅和一个大模型案例，但没有故障注入矩阵、状态一致性检查与恢复后的输出正确性细节。

- 显存中的哪些内容是故障后可信的、哪些必须回滚或重算，需要全文的状态机和一致性设计才能判断。
- Daemon 成为新的寿命与故障域；多进程隔离、显存泄漏与权限边界是部署关键。
- 摘要不足以判断训练优化器状态、推理 KV cache、CUDA graph 或 RDMA 注册的恢复覆盖率。

## 局限与后续工作

- 取得全文后按进程崩溃、GPU reset、驱动错误、节点失联分级故障注入，逐类核验恢复正确性和时长。

## 相关

- **同会议**：[[SOSP-2026]]；**相邻方向**：[[TensorHub-SOSP26]] 的权重传输与 [[CrystalLLM-SOSP26]] 的大规模训练调试。
- **会议记录**：[SOSP DOI](https://doi.org/10.1145/3830418.3843892)。
