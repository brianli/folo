---
title: "PD 分离与 AF 分离：大模型推理架构的技术解析与适用边界探讨"
source: "Essays"
category: "AI Infra"
group: "MiracleFarms"
url: "https://miraclefarms.github.io/notes/2026/08/25/pd-af-disaggregation-boundary/"
published: 2026-08-25T10:50:00+08:00
saved: 2026-09-27T16:29:03+08:00
folo_key: "mf-essays::https://miraclefarms.github.io/notes/2026/08/25/pd-af-disaggregation-boundary/"
tags:
  - "folo"
  - "AI_Infra"
  - "Essays"
---

# PD 分离与 AF 分离：大模型推理架构的技术解析与适用边界探讨

> [!info] Essays · AI Infra · 2026-08-25 10:50 · [原文](https://miraclefarms.github.io/notes/2026/08/25/pd-af-disaggregation-boundary/)

![[PD 分离与 AF 分离：大模型推理架构的技术解析与适用边界探讨-01.jpg]]

> **版本声明**：本文的框架状态描述基于 2026-08-25 前后的公开材料（vLLM `afd-plugin` main 分支、SGLang 0.5.12、NVIDIA Dynamo 1.0、TensorRT-LLM 1.3）；版本对齐信息见文末。

推理架构的拆分史看起来像一条单向阶梯：chunked prefill 把一个请求切成小块，P/D 分离把两个阶段搬到不同 GPU 池，AFD 再把一层里的 Attention 与 FFN 送去不同机器。阶梯的每一级都宣称拿到了新的自由度。但把最近三个月的论文和生产盘点放在一起看，会出现一个不那么整齐的画面：**每一层拆分都在用一种耦合换另一种耦合，换来的自由度只有在特定的模型稀疏结构、互联带宽层级和 SLO 形状下才结算成收益。**

这篇文章按六个问题走：两类算子为什么会在同一块卡上打架，P/D 拆开之后瓶颈搬到了哪里，AF 拆分的切口为什么落在”状态归属”上，拆开之后比例该怎么定，哪些条件下拆分会亏，以及这条边界正在被什么力量推着移动。每一节配一张来自一手材料的图，因为这个题目的分歧几乎都藏在曲线的形状里，而不在结论句里。

## 一、为什么要拆：两类算子的利用率会朝相反方向跑

> **本节论点**：拆分的起点是同一块 GPU 上两类算子对资源的诉求随负载反向发散——长上下文把 MoE FFN 饿死的同时，Attention 的利用率纹丝不动。 **本节核心参考**：
> 
> - ★★★ **FastAFD（Hao AI Lab @ UCSD，2026-07-06）**[\[1\]](https://haoailab.com/blogs/fastafd/) — GB200 NVL72 上共置 MoE 的 MFU 分解实测，本节主线判断的直接证据
> - ★★ 前期调研文章《LLMCompass：用 4.1% 误差的硬件评估框架，重写 LLM 推理的性价比公式》[\[19\]](https://miraclefarms.github.io/notes/2026/07/21/llmcompass-hardware-eval-framework/) — 从硬件设计空间侧量化 prefill/decode 的需求分裂
> - ★ DistServe[\[18\]](https://arxiv.org/abs/2401.09670) — P/D 分离的原始动机文献

P/D 分离最初的论证很干净：prefill 处理整段输入，算力受限；decode 一次只推进一个 token，内存带宽受限。前期调研文章里用 LLMCompass 做过一次硬件侧的量化，结论比直觉更极端——把算力砍掉四分之三，prefill 慢 3.25 倍，decode 只慢 0.1%；反过来把内存带宽从 800 GB/s 提到 2000 GB/s，decode 延迟改善 1.88 倍，prefill 只降 14.3%。[\[19\]](https://miraclefarms.github.io/notes/2026/07/21/llmcompass-hardware-eval-framework/) 两个阶段对硬件的诉求几乎正交，这为按阶段裁剪资源提供了物理依据。

AFD 的动机要晚几年才浮出来，因为它依赖 MoE 变成主流架构。FastAFD 在 GB200 NVL72 上做的一次 MFU 分解，把这个动机摊在了台面上。

![[PD 分离与 AF 分离：大模型推理架构的技术解析与适用边界探讨-02.png]]

*图 1：Qwen3-235B-A22B-FP8 在 GB200 NVL72 上共置部署（Attention 与 MoE 1:1，KV 容量 808,704 token/rank）。横轴是上下文长度，decode batch 由 KV 容量除以上下文长度得到——2K 时能装 395 条并发，128K 时只剩 6 条。红线 MoE FFN 的 MFU 从 28% 塌到 0.5%，蓝线 Attention 却从 5.4% 微升到 6.9%。这张图说明共置部署下两类算子的利用率随上下文长度朝相反方向走，是 AFD 全部论证的起点。来源：FastAFD 博客，图中数据。*

这条曲线解释了共置 MoE 的结构性困难。Decode 阶段能装多少并发请求由 KV cache 容量决定，上下文越长，同一份显存能装的请求越少；而 MoE 专家的 GEMM 效率取决于一次能攒到多少 token。**KV 容量给 Attention 定的上限，同时也成了 MoE FFN 的上限，但这两者本该由完全不同的因素决定。** 128K 上下文下 MoE FFN 只跑出 0.5% 的算力峰值，意味着这部分硬件几乎在空转。

到这里，两种分离的动机就分层清楚了。**P/D 分离切的是时间维度——同一个请求的两个阶段；AFD 切的是空间维度——同一时刻同一层里的两类算子。** 它们可以叠加，也确实在生产系统里叠加了——xDeepServe 在 CloudMatrix384 上就是先做 P/D 分离，再在 decode 侧做 MoE-Attention 解耦。[\[23\]](https://miraclefarms.github.io/notes/2026/05/26/xdeepserve-moe-disaggregated-serving/)

## 二、P/D 怎么拆：瓶颈从算力搬到了排队与传输

> **本节论点**：P/D 分离把 prefill 的算力问题解决之后，TTFT 的主要成分变成了排队和跨节点 KV 传输——生产集群上 prefill 执行本身只占 P95 TTFT 的 2–23%。 **本节核心参考**：
> 
> - ★★★ **Kairos: Towards Load-Aware Prefill Deflection for Disaggregated LLM Serving（arXiv，2026-07-02）**[\[5\]](https://arxiv.org/abs/2607.02043) — 生产式 A100 集群上的 TTFT 成分分解与 deflection 实验，本节主线证据
> - ★★ Le Compute《Prefill/decode disaggregation reaches production》[\[14\]](https://lecompute.fr/en/runtimes/disaggregation-prefill-decode-production/) — 三个 runtime 的原生支持状态盘点
> - ★ 前期调研文章《P/D 分离推理的失控点：Dynamo 路由博弈阅读》[\[22\]](https://miraclefarms.github.io/notes/2026/07/03/dynamo-disaggregated-inference-price-of-anarchy/) — 饱和拐点的行为刻画

到 2026 年，P/D 分离在框架侧已经不是可选项。Le Compute 在 7 月的盘点里列了三个原生支持的 runtime：NVIDIA Dynamo 1.0 负责 KV 亲和路由与拓扑感知放置，SGLang 0.5.12 把分离当作一等公民并同时提供 NIXL 与 MORI-IO 两套传输后端，TensorRT-LLM 1.3 引入 Helix Parallelism 在 Attention 层与 FFN 层之间切换并行策略。[\[14\]](https://lecompute.fr/en/runtimes/disaggregation-prefill-decode-production/)

框架成熟之后，问题换了一层。Kairos 这篇论文在一个生产式 A100 集群（2 个 prefill 节点 + 2 个 decode 节点）上做 TTFT 成分分解，得到一个反直觉的数字：**prefill 的实际执行时间只占 P95 TTFT 的 2–23%，剩下的全是排队和跨节点 KV 传输。**[\[5\]](https://arxiv.org/abs/2607.02043) 在突发、长尾的真实流量下，prefill 节点排队排到饱和，decode 节点的算力却大量闲置——因为 decode 是带宽受限的，它的算力本来就用不完。

这个观察直接推翻了一个很常见的部署直觉：”TTFT 上不去，就给 prefill 池加卡”。加卡只能压缩那 2–23%。

Kairos 的解法是让 decode 节点顺带把 prefill 干了。可行性来自另一条曲线：

![[PD 分离与 AF 分离：大模型推理架构的技术解析与适用边界探讨-03.png]]

*图 2：把 chunked prefill 注入正在进行的 decode batch 后，decode 单步延迟的增量 ΔT\_step。五个面板覆盖五类负载形态（低/高 prefill × 低/高 decode，以及长尾突发）。chunk 大小 χ ≤ 512 时曲线基本贴地（增量 ≤ 30 ms），χ=1024 起明显抬头，χ=2048 时增量冲到 200 ms 以上。这张图说明 decode 节点能以接近零的代价吸收 prefill 工作，前提是 chunk 切得足够小——它是 deflection 调度成立的物理前提。来源：Kairos 论文 Figure 1，图中数据。*

有了这条曲线，调度器就可以对每个排队请求同时估算”走 prefill 节点”和”走 decode 节点 chunked 插队”两条路径的 TTFT，挑更快的那条，同时守住 decode 侧的 TBT SLO。结果是 P95 TTFT 最多降低 81%，SLO 达成率最多提升 79%，每请求的路由开销在亚毫秒级。[\[5\]](https://arxiv.org/abs/2607.02043)

**P/D 分离真正需要调的对象是排队与传输，而 P:D 卡数比例只是这个问题的一个投影。** 前期调研里的两个发现从别的角度指向同一件事：Dynamo 在受控负载下会在并发 C=128 附近出现拐点，过了拐点 TTFT P99 从 74 ms 爆到 113 秒而 ITL P99 基本持平，说明爆炸发生在排队而非计算；[\[22\]](https://miraclefarms.github.io/notes/2026/07/03/dynamo-disaggregated-inference-price-of-anarchy/) 而在异构 P/D 的实验里，同样的 4P4D 配置，KV 用 BF16 时 SLA 断点在 0.1 req/s，换成 FP8 就抬到 1.0 req/s——传输量的十倍差距，直接决定了这套配置能不能用。[\[21\]](https://miraclefarms.github.io/notes/2026/07/03/heterogeneous-pd-inference-boundary/)

## 三、AF 怎么拆：切口画在状态归属上

> **本节论点**：AFD 的切分依据是状态归属——Attention 有状态且状态量随上下文增长，FFN 无状态且只吃 token 批量；这个性质决定了它只对 MoE 成立。 **本节核心参考**：
> 
> - ★★★ **Analytical Provisioning for AFD（arXiv 2601.21351，v1 2026-01-29 / v4 2026-08-14）**[\[2\]](https://arxiv.org/abs/2601.21351) — AFD 的状态语义与 decode 步内数据流刻画
> - ★★ MegaScale-Infer[\[8\]](https://arxiv.org/abs/2504.02263) 与 Step-3[\[9\]](https://arxiv.org/abs/2507.19427) — AFD 的两个奠基系统与首批生产数字
> - ★★ FastAFD[\[1\]](https://haoailab.com/blogs/fastafd/) — 把通信控制面放到 GPU 侧的第三条实现路线
> - ★ vLLM `afd-plugin`[\[13\]](https://github.com/vllm-project/afd-plugin) — 开源工程化落点

P/D 的切分依据是”请求走到了哪一步”，AFD 的依据换成了”这段计算持有什么状态”。

![[PD 分离与 AF 分离：大模型推理架构的技术解析与适用边界探讨-04.jpg]]

*图 3：一个 decode 步内 Attention（A）与 FFN（F）的角色分工。A 侧持有 KV cache，每个请求的状态随生成长度增长；F 侧完全无状态，每层之间 A↔F 通信传的也只是无状态的激活。下半部分展示状态更新：进行中的请求各加 1 个 token，完成的槽位由新 prefill 填回。这张图说明 AFD 的切分线画在状态归属上——正因为 F 侧无状态，它才能自由地把多个 A 侧 worker 的 token 聚到一起做大 batch GEMM。来源：arXiv 2601.21351 Figure 1。*

无状态这一条是全部工程可行性的来源。FFN worker 不需要知道 token 来自哪个请求、上下文多长，它只需要一批 token。于是可以让 M 个 Attention worker 各自按 KV 容量收请求，再把它们的 token 汇总到 N 个 FFN worker 上凑成大的专家 GEMM——第一节里”KV 容量给 MoE 定上限”的耦合就此被切断。

代价是每层要走两趟跨机通信。这一步的成本决定了 AFD 只在 MoE 上划算：MoE 本来就要为专家并行做 all-to-all，AFD 复用的是已有的通信形态；换成 dense transformer，A↔F 之间传激活的开销没有任何东西可以摊销。这也是为什么讨论 AFD 时几乎总能默认对方在说 MoE。

三个系统给出了三条实现路线。MegaScale-Infer（字节跳动）用 ping-pong micro-batch 在 A、F 两侧交替流水以隐藏通信，报告最高 1.90× 单 GPU 吞吐；[\[8\]](https://arxiv.org/abs/2504.02263) Step-3（阶跃星辰）把 AFD 和 MFA 注意力做成模型-系统协同设计，321B 模型在 Hopper 上 50 ms TPOT SLA 下跑到 4,039 token/s/GPU，同口径下 DeepSeek-V3 是 2,324；[\[9\]](https://arxiv.org/abs/2507.19427) xDeepServe 在 384 张 Ascend 910C 上把 288 Die 分给专家、480 Die 分给注意力，A2E/E2A 通信 172 μs / 193 μs，在 49 ms 的解码迭代里占比约 0.75%。[\[23\]](https://miraclefarms.github.io/notes/2026/05/26/xdeepserve-moe-disaggregated-serving/)

FastAFD 是第四条，也是差异最大的一条。前面几个系统都走 RDMA 加 CPU 侧控制，FastAFD 把 payload 和控制都留在 GPU 上，用 DeepEP 风格的 dispatch/combine 走 NVLink/NVSwitch，配合 MegaMoE 融合与 CUDA graph 的零开销调度。[\[1\]](https://haoailab.com/blogs/fastafd/) 代价写得很直白：吃 SM 资源。这条路线只在单 NVLink 域内成立，出了机架就退回 RDMA。

![[PD 分离与 AF 分离：大模型推理架构的技术解析与适用边界探讨-05.png]]

*图 4：GB200 NVL72、FP8、每根柱子取各自最优 attention:FFN 比例下的单 GPU 解码吞吐。MiniMax-M2.5 在 8K/16K 上下文分别为 1.45×（18 节点 17:1）与 1.35×（18 节点 17:1），Qwen3-235B 为 1.41×（8 节点 7:1）与 1.44×（12 节点 11:1）。这张图给出 AFD 在 NVL72 上的收益量级，同时也暴露了下一节的问题——每个数字都绑着一个特定比例。来源：FastAFD 博客，图中数据。*

工程化的落点则透出另一层信息。vLLM 主仓里两个 AFD 相关 RFC——#22799（AFD MVP 与硬件无关的 AFConnector）和 #27584（在线弹性 AFD）——都被关成了 not planned；[\[11\]](https://github.com/vllm-project/vllm/issues/22799)[\[12\]](https://github.com/vllm-project/vllm/issues/27584) 能力最终以外置插件 `vllm-project/afd-plugin` 落地，支持 CUDA 与 Ascend NPU 两套 connector，模型覆盖 DeepSeek-V2/V3/V3.2 与 Qwen3 MoE 系列。[\[13\]](https://github.com/vllm-project/afd-plugin) **社区把 AFD 放在插件而非主干，本身就是一个关于适用边界的判断：它还没有普适到该成为默认路径。**

## 四、拆到什么比例：自由度增加的同时，错配的代价也增加了

> **本节论点**：拆开只是把一个固定配置换成一条搜索曲线，曲线两端都可能低于未拆分基线；比例搜索因此从调优动作升级为部署前置条件。 **本节核心参考**：
> 
> - ★★★ **FastAFD 的 attention:FFN 比例扫描**[\[1\]](https://haoailab.com/blogs/fastafd/) — 同一模型上比例从 3:1 到 17:1 的完整曲线，本节主线证据
> - ★★ Analytical Provisioning for AFD[\[2\]](https://arxiv.org/abs/2601.21351) — 最优 A/F 比的闭式解与三个瓶颈区间
> - ★ 前期调研文章《vLLM AFD Plugin：把 MoE 的 Attention 与 FFN 拆成两套服务之后》[\[20\]](https://miraclefarms.github.io/notes/2026/07/24/vllm-afd-plugin-moe-serving/) — Ascend 侧的比例敏感度实测

拆分带来的自由度是一条曲线，收益取决于你停在曲线的哪个点。FastAFD 把这条曲线完整画了出来。

![[PD 分离与 AF 分离：大模型推理架构的技术解析与适用边界探讨-06.png]]

*图 5：横轴 attention:FFN 节点比例从 3:1 扫到 17:1，虚线为共置基线。左列 Qwen3-235B 在 8K 上下文于 7:1 处见顶（2518 tok/s/GPU，1.41×），继续加 attention 节点反而单调下滑，17:1 时约 1850 已逼近 1781 的基线，同时下排的 decode 单步延迟从 32.8 ms 一路涨到 48.7 ms；右列 MiniMax-M2.5 则单调上升到 17:1 才走平，延迟全程稳在 30.7–33.8 ms。这张图说明”最优比例”不是一个可移植的常数——两个都是 MoE、都在同一台 NVL72 上，曲线形状完全不同。来源：FastAFD 博客，图中数据。*

两条曲线形状不同的原因在模型本身。Qwen3-235B 每 token 激活 22B 参数，MiniMax-M2.5 只有 10B。激活参数越多，FFN 侧的工作量越重，能被 attention 侧掩盖的余量越小，于是更早地把 MoE 顶到关键路径上——曲线见顶后掉头。**同一个 AFD 实现、同一台机器、同一类模型架构，最优比例可以差出一倍以上；比例搜索因此属于部署前置条件，等上线之后再调已经太晚。**

Ascend 侧的实测给出了同一结论的另一个版本。vLLM AFD Plugin 在 DeepSeek-V3.2 W8A8 上对比 EP64 基线：16K 输入时 48A16F 比基线低 5.3%，64A16F 比基线高 11.3%；32K 输入对应为低 10.0% 与高 9.0%。[\[20\]](https://miraclefarms.github.io/notes/2026/07/24/vllm-afd-plugin-moe-serving/) 拆分本身没有带来任何保证，它只是把配置空间放大，让好配置和坏配置同时变得可达。

如果每次都要靠实跑扫描比例，AFD 的部署成本会高到劝退。2601.21351 这篇论文提供的是一条捷径：把 AFD 建模成随机负载下的排队系统，推出最优 A/F 比的 mean-field 闭式解，并把它分解成 Attention 瓶颈、通信瓶颈、FFN 瓶颈三个区间，再加一层高斯 barrier 修正来刻画跨 worker 同步开销。在 trace 校准的仿真里，闭式解预测的最优比与仿真搜出的最优比相差在 10% 以内。[\[2\]](https://arxiv.org/abs/2601.21351)

这类模型化方法有它自己的边界。前期调研在 AIConfigurator 的源码审查里留下过一条尚未消解的分歧：论文报告的搜索加速与配置收益成立于其 workload 假设内，但 rate matching 依赖固定经验因子、内存模型不建模 KV 容量约束，外推到 agent 场景那种频繁短请求时需要重新校准。[\[24\]](https://miraclefarms.github.io/notes/2026/05/26/nvidia-aiconfigurator-disaggregated-serving/) 闭式解和仿真都属于同一类工具，同一条警告适用。

## 五、什么时候不该拆：三个失效机制和一个结构性反例

> **本节论点**：AFD 有一个由互联带宽决定的天花板，越过之后加 FFN 节点不再涨利用率；而 instance 级 P/D 分离在 MoE 上有一个结构性缺陷——它要求两侧各持一份完整权重。 **本节核心参考**：
> 
> - ★★★ **Revealing the Challenges of Attention-FFN Disaggregation（arXiv 2602.09721，2026-02-10）**[\[3\]](https://arxiv.org/abs/2602.09721) — 扩展 roofline 给出的三个失效机制，本节主线证据
> - ★★★ **ExpertPlex（arXiv 2607.18002，2026-07-20）**[\[4\]](https://arxiv.org/abs/2607.18002) — 对 instance 级 P/D 分离的结构性反驳
> - ★★ TaiChi[\[17\]](https://arxiv.org/abs/2508.01989) — 聚合与分离之间的 SLO 判据
> - ★ 前期调研文章《AFD 没有统一最优解：Attention–FFN 拆分的收益边界》[\[7\]](https://miraclefarms.github.io/notes/2026/07/28/attention-ffn-disaggregation-design-space/) — 128 张 B200 的设计空间搜索

前四节都在讲怎么拆。这一节讲拆分会在哪里亏。

2602.09721 是目前唯一系统性给出 AFD 负面结论的工作。它把 roofline 模型扩展到同时考虑互联带宽、算术强度和硬件 FLOPS 利用率（HFU），然后在 H20、H800、H100、H200、B200、B300、GB200、GB300 八个平台上扫 DeepSeek-V3 的配置。

![[PD 分离与 AF 分离：大模型推理架构的技术解析与适用边界探讨-07.png]]

*图 6：DeepSeek-V3 在 H800 上归一化算术强度随 FFN 节点数 N\_F 的变化。蓝色虚线是不考虑带宽上限的理论值，随 N\_F 线性上升；红色实线是加入带宽天花板后的实际值，呈阶梯状——在 N\_F 约 2.5 到 8 之间完全走平（图中标注为 “Stable Intensity”），在 8 到 32 之间又出现一段更长的平台。平台段就是论文说的 dead zone：增加 FFN 实例既不涨算术强度也不涨 HFU，纯粹是浪费。这张图说明 AFD 的扩缩不是连续的，它踩在带宽决定的台阶上。来源：arXiv 2602.09721 Figure 2，图中数据。*

论文归纳出三个失效机制。第一个是这张图里的 dead zone：带宽封顶时增加 FFN 实例不改变任何东西。第二个是固定延迟预算下，规模越大单个算子的活跃时间越短，算子效率随之下降，抵消掉一部分扩展收益。第三个是节点粒度的离散性——AFD 只能整节点增减，而大规模 EP 可以连续调 batch，遇到负载倾斜时 AFD 付出的不均衡代价更高。三条合起来，论文给出的适用条件是：**Superpod 级互联带宽、粗粒度专家、较低稀疏度**——三个条件同时成立时 AFD 才有稳定收益。[\[3\]](https://arxiv.org/abs/2602.09721)

这个结论和 FastAFD 是一致的，只是符号相反。FastAFD 自己也承认，当 FFN 侧执行时间超过 attention 侧（T\_e > T\_a）时，MoE 会暴露在关键路径上，AFD 反而变差；而且”NVL72 上的共置基线本来就很强”。[\[1\]](https://haoailab.com/blogs/fastafd/) 两边说的是同一件事：AFD 的收益来自带宽充裕时的重新配平，带宽一旦成为约束，重新配平就失去了空间。

P/D 这一侧的反例来自 ExpertPlex，角度更结构化。

![[PD 分离与 AF 分离：大模型推理架构的技术解析与适用边界探讨-08.png]]

*图 7：论文对 prefill-decode 共置方案的问题刻画。左侧是 GPU 内按固定比例切分（prefill 2/3、decode 1/3）带来的网络干扰；右侧的时间线展示固定切分下的三类浪费：decode 启动被 prefill 阻塞、Attention 与 MoE 的资源诉求不同却共用同一份切分、以及动态专家计算结束后留下的气泡。这张图说明”按阶段切资源”在算子粒度上必然留下空档，是 ExpertPlex 转向专家共享的动机。来源：arXiv 2607.18002 Figure 2。*

ExpertPlex 的主张是：instance 级的 P/D 分离要求 prefill 池和 decode 池各持一份完整模型副本。配置好的 P:D 比例几乎必然与实时需求错配，一侧过配、另一侧过载，而权重的重复占用把这个错配的代价放大。它的做法是让专家参数跨 P/D 共享，只把轻量的 Attention 模块分离出去，再用 adaptive persistent kernel 在 tile 粒度上调度动态专家计算。结果是 goodput 比 instance 级 P/D 分离高 2.01×、比共置高 1.66×，并消除了 95% 以上的重复权重。[\[4\]](https://arxiv.org/abs/2607.18002)

**对 MoE 而言，该分的是 Attention 与 Expert，而 Prefill 与 Decode 的切分线正在失去意义。** 这是第三节那条”状态归属”逻辑推到底的自然结果：专家权重既不属于 prefill 也不属于 decode，按阶段复制它本来就没有依据。

选择聚合还是分离，TaiChi 提供了一条可直接用的判据：TTFT 紧、TPOT 松时聚合部署更优，TPOT 紧、TTFT 松时分离更优。[\[17\]](https://arxiv.org/abs/2508.01989) 前期调研在 128 张 B200 上的设计空间搜索给出了同向的结论——聚合部署拿下多数系统吞吐面板，AFD 则在所有用户交互延迟前沿上胜出；只有当聚合模式撞上单卡容量墙时（1M prefix 场景下聚合估算需要约 298 GiB 而 B200 只有 180 GiB，AFD 按角色切开后降到约 165 GiB），AFD 才从”更快”变成”唯一可行”。[\[7\]](https://miraclefarms.github.io/notes/2026/07/28/attention-ffn-disaggregation-design-space/)

把这些放在一起，可以整理成一张判定表：

| 条件                       | 倾向                   | 依据                                                                                                                   |
| ------------------------ | -------------------- | -------------------------------------------------------------------------------------------------------------------- |
| Dense 模型                 | 不拆 AF                | A↔F 激活传输无 all-to-all 可摊销[\[1\]](https://haoailab.com/blogs/fastafd/)                                                 |
| MoE + 单 NVLink 域 + 长上下文  | 拆 AF                 | KV 容量与专家 batch 解耦，1.35–1.45×[\[1\]](https://haoailab.com/blogs/fastafd/)                                             |
| MoE + 跨机 RDMA + 细粒度专家高稀疏 | 谨慎                   | dead zone 与离散扩缩惩罚[\[3\]](https://arxiv.org/abs/2602.09721)                                                           |
| TTFT 紧、TPOT 松            | 倾向聚合                 | goodput 判据[\[17\]](https://arxiv.org/abs/2508.01989)                                                                 |
| TPOT 紧、TTFT 松            | 倾向分离                 | 同上                                                                                                                   |
| 聚合模式单卡 OOM               | 必须拆                  | 298 GiB → 165 GiB[\[7\]](https://miraclefarms.github.io/notes/2026/07/28/attention-ffn-disaggregation-design-space/) |
| MoE 且 P:D 比例长期错配         | 考虑专家共享而非 instance 分离 | 消除 95%+ 重复权重[\[4\]](https://arxiv.org/abs/2607.18002)                                                                |

## 六、边界会移动：当分离从软件切分变成硬件分工

> **本节论点**：AFD 的适用边界正在被硬件重写——算子级调频和专用解码芯片都把”该不该拆”变成了”已经拆好了”。 **本节核心参考**：
> 
> - ★★★ **AFlex: Energy-Efficient LLM Serving via Disaggregated Attention–FFN and Flexible Frequency Scaling（arXiv 2608.01891，2026-08-03）**[\[6\]](https://arxiv.org/abs/2608.01891) — 算子级频率敏感度实测，本节主线证据
> - ★★ SemiAnalysis《GTC 2026 – The Inference Kingdom Expands》[\[15\]](https://newsletter.semianalysis.com/p/nvidia-the-inference-kingdom-expands) — NVIDIA LPX 机架的 Attention/FFN 硬件分工
> - ★★ FastAFD 的 Vera Rubin + LPX 投影[\[1\]](https://haoailab.com/blogs/fastafd/) — 新硬件下 AFD 收益的模型化外推
> - ★ Chiplog《Inside Attention-FFN disaggregation》[\[16\]](https://www.chiplog.io/p/inside-attention-ffn-disaggregation) — 硬件专用化叙事（部分付费墙，数字未能二次核对）

前五节都在软件层讨论。AFD 的第三个理由不在软件里。

![[PD 分离与 AF 分离：大模型推理架构的技术解析与适用边界探讨-09.png]]

*图 8：decode 阶段 Attention（DA）与 FFN（DF）算子对 GPU 频率的敏感度，横轴张量并行度、纵轴 batch size，数值越高越敏感。左上 DA 的延迟敏感度随 batch 与 TP 剧烈变化，从 0.07（batch 1、TP8）跨到 0.84（batch 256、TP1）；右上 DF 的分布明显更平，最低也有 0.46。下排能耗敏感度呈现另一套分布。这张图说明两类算子的能耗最优频率本就不同，且随 batch 与并行度漂移——共置时只能取一个折中频率，分离之后才谈得上分别调。来源：arXiv 2608.01891 Figure 3，图中数据。*

AFlex 基于这个观察做了一层全局调度器加算子级 DVFS 的双层控制，配合可变深度的 A/F 交错流水线，在守住 TTFT/TPOT SLO 的前提下把能耗压到比现有分离式方案低 49%、比调频方案低 48%。[\[6\]](https://arxiv.org/abs/2608.01891) **这条收益和吞吐、延迟都无关，它给”为什么要拆”补上了一个独立的理由。**

更彻底的一步发生在芯片层面。NVIDIA 以 200 亿美元拿下 Groq 的 IP 与团队后，在 GTC 2026 上发布了 LPX 机架——32 个计算托盘、每托盘 16 颗 LPU，共 512 颗，通过 Groq 的 C2C 协议互联而非 NVLink；单颗 LPU 配 500 MB 片上 SRAM 与 1.2 PFLOPS FP8 算力。SemiAnalysis 记录的官方定位是把 attention 映射到 GPU（需要动态 KV 访问和大 HBM 容量），把 FFN 映射到 LPU（无状态、确定性计算）。[\[15\]](https://newsletter.semianalysis.com/p/nvidia-the-inference-kingdom-expands)

这正是第三节那条状态归属逻辑的硬件版本。软件层争论了两年的”AFD 该不该拆”，在硬件层被回答成”已经拆好了，接口在这里”。

FastAFD 顺着这个方向做了一次外推。如果 LPX 承接 MoE 且能被 Rubin 侧剩余工作完全掩盖，加速比化简为 1/(1 − MoE 占比)：MiniMax-M2.5 在 8K/16K 上下文分别为 1.68× 与 1.57×，Qwen3-235B 为 1.75× 与 1.71×；若把边界推到 dense + MoE 一起卸载，上限可达 2.07–2.42×，代价是更紧的依赖关系和更快的 LPX dense kernel。配置假设是每个 Rubin 机架配两个 LPX 机架（合计 256 GB SRAM，容纳约 230 GB 的 checkpoint），每步每个 hidden 元素约传 3 字节。[\[1\]](https://haoailab.com/blogs/fastafd/)

这组数字全部来自模型化投影而非实测，需要按此对待。但方向是清楚的：第五节那个”Superpod 级互联带宽”的门槛，正在被专门为这种分工设计的机架式互联往下压。**判定表里的条件不会变，会变的是有多少部署环境天然满足这些条件。**

## 七、结论

回到开篇的判断。P/D 分离和 AF 分离都不是”更先进的架构”，它们是两次针对不同耦合的解耦：前者切断 prefill 与 decode 在同一批硬件上的资源竞争，后者切断 KV 容量对专家批量的隐性封顶。每一次解耦都引入新的耦合——P/D 引入了跨节点 KV 传输和 P:D 比例，AF 引入了每层两趟通信和 A:F 比例。收益是否为正，取决于新引入的耦合是否比被切断的那个更容易管理。

这个框架下，本文的判断成立于三个条件：模型是 MoE（否则 AF 分离的通信无处摊销）、互联带宽在 Superpod 级（否则会撞上 dead zone）、SLO 形状是 TPOT 紧而 TTFT 相对松（否则聚合部署往往更经济）。三个条件之外，共置部署仍然是一个体面的答案，尤其是在可以大量复制完整 replica 的吞吐型服务里。

三个问题仍然开放。第一，几乎所有 AFD 的公开数据都来自 GB200 NVL72 或 Ascend CloudMatrix 这类超节点，H20 及国产卡上的最优 A:F 配比缺少公开的数值锚点——而这恰恰是国内多数集群面对的环境。第二，Kairos 揭示的”prefill 只占 TTFT 的 2–23%”是在传统 chat 类 trace 上测的，agent 负载的请求形态（高 prefix 命中、频繁短请求、工具调用打断）会把这个比例推向哪一侧还没有公开数据。第三，ExpertPlex 指向的专家共享路线如果成立，P/D 分离在 MoE 上的地位需要重新评估，但目前只有一篇论文，缺少独立复现。

对于要做部署决策的人，一个可操作的顺序是：先用 SLO 形状判断该聚合还是该分离，再用模型稀疏结构判断该不该做 AF 分离，最后才去搜比例。反过来做——先拆再想为什么——大概率会停在曲线的错误一端。

- - -

## 参考资料

\[1\] [FastAFD: Open-Source Large-Scale Attention-FFN Disaggregation on Blackwell NVL72](https://haoailab.com/blogs/fastafd/)

\[2\] [Analytical Provisioning for Attention-FFN Disaggregated LLM Serving under Stochastic Workloads](https://arxiv.org/abs/2601.21351)

\[3\] [Revealing the Challenges of Attention-FFN Disaggregation for Modern MoE Models and Hardware Systems](https://arxiv.org/abs/2602.09721)

\[4\] [ExpertPlex: A High-Goodput Disaggregated Serving System for MoE LLMs with Adaptive Persistent Kernels](https://arxiv.org/abs/2607.18002)

\[5\] [Towards Load-Aware Prefill Deflection for Disaggregated LLM Serving](https://arxiv.org/abs/2607.02043)

\[6\] [Energy-Efficient LLM Serving via Disaggregated Attention–FFN and Flexible Frequency Scaling](https://arxiv.org/abs/2608.01891)

\[7\] [前期调研：AFD 没有统一最优解——Attention–FFN 拆分的收益边界](https://miraclefarms.github.io/notes/2026/07/28/attention-ffn-disaggregation-design-space/)

\[8\] [MegaScale-Infer: Serving Mixture-of-Experts at Scale with Disaggregated Expert Parallelism](https://arxiv.org/abs/2504.02263)

\[9\] [Step-3 is Large yet Affordable: Model-system Co-design for Cost-effective Decoding](https://arxiv.org/abs/2507.19427)

\[10\] [Announcing vLLM AFD Plugin: Disaggregating Attention and FFN for Flexible MoE Serving](https://vllm.ai/blog/2026-07-23-vllm-afd-plugin)

\[11\] [vLLM RFC #22799: ATTN-FFN Disaggregation for MoE Models](https://github.com/vllm-project/vllm/issues/22799)

\[12\] [vLLM RFC #27584: Elastic Attn-FFN Disaggregation](https://github.com/vllm-project/vllm/issues/27584)

\[13\] [vllm-project/afd-plugin](https://github.com/vllm-project/afd-plugin)

\[14\] [Prefill/decode disaggregation reaches production: state of play, May 2026](https://lecompute.fr/en/runtimes/disaggregation-prefill-decode-production/)

\[15\] [GTC 2026 – The Inference Kingdom Expands](https://newsletter.semianalysis.com/p/nvidia-the-inference-kingdom-expands)

\[16\] [Inside Attention-FFN disaggregation: What Groq LP40 will look like](https://www.chiplog.io/p/inside-attention-ffn-disaggregation)

\[17\] [Prefill-Decode Aggregation or Disaggregation? Unifying Both for Goodput-Optimized LLM Serving](https://arxiv.org/abs/2508.01989)

\[18\] [DistServe: Disaggregating Prefill and Decoding for Goodput-optimized Large Language Model Serving](https://arxiv.org/abs/2401.09670)

\[19\] [前期调研：LLMCompass——用 4.1% 误差的硬件评估框架，重写 LLM 推理的性价比公式](https://miraclefarms.github.io/notes/2026/07/21/llmcompass-hardware-eval-framework/)

\[20\] [前期调研：vLLM AFD Plugin——把 MoE 的 Attention 与 FFN 拆成两套服务之后](https://miraclefarms.github.io/notes/2026/07/24/vllm-afd-plugin-moe-serving/)

\[21\] [前期调研：异构 P/D 推理的真正边界——算力、格式与 KV 所有权](https://miraclefarms.github.io/notes/2026/07/03/heterogeneous-pd-inference-boundary/)

\[22\] [前期调研：P/D 分离推理的失控点——Dynamo 路由博弈阅读](https://miraclefarms.github.io/notes/2026/07/03/dynamo-disaggregated-inference-price-of-anarchy/)

\[23\] [前期调研：xDeepServe——在 384 NPU 上把 MoE 推理拆成流水线](https://miraclefarms.github.io/notes/2026/05/26/xdeepserve-moe-disaggregated-serving/)

\[24\] [前期调研：AIConfigurator——把分离式推理的配置搜索从 GPU 搬到 CPU](https://miraclefarms.github.io/notes/2026/05/26/nvidia-aiconfigurator-disaggregated-serving/)

\[25\] [SGLang Roadmap: Prefill-Decode Disaggregation (2026 Q2)](https://github.com/sgl-project/sglang/issues/21703)

\[26\] [Gimbal: Coordinated Scheduling for MoE LLM Serving](https://arxiv.org/abs/2606.15177)

### 版本对齐信息

| 材料                        | 版本 / 时间                                  | 说明                                                                                                                       |
| ------------------------- | ---------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `vllm-project/afd-plugin` | main 分支，查询日期 2026-08-25                  | 外置插件，非 vLLM 主干；README 明确列出 model runner v2 不支持、CUDA graph 仅 decode-only 等限制                                              |
| vLLM RFC #22799 / #27584  | 均为 closed as not planned，查询日期 2026-08-25 | 主干未采纳，不视为稳定 API                                                                                                          |
| SGLang                    | 0.5.12（2026-05）                          | PD 分离一等公民，NIXL + MORI-IO 双后端；来源为二手盘点[\[14\]](https://lecompute.fr/en/runtimes/disaggregation-prefill-decode-production/) |
| NVIDIA Dynamo             | 1.0（2026-03）                             | KV 亲和路由与拓扑感知放置                                                                                                           |
| TensorRT-LLM              | 1.3                                      | Helix Parallelism                                                                                                        |
| arXiv 2601.21351          | v4，2026-08-14                            | 本文引用的闭式解结论以 v4 为准                                                                                                        |

## 附录

以下支线与主线相关，但展开会冲淡各节的单一主题，在此列出去向。

**A. AFD 之外的 MoE 调度手段。** Gimbal 走的是不拆算子的路线：用 KV 占用、队列压力、专家负载三类后端信号做细粒度 DP engine 调度，再用混合整数非线性规划做专家放置，报告 TTFT 降低 42.9%、TPOT 降低 33.3%。[\[26\]](https://arxiv.org/abs/2606.15177) 它和 AFD 解决的是同一个利用率问题，但落点在调度而非拓扑，两者可叠加。

**B. SGLang 侧尚未闭合的工程债。** SGLang 2026 Q2 的 PD 分离路线图把”与全部并行策略及投机解码的完全兼容”和”生产级可靠性”列为目标，这份清单是判断 P/D 分离在框架侧还剩多少未完成工作的一手材料。[\[25\]](https://github.com/sgl-project/sglang/issues/21703)

**C. 硬件专用化的产业叙事。** Chiplog 从 attention 要容量、FFN 要带宽的角度推演 Groq 系列芯片的形态与 NVIDIA 收购逻辑。[\[16\]](https://www.chiplog.io/p/inside-attention-ffn-disaggregation) 该文对具体部署规模与降本倍数的引述位于付费墙之后，本文未能二次核对，因此正文只采用了可从 SemiAnalysis 与一手论文验证的部分。
