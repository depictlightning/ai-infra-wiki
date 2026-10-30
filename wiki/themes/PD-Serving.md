---
type: theme
topic: PD-Serving
title: Prefill and Decode Serving
theme_kind: lens
member_tag: lens/pd-serving
candidate_tags: [disaggregated-serving, chunked-prefill]
paper_count: 20
first_generated: 2026-10-29
last_updated: 2026-10-30
tags: [topic-overview, llm-serving, scheduling]
---

# Prefill and Decode Serving

本专题研究 LLM 服务中预填充（prefill）与解码（decode）的混合、分离及中间方案。预填充并行处理输入并建立 KV cache；解码逐步生成后续 token。前者通常更依赖计算吞吐，后者在小批量时常受权重和 KV 读取带宽限制，具体瓶颈随模型、批量和硬件改变。

如何用模拟器评估阶段配比、数据传输与队列，见 [[Inference-Simulation]]。

## 纳入范围

核心论文直接研究阶段调度、放置或分离部署中的数据路径，且已有本站全文笔记。通用 KV 优化和只在背景中提及阶段分离的论文不自动纳入；多模态阶段组合若涉及 P/D 也纳入。资源和缓存的更广泛讨论分别见 [[LLM-Serving]] 和 [[KV-Cache-Systems]]。

## 核心论文

- [[DCP-OSDI26]] — 在流水线并行中动态选择 prefill 块并重排 decode，控制阶段失衡。
- [[EcoServe-OSDI26]] — 各完整实例交替执行阶段，多个实例错峰，减少跨实例 KV 搬运。
- [[BatchLLM-MLSys26]] — 利用离线批次可重排的条件，联合前缀共享和 P/D 混合组批。
- [[TriInfer-MLSys26]] — 根据多模态流量选择编码、预填充和解码的组合及实例比例。
- [[DualPath-SIGCOMM26]] — 在分离部署中利用 decode 节点的存储读取能力，缓解旧 KV 读取瓶颈。
- [[SarathiServe-OSDI24]]（2024）— 用分块预填充与 decode 优先组批控制共置干扰。
- [[DistServe-OSDI24]]（2024）— 分离阶段并结合 SLO 和互连搜索配置。
- [[Splitwise-ISCA24]]（2024）— 以阶段分离与混合池探索异构硬件、功耗和成本。
- [[Mooncake-FAST25]]（2025）— 将分离部署与共享 KV cache 和缓存感知调度结合。
- [[TaiChi-arXiv25]]（2025 预印本）— 使用不同块大小的实例和请求级调度，探索混合与分离之间的配置。
- [[DuetServe-ICML26]]（2026）— 预测 token 间延迟，按需在 GPU 内划分 SM 并发执行两个阶段。
- [[FluidPD-arXiv26]]（2026 预印本）— 在固定 GPU 数量下借用 decode 余量，并按持续压力切换 P/D 实例角色。
- [[SplitScaling-IJCAI26]]（2026）— 用强化学习分别扩缩两个阶段的实例，并重分配等待中的请求。
- [[ThunderServe-MLSys25]]（2025）— 联合优化异构 GPU 分组、P/D 角色和通信路径，并在负载变化后轻量重调度。
- [[TetriInfer-arXiv24]]（2024 预印本）— 早期阶段分离工作，结合分块、输出长度预测和角色切换。
- [[Seesaw-MLSys25]]（2025）— 离线吞吐场景中随阶段重新分片，在同一批 GPU 上切换并行方式。
- [[PDServe-arXiv24]]（2024 预印本）— 面向大规模部署组织阶段实例组，按阶段能力匹配调整配比。
- [[Libra-NSDI26]]（2026）— 在统一实例池内按请求切分阶段工作，并以双层调度应对负载不均。
- [[EPDServe-ICML25]]（2025）— 分离多模态编码、预填充和解码，规划并动态调整阶段资源。
- [[MuxWise-ASPLOS26]]（2026）— 在同一 GPU 内动态划分 SM，两个阶段共享 KV 池并并发执行。

各论文的全文来源、图表核对范围和审阅状态见对应论文页。

## 代表工作与阅读路线

先读 [[DistServe-OSDI24]]、[[Splitwise-ISCA24]] 和 [[Mooncake-FAST25]]：它们分别从 SLO 配置、阶段分离与成本、以及生产服务的 KV cache 组织出发，给后续方案提供问题背景。[[SarathiServe-OSDI24]] 是混合执行与分块调度的重要对照。[[TetriInfer-arXiv24]] 与这些早期分离工作同期，DistServe 和 Sarathi-Serve 的原文均将其作为相关研究引用，但它的网络交接实验存在模拟边界。

