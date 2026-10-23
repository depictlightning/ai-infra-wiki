# AI 基础设施论文 Wiki

本分支目前整理 OSDI 2026 的 49 篇 AI 基础设施论文、用户人工选定的 SOSP 2026 的 18 篇，以及单独指定的 SIGCOMM 2026 论文 [[DualPath-SIGCOMM26]]。SOSP 中 11 篇已按全文阅读，7 篇只有公开摘要；每篇页面和会议综述都标明证据等级。

## Conferences

- [[OSDI-2026]] — 49 篇：推理服务、KV 缓存、训练与 RL、GPU 软件栈、可靠性、检索与数据路径。
- [[SOSP-2026]] — 人工选定 18 篇：推理与服务、训练与工作流、GPU 内核与调度、模型存储、可观测性。
- [[SIGCOMM-2026]] — 指定收录 1 篇：分离式 KV cache 存储 I/O 与智能体推理。

## 主题

专题根据论文标签发现候选，再人工确定核心论文；标签相同不代表研究问题或实验条件相同。当前先建立一篇导读，后续按同一格式扩展。

### 系统领域

- [[KV-Cache-Systems]] — 5 篇 | KV cache 的存放位置、搬运路径与复用调度。
- [[LLM-Serving]] — 9 篇 | 在线请求与资源调度、离线批处理及确定性服务。

## 阅读入口

- 长上下文缓存：[[Strata-OSDI26]]、[[ECHO-OSDI26]]、[[DirectKV-OSDI26]]
- 推理服务：[[BatchGen-OSDI26]]、[[EcoServe-OSDI26]]、[[Prism-OSDI26]]
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
