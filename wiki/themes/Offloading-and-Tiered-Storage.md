---
type: theme
topic: Offloading-and-Tiered-Storage
theme_kind: lens
member_tag: lens/offloading-and-tiered-storage
paper_count: 19
first_generated: 2026-10-27
last_updated: 2026-10-27
tags: [topic-overview, optimization-method]
---

# 卸载与分层存储

本专题按优化方法组织站内论文，可与 [[KV-Cache-Systems]]、[[LLM-Serving]] 等系统领域专题交叉。以下是本站依据论文页「核心方法」作的归类；明确的副机制也可纳入，理由在条目中说明。

## 纳入范围

收录把数据、状态或处理工作部署到 CPU、主机内存、CXL、SSD 等层级，并优化装载、卸载或跨层访问的工作。仅把内存分配器换成共享池，不足以证明存在卸载；具体回载通路也可作为明确的副机制纳入。核心名单由人工确认，标签不自动决定成员；「仅公开摘要」表示尚未取得全文。

## 核心论文

- [[HiSparse-arXiv26]] — 主机 DRAM 保留完整注意力 KV，GPU 热缓存承载当前及近期所选记录。
- [[Chen-LLMDataPipelines-OSDI26]] — 把多模态转换下推至存储节点，缓解训练端 CPU 数据加载瓶颈。
- [[Cocoon-OSDI26]] — 将噪声历史分在 GPU、CPU 和 CXL-NMP，并就地执行 GEMV。
- [[CoPilotIO-OSDI26]] — 卸载 NVMe completion 控制工作到 CPU，数据 payload 仍直接进入 GPU。
- [[DirectKV-OSDI26]] — 在主机保存 KV，GPU 经 NVLink-C2C 直接读取。
- [[DualPath-SIGCOMM26]] — 优化外部 KV 存储经解码/预填充节点回载的两条 I/O 路径。
- [[ECHO-OSDI26]] — 完整 MLA KV 放主机，GPU 只保留所选 token 的缓存。
- [[FlowANN-OSDI26]] — 把图短边留在 GPU、其余图边放 CPU，并异步取回。
- [[Janus-SOSP26]] — 用主机 pinned memory 保存温热模型，激活时 H2D 装载。
- [[KAIROX-OSDI26]] — 在 CPU/GPU 间分配活跃神经元组的存储和计算。
- [[KTransformers-SOSP25]] — 路由专家常驻 CPU 内存并在 CPU 执行，注意力等放 GPU。
- [[Nixie-OSDI26]] — 以 GPU、pinned、paged 和 disk 四层管理应用切换。
- [[Prism-OSDI26]] — 把长期闲置模型驱逐到 CPU DRAM，并按请求重新激活。
- [[RLinf-OSDI26]] — 共置 RL 角色通过 device lock 按依赖执行 onload/offload。
- [[Seer-OSDI26]] — 以 DRAM/SSD 分层 KV pool 支持 rollout chunk 跨实例迁移。
- [[Strata-OSDI26]] — 管理 GPU、主机与外部存储的 KV 备份、加载和淘汰。
- [[Umap-OSDI26]] — 用受限用户态内存缓存承载 DFS 后备矩阵并异步读写。
- [[Wang-LocalMoEInference-OSDI26]] — CPU DRAM 保存完整专家，长 prefill 流式装载到 GPU。
- [[Weave-OSDI26]] — RL 阶段结束时把状态卸载至主机，随后以已缓存状态热启动。

## 主题导读

[[ECHO-OSDI26]] 按需回载稀疏 KV，[[DirectKV-OSDI26]] 直接读取主机 KV，[[Strata-OSDI26]] 管理多层缓存。[[Prism-OSDI26]] 与 [[Janus-SOSP26]] 还有闲置模型的主机温热状态；[[RLinf-OSDI26]] 和 [[Weave-OSDI26]] 通过阶段性 onload/offload 共享设备。节省设备驻留并不自动减少总搬运量。