然后读 [[PDServe-arXiv24]] 看大规模生产系统如何组织和调整阶段组；[[Seesaw-MLSys25]] 的相关工作引用了该论文。读 [[ThunderServe-MLSys25]] 看异构云硬件下如何同时选分组、角色与路径；读 [[Seesaw-MLSys25]] 看离线吞吐任务中如何在同一 GPU 组上转换并行方式。继续研究在线负载变化，可读 [[TaiChi-arXiv25]]、[[FluidPD-arXiv26]] 和 [[Libra-NSDI26]]；[[SplitScaling-IJCAI26]] 提供扩缩容模拟对照。GPU 内资源划分可对照 [[MuxWise-ASPLOS26]] 与 [[DuetServe-ICML26]]。多模态分配可先读 [[EPDServe-ICML25]]，再读 [[TriInfer-MLSys26]]。

此处的代表性依据是论文问题、会议发表、后续论文中的引用关系和研究覆盖面，并非按引用量排名。预印本与会议版可能被文献数据库分别收录，当前没有完成可靠的跨版本引用量归并；较新的论文也不能仅凭新近发表就断言影响力已经稳定。

## 如何比较混合与分离

同一实例混合执行时，请求的 KV 留在本地，资源可由两个阶段共享；长 prefill 却可能推迟已有请求的下一个 token。分块缩短单次占用，但过小的块也会降低计算效率。[[DCP-OSDI26]] 用 SLO 反馈调整块大小；[[BatchLLM-MLSys26]] 的重排依赖离线请求可预先汇总，不应直接推广到实时在线服务。

[[SarathiServe-OSDI24]] 将长 prefill 切块并优先调度已有 decode。[[DuetServe-ICML26]] 进一步在预测 token 间延迟超标时划分同一 GPU 的 SM，允许两阶段并发；SM 划分仍共享内存带宽，两篇论文的硬件和基线也不同。

[[MuxWise-ASPLOS26]] 也在 GPU 内划分 SM，并保持两阶段共享 KV 池；它按 decode 延迟估计与竞争余量分配 SM，以按层 prefill 和异步查询减少调度空档。其实验主要以 P99 TBT 达标评估容量，不应与双 SLO 请求达标率直接比较。SGLang 官方博客提供同作者团队的 PD-Multiplexing 早期实现线索，但具体版本和实验需要分别核对。

[[NanoFlow-OSDI25]] 同样在一个实例内混合阶段，但优化对象是不同子批次的计算、访存与通信操作重叠；它没有按阶段固定划分 SM，也没有求解 P:D 实例数量。其目标和实验指标与上述阶段调度方案不同，可作为 GPU 内执行的对照。

分离部署把阶段放到不同实例，可独立选择批量、并行配置和资源比例。代价是模型副本、阶段交接和 KV 传输，以及两侧容量失衡。prefill 完成的速率若超过 decode 消化能力，排队与 KV 占用会增长；反过来 decode 资源可能空闲。[[PragmaticDisaggregation-MLSys26]] 直接研究这些取舍，其笔记已通读但仍待复核，尤其一处正文与图表的倍数不一致，故暂列候选。

[[DistServe-OSDI24]] 以阶段 SLO 和互连拓扑搜索资源与并行配置；[[Splitwise-ISCA24]] 还比较异构 GPU 的成本、功耗与吞吐，其大规模配置结果主要来自模拟。[[Mooncake-FAST25]] 在分离计算之外引入共享 KV cache，减少重复预填充，但缓存读取与跨阶段传输都占用网络。

[[EcoServe-OSDI26]] 在时间上隔离阶段并让多个完整实例错峰，避免阶段间转移请求 KV；它依赖足够的副本和并发，长窗口仍可能增加等待。[[TriInfer-MLSys26]] 则选择不同阶段组合，说明混合与分离可以是部署配置的连续选择，而非固定二选一。

[[TaiChi-arXiv25]] 通过不同块大小的实例和请求级调度在混合与分离之间调整，将有余量请求的一部分延迟预算分配给其他请求。其单机实验支持所测配置的收益，不能据此断定对所有集群和 SLO 都最优。

分离后的数据路径还包括从外部存储读取可复用的旧 KV。[[DualPath-SIGCOMM26]] 让 decode 侧参与旧 KV 读取，并经过计算网络供 prefill 使用；这与 prefill 完成后把新 KV 交给 decode 是不同流量。其收益依赖高前缀复用、网络拓扑和可用带宽。

## P/D 实例数量如何分配

