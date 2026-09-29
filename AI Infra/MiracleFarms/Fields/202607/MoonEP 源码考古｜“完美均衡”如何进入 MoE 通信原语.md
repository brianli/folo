---
title: "MoonEP 源码考古｜“完美均衡”如何进入 MoE 通信原语"
source: "Fields"
category: "AI Infra"
group: "MiracleFarms"
url: "https://miraclefarms.github.io/notes/2026/07/29/moonep-perfect-balance-source-notes/"
published: 2026-07-29T17:40:00+08:00
saved: 2026-09-27T16:30:43+08:00
folo_key: "mf-fields::https://miraclefarms.github.io/notes/2026/07/29/moonep-perfect-balance-source-notes/"
tags:
  - "folo"
  - "AI_Infra"
  - "Fields"
---

# MoonEP 源码考古｜“完美均衡”如何进入 MoE 通信原语

> [!info] Fields · AI Infra · 2026-07-29 17:40 · [原文](https://miraclefarms.github.io/notes/2026/07/29/moonep-perfect-balance-source-notes/)

![[MoonEP 源码考古｜“完美均衡”如何进入 MoE 通信原语-01.png]]

> **版本声明**：本文分析基于 MoonshotAI/MoonEP commit `0f385f038fc3`（2026-07-28）；该提交相对初始代码只补充了 AcclEP 致谢，正文所述实现来自首个代码提交 `51e64aa55310`（2026-07-27）。

把 NCCL、DeepEP、EPLB、ECHO、UltraEP 和 MoonEP 串成一棵向下生长的技术树，很容易得到一个诱人的结论：MoE Expert Parallelism 正沿着单一路线迭代，MoonEP 位于最新一层。源码给出的画面更复杂。**MoonEP 的关键动作是把在线负载计划、热点专家权重预取、token dispatch/combine、静态计算形状和训练反向梯度归并塞进同一个执行契约。** 它吸收了多条路线的想法，却没有取代这些路线各自解决的通信、异构适配和跨节点扩展问题。

这次源码阅读也修正了 README 中一句过于顺滑的表述。MoonEP 确实保证每个 rank 接收 `S × K` 个真实 routed token；物理计算 buffer 还要容纳每个 VM group 的 padding，实际分配长度是 `NvS = S × K + padding_extra`。所谓“完美均衡”精确指向 rank 级真实 token 负载，不等于所有专家同样热，也不等于物理 buffer 严格只有 `S × K` 行。

## 一、原图抓住了趋势，却把四种关系画成了一种箭头

原图最准确的部分，是看到了 EP 优化对象正在扩张。DeepEP 把 MoE 的 dispatch/combine 从普通 All-to-All 中拆出来，EPLB 一类控制逻辑处理长期专家热度，ECHO 与 UltraEP 把复制决策推进到运行时，MoonEP 再把这些动作压进静态 shape 的通信执行路径。**MoE 的瓶颈已经从“怎样搬 token”扩展为“怎样同时约束通信、热点专家、计算 straggler 和动态内存”。**

问题出在箭头。原图用同一种视觉关系表达了软件依赖、后端移植、算法启发和同类竞争。UCCL-EP 提供 DeepEP-compatible API，并把 GPU/NIC 组合扩展到 NVIDIA、AMD、EFA、Broadcom 等平台[\[1\]](https://github.com/uccl-project/uccl/tree/main/ep)；ROCm/DeepEP 是 AMD 维护的 DeepEP 实现[\[2\]](https://github.com/ROCm/DeepEP)；MoRI 则是更通用的 Modular RDMA Interface[\[3\]](https://github.com/ROCm/mori)。三者都与异构 EP 有关，角色并不相同，把 `rocm/mori` 合并成一个节点会丢掉这层区别。

NCCL 与 DeepEP 也需要版本限定。DeepEP V2 已经从 NVSHMEM 切到 NCCL GIN，并把高吞吐与低延迟接口统一进 `ElasticBuffer`；V1 的关键路径并非建立在 NCCL GIN 上[\[4\]](https://github.com/deepseek-ai/DeepEP)。同时，NVIDIA 另有从 NCCL Device API 原生构建的 NCCL EP，提供 `ncclEpDispatch` 与 `ncclEpCombine` 两个统一原语[\[5\]](https://arxiv.org/abs/2603.13606)。因此，`NCCL → DeepEP → MoonEP` 无法概括当前通信路线。

更合适的画法是三条交叉路径：通信底座与专用 dispatcher、专家负载均衡控制、通信与实时复制的一体化执行。UltraEP 的官方说明也只说通信 kernel 借鉴 DeepEP 与 HybridEP，均衡设计受 EPLB 和 LPLB 启发[\[6\]](https://github.com/Dots-Infra/UltraEP#acknowledgement)；MoonEP 则在 README 中致谢 DeepEP、ECHO、UltraEP 与 AcclEP[\[7\]](https://github.com/MoonshotAI/MoonEP/blob/0f385f038fc33bec22e3bcf5a07a8a22693e754c/README.md#acknowledgments)。这些措辞支持 influence graph，不支持“后者依赖前者运行”或“后者已经替代前者”。

## 二、`Buffer` 先把动态问题框进一组静态上界

MoonEP 的入口很短：调用方创建一个 `Buffer(S, H, K, E, R)`，随后按 `dispatch → prefetch_weight → expert GEMM → combine` 执行；训练反向再复用同一份 plan 做 redispatch、combine 和 `reduce_grad`[\[8\]](https://github.com/MoonshotAI/MoonEP/blob/0f385f038fc33bec22e3bcf5a07a8a22693e754c/moonep/api.py)。接口里最重要的参数没有藏在调优选项中：`S` 是每 rank 输入 token 数，`K` 是 top-k，`E` 是 EP group 内总专家数，`R` 是 rank 数，`B` 是每 rank 的权重预取槽位。

初始化阶段已经把运行时自由度压缩掉一大半。源码要求 `E % R == 0`，默认 `B = E / R`，通信 buffer 和 planner scratch 一次性分配；`R` 必须等于 process group 的 world size。真实 token 容量 `NvS_capacity` 固定为 `S × K`，随后为每个非空 VM group 向 `token_padding` 对齐预留额外空间：

```
NvS_capacity = S × K
padding_extra = (token_padding - 1) × 2 × E / R
NvS = NvS_capacity + padding_extra
```

以 README 的示例 `S=4096、K=8、E=256、R=8、token_padding=128` 计算，每个 rank 的真实 routed token 是 32,768，逻辑 buffer 还会为分组 padding 预留 8,128 行，`NvS` 达到 40,896。**静态 shape 来自可证明的容量上界，并非消除了 padding 成本。** 源码进一步限制 `R ≤ 128`、`K ≤ 32`，以保证 rank bitset、dedup mask 与 int32 编码成立[\[9\]](https://github.com/MoonshotAI/MoonEP/blob/0f385f038fc33bec22e3bcf5a07a8a22693e754c/moonep/planning.py#L1267-L1290)。

这个设计对 CUDA Graph 友好：每层不再把动态 token count 拉回 CPU 决定 tensor shape，Grouped GEMM 只消费固定的 `[E+B, H, H']` 权重视图和 planner 产出的 `cu_seqlens[E+B]`。代价同样清楚——调用方必须接受 MoonEP 的连续对称内存布局、固定容量和生命周期管理，buffer 销毁还要发生在 process group teardown 之前。

## 三、planner 怎样把任意路由压成每 rank 恰好 `S × K`

`planning.py` 是整个仓库最集中的设计说明：一个 cooperative grid launch 在同一 kernel 内完成 Phase A/B/C/D，通过软件 grid barrier 和 NVLink meta buffer 上的 system-scope atomic barrier 同步[\[10\]](https://github.com/MoonshotAI/MoonEP/blob/0f385f038fc33bec22e3bcf5a07a8a22693e754c/moonep/planning.py#L1-L8)。生产实现超过 1,300 行，仓库还放了一份 300 多行的 PyTorch reference，后者更容易看清算法骨架[\[11\]](https://github.com/MoonshotAI/MoonEP/blob/0f385f038fc33bec22e3bcf5a07a8a22693e754c/tests/planning_reference.py)。

planner 先汇总所有 source rank 的 `tokens_per_expert`，再按专家 home group 计算每组总负载。每个 destination rank 的容量 `CAP` 固定为 `S × K`。`balance = group_tokens - CAP` 为正表示超载，为负表示尚有空间；算法反复选择最超载的 home group 与空闲最多的 destination rank，用后者的缺口作为迁移额度。下一步再从超载 home group 中反复挑剩余 token 最多的专家，把它的一部分 token 配给接收 rank。

这里有两个容易被“near-optimal planner”宣传语遮住的实现事实。第一，当前 constructive planner 让每个 destination rank 最多从一个 remote home group 接收 token，因此训练时要求 `B = E/R`，保证该 home group 的所有候选专家都能落入本地预取槽。第二，迁移单位可以是某个专家的一部分 token；planner 复制的是专家权重，随后把该专家的部分 token 改送副本。**均衡发生在 rank 总 token 数这一层，router 的专家选择语义没有被改写。**

最终 plan 不只是一张 rank 映射表。`MoonEPCommPlan` 保存 `dst`、`experts_to_copy`、padding 清零区间、remote statistics，以及处理同一 token 的多个 top-k expert 落到同一 destination rank 时所需的 dedup 结构。`dst` 的非负值表示真正拷贝 hidden state 的 primary entry，后续重复 entry 编成 `-raw_dst-1`，仍保留各自 routing weight。这样，一个 token 发往同一 rank 的多个专家时，hidden state 只跨 NVLink 写一次，dispatch epilogue 再在目标 shard 内展开副本。

## 四、对称内存把“复制专家”变成固定行号

planner 只能回答“哪些 token 去哪里”；Grouped GEMM 还需要在目标 rank 找到对应权重。MoonEP 为每个 projection 建立完全一致的连续 VMM 视图 `[E+B, H, H']`。前 `E` 行对应全局专家，每段物理内存仍归 home rank 所有，但所有 rank 都能通过 NVLink 映射访问；后 `B` 行是本地 prefetch slot。planner 选出的 remote expert 会被复制到这些固定槽位，`cu_seqlens` 决定本轮哪些行实际参与计算。

![[MoonEP 源码考古｜“完美均衡”如何进入 MoE 通信原语-02.png]]

*图 2：所有 rank 看到相同的 `[E+B, H, H']` 行号空间；黄色区域是各自拥有的专家，粉色区域是跨层共享的预取槽。固定行号让动态副本仍能进入静态 Grouped GEMM。来源：MoonEP 官方仓库原图。*

图 2 解释了 README 中“额外成本不是每层 `B` 份专家”的来源：prefetch 物理池是 process-global，多个 layer 的 `[E, E+B)` 虚拟区间映射到同一批槽位。这个技巧显著压低冗余权重的常驻显存，却把调度约束推给执行时序——同一物理池无法被任意并发 layer 无条件复用。

token 路径也围绕同一块 symmetric buffer 设计。dispatch kernel 把 hidden state 直接写到远端最终 expert-grouped offset；`zero_copy=True` 时，API 把本地 NVLink shard 的 view 直接交给 expert FFN，combine 再从同一 view 读回，省掉 comm buffer 与 user buffer 之间的边界拷贝。**这里的 zero-copy 是以 aliasing contract 换来的：下一次 dispatch/combine 会覆盖该 view，autograd 若要为 backward 保存激活，必须退回 `zero_copy=False`。** 源码用 `data_ptr()` 断言 combine 收到的就是 dispatch 暴露的那块 view，避免调用方把普通 tensor 误当成零拷贝输入。

## 五、训练闭环落在 `reduce_grad`，不止是 forward cloning

只在 forward 复制热点专家，容易把系统做成推理专用优化。训练还要回答一个更棘手的问题：副本产生的权重梯度怎样回到唯一的 home expert，同时不被框架原有的 data-parallel gradient reduce 重复计算。

MoonEP 为每个 projection 建立 fp32 `[E+B, H, H']` grad view。`[0,E)` 是 owner rank 的参数梯度，`[E,E+B)` 是副本临时梯度，背后接到独立 reduce buffer；每个 rank 又把所有 rank 的 reduce buffer 映射成 `[R,B,H,H']`。反向阶段由 home rank 远程读取各处属于自己的副本梯度，累加进本地参数梯度，再把消费过的 slot 清零[\[12\]](https://github.com/MoonshotAI/MoonEP/blob/0f385f038fc33bec22e3bcf5a07a8a22693e754c/moonep/grad_reduce.py)。

同一份 forward plan 还会被反向复用。combine backward 用 `dispatch(plan=saved_plan)` 把输出梯度重新散射到 VM group 顺序，跳过 planning；dispatch backward 用 combine 把每个 token 的 `K` 份梯度加回 token-major 布局。**静态 shape、token 去向、权重副本和梯度归并共享同一份 plan，这才是 MoonEP 相比“在 dispatcher 外挂一个 balancer”更紧的地方。**

ECHO 已经展示过相似的训练闭环：forward 克隆热点专家并重定向溢出 token，backward 把副本梯度归并回 home expert，同时缓解 straggler 与 CUDA Graph 的 worst-case buffer 浪费[\[13\]](https://arxiv.org/html/2603.07685v2#S4.SS3.SSS7)。UltraEP 也会基于每个 microbatch、每层的 post-gating 精确负载实时生成副本计划，并把 weight sync 和 gradient reduce 放进训练关键路径[\[14\]](https://github.com/Dots-Infra/UltraEP#2-forward-and-backward)。MoonEP 的差异集中在执行布局：它追求 rank 级严格 `S × K`、直接写最终 expert-grouped 位置，并让 planner 输出直接驱动对称内存中的 VM group 行号。

## 六、官方 benchmark 支持“抗倾斜”，尚未支持“普遍更快”

MoonEP README 的两组 benchmark 都运行在 H20、EP=8。通信对比固定 `E=384、H=7168、K=8、S=8192/rank、32 SMs`，横轴 `MaxVio` 表示最热专家相对均值超出的倍数[\[15\]](https://github.com/MoonshotAI/MoonEP/blob/0f385f038fc33bec22e3bcf5a07a8a22693e754c/benchmarks/bench_vs_deepep.py)。

![[MoonEP 源码考古｜“完美均衡”如何进入 MoE 通信原语-03.png]]

*图 3：官方图中数据。MoonEP 柱子包含 communication、planning 与 prefetch；DeepEP V2 为橙色。低倾斜时 MoonEP dispatch 未占优，倾斜加剧后 combine 和总 dispatch 路径保持近似平坦。来源：MoonEP 官方仓库。*

从图中读数，`MaxVio=0.2` 时 MoonEP dispatch forward 为 2598μs，DeepEP V2 为 2296μs，前者慢约 13.2%；到 `MaxVio=20`，两者分别是 2413μs 和 2604μs，MoonEP 快约 7.3%。combine forward 的差距从 1975μs 对 2126μs扩大到 1954μs 对 2468μs，优势约从 7.1% 增至 20.8%。这些数字把工程判断限定得很清楚：**MoonEP 购买的是负载倾斜下的稳定延迟和固定内存形状，均衡流量下还要支付 planning 与 prefetch。**

![[MoonEP 源码考古｜“完美均衡”如何进入 MoE 通信原语-04.png]]

*图 4：官方端到端训练图把 MoonEP iteration time 归一为 1。DeepEP 在 `MaxVio=0.2/1` 时分别快约 1.9%/0.5%，从 5 开始慢 3.9%，到 15 慢 11.1%，20 时标记 OOM。数据仅来自项目方 H20、EP=8 实验。*

证据边界比曲线本身更重要。仓库提供了 611 行的通信对比脚本，能看到相同 routing、warmup/iteration 与 planning/prefetch 计时方式；但没有提交生成图 3 的原始 CSV。图 4 更缺少对应的端到端性能脚本和原始测量记录，`tests/test_e2e.py` 是正确性 smoke test，无法复现这条训练曲线。当前仓库一共只有两个 commit，首个 commit 一次性加入约 1.29 万行代码，第二个只修改致谢。**这足以支撑源码机制分析，距离跨团队、跨拓扑的生产验证仍有明显距离。**

## 七、哪些边界会直接影响选型

当前实现围绕一个 NVLink domain 展开。它用 CUDA VMM 建立跨 rank 映射，通过 Unix domain socket 传递 POSIX file descriptor，用 NVLink multicast mapping 广播规划元数据；测试命令明确要求多 GPU 与 NVLink。仓库中没有跨节点 RDMA transport，README 公布的设备支持只有 NVIDIA GPU，Zhenwu PPU 仍处于 under review[\[16\]](https://github.com/MoonshotAI/MoonEP#supported-devices)。因此，把 H20、EP=8 的结果外推到 EP64、NVL72、InfiniBand scale-out 或异构 GPU 集群都缺少依据。

负载均衡是否需要进入运行时，也取决于模型训练方式和生产流量。前期调研记录了两组相反场景：xDeepServe 的 DeepSeek-R1 最热专家达到均值 30 倍，EPLB 把解码前向延迟降低 40% 以上；MiMo V2.5 借助训练时负载均衡目标，把生产专家负载度做到 0.8495，并称无需额外推理时均衡。Kimi K3 又把 Quantile Balancing 放进训练过程，但尚未披露生产流量上的专家负载[\[17\]](https://miraclefarms.github.io/notes/2026/07/20/kimi-k3-sparse-moe-training-stability/)。

这组边界给出了一条更实用的选型顺序：先量生产 routing 的 rank-level imbalance，再看静态 CUDA Graph 与内存碎片是否构成瓶颈，最后评估复制权重和归并梯度的代价能否被计算覆盖。训练已经把路由压得足够均匀时，MoonEP 的额外 planner/prefetch 可能只剩成本；热点随 layer、microbatch 和流量快速变化，历史 EPLB 又跟不上时，它的集成路径才开始显出意义。

## 八、结论：MoonEP 是一次强耦合集成，技术版图仍是多条路线

MoonEP 源码把“完美均衡”落实成了一个可检查的不变量：每个 rank 承担 `S × K` 个真实 routed token。planner 用当前 gating 结果把超载 home group 的部分专家 token 分给空闲 rank，prefetch 把必要权重放进固定槽位，dispatch 直写最终 expert-grouped buffer，backward 再沿同一份 plan 归并梯度。**它把通信、均衡和静态执行形状做成了一个整体，代价是更强的内存布局契约、NVLink 域假设和运行时复制开销。**

原图若改成 influence graph，MoonEP 仍应放在通信与实时均衡的交叉处；箭头需要区分实现后端、代码移植、设计启发和竞争路线。DeepEP、HybridEP、NCCL EP、UCCL-EP、ROCm/DeepEP 与 MoRI 继续回答不同硬件和拓扑上的数据搬运问题，EPLB/LPLB 继续代表较松耦合的控制路径，ECHO 与 UltraEP 则与 MoonEP 共享在线复制的设计空间。

下一步要看的已经不在 H20、EP=8 的自测图里：相同模型与训练配方下，MoonEP 在 NVL72 和跨节点 EP 上能否维持严格均衡；prefetch pool 遇到 pipeline/VPP 并发时怎样隔离；zero-copy 与真实 autograd、activation checkpointing 集成后还剩多少收益；训练时已均衡的模型是否仍能覆盖 planner 成本。这些问题回答之前，MoonEP 更适合被理解为一份清晰、激进、值得复现的系统设计，而非 MoE EP 的最终答案。

- - -

## 参考资料

\[1\] [UCCL-EP：DeepEP-compatible heterogeneous EP implementation](https://github.com/uccl-project/uccl/tree/main/ep)

\[2\] [ROCm/DeepEP](https://github.com/ROCm/DeepEP)

\[3\] [ROCm MoRI：Modular RDMA Interface](https://github.com/ROCm/mori)

\[4\] [DeepEP：V2 and NCCL GIN backend](https://github.com/deepseek-ai/DeepEP)

\[5\] [NCCL EP：Towards a Unified Expert Parallel Communication API for NCCL](https://arxiv.org/abs/2603.13606)

\[6\] [UltraEP：Acknowledgement and design lineage](https://github.com/Dots-Infra/UltraEP#acknowledgement)

\[7\] [MoonEP README and acknowledgments](https://github.com/MoonshotAI/MoonEP/blob/0f385f038fc33bec22e3bcf5a07a8a22693e754c/README.md)

\[8\] [MoonEP top-level API](https://github.com/MoonshotAI/MoonEP/blob/0f385f038fc33bec22e3bcf5a07a8a22693e754c/moonep/api.py)

\[9\] [MoonEP planner encoding bounds](https://github.com/MoonshotAI/MoonEP/blob/0f385f038fc33bec22e3bcf5a07a8a22693e754c/moonep/planning.py#L1267-L1290)

\[10\] [MoonEP cooperative GPU planning kernel](https://github.com/MoonshotAI/MoonEP/blob/0f385f038fc33bec22e3bcf5a07a8a22693e754c/moonep/planning.py)

\[11\] [MoonEP PyTorch planning reference](https://github.com/MoonshotAI/MoonEP/blob/0f385f038fc33bec22e3bcf5a07a8a22693e754c/tests/planning_reference.py)

\[12\] [MoonEP gradient reduction kernel](https://github.com/MoonshotAI/MoonEP/blob/0f385f038fc33bec22e3bcf5a07a8a22693e754c/moonep/grad_reduce.py)

\[13\] [Megatron-Core Technical Report：ECHO](https://arxiv.org/html/2603.07685v2#S4.SS3.SSS7)

\[14\] [UltraEP forward and backward integration](https://github.com/Dots-Infra/UltraEP#2-forward-and-backward)

\[15\] [MoonEP benchmark versus DeepEP V2](https://github.com/MoonshotAI/MoonEP/blob/0f385f038fc33bec22e3bcf5a07a8a22693e754c/benchmarks/bench_vs_deepep.py)

\[16\] [MoonEP supported devices](https://github.com/MoonshotAI/MoonEP#supported-devices)

\[17\] [前期调研：Kimi K3 稀疏 MoE 训练稳定性](https://miraclefarms.github.io/notes/2026/07/20/kimi-k3-sparse-moe-training-stability/)

### 版本对齐信息

| 项目                   | 版本                         | 日期                      | 本文使用范围                                                      |
| -------------------- | -------------------------- | ----------------------- | ----------------------------------------------------------- |
| MoonshotAI/MoonEP    | commit `0f385f038fc3`      | 2026-07-28              | README、API、planner、dispatch/combine、grad reduce 与 benchmark |
| MoonshotAI/MoonEP    | code commit `51e64aa55310` | 2026-07-27              | 全部源码；后续 commit 仅追加 AcclEP 致谢                                |
| UltraEP              | v1.0.0 README / arXiv v3   | 2026-07-15 / 2026-06-18 | 设计借鉴关系与实时复制对照                                               |
| Megatron-Core report | arXiv `2603.07685v2`       | 2026-03-10              | ECHO 机制与训练闭环                                                |
| NCCL EP              | arXiv `2603.13606`         | 2026-03-13              | 原生 NCCL Device API 路线                                       |
