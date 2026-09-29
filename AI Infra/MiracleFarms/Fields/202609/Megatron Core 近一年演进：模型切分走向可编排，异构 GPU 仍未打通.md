---
title: "Megatron Core 近一年演进：模型切分走向可编排，异构 GPU 仍未打通"
source: "Fields"
category: "AI Infra"
group: "MiracleFarms"
url: "https://miraclefarms.github.io/notes/2026/09/14/megatron-core-year-in-review-model-partitioning/"
published: 2026-09-14T12:20:00+08:00
saved: 2026-09-27T16:29:49+08:00
folo_key: "mf-fields::https://miraclefarms.github.io/notes/2026/09/14/megatron-core-year-in-review-model-partitioning/"
tags:
  - "folo"
  - "AI_Infra"
  - "Fields"
---

# Megatron Core 近一年演进：模型切分走向可编排，异构 GPU 仍未打通

> [!info] Fields · AI Infra · 2026-09-14 12:20 · [原文](https://miraclefarms.github.io/notes/2026/09/14/megatron-core-year-in-review-model-partitioning/)

![[Megatron Core 近一年演进：模型切分走向可编排，异构 GPU 仍未打-01.png]]

> **版本声明**：本文分析基于 NVIDIA/Megatron-LM commit `3703d4e33a3a`（2026-09-12）[\[7\]](https://github.com/NVIDIA/Megatron-LM/commit/3703d4e33a3a2b2d11ebcc8e41f45af7ce7d1eda)；release 范围为 2025-09-14 至 2026-09-14，尚未进入正式版本的 main 分支能力会单独标注。

如果只扫 release note，Megatron Core 过去一年像是在不停加功能：DeepSeek V3.2、GDN、MTP、Qwen3-VL、MIMO、HybridModel、GTP，外加一长串 CUDA Graph、MoE 和 checkpoint 优化。把代码按“模型究竟在哪里被切开”重新排一遍，主线清楚得多：**Megatron 正在把切分单位从整齐的 Transformer 层，扩展到不等长 pipeline stage、独立子层、独立模态模块和单个权重。**

这条主线也有一道很容易写错的边界。仓库里频繁出现 heterogeneous，但大多数时候说的是模型结构和并行布局异构，并不等于 A100、H100、4090 混在一个训练作业里就会自动均衡。2026 年 1 月，维护者对这个问题给过一句很直接的答复：Megatron Core 当前不支持 heterogeneous GPU training，所有并行策略仍假设同构 GPU[\[22\]](https://github.com/NVIDIA/Megatron-LM/issues/1757#issuecomment-3704491137)。

下面这篇 field note 想把两件事拆开：Megatron 已经把模型异构推进到了什么粒度；跨代、跨架构 GPU 协同训练还缺哪些调度层。

## 一、六个功能版本，变化集中在“切分对象”

近一年恰好覆盖 Megatron Core 0.14 到 0.19 六个 feature release。0.15.1–0.15.3、0.16.1、0.17.1、0.18.2 属于补丁发布，官方没有给出新的 feature milestone，下面不把安全修复和回补提交硬凑成“新能力”。

| 版本            | 最重要的功能与性能变化                                                                                                                                 | 新模型 / 新架构支持                                                                                  | 对模型切分的影响                                                                                                                                                   |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 0.14（2025-10） | HyperCommGrid、ModelCommProcessGroup、EP All-to-All overlap、MoE router fusion；动态推理加入多 batch CUDA Graph                                        | MiMo video VLM、MIMO AVLM，ModelOpt 增加 DeepSeek / Qwen 配置                                      | 通信组从全局固定组合，开始变成可描述的 N 维网格；custom PP layout 正式进入可用版本[\[1\]](https://github.com/NVIDIA/Megatron-LM/releases/tag/core_v0.14.0)                                |
| 0.15（2025-12） | QKV 预处理融合达到 **3 倍预处理加速、10%–14% 端到端提升**；CUDA Graph runner cache 在动态推理中最高 2 倍                                                                 | GPT-OSS 加 YaRN 与 ModelOpt 导入导出，DeepSeek V3 / FP8 路径继续补齐                                      | FSDP 开始支持 joint training of parallel modules；切分后的模块不再只能共享一套优化与状态管理[\[2\]](https://github.com/NVIDIA/Megatron-LM/releases/tag/core_v0.15.0)                 |
| 0.16（2026-02） | optimizer state / master weight CPU offload、细粒度 activation offload、partial CUDA Graph + EP overlap、HybridEP 的 EP64 与 NVL8+IB 优化             | DeepSeek V3.2                                                                                | MTP 可以放进独立 pipeline stage，模型末端的额外预测头不必继续挤在最后一个 decoder stage[\[3\]](https://github.com/NVIDIA/Megatron-LM/releases/tag/core_v0.16.0)                       |
| 0.17（2026-04） | absorbed MLA、MLA down-projection fusion、MTP speculative decoding、inference-optimized MoE；CUDA Graph 覆盖 optimizer、vision encoder、Mamba block | GDN hybrid、Qwen3-VL、GPT-OSS、Nemotron 3 Super                                                 | fVPP 允许混合层按 pattern 指定 PP/VPP 边界；MIMO 增加独立 optimizer、checkpoint 与 multimodule 1F1B[\[4\]](https://github.com/NVIDIA/Megatron-LM/releases/tag/core_v0.17.0) |
| 0.18（2026-06） | Megatron-FSDP 与 EP A2A overlap 接通，HybridEP 融合 permute / dispatch，THD SFT 支持 PP，chunked MLP 进入训练                                             | Flextron；MIMO core primitive                                                                 | encoder 与 LLM 可在同一批 rank 上采用不同 TP/DP，也可经 bridge communicator 变换边界张量[\[5\]](https://github.com/NVIDIA/Megatron-LM/releases/tag/core_v0.18.0)                |
| 0.19（2026-08） | quantile balancing router、packed-sequence MoE dispatch、GDN unified All-to-All、CUDA Graph 兼容 activation offload、流式量化 checkpoint load         | HybridModel、DeepSeek Sparse Attention；DeepSeek V4 已迁到 dev，Nemotron MoE VLM / RADIO 示例进入 MIMO | MIMO non-colocated 路径端到端打通；一个 rank span 可同时拥有 dense 与 expert 两套 factorization[\[6\]](https://github.com/NVIDIA/Megatron-LM/releases/tag/core_v0.19.0)      |

表里最醒目的数字来自 kernel 和 CUDA Graph，切分层的收益却更难用一个百分比概括。原因很现实：**切分 API 只负责扩大可行域，最终吞吐取决于用户有没有把层耗时、显存峰值和链路放在同一张账上。** 0.18 还记录过 H100 上 T5 性能约 10% 的已知回退，而 GB200 不受影响[\[5\]](https://github.com/NVIDIA/Megatron-LM/releases/tag/core_v0.18.0)。版本升级从来不等于所有架构同步变快。

## 二、PP 已支持不同 stage 放不同数量的 layer

早期 Megatron 的默认分层逻辑很规整：`num_layers` 平均除以 PP size；首尾 stage 可以单独指定层数，中间 stage 仍需均分。这个接口能补偿 embedding、loss 或末端 head 的额外工作，却很难表达 61 层 decoder 加一个 MTP 的 DeepSeek-V3 式模型。

现在的 `--pipeline-model-parallel-layout` 直接写每个物理 / 虚拟 stage 持有哪些层。官方示例 `Et*3|(tt|)*29,m|L` 把 61 个 decoder、一个 MTP、embedding 与 loss 排进 PP16、VPP2：PP rank 0 的第一个 virtual stage 放 3 层，rank 1–13 各放 2 层，rank 14 的第二个 virtual stage 只放 MTP，rank 15 的第二个 virtual stage 只做 loss[\[8\]](https://docs.nvidia.com/megatron-core/developer-guide/latest/user-guide/features/pipeline_parallel_layout.html)。空 stage 也合法。

代码里的数据结构是一个 `PP rank × VPP rank` 矩阵。`get_num_layers_to_build()` 逐格计数，`get_layer_offset()` 沿 VPP-major、PP-minor 的顺序累加 offset；embedding、decoder、MTP 和 loss 都成为显式 layer type[\[9\]](https://github.com/NVIDIA/Megatron-LM/blob/3703d4e33a3a2b2d11ebcc8e41f45af7ce7d1eda/megatron/core/transformer/pipeline_parallel_layer_layout.py#L13-L171)。**所以“不同 stage 不同 layer 数量”已经支持，而且能细到每个 virtual stage。**

但它是静态布局语言。配置文件不会先测 H100 每层 18 ms、A100 每层 31 ms，再替你求出 5∶3 的层数比例；它也不理解哪一段跨 NVLink、哪一段跨 IB。用户得先做 profile，再把结果翻译成 layout 字符串，最后用 launcher 的 rank placement 把对应 PP rank 放到目标 GPU。这里的自动化还停在框架外。

MTP standalone 是这套布局的一个典型收益。过去额外预测层堆在末端，最后一个 stage 同时承担 decoder 尾部、MTP 和 loss，VPP 再平均也会歪。0.16 之后，MTP 可以独占一个 virtual stage；当前校验仍要求全部 MTP 位于同一个 virtual stage，且不能落在第一个 PP rank[\[10\]](https://docs.nvidia.com/megatron-core/developer-guide/latest/user-guide/features/multi_token_prediction.html)。灵活性增加了，边界仍然明确。

## 三、HybridModel 把切分粒度推进到 Transformer block 内部

模型变复杂后，“一层”本身就不再等价。一个传统 GPT block 把 self-attention 和 MLP / MoE 绑在同一层号上；Mamba hybrid 会穿插状态空间层，DeepSeek 路径又加 MLA、DSA、MTP。按层数均分，很可能每个 stage 都是 8 层，计算量却差一截。

HybridModel 的处理很干脆：一个 pattern 字符代表一个 layer family。`M` 是 Mamba-2，`G` 是 GDN，`*` 是 self-attention，`D` 是 DSA，`-` 是 dense MLP，`E` 是 MoE；原来一个 GPT block 会被展开为 `*-` 或 `*E` 两个可独立编排的位置[\[11\]](https://github.com/NVIDIA/Megatron-LM/blob/3703d4e33a3a2b2d11ebcc8e41f45af7ce7d1eda/docs/user-guide/hybrid-model-migration.md#L18-L69)。`|` 直接插 pipeline segment 边界，`/` 追加 MTP pattern。0.17 合入的 fVPP 就建立在这套表示上[\[12\]](https://github.com/NVIDIA/Megatron-LM/pull/3377)。

这给了 PP 更细的手术刀。某个 attention 很重、相邻 MLP 较轻时，边界可以落在二者之间；MoE 层、GDN 层和 Mamba 层也不必被当作同成本的“第 N 层”。**细粒度切分已经从 transformer block 进入 layer family，但仍没进入任意算子 DAG。** 通信边界越多，activation 的 shape、残差连接和 checkpoint 映射越容易出错，所以 HybridModel 选择了有限符号表，而非让用户在任意 op 后画一条竖线。

0.19 之后的 main 又往下走了一层。Generalized Tensor Parallelism（GTP）把权重按 `TP × GTP_remat` 切分，在每次 GEMM 前逐权重异步 all-gather，用完释放，反向再 reduce-scatter；它能和 TP、SP、EP、DDP、CUDA Graph 组合[\[13\]](https://github.com/NVIDIA/Megatron-LM/blob/3703d4e33a3a2b2d11ebcc8e41f45af7ce7d1eda/docs/api-guide/core/generalized_tensor_parallel.md#L1-L43)。官方 GB200 NVL72 实验用约 280B 的 hybrid Mamba-MoE proxy，在 128 到 3072 GPU 上保持单域基线 **93% 以上弱扩展效率**，reserved memory 随 DP 扩大从 137 GB 降到 104 GB[\[14\]](https://github.com/NVIDIA/Megatron-LM/blob/3703d4e33a3a2b2d11ebcc8e41f45af7ce7d1eda/docs/api-guide/core/generalized_tensor_parallel.md#L198-L218)。

这部分截至 2026-09-12 仍标为 experimental，并要求 Transformer Engine 2.19 以上；它属于 0.19 之后的 main 能力，不能倒写成 0.19 的稳定承诺。它解决的是单个权重的存储和通信粒度，也不会自动决定哪张 GPU 应该多拿几层。

## 四、多模态切分：MIMO 让每个模块拥有自己的并行网格

多模态把“均匀切层”推到了极限。vision encoder 输入有限、attention 形态固定，LLM 却可能面对 128K 融合序列；把两者绑在相同 TP/CP/PP/DP 上，小 encoder 会继承为大 LLM 设计的通信代价。

![[Megatron Core 近一年演进：模型切分走向可编排，异构 GPU 仍未打-02.png]]

 *图 1：MIMO 的两种 placement。colocated 模式让同一组物理 rank 以 encoder 与 LLM 两套逻辑网格重新解释；non-colocated 模式把模块放在不相交 rank 集合，经 boundary communicator 交接 activation 与 gradient。来源：Heterogeneous Parallelism for Multimodal Large Language Model Training。*

MIMO 给出的答案是每个模块一张 `HyperCommGrid`。同一批物理 rank 可以采用 colocated 模式：encoder 用 TP2/DP2，LLM 用 TP4/DP1；也可以采用 non-colocated 模式，把 encoder 和 LLM 放在完全不相交的 rank span。边界 communicator 在前向重排 activation，在反向把 gradient 按原布局送回[\[15\]](https://arxiv.org/abs/2605.27678)。

这套抽象不只换 TP/DP。PR #3211 允许 encoder 与 language module 使用独立 PP grid[\[16\]](https://github.com/NVIDIA/Megatron-LM/pull/3211)；当前 topology builder 为每个模块分别创建 TP、CP、GTP、DP、PP，expert view 还能在同一 rank span 上换成 ETP、EP、expert-DP，并只共享 PP 维[\[17\]](https://github.com/NVIDIA/Megatron-LM/blob/3703d4e33a3a2b2d11ebcc8e41f45af7ce7d1eda/examples/mimo/training/topology.py#L23-L177)。0.19 的端到端 non-colocated 训练把 training loop、dual gradient finalization、checkpoint 和 optimizer 都接通了[\[18\]](https://github.com/NVIDIA/Megatron-LM/pull/5602)。

![[Megatron Core 近一年演进：模型切分走向可编排，异构 GPU 仍未打-03.png]]

 *图 2：MIMO 论文的 operating regime 汇总。共享 GPU 且显存有余量时，colocated 先通过独立 layout 解开 encoder 与 LLM；encoder 压力上升后，non-colocated 的独立 rank island 开始占优。图中是相对优化过的 homogeneous baseline 的增益，不是对未经调优基线的对比。来源同图 1。*

论文报告 colocated heterogeneous 最高提升 49.3% TFLOPS/GPU；non-colocated 最高提升 13.0% aggregate token throughput 与 9.6% TFLOPS/GPU[\[15\]](https://arxiv.org/abs/2605.27678)。图 2 也揭示了边界：小 encoder 往往适合 colocated，120B LLM 搭配更大 encoder 时才更容易由 non-colocated 获益。没有一条 placement 规则能覆盖所有模型尺寸与 vision-token 比例。

框架在调优层补了三块短板。encoder prefetch 用有界 FIFO 提前准备 frozen encoder feature，去掉不必要的 H2D 与 GPU→CPU 同步[\[19\]](https://github.com/NVIDIA/Megatron-LM/pull/5833)；MIMO wrapper 把 `no_sync`、grad reduce 和 param gather 的生命周期暴露给 stock training loop，使各模块 DDP overlap 真正生效[\[20\]](https://github.com/NVIDIA/Megatron-LM/pull/5979)；随后又批量化 bridge fan-in / fan-out P2P，并允许各活动模块的第一个 PP stage 做梯度 overlap[\[21\]](https://github.com/NVIDIA/Megatron-LM/pull/6606)。

这些优化是在已选 layout 上隐藏通信。它们还不是 cost-model 驱动的自动布局器。

## 五、负载均衡已经分成三层，硬件层仍是空白

读到这里，load balancing 这个词至少有三种含义。

第一层是 MoE token balance。0.19 的 quantile balancing router 根据 expert routing score 的分位数更新 per-expert bias，以无 auxiliary loss 的方式纠正热点[\[23\]](https://github.com/NVIDIA/Megatron-LM/pull/5349)；routing trace、expert concentration 和 one-layer-ahead predictability 工具则帮助判断失衡来自数据、router 还是通信。DeepEP、HybridEP、NCCL dispatcher、packed sequence 与 permute fusion 处理的是 token 到 expert 的动态不均匀。

第二层是模型结构 balance。custom PP layout、fVPP、HybridModel 和 MIMO 都在扩充“哪些工作能被分开”的表达力。GDN 的 THD 路径把 per-sequence All-to-All 合成一次 unified All-to-All，在 Qwen3-Next proxy、16K sequence、CP4 条件下把 iteration time 改善约 7%–10%[\[24\]](https://github.com/NVIDIA/Megatron-LM/pull/4913)。这里处理的是变长序列造成的通信碎片，并非 GPU 快慢差。

第三层才是硬件 balance：给每种 GPU 建算子 profile，考虑显存、NVLink / PCIe / IB 拓扑，搜索每个 PP stage 的 layer family、micro-batch 数与 rank placement。**Megatron Core 没有这层 profiler、cost estimator 和 planner。**

如果只做同构集群里的手工调优，我会把每个 stage 的目标时间写成：

`T_stage = T_forward + T_backward + T_exposed_P2P + T_unhidden_collective`

然后按最大 `T_stage` 最小化，而不是按 layer 数量相等。HybridModel 里 attention、MoE、Mamba 与 GDN 要分别测；多模态里 encoder、projection 和 LLM 也要分开测。显存约束要用峰值 activation + optimizer / grad state，而非参数量近似。layout 字符串只是求解结果的载体。

换成混合 GPU 后，这个实验方案可以帮助理解瓶颈，却不能把配置变成官方支持。起初我也想过，既然 custom layout 可以让 H100 放 6 层、A100 放 3 层，手工绑 rank 不就够了吗？问题出在余下的全栈：不同架构的 kernel 可用性、FP8 / FP4 recipe、collective 速度、CUDA Graph、内存容量与 straggler 行为仍会进入同一个同步训练作业。缺了硬件感知 planner 与运行时反馈，静态层数比例只能覆盖稳态计算，覆盖不了通信尾部和动态抖动。

## 六、interleaved PP 在异构场景里的四个局限

Interleaved 1F1B 的直觉是把每个物理 PP stage 再切成多个 model chunk，让后面的 virtual chunk 提前填补流水线首尾气泡。同构集群、规则 Transformer 上，这通常有效。异构 stage 下，四个限制会一起冒出来。

**第一，所有物理 PP rank 共享同一个 VPP size。** `PipelineParallelLayerLayout` 的 stage 总数必须能被 PP size 整除，随后固定转换为 `PP × VPP` 矩阵；schedule 也把 `len(model)` 当成全局 `num_model_chunks`[\[25\]](https://github.com/NVIDIA/Megatron-LM/blob/3703d4e33a3a2b2d11ebcc8e41f45af7ce7d1eda/megatron/core/pipeline_parallel/schedules.py#L1019-L1083)。每格可以放不同层数甚至为空，但快 GPU 不能天然拥有 5 个 chunk、慢 GPU 只有 2 个 chunk。用户只能用空格子和不同层数模拟，调度轮次仍按统一 chunk 网格推进。

**第二，interleaving 缩短的是拓扑气泡，无法自动消掉速度失配。** 一个慢 stage 的 virtual chunk 仍会挡住相邻 stage。若切得更碎，P2P 次数与 activation stash 随之增加。社区在 non-interleaved P2P overlap 的讨论里给过一个很实用的反例：micro-batch 已很多或 PP 很小时，原始气泡本来就小，VPP 新增的 P2P 和 per-chunk activation 保存可能让收益转负[\[26\]](https://github.com/NVIDIA/Megatron-LM/issues/4385#issuecomment-3834875050)。

**第三，异构模型边界经常伴随 tensor shape 变化，而当前 interleaved schedule 明确拒绝 `adjust_tensor_shapes_fn`。** MIMO 为此走 graph-aware multimodule communicator 与独立调度扩展，不能把任意 encoder→LLM 边界直接塞进通用 interleaved 1F1B[\[25\]](https://github.com/NVIDIA/Megatron-LM/blob/3703d4e33a3a2b2d11ebcc8e41f45af7ce7d1eda/megatron/core/pipeline_parallel/schedules.py#L1074-L1083)。

**第四，MoE overlap 会放大 collective 顺序约束。** Issue #1810 记录了 uneven stage 配合 `--overlap-moe-expert-parallel-comm` 在大规模作业里可能 deadlock 的案例，提报者后来也撤回了最初的单 event 根因判断，维护者未能稳定复现；截至本文版本该 issue 仍开放[\[27\]](https://github.com/NVIDIA/Megatron-LM/issues/1810)。这条证据只够支持“组合路径要做规模回归”，还不足以宣布 uneven fVPP 本身存在确定缺陷。

因此，异构 PP 的优化对象应该是“每个 virtual chunk 的时间 + 边界通信 + 调度顺序”，层数只是输入变量。interleaving 是执行模板，不是负载均衡算法。

## 七、不同架构 GPU 能否一起训练一个大模型

就 Megatron Core 官方边界而言，答案很明确：**通用的异构 GPU 协同训练尚不支持。** 维护者明确说所有并行策略假设 homogeneous GPU，并把用户引向社区项目 MegatronApp[\[22\]](https://github.com/NVIDIA/Megatron-LM/issues/1757#issuecomment-3704491137)。同属 NVIDIA 的跨代 GPU 也许能被 PyTorch / NCCL 拉进同一 world，但“能启动”不等于有受支持的负载均衡、数值配方和性能保证；跨 CUDA、ROCm 或其他加速器后端，stock topology builder 直接使用 NCCL，更谈不上透明协同。

MIMO 的 non-colocated rank island 提供了一个很有诱惑力的形状：把 vision encoder 放旧卡，LLM 放新卡，中间只传 boundary tensor。论文和代码证明了模块级独立网格与边界 autograd 可以成立，却没有证明跨 GPU architecture 的 kernel、精度和同步组合已经被支持。**它是硬件异构调度可以复用的模型抽象，不是硬件异构产品能力。**

社区已经给出几条补层方案。MegatronApp 的 MegaFBD 把 forward 与 backward 拆成不同逻辑进程和 rank，让两阶段采用不同并行度与资源映射；MegaDPP 用运行进度动态选择下一 micro-batch，尝试缓解 stage 抖动[\[28\]](https://github.com/OpenSQZ/MegatronApp)。技术报告声称整套工具链获得双位数吞吐与利用率提升，但没有给出 MegaFBD 在跨代 GPU 大模型训练上的独立完整消融，现阶段更适合作为工程原型看待[\[29\]](https://arxiv.org/abs/2507.19845)。

自动并行系统走得更远。Metis 把 GPU 类型写进 profiler、cost estimator 与 planner，在 GPT-3、MoE、Wide-ResNet 和三类 GPU 组合上相对传统方案报告 1.05–8.43 倍加速，并在 stage 内偏好 DP，避开异构网络上高频 TP collective[\[30\]](https://www.usenix.org/conference/atc24/presentation/um)。FlashFlex 允许 DP/PP/TP 全非对称切分，用分层图分区在 7B–30B 模型上逼近 FLOP 等价同构集群，最小 MFU 差距为 0.30%[\[31\]](https://arxiv.org/abs/2409.01143)。H2 则把跨厂商 runtime、device-direct RDMA 与 HeteroPP / HeteroAuto 放在一起，在千卡级异构集群训练 100B 模型时相对同构 baseline 最高快 16.37%[\[32\]](https://arxiv.org/abs/2505.17548)。更早的 HetPipe 用 virtual worker + pipeline model parallel + WSP 证明旧卡也能组成有效训练单元，实验中收敛时间最多缩短 49%[\[33\]](https://www.usenix.org/conference/atc20/presentation/park)。

这些系统共同补的是 Megatron 上游缺失的 planner 和 runtime。它们大多不是切换一个 Megatron flag 就能得到的能力，生产采用需要重新评估 checkpoint、optimizer、数值一致性、通信后端与故障恢复。

## 八、今天怎么选切分策略

| 场景                                       | 当前可用路径                           | 需要自己补的部分                                                     |
| ---------------------------------------- | -------------------------------- | ------------------------------------------------------------ |
| 同构 GPU、规则 dense Transformer              | TP / PP / VPP / DP / CP 组合       | 按 profile 选择并行度，避免 VPP 过细造成额外 P2P                            |
| 同构 GPU、首尾或 MTP 负载不均                      | `pipeline_model_parallel_layout` | 手工给每个 PP/VPP stage 分层，验证 overlap 组合                          |
| Mamba / GDN / DSA / attention / MoE 混合架构 | HybridModel pattern + fVPP       | 按 layer family 测时；checkpoint 迁移与数值等价验证                       |
| vision / audio encoder + LLM             | MIMO colocated 或 non-colocated   | 独立 module grid、boundary 通信、encoder prefetch 与 DDP overlap 调优 |
| 单权重显存压力，GB200 级高速域                       | experimental GTP                 | TE 版本、权重 gather 域、低精度通信与 prefetch chain 调优                   |
| A100 + H100 / 3080 + 4090                | Megatron Core 无原生支持              | 外部 profiler / planner、rank placement、动态调度与全栈兼容验证             |
| 跨厂商 GPU / NPU                            | 使用 H2、FlashFlex、Metis 一类独立系统思路   | 统一 runtime、跨后端 collective、checkpoint 与算子生态                   |

我的结论落在中间：Megatron 已经具备很强的“切分表达能力”，离“异构硬件自动并行系统”仍隔着一层完整的优化器。0.14 的 custom PP layout 让不同 stage 放不同数量的层；0.17 的 fVPP 与 HybridModel 把边界推进到 attention、MLP、MoE、Mamba、GDN；0.19 的 MIMO 让模块拥有独立 rank grid；最新 main 的 GTP 又把权重本身拆得更细。**这条演进解决了异构训练的表示问题，硬件 profile、代价建模、自动搜索与运行时重平衡还留在框架外。**

所以，当一份配置里出现 `heterogeneous`，先问清它说的是 layer、module、rank layout，还是 GPU architecture。四个词看起来挨得很近，落到 3000 张卡上，差的是一整套调度系统。

- - -

## 参考资料

\[1\] [Megatron Core 0.14.0 release](https://github.com/NVIDIA/Megatron-LM/releases/tag/core_v0.14.0)

\[2\] [Megatron Core 0.15.0 release](https://github.com/NVIDIA/Megatron-LM/releases/tag/core_v0.15.0)

\[3\] [Megatron Core 0.16.0 release](https://github.com/NVIDIA/Megatron-LM/releases/tag/core_v0.16.0)

\[4\] [Megatron Core 0.17.0 release](https://github.com/NVIDIA/Megatron-LM/releases/tag/core_v0.17.0)

\[5\] [Megatron Core 0.18.0 release](https://github.com/NVIDIA/Megatron-LM/releases/tag/core_v0.18.0)

\[6\] [Megatron Core 0.19.0 release](https://github.com/NVIDIA/Megatron-LM/releases/tag/core_v0.19.0)

\[7\] [Megatron-LM commit 3703d4e33a3a](https://github.com/NVIDIA/Megatron-LM/commit/3703d4e33a3a2b2d11ebcc8e41f45af7ce7d1eda)

\[8\] [Megatron Core: Custom Pipeline Model Parallel Layout](https://docs.nvidia.com/megatron-core/developer-guide/latest/user-guide/features/pipeline_parallel_layout.html)

\[9\] [PipelineParallelLayerLayout source at 3703d4e33a3a](https://github.com/NVIDIA/Megatron-LM/blob/3703d4e33a3a2b2d11ebcc8e41f45af7ce7d1eda/megatron/core/transformer/pipeline_parallel_layer_layout.py#L13-L171)

\[10\] [Megatron Core: Multi-Token Prediction pipeline layout](https://docs.nvidia.com/megatron-core/developer-guide/latest/user-guide/features/multi_token_prediction.html)

\[11\] [GPTModel to HybridModel migration guide](https://github.com/NVIDIA/Megatron-LM/blob/3703d4e33a3a2b2d11ebcc8e41f45af7ce7d1eda/docs/user-guide/hybrid-model-migration.md#L18-L69)

\[12\] [Megatron-LM #3377: Add flexible virtual pipeline parallel to hybrid model](https://github.com/NVIDIA/Megatron-LM/pull/3377)

\[13\] [Megatron Core: Generalized Tensor Parallelism design](https://github.com/NVIDIA/Megatron-LM/blob/3703d4e33a3a2b2d11ebcc8e41f45af7ce7d1eda/docs/api-guide/core/generalized_tensor_parallel.md#L1-L43)

\[14\] [GTP weak-scaling results at 128–3072 GPUs](https://github.com/NVIDIA/Megatron-LM/blob/3703d4e33a3a2b2d11ebcc8e41f45af7ce7d1eda/docs/api-guide/core/generalized_tensor_parallel.md#L198-L218)

\[15\] [Heterogeneous Parallelism for Multimodal Large Language Model Training](https://arxiv.org/abs/2605.27678)

\[16\] [Megatron-LM #3211: Multi-module heterogeneous parallelism for MIMO](https://github.com/NVIDIA/Megatron-LM/pull/3211)

\[17\] [MIMO per-module topology builder at 3703d4e33a3a](https://github.com/NVIDIA/Megatron-LM/blob/3703d4e33a3a2b2d11ebcc8e41f45af7ce7d1eda/examples/mimo/training/topology.py#L23-L177)

\[18\] [Megatron-LM #5602: End-to-end non-colocated MIMO training](https://github.com/NVIDIA/Megatron-LM/pull/5602)

\[19\] [Megatron-LM #5833: Encoder prefetch for heterogeneous MIMO training](https://github.com/NVIDIA/Megatron-LM/pull/5833)

\[20\] [Megatron-LM #5979: DDP communication overlap for MIMO](https://github.com/NVIDIA/Megatron-LM/pull/5979)

\[21\] [Megatron-LM #6606: Optimize MIMO pipeline communication](https://github.com/NVIDIA/Megatron-LM/pull/6606)

\[22\] [Megatron-LM maintainer answer on heterogeneous GPU training](https://github.com/NVIDIA/Megatron-LM/issues/1757#issuecomment-3704491137)

\[23\] [Megatron-LM #5349: Quantile Balancing in MoE](https://github.com/NVIDIA/Megatron-LM/pull/5349)

\[24\] [Megatron-LM #4913: Fuse per-sequence All-to-All in GDN](https://github.com/NVIDIA/Megatron-LM/pull/4913)

\[25\] [Interleaved pipeline schedule at 3703d4e33a3a](https://github.com/NVIDIA/Megatron-LM/blob/3703d4e33a3a2b2d11ebcc8e41f45af7ce7d1eda/megatron/core/pipeline_parallel/schedules.py#L1019-L1083)

\[26\] [Megatron-LM #4385: Non-interleaved P2P overlap discussion](https://github.com/NVIDIA/Megatron-LM/issues/4385)

\[27\] [Megatron-LM #1810: Open uneven-stage + MoE overlap deadlock report](https://github.com/NVIDIA/Megatron-LM/issues/1810)

\[28\] [OpenSQZ MegatronApp](https://github.com/OpenSQZ/MegatronApp)

\[29\] [MegatronApp: Efficient and Comprehensive Management on Distributed LLM Training](https://arxiv.org/abs/2507.19845)

\[30\] [Metis: Fast Automatic Distributed Training on Heterogeneous GPUs](https://www.usenix.org/conference/atc24/presentation/um)

\[31\] [FlashFlex: Accommodating Large Language Model Training over Heterogeneous Environment](https://arxiv.org/abs/2409.01143)

\[32\] [H2: Towards Efficient Large-Scale LLM Training on Hyper-Heterogeneous Cluster over 1,000 Chips](https://arxiv.org/abs/2505.17548)

\[33\] [HetPipe: Enabling Large DNN Training on Heterogeneous GPU Clusters](https://www.usenix.org/conference/atc20/presentation/park)

### 版本对齐信息

| 对象                                  | 对齐版本 / commit                        | 状态                              |
| ----------------------------------- | ------------------------------------ | ------------------------------- |
| Megatron Core feature releases      | 0.14.0–0.19.0                        | 已发布                             |
| Megatron-LM main                    | `3703d4e33a3a`，2026-09-12            | 已合入 main；GTP 等能力仍标 experimental |
| fVPP                                | PR #3377，merge commit `7418b1b8fbdd` | 已合并，0.17 release highlight      |
| MIMO non-colocated E2E              | PR #5602，merge commit `052209940d60` | 已合并，0.19 release highlight      |
| MIMO pipeline communication         | PR #6606，merge commit `f481e6361520` | 已合并，0.19 tag 同日附近进入 main        |
| uneven stage + MoE overlap deadlock | Issue #1810                          | 开放 issue；根因与复现尚无定论              |
