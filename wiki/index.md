# AI 基础设施 Wiki

本分支目前整理 OSDI 2026 的 49 篇 AI 基础设施论文、用户人工选定的 SOSP 2026 的 18 篇、SOSP 2025 论文 [[KTransformers-SOSP25]]、SIGCOMM 2026 论文 [[DualPath-SIGCOMM26]]、arXiv 预印本 [[DSpark-arXiv26]]、[[HySparse-arXiv26]] 与 [[HySparse2-arXiv26]]，以及技术报告 [[UnlimitedOCR2-TechReport26]]。SOSP 2026 中 11 篇已按全文阅读，7 篇只有公开摘要；每篇页面和会议综述都标明证据等级。

## 论文来源

- [[OSDI-2026]] — 49 篇：推理服务、KV 缓存、训练与 RL、GPU 软件栈、可靠性、检索与数据路径。
- [[SOSP-2025]] — 指定收录 1 篇：KTransformers 的 CPU/GPU 混合 MoE 推理。
- [[SOSP-2026]] — 人工选定 18 篇：推理与服务、训练与工作流、GPU 内核与调度、模型存储、可观测性。
- [[SIGCOMM-2026]] — 指定收录 1 篇：分离式 KV cache 存储 I/O 与智能体推理。
- arXiv 2026 — [[DSpark-arXiv26]]：按置信度调度验证的半自回归推测解码；[[HySparse-arXiv26]]、[[HySparse2-arXiv26]]：跨层共享 KV 的混合稀疏注意力。
- 技术报告 2026 — [[UnlimitedOCR2-TechReport26]]：稀疏视觉参考与分块长文档解析。

## 主题

专题根据标签发现候选，再人工确定收录内容；标签相同不代表研究问题或实验条件相同。论文与博客分别列出，博客的阅读状态在各自页面标明。

### 系统领域

- [[KV-Cache-Systems]] — 8 篇论文 | KV cache 的跨层共享、驻留、搬运路径与复用调度。
- [[LLM-Serving]] — 11 篇论文 | 在线请求与资源调度、推测解码、离线批处理及确定性服务。

### 策展视角

- [[DeepSeek]] — 2 篇已有全文笔记的核心论文 | 从模型结构、MoE 通信、KV 存储到生产推理的官方资料与博客阅读路线。
- [[vLLM-Serving]] — 0 篇站内核心论文页 | 从 PagedAttention、V1 到 vLLM-Omni 的阅读框架，官方报告与周边论文待分批整理。

### 按优化方法阅读

这些专题回答论文采用什么优化方法，可与上面的系统领域专题交叉。机制标签另记录计算量、状态字节、设备驻留、数据搬运、通信开销、执行利用率、流水重叠、串行轮次与等待等作用位置；论文数按当前人工确认的核心成员统计。同一论文可跨专题，篇数不能相加。各专题条目写明归类理由，邻近机制单列且不计入核心成员。

- [[Sparsification]] — 6 篇 | 稀疏化与加速。
- [[Lossless-Compression]] — 2 篇 | 无损压缩。
- [[Quantization-and-Low-Precision]] — 3 篇 | 量化与低精度计算。
- [[Operator-and-Compiler-Optimization]] — 18 篇 | 算子、编译与执行加速。
- [[Compute-and-State-Reuse]] — 20 篇 | 计算与状态复用。
- [[Speculative-Decoding-and-Multi-Token]] — 4 篇 | 投机解码与多 Token 生成。
- [[Scheduling-and-Resource-Management]] — 33 篇 | 调度与资源管理。
- [[Offloading-and-Tiered-Storage]] — 18 篇 | 卸载与分层存储。
- [[Prefetching-and-Access-Prediction]] — 9 篇 | 预取与访问预测。
- [[Communication-and-Parallelism]] — 19 篇 | 通信与并行优化。

## 博客

- [[vLLM-Hybrid-HiSparse]] — vLLM 稀疏 KV 动态卸载的机制与资料导航；所属专题：[[KV-Cache-Systems]]、[[vLLM-Serving]]。
- [[DeepSeek-V4.1-Flash-Serving]] — DeepSeek V4.1 Flash 大规模推理部署实践；所属专题：[[LLM-Serving]]、[[KV-Cache-Systems]]、[[DeepSeek]]。
- [[DeepSeek-EPLB]] — kaiyuan 解读 DeepSeek 的专家复制与分层负载均衡；所属专题：[[LLM-Serving]]、[[DeepSeek]]。
- [[Zartbot-DualPath]] — zartbot 的 DualPath 讲解；目前仅记录原文入口，尚未核对正文。所属专题：[[LLM-Serving]]、[[DeepSeek]]。

## 阅读入口

- 长上下文缓存：[[HySparse-arXiv26]]、[[HySparse2-arXiv26]]、[[Strata-OSDI26]]、[[ECHO-OSDI26]]、[[DirectKV-OSDI26]]
- 多页文档解析：[[UnlimitedOCR2-TechReport26]]
- 推理服务：[[KTransformers-SOSP25]]、[[BatchGen-OSDI26]]、[[EcoServe-OSDI26]]、[[Prism-OSDI26]]、[[DSpark-arXiv26]]
- 训练与 RL：[[Tessera-OSDI26]]、[[RollArt-OSDI26]]、[[Weave-OSDI26]]
- GPU 软件栈：[[MPK-OSDI26]]、[[Syncopate-OSDI26]]、[[UCCL-Tran-OSDI26]]
- 可靠性：[[AEGIS-OSDI26]]、[[SDCHunter-OSDI26]]、[[OpGuard-OSDI26]]
- 检索与数据：[[FlowANN-OSDI26]]、[[Helmsman-OSDI26]]、[[Umap-OSDI26]]

### SOSP 2026 阅读入口

- 推理服务：[[Janus-SOSP26]]、[[ACDC-SOSP26]]、[[LLM-42-SOSP26]]；仅摘要：[[SANDHI-SOSP26]]、[[StreamInfer-SOSP26]]、[[Bedrock-WLM-SOSP26]]。
- 训练与工作流：[[TensorHub-SOSP26]]、[[CrystalLLM-SOSP26]]、[[DiFlow-SOSP26]]。
- GPU 软件栈：[[Ave-SOSP26]]、[[MeldKernel-SOSP26]]；仅摘要：[[MorphX-SOSP26]]、[[EnerTune-SOSP26]]。
- 状态与存储：[[TensorDex-SOSP26]]、[[llmovoice-SOSP26]]；仅摘要：[[Anchor-SOSP26]]。
- 可观测性：[[QProf-SOSP26]]；仅摘要：[[LMTracer-SOSP26]]。

完整名单、证据边界和跨论文比较见 [[OSDI-2026]]、[[SOSP-2026]]。

### SIGCOMM 2026 阅读入口

- [[DualPath-SIGCOMM26]]：利用解码节点的存储网卡读取旧 KV cache；已按 SIGCOMM 正式版全文核对。

## 致谢

感谢 [lqhl/awesome-system-papers](https://github.com/lqhl/awesome-system-papers/) 开放系统论文收藏与整理成果。本 Wiki 在该仓库基础上整理 AI 基础设施方向的论文与阅读笔记。