阶段配比取决于各侧处理能力、输入与输出长度、缓存命中、SLO、并行配置和网络。prefill 的请求产出能力与 decode 持续处理请求的能力长期不匹配时，一侧会排队或让另一侧闲置。输入与输出 token 数的比例不能直接作为 GPU 数量的比例。

- **画像与离线配置**：[[DistServe-OSDI24]] 用阶段画像和 SLO 模拟选择并行配置、实例数与放置；[[Splitwise-ISCA24]] 在吞吐、成本、功耗约束下模拟机器池，并用混合池吸收负载波动。
- **比例敏感性**：[[Mooncake-FAST25]] 在其 16 节点合成负载和指定 TTFT/TBT 门槛下观察到约 1:1 的较优配比，该结果不应推广为通用比例。[[PragmaticDisaggregation-MLSys26]] 以两阶段速率匹配搜索整数资源分配；其笔记仍有图文倍数问题待复核。
- **请求与实例层弹性**：[[TaiChi-arXiv25]] 按实例能力和请求延迟余量选择混合与迁移；[[FluidPD-arXiv26]] 在既定 GPU 总量下，短期借用 decode 余量，持续失衡时切换实例角色。后者是单机实验。
- **请求切分而非固定配池**：[[Libra-NSDI26]] 在统一实例池中按负载决定请求切分点，同时控制混合批次和 KV 传输，关注 P99 token 延迟下的有效输出速率。
- **GPU 内算力划分**：[[MuxWise-ASPLOS26]] 在单实例内按 decode 目标分配 SM；它调整的是每张 GPU 的阶段计算份额，不是集群 P/D 实例数量。
- **动态扩缩容**：[[SplitScaling-IJCAI26]] 根据流量、队列和实例状态分别扩缩 P/D 池；其主要结果来自轨迹驱动模拟，扩容时需要额外资源并等待实例就绪。
- **异构资源与配比联合选择**：[[ThunderServe-MLSys25]] 将 GPU 分组、阶段角色、并行策略和 P/D 通信路径一起考虑，负载变化时在保留原分组的条件下调整角色。
- **固定资源下的替代路径**：[[TetriInfer-arXiv24]] 提出分离实例的角色切换；[[Seesaw-MLSys25]] 则不建立固定 P/D 实例池，而在同一组 GPU 上切换 TP/PP，主要优化离线吞吐。
- **组级调配与生产运维**：[[PDServe-arXiv24]] 根据两阶段的实例处理能力估计目标配比，并逐组重建连接；它报告了大规模商业部署，但配比实验是归一化的固定并发测试。
- **多模态分配**：[[EPDServe-ICML25]] 在三个独立阶段中搜索资源配置并按负载切换角色；[[TriInfer-MLSys26]] 还考虑阶段组合。三阶段配置不能直接视为普通两阶段 P/D 比例。

## 近期检索候选

以下根据论文发布页和摘要筛选，尚未全部完成本地全文解析和论文笔记，不计入核心论文：

- [DOPD（2025，2026 更新）](https://arxiv.org/abs/2511.20982v3)：根据负载监控动态调整 P/D 实例配比，并联合请求调度。
- [TokenScale（2025 预印本）](https://arxiv.org/abs/2512.03416v1)：用 token 速度作为扩缩容信号，并允许部分 decode 实例应对 prefill 突发。
- [HeteroScale（2025 预印本）](https://arxiv.org/abs/2508.19559v1)：研究异构硬件和网络条件下 P/D 联合扩缩容；作者报告了大规模生产部署。已取得 PDF，全文仍待本地解析。
- [Crossflow（2026 预印本）](https://arxiv.org/abs/2609.27085v1)：面向智能体流量，在不改变节点角色的条件下借用 decode 侧容量；需要与实际角色切换区分。

候选的设计描述仅来自摘要，实验设置、基线及结果尚未核验。

## 研究问题与评测

- **比较口径**：同时记录 TTFT、逐 token 间隔或 TPOT 的定义和分位数，以及满足各项 SLO 的请求比例和单位 GPU goodput；不同论文的达标规则不能直接互换。
- **负载与缓存**：改变输入和输出长度分布、到达突发、并发以及前缀命中率；区分在线尾延迟与离线完成时间。
- **资源与通信**：记录总 GPU、模型副本、KV 占用、网络拓扑、有效传输带宽和通信与计算的重叠；检查阶段配比与队列积压，而不只报告单阶段吞吐。
- **动态决策**：研究预测误差、模型加载与角色切换成本，衡量动态配比是否能在负载变化的时间尺度内收回成本。
