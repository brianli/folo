---
title: "DeepSeek-V4.1-Flash 读后：把 KV cache 压到 890 字节之后，瓶颈换了个位置"
source: "Readings"
category: "AI Infra"
group: "MiracleFarms"
url: "https://miraclefarms.github.io/notes/2026/09/10/deepseek-v41-flash-kv-cache-compression/"
published: 2026-09-10T15:00:00+08:00
saved: 2026-09-27T16:32:37+08:00
folo_key: "mf-readings::https://miraclefarms.github.io/notes/2026/09/10/deepseek-v41-flash-kv-cache-compression/"
tags:
  - "folo"
  - "AI_Infra"
  - "Readings"
---

# DeepSeek-V4.1-Flash 读后：把 KV cache 压到 890 字节之后，瓶颈换了个位置

> [!info] Readings · AI Infra · 2026-09-10 15:00 · [原文](https://miraclefarms.github.io/notes/2026/09/10/deepseek-v41-flash-kv-cache-compression/)

![[DeepSeek-V4.1-Flash 读后：把 KV cache 压到 890-01.jpg]]

一个 token 的全局 KV cache，DeepSeek-V1 要 389,120 字节，V4.1-Flash 要 890 字节[\[1\]](https://huggingface.co/deepseek-ai/DeepSeek-V4.1-Flash/blob/main/DeepSeek_V41_Tech_Report.pdf)。中间隔着 437 倍和不到三年。

这份技术报告用整整一节讲清了为什么继续压：稀疏注意力已经把长序列的算力成本降下来了，于是排在后面的约束浮了上来——全局 KV 要常驻 HBM，持久化 KV 要占 SSD 和 host memory，跨机迁移要吃 I/O 和互连带宽。**长任务 agent 的绑定约束已经从算力挪到了存储容量和数据搬运，V4.1-Flash 把架构预算几乎全押在这一侧。** 这个判断决定了报告里几乎每一个设计选择，也决定了它们的代价落在哪里。

![[DeepSeek-V4.1-Flash 读后：把 KV cache 压到 890-02.png]]

*图 1：这张图把 KV cache 压缩画成了一条产品线而非单点优化——V1→V3.2 的 8.1 倍来自 MLA，V3.2→V4-Flash 的 13.7 倍来自压缩加稀疏，V4-Flash→V4.1-Flash 的 3.9 倍则来自本文要拆的跨层复用与 FP4。来源：DeepSeek-V4.1-Flash 技术报告 Figure 1(b)。*

## 一、这是一篇成本报告

先看清楚这个模型的形状，后面的设计动机才立得住。

V4.1-Flash 有 552B backbone 参数外加 196B Engram 参数，40 层拆成 20 层因果编码器加 20 层解码器，prefill 时每 token 激活 8B、decode 时激活 16B，原生吃图像和文本，上下文 1M[\[2\]](https://huggingface.co/deepseek-ai/DeepSeek-V4.1-Flash)。和它的前一代 V4-Flash（284B 总参、13B 激活）比，参数规模几乎翻倍，同序列长度下的运行时 KV cache 却只有约 1/4，持久化 KV cache 只有约 1/8。

这组数字的排列方式本身就是主张：**参数可以变大，KV cache 必须变小。** 参数是一次性成本，按卡摊；KV cache 是按请求、按 token、按会话生命周期计费的持续成本，agent 跑得越久，它越贵。前期调研文章《DeepSeek V4 的注意力架构抉择》[\[3\]](https://miraclefarms.github.io/notes/2026/05/28/deepseek-v4-hybrid-attention/) 拆过上一代的 CSA/HCA 混合编排如何在 1M context 下把 KV cache 压到标准 GQA 的个位数百分比；这一代要回答的是压完之后剩下的那部分怎么办。

DeepSeek 自己把这条线索交代得很直白：V4[\[9\]](https://arxiv.org/abs/2606.19348) 的全局分支维护 main KV 和 indexer K，局部分支是滑动窗口注意力（SWA），窗口固定则 SWA 的存储与序列长度无关，所以序列一长，主导运行时占用的是全局 KV。这就把优化目标锁死了——**简化全局分支，基本不动局部设计**。三个动作依次落在架构、精度和部署三层，下面逐个看。

## 二、CED：prefill 只跑半个网络

agentic 负载的形状和 chat 不一样：工具调用密集，每一轮都带着增长的上下文回来，cache 一旦没命中，prefill 就要从头算。报告直接把这个当成主要痛点。

因果编码器-解码器（CED）的做法借自 YoCo[\[4\]](https://arxiv.org/abs/2405.05254)：把网络下半部分当作编码器；上半部分（解码器）的全局 KV 用层相关的投影权重直接从第 L/2 层的隐藏状态投影出来，各层自己的隐藏状态不再参与这一步。于是 prefill 阶段只需要跑完前 20 层，后 20 层的全局 KV 顺带就有了，序列长度 N 远大于窗口时，prefill 复杂度从 O(NL) 降到约 O(NL/2)。

![[DeepSeek-V4.1-Flash 读后：把 KV cache 压到 890-03.png]]

*图 2：编码器侧前两层是纯 SWA，其余 18 层按「1 层 Full + 5 层 Reuse」分三组；解码器侧 20 层分五组，第一组由 Full Mode 层建立候选池，后四组各由一层 Reindex 重新选择。图右的 Candidate Pool 是分层稀疏索引器的共享搜索域。来源：技术报告 Figure 3。*

代价藏在 SWA 上。CED 保留了逐层计算局部 KV 的设计——每层的局部 K/V 仍来自本层隐藏状态，这样局部表示的计算深度不打折。但代价是解码器的 SWA KV 没法白拿：严格重建它需要把最近 n\_win × L/2 个 token 再过一遍解码器各层。多轮对话里每轮 prompt 都很短、缓存前缀很长时，这笔开销并不小。

报告在这里引了 PowerAttention 的结论[\[5\]](https://arxiv.org/abs/2503.03588)——SWA 的实际有效感受野远小于理论上的 n\_win × L/2，然后据此只重放最后 n\_win 个 token。这是全文第一次出现”用近似换存储”的模式，第四节会看到它被推到更远的地方。**CED 的净效果是把 prefill 计算砍掉近一半，而 prefill 正是 agentic 场景里 cache 未命中时最贵的那一段。**

## 三、CSA2：把复用拆成三种模式

CSA2 的立论方式比它的机制更值得先读一遍。报告把 KV 成本拆成三个相乘的维度：条目大小（GQA 减头、MLA 共享小 latent）、序列维度（每 m 个 token 压成一条）、层维度（某些层复用别的层的缓存和选择）。然后它逐一点名了层维度上的前人工作——IndexCache 跨层复用 Top-K 索引、YOIO 全网共享一次稀疏路由、HySparse 让稀疏层复用密集层的 KV——并指出这些方法各自只覆盖了一个维度[\[6\]](https://arxiv.org/abs/2405.12981)。索引复用省不了 main KV 存储，全网共享路由会伤性能，混合设计还留着全注意力层。

CSA2 的答案是把缓存共享和索引复用解耦，用三种静态分配的模式覆盖不同组合。

![[DeepSeek-V4.1-Flash 读后：把 KV cache 压到 890-04.png]]

*图 3：绿色是本层现算，黄色是复用自最近一个 Full Mode 层的 main KV 和 indexer K，红色是复用自最近一个产索引层的 Top-K 索引。三种模式的共同点在最左侧——main Q 和 SWA KV 永远由本层自己算，这是”复用不塌陷”的关键。来源：技术报告 Figure 4。*

Full Mode 自己算 main KV、从中投影出 indexer K、跑索引器产出新的 Top-K；Reindex Mode 复用前面层的 main KV 和 indexer K，但用自己的 indexer Q 重新打分、选出不同的 Top-K；Reuse Mode 连 Top-K 一起复用，直接做稀疏注意力，索引器完全不跑。三种模式里每层都保留自己的 main Q 和 SWA KV——**共享的是”看什么”，不共享”怎么看”，这是复用不塌陷成全网单一路由的分界线。**

按 Figure 3 的编排，编码器 18 层 CSA2 里只有 3 层是 Full，解码器 20 层里 1 层 Full、4 层 Reindex，其余 15 层全是 Reuse。也就是说四十层网络里真正维护独立全局 KV 的只有 4 层。

解码器还多一层压缩：分层稀疏索引器。跨层复用减少了索引器的调用次数，但剩下的索引器仍要给整个可见上下文打分，1M context 下这笔开销自己就是瓶颈。做法是让解码器第一个 Full Mode 层在打分之后额外做一次块级选择——每块取块内最高分，选出最多 2,048 块 × 8 个位置共 16,384 个候选位置，后续 Reindex 层只在这个池子里选自己的 Top-512。**候选池大小固定，深层索引器的每 query 开销就从随上下文线性增长变成常数。** 报告特意说明这个机制是训练感知的：候选限制在后训练阶段就以同样方式施加，深层索引器是在它推理时面对的同一个搜索域里被优化出来的。

精度层是最后一刀。V4 已经对 indexer 的 Q/K 做 FP4 量化感知训练，V4.1 把 QAT 扩到 main KV。这里 FP4 只负责省存储，不承担加速矩阵乘的任务，所以他们敢选更准的格式——E2M1 配每 16 通道一个 E4M3 scale，照抄 NVFP4[\[7\]](https://developer.nvidia.com/blog/introducing-nvfp4-for-efficient-and-accurate-low-precision-inference/) 但去掉第二级全局 scale。理由算得很实：RMSNorm 权重最大约 1，512 通道 KV latent 归一化后 L2 范数上界约 22.6，训练中实测最大幅值约 10，而该格式支持到 2688，动态范围绰绰有余。SWA KV 因为对量化敏感，仍留在 FP8。

![[DeepSeek-V4.1-Flash 读后：把 KV cache 压到 890-05.png]]

*图 4：把 BF16/FP8/FP4 按 1、0.5、0.25 加权后，上下文扩大 256 倍，V4.1-Flash 的单 token decode FLOPs 只涨约 1/4——曲线几乎是平的，而 V4-Flash 在 256K 之后已经开始明显上翘。这张图解释了为什么下一节讨论的是存储而不是算力。来源：技术报告 Figure 2。*

## 四、SWA Bounded Replay：用一次有界重算换掉一层持久化存储

前三节都是架构，这一节是部署，也是全篇最有意思的一处判断。

V4 的持久化 KV cache 里，SWA KV 占了将近一半。它的访问模式和这层存储的设计假设完全不匹配：全局 KV 是长尾复用，配一个够大的 SSD 池能让它驻留 72 小时以上；SWA KV 只在活跃会话内几分钟的窗口里被用到，会话一结束或下一轮一开始就是死数据。V4 报告里提过 Zero SWA Caching——干脆不存、缺了就重算——但精确恢复要在 L × n\_win 个 token 上跑完整前向，生产里被判定为成本过高。

V4.1 的做法是放弃精确性：只重放最近 n\_win 个 token，并把 SWA 截断到这段重放区间内。对起始于位置 s 的重放，位置 i 的 query 只注意 \[max(s, i-W+1), i\] 范围内的 SWA key。重建出来的状态在数学上和完整前向不等价，报告的说法是对回答质量影响可以忽略，并且为保险起见在后训练阶段模拟了同样的重放做训练感知适配。

于是持久化层的账变了：SWA KV 从 SSD 挪到各机 10% host DRAM 组成的分布式内存池，容量小得多但 TTL 只有几分钟、周转极快；全局 KV 继续留在持久化缓存里保证 72 小时寿命；偶发的”命中全局 KV 但漏了 SWA KV”由 Encoder SWA Bounded Replay 兜底，重算 n\_win 个 token 而不是 L × n\_win 个。报告自己给这个设计下的判断是——它把灾难性的 miss 变成了一次廉价的优雅降级，正因如此才敢把 SWA KV 从持久化层整个拿掉。两项相乘（不存 SWA 约省一半，全局 KV 再压到 1/4），持久化占用降到 V4 的约 1/8。

这里有一处和前期调研对得上的张力。前期调研文章《AgentX 与 agentic 推理》[\[8\]](https://miraclefarms.github.io/notes/2026/09/09/agentx-agentic-inference-cuda-moat/) 记录过一个现象：混合注意力模型的前缀缓存失效主要发生在命中之后的滑动窗口门而非匹配阶段——前缀找到了，却因为缺少对应的窗口尾巴被丢弃，因此保留策略应当针对”窗口尾巴”。vLLM 和 ROCm/ATOM 走的是保留这条路，用选择性检查点把窗口尾巴留住，命中率从个位数拉到 95% 以上。DeepSeek 在同一个问题上选了重算，而且接受重算结果和原状态不等价。

两条路径面对的是同一个约束，代价结构不同：保留派花的是存储和缓存管理复杂度，重算派花的是每次 prefill 的固定开销加一点点精度。**重算派能成立的前提是模型自己在后训练里被喂过这种近似状态——这是一个只有模型厂商能做、推理框架做不了的选择。** 引擎侧的优化必须保持数学等价，模型侧不必。这条分界线大概会继续把两边的解法推向不同方向。

## 五、代价落在哪：三处退步和一条自陈的边界

压缩不会白来。报告给的对照表里，代价的分布挺清楚。

先看收益。后训练之后，V4.1-Flash 在 agentic 侧的成绩确实压过了同代：Terminal-Bench 2.1 拿到 90.6%，高于 Opus-5 的 89.1% 和 GLM-5.3 的 88.2%；DeepSWE v1.1 74.2%，比前代 V4-Flash 的 54.4% 高出近 20 个点，也略高于 Opus-5 的 74.0% 和 GPT-5.6 Sol 的 73.0%；Codeforces rating 3471，越过了参数量三倍于它的 V4-Pro（3348）；AutomationBench 54.8%、Agents’ Last Exam 31.8% 都是表内最好。

退步集中在三处，而且报告没有藏。

第一处在基座模型的长上下文分数：LongBench-V2 只有 45.2，低于 V4-Pro 的 51.5，仅比 V4-Flash 的 44.7 高半个点。一个把全部工程预算押在长上下文效率上的模型，在长上下文理解基准上没跑赢上一代的 Pro，这个对照本身就说明”能便宜地放下 1M token”和”能用好 1M token”是两件事。同表里 MGSM 从 V4-Flash 的 85.7 掉到 80.2，MATH 61.1 也低于 V4-Pro 的 64.5。

第二处在难度上限。Terminal-Bench 3.0 拿 30.0%、4.0 拿 31.2%，而 Opus-5 分别是 43.3% 和 51.8%；ProgramBench 20.3% 对 Opus-5 的 37.0%；HLE 36.8%（纯文本子集 39.1%）甚至略低于前代 V4-Flash 的 37.8%，离 Opus-5 的 56.3% 更远。报告自己的措辞是：在需要专家级领域知识的科学向 agent 任务上，与巨型模型的差距仍在，而且基准分数接近不等于前沿能力对齐。

第三处是自陈的鲁棒性边界，写在结论里：CSA2 可能的选择错误和 SWA Bounded Replay 的近似状态重建，都可能在未测到的边界情况下造成能力退化，重点关注方向是长上下文上的稀疏检索，以及缓存恢复边界处的 SWA 状态重建。**把这两个风险点写成待验证项，等于承认第三节和第四节那两次”用近似换存储”的收益是有条件的，条件是分布内。**

补偿这些代价的一个工程手段是可控推理强度。RL 阶段把 1–100 的标量 effort 作为条件信号，长度惩罚系数随 effort 指数衰减，部署时同一份权重就能在成本-质量曲线上移动；公开 API[\[10\]](https://api-docs.deepseek.com/news/news260424/) 的 max / high / low 三档分别对应 100 / 75 / 50。effort 从 25 升到 100，八个推理基准均分从 67.1% 到 76.3%，DeepSWE 从 66.0% 到 74.2%，Terminal-Bench 2.1 从 82.4% 到 90.6%，代价是约 2.5 倍输出 token。**收益是前置的：60–80 档已经拿回大部分精度而只花不到一半 token 预算，从 80 到 100 会把 agent 轨迹拉长 1.6–1.8 倍换取边际改善。** 这条曲线的形状比它的端点更有部署价值。

## 六、开源框架支持现状与选型建议

模型发布当天，能不能跑、跑到什么程度，决定了报告里那些数字对你有没有意义。截至 2026-09-10，只有两个开源推理栈动了：vLLM 和 SGLang，两边的 day-0 PR 在同一天 UTC 06:00 前后相隔不到一分钟开出来，**并且都还没合并**。

| 栈                                      | 入口                                                                                                                                 | 规模                                                                              | 覆盖到哪一层                                                                                                                                                                                                                                                                                              | 状态（2026-09-10 查询）                                 |
| -------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------- |
| 官方参考实现                                 | HF repo `inference/`[\[14\]](https://huggingface.co/deepseek-ai/DeepSeek-V4.1-Flash/tree/main/inference)                           | `model.py` / `kernel.py` / `engram.py` / `vision.py` / `generate.py` + `run.sh` | 完整前向与多模态，含 Engram 与自定义 kernel                                                                                                                                                                                                                                                                       | 随权重同时发布，定位是正确性基准                                  |
| vLLM                                   | PR #56201[\[11\]](https://github.com/vllm-project/vllm/pull/56201)、#56208[\[12\]](https://github.com/vllm-project/vllm/pull/56208) | 147 文件，+20,032 / −542，7 commits                                                 | `models/deepseek_v4_1/` 下 NVIDIA 与 AMD 双路径齐全：`vl_model.py` 多模态、`dspark.py` 推测解码、`compressor.py` / `sparse_mla.py` / `attention.py` 覆盖 CSA2、`common/engram.py` 与三套 mHC kernel（tilelang / triton / torch）、`v1/attention/backends/mla/sparse_swa.py`                                                   | OPEN；有合并冲突待 rebase，CI 5 通过 1 失败 3 未跑，仅 1 条 review |
| SGLang                                 | PR #38798[\[13\]](https://github.com/sgl-project/sglang/pull/38798)                                                                | 150 文件，+15,141 / −907                                                           | `layers/attention/dsv4/dsv41_sparse.py` 加 FP4 indexer / topk / sparse prefill kernel 覆盖 CSA2；`layers/engram.py` 配三个 Engram kernel；`layernorm/mhc.py`；`mem_cache/` 下 `dsv41_request_window.py`、`deepseek_v4_memory_pool.py`、`hicache_storage.py`；`model_executor/encoder_swa_replay.py`（唯一实现有界重放的一处） | OPEN；`run-ci` 标签未加，CI 尚未运行                        |
| TensorRT-LLM / llama.cpp / vLLM Ascend | —                                                                                                                                  | —                                                                               | 未见 V4.1 相关 PR                                                                                                                                                                                                                                                                                       | 无                                                 |

两份 PR 的差异落在一个具体文件上。**SGLang 的 `encoder_swa_replay.py` 是目前唯一把第四节那套 Encoder SWA Bounded Replay 实现出来的开源代码**；vLLM 这 147 个文件里没有对应模块，`sparse_swa.py` 是稀疏加滑动窗口的注意力后端，解决的是另一件事。这个差异直接对应报告里两条不同的收益：运行时 KV 降到 1/4 靠的是 CSA2 加 FP4，两边都有；持久化 KV 降到 1/8 靠的是不存 SWA KV 加有界重放，眼下只有 SGLang 侧铺了路。

据此给三条选型判断。

**要复现报告里的成本数字，现在只有 SGLang 一条路。** 持久化占用那个 1/8 是这次发布最直接的省钱项——它决定 SSD 池要开多大、跨机迁移要吃多少带宽。缺了有界重放，SWA KV 要么继续持久化（省不掉那一半），要么每次 miss 走全量重算（把灾难性 miss 留在原地）。愿意等 PR 合并并自己压测的团队，这条线值得盯。

**要多模态、推测解码或 AMD 卡，vLLM 的覆盖面更齐。** NVIDIA 与 AMD 两套 `model.py` / `vl_model.py` / `dspark.py` 都在同一份 PR 里，加上 Rust 前端补齐了 V4.1 的对话模板与 DSML 工具调用解析，端到端服务形态比 SGLang 侧完整。代价是它还有合并冲突和一项 CI 失败，短期内 API 表面大概率还会变。

**这周想验证模型能力而不是服务性能，直接用官方参考实现。** `inference/` 目录带完整前向、Engram 和视觉分支，跑得慢但语义正确，可以作为后面两个栈的对拍基准。两份社区 PR 在 CI 转绿并合并之前都别进生产——**这两个 PR 的 CI 状态，眼下比任何一篇部署教程都更能说明成熟度。**

## 七、结论

回到开篇那个判断。这份报告的三个主要设计——CED 的非对称激活、CSA2 的跨层复用、SWA Bounded Replay 的有界重算——指向同一个成本模型：**在长任务 agent 的负载下，能省的算力已经省得差不多了，接下来的每一分成本降幅都要从存储容量、数据搬运和缓存生命周期里抠。** 报告把参数从 284B 加到 552B 的同时把 KV cache 压到 1/4，这个方向选择本身就是结论。

它成立的边界也清楚。收益建立在 agentic 负载的两个特征上——输入远重于输出、前缀高度可复用；换成输出密集或前缀不可复用的场景，prefill 减半和持久化缓存减压都会大幅贬值。近似的部分（Top-K 选择、重放出来的 SWA 状态）目前只有”内部评测未见系统性退化”这一层保证，报告自己把它列为待测边界。至于后训练，报告明确说没有算法创新，全部增益来自数据和环境流水线的规模化——这句话在多大程度上可复现，取决于别人有没有它那套能跑到百万级并发沙箱的基础设施。

还开放着的问题是那条 LongBench-V2 曲线。当把 1M token 放进 HBM 的成本降到 890 字节每 token 之后，限制长上下文能力的因素就不再是能不能放下了。下一代要证明的东西，从”压得够不够小”换成了”选得够不够准”。

- - -

## 参考资料

\[1\] [DeepSeek-V4.1-Flash: Pushing the Limits of KV Cache Compression（技术报告 PDF）](https://huggingface.co/deepseek-ai/DeepSeek-V4.1-Flash/blob/main/DeepSeek_V41_Tech_Report.pdf)

\[2\] [deepseek-ai/DeepSeek-V4.1-Flash · Hugging Face](https://huggingface.co/deepseek-ai/DeepSeek-V4.1-Flash)

\[3\] [前期调研：DeepSeek V4 的注意力架构抉择——从 2% KV Cache 反推 CSA/HCA 的工程逻辑](https://miraclefarms.github.io/notes/2026/05/28/deepseek-v4-hybrid-attention/)

\[4\] [You Only Cache Once: Decoder-Decoder Architectures for Language Models](https://arxiv.org/abs/2405.05254)

\[5\] [PowerAttention: Exponentially Scaling of Receptive Fields for Effective Sparse Attention](https://arxiv.org/abs/2503.03588)

\[6\] [Reducing Transformer Key-Value Cache Size with Cross-Layer Attention](https://arxiv.org/abs/2405.12981)

\[7\] [Introducing NVFP4 for Efficient and Accurate Low-Precision Inference](https://developer.nvidia.com/blog/introducing-nvfp4-for-efficient-and-accurate-low-precision-inference/)

\[8\] [前期调研：AgentX 与 agentic 推理的 CUDA 护城河](https://miraclefarms.github.io/notes/2026/09/09/agentx-agentic-inference-cuda-moat/)

\[9\] [DeepSeek-V4: Towards Highly Efficient Million-Token Context Intelligence](https://arxiv.org/abs/2606.19348)

\[10\] \[DeepSeek V4 Preview Release

DeepSeek API Docs\](https://api-docs.deepseek.com/news/news260424/)

\[11\] [vLLM PR #56201: \[Model\] Support DeepSeek-V4.1-Flash](https://github.com/vllm-project/vllm/pull/56201)

\[12\] [vLLM PR #56208: \[Model\] Support DeepSeek-V4.1-Flash in Rust and Python frontends](https://github.com/vllm-project/vllm/pull/56208)

\[13\] [SGLang PR #38798: \[Model\] Add DeepSeek V4.1 support](https://github.com/sgl-project/sglang/pull/38798)

\[14\] [DeepSeek-V4.1-Flash 官方参考实现（HF repo inference/）](https://huggingface.co/deepseek-ai/DeepSeek-V4.1-Flash/tree/main/inference)
