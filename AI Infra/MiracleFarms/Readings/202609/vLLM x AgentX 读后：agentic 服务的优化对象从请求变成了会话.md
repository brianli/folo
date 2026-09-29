---
title: "vLLM x AgentX 读后：agentic 服务的优化对象从请求变成了会话"
source: "Readings"
category: "AI Infra"
group: "MiracleFarms"
url: "https://miraclefarms.github.io/notes/2026/09/09/vllm-agentx-agentic-serving-stack/"
published: 2026-09-09T21:30:00+08:00
saved: 2026-09-27T16:26:43+08:00
folo_key: "mf-readings::https://miraclefarms.github.io/notes/2026/09/09/vllm-agentx-agentic-serving-stack/"
tags:
  - "folo"
  - "AI_Infra"
  - "Readings"
---

# vLLM x AgentX 读后：agentic 服务的优化对象从请求变成了会话

> [!info] Readings · AI Infra · 2026-09-09 21:30 · [原文](https://miraclefarms.github.io/notes/2026/09/09/vllm-agentx-agentic-serving-stack/)

![[vLLM x AgentX 读后：agentic 服务的优化对象从请求变成了会话-01.jpg]]

vLLM 在 AgentX 上报出的最好成绩是 DeepSeek V4 Pro 用 12 张 GB300 撑住 256 个并发会话、P90 每用户 58.3 tok/s 时，每 GPU 秒推进 83K token[\[1\]](https://vllm.ai/blog/2026-09-08-vllm-agentx)。把这个成绩拆开看，贡献最大的几项几乎没有一项来自”某个算子更快了”。**排在最前面的是一个调度阈值：把单个请求每步能占用的 token 预算从无上限压到 512，仅此一项就让 B300 上的 DeepSeek V4 Pro 每 GPU 秒总吞吐最高提升 93%、P90 交互度改善约 2.3 倍。** 一个参数的收益盖过大部分 kernel 工作，这件事本身就说明 agentic 负载重排了优化的优先级。

前期调研文章《AgentX 读后》的判断是，agentic 推理的成本大头已经从 attention 的 FLOPs 转移到会话状态的保存、搬运、路由与重建这条路径上[\[3\]](https://miraclefarms.github.io/notes/2026/09/09/agentx-agentic-inference-cuda-moat/)。那篇文章站在基准和厂商差距的角度；vLLM 这篇是同一件事的另一半——引擎侧把这条路径逐段修过来之后，具体改了什么，以及哪些看起来合理的直觉被端到端测量推翻。它按三个平面组织：数据面让 KV 活着且离算力近，执行面按模型架构选并行，控制面配 P/D 速率与请求调度。读完三个平面会发现它们收敛到同一个变量：**一个会话的 KV 前缀能不能持续待在离它下一轮计算最近的地方。**

## 一、先把负载的形状记住

AgentX 从真实 agentic 编码 trace 里统计出四条特征[\[2\]](https://newsletter.semianalysis.com/p/agentx-inferencexv3-does-cuda-moat)：会话中位 43 轮；输入中位 142K token、输出中位 444 token；前缀缓存命中率超过 96%；44% 的会话至少包含一个子代理，这些会话的子代理 rollout 中位数是 4 次。

agentic 流量的份额还在涨：2026 年 6 月 OpenAI 披露，在企业客户里 Codex 贡献了 Codex 与 ChatGPT 合计输出 token 的 64%[\[4\]](https://openai.com/signals/enterprise-data/)。

这些数字来自 agentic 会话的构造方式。每一轮把最新的工具结果追加到累积上下文后，整份重新发给模型，所以输入持续膨胀，而每轮只带来一小段新的 prefill，请求的绝大部分是引擎已经见过的前缀。输入输出比落在 320:1 这个量级，与聊天场景完全不是一回事。

前期沉淀里有两组可以对照的数据：Codex 与 SWE-bench Pro 的 610 条 agentic trace 中位 33 轮、第 30 轮上下文约 80K token；DualPath 采集的 500 条生产编码轨迹平均 157 轮、上下文 32.7K token、每轮 append 仅 429 token、KV 命中率 98.7%。三组数据的轮数从 33 分布到 157，差异不小，但”每轮新增几百 token、命中率 90% 以上”这条形状高度一致——AgentX 的输出中位 444 token 和 DualPath 的每轮 append 429 token 几乎重合。**引擎每一步真正要算的新 token 只有几百个，其余 99% 以上的上下文以 KV 的形式存在于某个地方，工程问题因此从”算得多快”变成”这批 KV 在不在手边”。**

子代理那条特征单独制造了一类压力。分叉出来的子代理要么继承父上下文、要么全新开始，结果再并回父会话。KV 的访问模式因此从一条线变成一棵树，缓存的保留边界不能只按轮次划分——这一点在第五节会变成具体的策略选择。

## 二、三个平面：把优化对象从请求换成会话

![[vLLM x AgentX 读后：agentic 服务的优化对象从请求变成了会话-02.png]]

*图 1：vLLM 把 agentic 服务的优化拆成三个平面——数据面管理分布式共享 KV cache，执行面把每个模型映射到合适的并行方式与 kernel，控制面协调 P/D 比例与请求调度。这张图是全文的骨架，后面每一节都能在图上找到位置。来源：vLLM 官方博客 Figure 3[\[1\]](https://vllm.ai/blog/2026-09-08-vllm-agentx)。*

这个划分本身携带信息。数据面里出现的是”分布式共享 KV cache”，控制面里出现的是”P/D 比例”和”请求调度”——两者都超出单个引擎进程的范围。换句话说，**vLLM 把引擎内部的内存管理和集群层面的状态调度放进了同一张图讨论。**

前期沉淀里 inference-system 域 engine-architecture 节点的立场是”推理系统控制面数据面分层，调度/内存/传输上移集群本体”。这张图是那条判断的又一个实例，而且是第一次由引擎自己以这种方式画出来。三个平面接下来分别展开。

## 三、数据面之一：混合注意力让静态显存分区失效

KV cache 管理从 PagedAttention 起就是 vLLM 的核心，长上下文把容量压力推得更高。现代混合模型让分配问题又复杂一层：滑动窗口注意力、线性注意力与全注意力混在同一个模型里，各自缓存块的大小和生命周期都不一样。

![[vLLM x AgentX 读后：agentic 服务的优化对象从请求变成了会话-03.png]]

*图 2：一个页大小服务所有注意力类型，一个 block pool 在它们之间共享。这张图解释了为什么 vLLM 拒绝按注意力类型静态切分显存——最优切分点会随并发、上下文长度和复用模式移动，静态划分等于把一个动态最优值写死。来源：vLLM 官方博客 Figure 4[\[1\]](https://vllm.ai/blog/2026-09-08-vllm-agentx)。*

全注意力的 KV 随序列长度线性增长，滑动窗口只需要固定窗口，线性注意力的 recurrent state 是固定大小的快照。三种生命周期和缩放规则不同，最优的容量划分随负载移动。共享池让 vLLM 按需动态再分配，而不必在启动时押一个比例。

DeepSeek V4[\[6\]](https://vllm.ai/blog/2026-04-24-deepseek-v4) 提供了一个碎片化的反面案例。它最初的 KV cache 布局把不同缓存类型切成三个尺寸桶、分配 92 个独立张量。碎片化在三处同时收税：padding 浪费显存、P/D 传输要携带更多描述符、offload 的搬运粒度被打散。新的 packed 布局把所有 cache group 和层收进每个 block 一块连续的后备分配[\[7\]](https://github.com/vllm-project/vllm/pull/44577)；开启 FP4 indexer 时还允许更小的分配单元，省下约 10% 的 KV cache 显存[\[8\]](https://github.com/vllm-project/vllm/pull/48993)。

**显存布局在 agentic 负载里同时是三件事的前置条件——容量、P/D 传输效率、卸载粒度。** 单机时代它只影响第一件；一旦 KV 要跨节点搬、要落 CPU 和磁盘，张量的排布方式就直接决定搬运的元数据开销。

## 四、数据面之二：把前缀缓存做成分层的分布式池

为了让前缀缓存活过 GPU 显存容量、并跨引擎实例存在，vLLM 集成了 Mooncake Store[\[9\]](https://github.com/kvcache-ai/Mooncake) 作为分布式 KV cache 池，设计在此前的博客里讲过[\[5\]](https://vllm.ai/blog/2026-05-06-mooncake-store)。这次的更新集中在容量、效率和保留策略三个方向。

分层是容量方向的主要改动。vLLM 的 Mooncake Store `standalone-store` 模式让一个外部 Mooncake client 拥有 CPU 池和磁盘层，vLLM worker 退化成纯粹的请求方[\[10\]](https://docs.vllm.ai/en/latest/features/mooncake_store_connector_usage/#configure-mooncake)。每个节点起一个独立的 Mooncake client，就能用 CPU 内存和磁盘自由扩池，甚至加入只有 CPU 的节点。这套分布式共享池还和 Dynamo[\[11\]](https://github.com/ai-dynamo/dynamo)、llm-d[\[12\]](https://github.com/llm-d/llm-d) 这类 router 打通了——请求在任意实例都能拿到命中，路由策略因此被简化。

前期沉淀记录过这条路径的起点：vLLM × Mooncake Store 把 agentic 场景的跨实例命中率从 1.7% 提到 92.2%、P50 TTFT 降低 46 倍。这个数字和 AgentX 报告的 96% 前缀命中率口径不同——后者是负载自身可复用比例的上界，前者是系统实际拿到的比例。两个数字接近，说明这条路径已经把可复用的部分吃得差不多了。

效率方向的开销落在一个容易被忽略的地方：CPU。混合模型必须为每种注意力类型分别构造 key 并分别查找，查表次数随类型数量翻倍，而这些工作原本坐在调度器的关键路径上。四个 PR 把这块成本压下去——更高效的数据结构、异步查找、把工作移出调度器关键路径、收发操作并行化[\[13\]](https://github.com/vllm-project/vllm/pull/46188)[\[14\]](https://github.com/vllm-project/vllm/pull/45444)[\[15\]](https://github.com/vllm-project/vllm/pull/45659)[\[16\]](https://github.com/vllm-project/vllm/pull/47317)。**在 agentic 负载上，调度器每一轮都要重查一次十几万 token 的前缀，CPU 侧的查表开销会被轮次数量放大成结构性瓶颈。**

## 五、数据面之三：混合注意力把问题从”存哪些 token”换成”在哪个边界存”

混合模型的前缀复用有一个额外条件：复用边界上必须同时持有对应的线性状态或滑动窗口缓存。全注意力的 KV 可以按 token 切片复用，recurrent state 不行——它是一份不可切片的快照，只能整段命中。每个 token 都存快照太贵，vLLM 用两条互补策略解决。

第一条是间隔式保留：每轮自动保留 prompt 末尾的缓存与线性状态[\[17\]](https://github.com/vllm-project/vllm/pull/43447)。后续轮次以及从该上下文 fork 出的子代理，通常是重放并延长某一轮的上下文，正好落在这些边界上。

第二条针对间隔式抓不到的情况。共享前缀常常在一轮内部就结束，轮末边界上没有对应的 checkpoint。Marconi 式选择性保留在一个前缀被第二次观察到时才留 checkpoint[\[18\]](https://github.com/vllm-project/vllm/pull/47782)：请求碰到一个此前见过但没有 checkpoint 的前缀时，vLLM 重算缺失状态并在该边界存下，之后共享这个前缀的请求就能复用。用”出现两次”作为值得保存的信号，避免为一次性前缀付存储成本；技术细节在 vLLM 的 Kimi K3 博客里展开[\[19\]](https://vllm.ai/blog/2026-07-27-k3)。

这条线在前期沉淀里有连续的记录：vLLM 曾在 #37898 引入 Marconi 式准入策略，理由正是 recurrent state 快照不可切片、只能整段命中；Kimi K3 则必须把存储粒度（1024–6144 token 的物理块）与匹配粒度（512 token 的 prefix hash）解耦，一次命中要在同一边界同时拥有 MLA 哈希和每个 KDA state group 的 checkpoint。**混合注意力把前缀缓存的设计变量从”存哪些 token”换成了”在哪些边界存整份状态”，而两条保留策略分别对应轮边界与轮内共享边界这两类信号。**

## 六、执行面之一：MLA 让 TP 失去意义，DCP 补位

现代推理系统摊开的并行轴有张量并行（TP）、数据并行（DP）、专家并行（EP）、流水并行（PP）和上下文并行（CP）。哪一种最优取决于模型架构、硬件拓扑、负载形状和延迟 SLO，vLLM 用 Kimi K3 和 DeepSeek V4 两个模型说明这句话具体意味着什么。

Kimi K3 用的是多头潜在注意力（MLA）加 Kimi Delta Attention（KDA）。**MLA 把 KV 压进单个潜在空间、只有一个头，普通张量并行在它身上退化成纯复制——显存没省，带宽照吃。**

解码上下文并行（DCP）沿序列维切分缓存，每个 rank 只持有 1/N 的 KV 状态[\[20\]](https://vllm.ai/blog/2026-08-07-decode-context-parallelism)。它在 agentic 负载上有两个收益：MLA attention 访存受限、代价随上下文增长，agentic 前缀越长，attention 在每个 decode step 里占的比重越大，切开就直接缩短这一步；不复制 KV 也让引擎能同时挂住更多序列而不卡在 KV 准入上，吞吐随之抬高。

![[vLLM x AgentX 读后：agentic 服务的优化对象从请求变成了会话-04.png]]

*图 3：横轴是每 rank 的序列数（并发），纵轴是 P50 TPOT。DCP8 的曲线整体在 TP8 之下，并且能延伸到更高的并发点——前者说明单步更快，后者说明不复制 KV 换来了更大的准入空间。来源：vLLM 官方博客 Figure 6[\[1\]](https://vllm.ai/blog/2026-09-08-vllm-agentx)。*

DCP 的代价是通信。KV 按序列切开之后，每个 MLA decode 层都需要一次 query gather 和一次 partial output 归约。vLLM 绕开了 NCCL：用对等 GPU 可以直接读写的对称内存缓冲区，query 被 multicast 直接送进 attention kernel 消费的缓冲区，每张 GPU 把自己的部分注意力输出和 log-sum-exp 统计直接写进对端的接收槽，各 rank 本地用 online softmax 合并结果。这些 GPU 到 GPU 的写入被融合进同一个 kernel，相对默认 DCP8 实现每层延迟降低约 13%。原本的 all-gather、staging copy、all-to-all、unpack 四段式路径，被压成几次融合进计算的远程写。

## 七、执行面之二：同一个 DCP，换个 scale-up domain 结论就翻转

![[vLLM x AgentX 读后：agentic 服务的优化对象从请求变成了会话-05.png]]

*图 4：每 rank batch size 超过 3 之后，宽 EP（DEP16）的曲线走到 DCP8 之下。交叉点的存在意味着”K3 应该用 DCP”这句话在没有指定并发和 scale-up 规模时并不成立。来源：vLLM 官方博客 Figure 8[\[1\]](https://vllm.ai/blog/2026-09-08-vllm-agentx)。*

在 NVL72 这一级的 scale-up domain 上，宽 EP 配合数据并行（DEP）比 DCP 扩得更好，在相同的解码延迟 SLO 下给出更高吞吐。原因是 DCP 规模变大、跨到多节点之后，切分注意力的通信代价超过它省下的计算；DEP 把请求连同它的 KV 分配给不同的数据并行 rank，注意力路径完全本地，只把 MoE 专家切开。

前期沉淀里 inference-parallelism 节点记录过一条相关判断：在 GB300 NVL72 上 Kimi K3 的 8 GPU replica 可以跨两个 4 GPU 节点组合 TP 与 EP，说明并行边界应按高带宽 NVLink domain 划分而非机箱划分。这次的数据给那条判断补了一个方向相反的用法：**domain 变大不只是放宽了并行度的上限，它会改变哪一种并行更优——同一个模型、同一个引擎，scale-up domain 从 8 卡换到 72 卡，最优策略换了一种。**

## 八、执行面之三：DeepSeek V4 为什么 TP 和 DCP 都不合适

DeepSeek V4 同样有 MLA 式的 KV cache，在 TP 下复制、内存使用低效。除此之外，它的压缩稀疏注意力让 TP 的按头切分在计算上也不划算，原文给了三条并列的理由。

压缩器路径对每个压缩位置只产生一份共享的 KV 表示，而没有独立的按头状态，TP 因此无法沿 KV 头维切分计算，每个 rank 都要重复整份压缩器工作。indexer 虽然有 64 个头，但每个 token 只产出一次全局 top-k 选择，当前的 TP 路径把整个 indexer 复制到每个 rank——这样避开了 top-k 之前的稠密分数归约，代价是重复计算。稀疏 MLA 的主要开销落在扫描和收集 top-k 的 KV 条目上，属于访存受限，TP 在每个 rank 上重复这部分工作，只把更便宜的按头计算分掉了。

三条合起来是同一个意思：**TP 的切分维度是”头”，而这个注意力栈的工作量分布在共享表示、全局 top-k 和访存扫描上，两者对不上。**

预填充上下文并行（PCP）切的是 prompt 序列（query 张量），把压缩器和 indexer 的工作分到各 rank，同时给稀疏 MLA 一个更宽、head-local 的形状。32K prompt 上 PCP8 相对 TP8 有 2.65× 的 prefill 加速，TTFT 明显下降。但它仍然在各 rank 复制解码侧状态，所以只适合专用的 prefill worker——这一条正好解释了为什么它出现在 P/D 分离的语境里而不是聚合部署里。

DEP 把请求和解码 token 分散到各数据并行 rank，注意力路径完全本地，因此成为 DeepSeek V4 多数配置的默认。至于 DCP 为什么在 V4 上不灵，留到第十二节的 bitter lesson 里说。

## 九、控制面之一：给单请求设配额，给 prefill 定节拍

agentic 服务把两类请求混在一起：频繁的追加型请求复用长前缀、只需要很短的 prefill；偶发的全新长 prefill 跨几万 token。这带来两个调度问题——实例内部，一个长 prefill 会挡住短的交互轮次；跨 DEP rank，prefill 落点不均会制造负载失衡。

![[vLLM x AgentX 读后：agentic 服务的优化对象从请求变成了会话-06.gif]]

*图 5：单个 rank 队列的会话视图。左侧没有 chunk 上限，一个长 prefill 吃掉整个预算，已经缓存好的短轮次只能等；右侧设了 512 token 上限，短轮次每一步都能挤进同一批并更早开始解码。这张图解释了为什么一个阈值参数能带来 93% 的吞吐提升。来源：vLLM 官方博客 Figure 9[\[1\]](https://vllm.ai/blog/2026-09-08-vllm-agentx)。*

vLLM 的分块预填充调度器默认按先进先出跑，一个长 prefill 可以一步接一步占满整个 token 预算，同一 rank 上排队的短轮次在它跑完之前完全排不进去。`--long-prefill-token-threshold` 给单个请求每步能调度的 token 数设上限，512 的阈值下长 prefill 会留出空间。B300 上的 DeepSeek V4 Pro 因此把每 GPU 秒总吞吐最高提升 93%、P90 交互度改善约 2.3 倍。代价是那个长请求自己的 TTFT 变高，对 TTFT 敏感的部署应当把阈值调大。

前期沉淀里 request-scheduling 节点有一条判断说得更早：KV cache 把请求大小从一维时间撑成时间×内存的二维约束，调度的核心控制变量因此从”先跑哪个请求”变成”token 预算如何切分给 prefill 与 decode”。**这个阈值是那条判断在 agentic 负载上最直接的兑现——收益来自给单个请求的预算占用设一个上限，而不来自更聪明的排序函数。**

第二级控制针对 DEP。MoE 的 all-to-all 通信强制各 rank 锁步推进，一个 rank 在做 prefill 就拖慢整组；prefill 分散落在不同 step 上时，这笔罚金要反复付。`--prefill-schedule-interval` 只在每第 N 个引擎步准入 prefill 工作，计数器跨数据并行 rank 对齐，把 prefill 集中到同一批 step 上，其余 step 变成纯解码。两个参数在做同一件事：把”谁能占用这一步”从自由竞争改成有配额、有节拍。

## 十、控制面之二：P/D 比例是量出来的

优化单个引擎不足以找到分布式部署的最佳延迟-成本点，堆更多 GPU 或者上分离也不会自动改善前沿——prefill 和 decode 两级必须速率匹配。vLLM 用的是一套两阶段标准流程，并且明说它可以由 agentic workflow 自动执行。

第一阶段是饱和剖析：分别跑纯 prefill 和纯 decode 部署，扫并行策略（例如 TP 与宽 EP）和部署规模（8、16、32 卡），并发一路加到吞吐饱和，产出一张饱和表——每个（并行策略，规模）组合的最大 prefill/decode 请求速率。第二阶段是 P/D 扫描：从每个配置的饱和点推出 P/D 比例，再在合起来的分离式部署上扫并发，采集整个工作区间的指标。

前期沉淀里 pd-disaggregation 节点记录过失配的后果：P/D worker 比例失配会导致一侧空转、另一侧排队，GPU 利用率和用户延迟同时变差。同一个域里还挂着一条尚未解决的活跃矛盾——AIConfigurator 这类性能建模工具的宣称边界，论文数字与源码里的经验因子对不上。vLLM 这套方法绕开了那个争议：它不建模，直接测两端的饱和点。**代价是每个（模型，硬件，并行策略，规模）组合都要真跑一遍；在配置空间已经大到无法穷举的时候，两阶段测量把搜索压成两步，比拟合一个跨模型的性能模型更容易保证结论不出错。**

## 十一、闭环：模型专属 kernel 与社区贡献

agentic 负载也把 kernel 的瓶颈挪了位置——长上下文注意力、推测解码和通信取代通用矩阵乘成为主战场。原文挑了几项有端到端实测的改动，并且强调这些 kernel 全部开源，其中一部分已经被其他开源引擎采用。

MiniMax M3 这边，一个用 CuteDSL 写的长上下文 indexer 把 GB300 上报告的 indexer 延迟改善约 3% 到 31%，幅度随形状变化[\[21\]](https://github.com/vllm-project/vllm/pull/48582)；上游化的 MSA top-k 路径把最坏情况下的 kernel 性能提升最多 4 倍，AgentX 端到端吞吐提升约 7%；推测验证路径在中等 batch 的解码上带来约 20% 的改善。

Kimi K3 这边，GEMM 与 reduce-scatter 的融合改善了序列并行的通信[\[22\]](https://github.com/vllm-project/vllm/pull/52079)，latent-tail MoE 融合把端到端延迟降低约 5%[\[23\]](https://github.com/vllm-project/vllm/pull/53152)。DeepSeek V4 这边主要来自社区贡献：MXFP4 MoE 与 HCA 压缩的改进[\[24\]](https://github.com/vllm-project/vllm/pull/43584)[\[25\]](https://github.com/vllm-project/vllm/pull/44230)、多流 C4A[\[26\]](https://github.com/vllm-project/vllm/pull/42925)，以及基于簇的 top-k[\[27\]](https://github.com/vllm-project/vllm/pull/43008)。

把这一节的数字排在一起会看到一个梯度：**单个 kernel 的改善动辄 4 倍或 31%，落到端到端只剩 7% 与 5%，而第九节那个调度阈值的端到端收益是 93%。** kernel 工作依然重要——它是长期复利，那个 4 倍的最坏情况修复正是 P99 不再爆炸的原因。梯度说明的是另一件事：在 agentic 负载上，单请求路径上的算子优化已经不是最容易拿到的收益，每个会话在系统里怎么排队、怎么落位才是。

## 十二、成绩单怎么读，以及口径在哪里

![[vLLM x AgentX 读后：agentic 服务的优化对象从请求变成了会话-07.png]]

*图 6：横轴是 P90 交互度（每用户 tok/s），纵轴是每 1 美元 TCO 的总 token 数，三个模型各取最好的 vLLM 配置。任何”A 比 B 好”的说法都只在这条曲线的某一段成立，读结论前先确定自己的延迟 SLO 落在横轴哪个位置。来源：vLLM 官方博客 Figure 1，数据来自 SemiAnalysis AgentX[\[1\]](https://vllm.ai/blog/2026-09-08-vllm-agentx)。*

在 P90 交互度维持 50 tok/s 以上这个常见且苛刻的 SLO 下，三个开源前沿模型的最高吞吐配置分别是：DeepSeek V4 Pro 1.6T 用 12 张 GB300 服务 256 并发，83K TPGS，P90 58.3 tok/s；MiniMax M3 428B 只用 2 张 B300 服务 24 并发，70K TPGS，P90 74.2 tok/s；Kimi K3 2.8T 用 16 张 GB300 服务 48 并发，11.8K TPGS，P90 62.7 tok/s[\[28\]](https://inferencex.semianalysis.com/inference/agentic/439873)[\[29\]](https://inferencex.semianalysis.com/inference/agentic/439907)[\[30\]](https://inferencex.semianalysis.com/inference/agentic/441066)。

TPGS 的口径必须先说清楚：它统计输入、输出和缓存 token 的总和。在 96% 命中率的负载上，这个口径里绝大部分 token 是缓存命中的，实际计算量远小于数字本身。它衡量的是”系统每 GPU 秒推进了多少 token 的会话进度”，与”每 GPU 秒算了多少 token”不是同一个量。Kimi K3 的 11.8K 与 DeepSeek V4 Pro 的 83K 之间那 7 倍差距，主要反映 2.8T 与 1.6T 的参数量差别以及 16 卡与 12 卡的部署规模，不宜直接读成引擎效率差。

成本对比的口径同样需要交代。三个模型对 Opus 5 API 报价的成本优势分别是 106×、85× 和 14.6×：DeepSeek V4 Pro 的 GPU TCO 是每小时 27.72 美元，处理同样的 token 量按 Opus 5 计价约 2926 美元；MiniMax M3 是 4.52 对 384 美元；Kimi K3 是 36.96 对 538 美元。Opus 5 的等价成本按缓存输入 0.50 美元/M、未缓存输入 5 美元/M、输出 25 美元/M 计算，假设理论完美命中率，且不计 cache-write 费用和长上下文溢价——原文明确说这个假设对 Opus 有利。**GPU TCO 与 API 报价属于不同类别的量，前者是自建成本，后者含毛利、可用性承诺、SLA 和模型质量，所以这组倍数说明的是自建开源模型服务与购买闭源 API 之间的价差量级，不能当作模型能力的比较。**

K3 的 14.6× 比另外两个低一个量级，原因也在表里：2.8T 参数需要 16 张 GB300，硬件 TCO 每小时 36.96 美元是三者最高，而它的 token 产出并没有成比例更高。整套结果跑在 SemiAnalysis 独立运行的公开基础设施上，harness 开源、每条结果都链回可复现的 run[\[32\]](https://github.com/SemiAnalysisAI/agentx-harness)[\[31\]](https://inferencex.semianalysis.com/inference?i_seq=agentic-traces&i_xmode=interactivity&g_runid=33418433573&i_best=0&i_active=b200_vllm%2Cb300_vllm%2Cgb200_dynamo-vllm%2Cgb300_dynamo-vllm&i_hc=1&i_advlabel=0&i_label=0)。

## 十三、三条被端到端测量推翻的直觉

原文单列了一节讲失败的尝试，这一节的信息密度高于成功案例——每条失败都划掉了一片搜索空间。

### 13.1 流水并行不适合温启动的轮次

PP 以及分块流水并行在长而全新的 prompt 上表现很好，大 prefill 提供足够的工作填满流水级，吞吐几乎线性扩展、通信代价很小[\[33\]](https://docs.vllm.ai/projects/ascend/en/latest/user_guide/feature_guide/dynamic_chunk_pipeline_parallel.html)。但 agentic 的多数轮次系统提示和历史都已缓存，每个新请求只加几百到几千 token，填不满流水线，气泡吃掉大部分潜在收益。边界很清楚：冷启动、计算密集的 prefill 适用；占据 agentic 会话主体的温启动、前缀密集轮次不适用。

### 13.2 DCP 不能从 Kimi K3 平移到 DeepSeek V4

DCP 在纯 MLA 模型（DeepSeek R1、Kimi K2.5、K2.7）和混合 MLA 模型（Kimi K3）上都好用，但 V4 的压缩稀疏注意力和高度压缩注意力包含 indexer、额外的压缩器和主注意力操作，上下文并行必须切分并协调所有这些子层，通信和实现复杂度都大幅上升。vLLM 在通信与计算重叠、以及对应 kernel 上投入很多之后，DCP 只是追平了 DEP。**投入很大、结论是持平，这类记录比成功案例更有参考价值——它标出了一条别人不必再走的路。**

### 13.3 负载均衡输给了会话粘性路由

这条与前期沉淀的立场直接相撞，值得单独展开。聚合式 DEP 部署里各 rank 的 KV 使用量差异很大，自然反应是按队列深度、运行 token 数或当前 KV 占用来均衡请求。在 AgentX 上，这些策略全部输给了简单的会话粘性路由。原因是缓存局部性：许多 agentic 会话的轮间间隔很短，下一轮到达时前缀往往还驻留在上一次服务它的 GPU 上。把会话迁到负载更低的 rank 会强制系统去取 KV——即便前缀在分布式 KV 池里被完整保留。传输是异步的、能和计算重叠，但并不免费：预取的 block 会临时占用目标 rank 的 GPU KV 容量，压低它能准入的序列数。最终系统得到一个更均衡的队列，同时处理更少的并发请求。

前期沉淀在这一点上给出的是相反方向的结论。Dynamo 的实验记录显示，并发 128 饱和之后提高 router temperature、保留 overlap weight，可以把 70B 1P/5D 的饱和 PoA 从 66.42 降到 21.53，吞吐代价约 13%——亲和在饱和后应当向负载均衡让步；llm-d v0.9 则把非局部决策分成亲和直到饱和、负载溢出、P2P KV pull 三段。kvcache 域 routing 节点的当前立场写的正是”饱和后亲和应让位负载均衡”。

**两边并不真的冲突，判据是轮间间隔与前缀驻留时间的关系。** Dynamo 和 llm-d 的实验里触发让步的是实例饱和——队列已经排到 TTFT 爆炸，此时迁移的收益足以盖过重取成本；AgentX 的会话轮间间隔短到前缀还没被驱逐，迁移成本是确定的（重取加占容量），均衡收益却是不确定的（目标 rank 也在跑别的会话）。原文的措辞值得原样记住：路由决策必须把每个 worker 上已经驻留了什么状态当成一类输入，而不只是排了多少活。这正是 routing 节点的立场需要补上的边界条件。

## 十四、路线图上还没做的部分

下一步是把 agentic 结构在整个服务栈里显式化，原文给了四个方向。

控制面上，首轮请求和第二轮之后的请求应当分开路由。首轮需要长的全新 prefill 去填满前缀缓存，第二轮之后有高复用、只需要短的追加 prefill；分开之后既避开队头阻塞，也允许两侧用不同的引擎配置和并行方式，例如首轮用 PCP 和 CPP。前期沉淀里已经记录过同类做法的实测：PPD 把 Turn 2+ 请求拆成 full prefill 与 append-prefill 两条路径，Turn 2+ 的 TTFT 降低 48%–73%，KV 传输负载削减约 75%。vLLM 把它列进路线图，说明这条路径正在从论文走向主流引擎的默认行为。

数据面和执行面上的三项都围绕”让引擎知道会话结构”。Agent hints 让 agentic 框架或 harness 随请求携带会话结构、潜在分叉点与缓存位置、工具调用延迟、会话生命周期等提示，先通过标准化 API 消费，再用来指导调度与驱逐。前期沉淀里 Dynamo 的 agent hints 在 coding agent 场景做到最高 4× 更低 TTFT、1.5× 更高吞吐，它的 `cache_control` 还能把指定 prefix node 在 worker 的 radix tree 里 pin 住、防止工具调用等待间隙被 LRU 驱逐——接口层面的事实标准正在形成。可编程 KV cache 则把预取、驱逐、软钉住的策略开放给用户，按各自负载模式配置。

第四项与第十二节那条反直觉结论正好互补：会话级 KV 管理利用轮间空隙，把保留的 KV 状态提前搬到可能服务下一轮的 worker 上。既然迁移的代价来自”下一轮到达时才去取”，那就在空隙里提前搬——**这是把粘性路由的约束条件改掉，而不是接受它。**

## 十五、结论与边界

回到开篇的判断。数据面在保证前缀留得住：统一页让混合注意力共享一个池，分层池把容量延伸到 CPU 和磁盘，两条保留策略决定在哪些边界存下不可切片的状态。执行面在前缀留得住的前提下让解码步更短：DCP 切 KV、DEP 保持注意力本地、PCP 分摊压缩器与 indexer。控制面决定谁能占用这一步、以及下一轮该去哪台机器。**三个平面的每一处改动都在回答同一个问题——这个会话的 KV 现在在哪、下一轮它还在不在那儿。**

边界要说清楚几条。

这是引擎团队的自述（工作由 Inferact 主导、vLLM 社区支持[\[34\]](https://inferact.ai/)），配置针对 AgentX 这一个负载调优。基准本身由 SemiAnalysis 独立运行、结果和 harness 公开可复现，这一层是第三方的；但哪种并行、哪个阈值最好，仍然是 vLLM 自己的选择，换一个 agentic harness 的会话结构（更长的轮间间隔、更少的子代理分叉）结论可能移动。

所有并行结论都绑定具体的模型架构与硬件世代。DEP16 在每 rank batch 超过 3 时胜过 DCP8 这条，换一代 scale-up domain 或换一个注意力栈就要重测；这也正是原文自己从 DeepSeek V4 的 DCP 尝试里得到的教训。

93%、2.65×、13%、10% 这些数字各自来自不同的模型-硬件组合——B300 上的 DeepSeek V4 Pro、32K prompt 的 PCP8 对 TP8、DCP8 每层、开 FP4 indexer 的 packed 布局——不能叠加成一个总收益。

最后留一个开放问题。三条 bitter lesson 里有两条在说同一件事：通用最优策略不存在。如果并行、调度、路由都必须跟着模型架构和会话结构走，那么配置搜索本身就会成为主要成本。vLLM 给的答案是两阶段测量加自动化 workflow。这个答案能不能跟上模型发布的节奏，是下一轮该看的东西。

- - -

## 参考资料

\[1\] [vLLM x AgentX: Optimizing for Real-World Agentic Serving](https://vllm.ai/blog/2026-09-08-vllm-agentx)

\[2\] [AgentX & InferenceXv3: Does the CUDA Moat Still Hold?](https://newsletter.semianalysis.com/p/agentx-inferencexv3-does-cuda-moat)

\[3\] [前期调研：AgentX 读后——agentic 推理把 CUDA 护城河从 kernel 挪到了会话状态](https://miraclefarms.github.io/notes/2026/09/09/agentx-agentic-inference-cuda-moat/)

\[4\] [OpenAI Enterprise Signals: Codex output token share](https://openai.com/signals/enterprise-data/)

\[5\] [Serving Agentic Workloads at Scale with vLLM x Mooncake](https://vllm.ai/blog/2026-05-06-mooncake-store)

\[6\] [DeepSeek V4 in vLLM](https://vllm.ai/blog/2026-04-24-deepseek-v4)

\[7\] [vLLM PR #44577: Packed KV cache layout](https://github.com/vllm-project/vllm/pull/44577)

\[8\] [vLLM PR #48993: Smaller allocation unit with the FP4 indexer](https://github.com/vllm-project/vllm/pull/48993)

\[9\] [Mooncake Store](https://github.com/kvcache-ai/Mooncake)

\[10\] [vLLM Docs: Mooncake Store connector usage](https://docs.vllm.ai/en/latest/features/mooncake_store_connector_usage/#configure-mooncake)

\[11\] [NVIDIA Dynamo](https://github.com/ai-dynamo/dynamo)

\[12\] [llm-d](https://github.com/llm-d/llm-d)

\[13\] [vLLM PR #46188: Hybrid KV lookup data structures](https://github.com/vllm-project/vllm/pull/46188)

\[14\] [vLLM PR #45444: Asynchronous lookup path](https://github.com/vllm-project/vllm/pull/45444)

\[15\] [vLLM PR #45659: Move lookup work off the scheduler critical path](https://github.com/vllm-project/vllm/pull/45659)

\[16\] [vLLM PR #47317: Parallel send and receive](https://github.com/vllm-project/vllm/pull/47317)

\[17\] [vLLM PR #43447: Interval-based retention](https://github.com/vllm-project/vllm/pull/43447)

\[18\] [vLLM PR #47782: Marconi-style selective retention](https://github.com/vllm-project/vllm/pull/47782)

\[19\] [Serving Kimi K3 with vLLM](https://vllm.ai/blog/2026-07-27-k3)

\[20\] [Decode Context Parallelism in vLLM](https://vllm.ai/blog/2026-08-07-decode-context-parallelism)

\[21\] [vLLM PR #48582: CuteDSL long-context indexer](https://github.com/vllm-project/vllm/pull/48582)

\[22\] [vLLM PR #52079: GEMM and reduce-scatter fusion](https://github.com/vllm-project/vllm/pull/52079)

\[23\] [vLLM PR #53152: Latent-tail MoE fusion](https://github.com/vllm-project/vllm/pull/53152)

\[24\] [vLLM PR #43584: MXFP4 MoE improvements](https://github.com/vllm-project/vllm/pull/43584)

\[25\] [vLLM PR #44230: HCA compression improvements](https://github.com/vllm-project/vllm/pull/44230)

\[26\] [vLLM PR #42925: Multi-stream C4A](https://github.com/vllm-project/vllm/pull/42925)

\[27\] [vLLM PR #43008: Cluster-based top-k](https://github.com/vllm-project/vllm/pull/43008)

\[28\] [AgentX run: DeepSeek V4 Pro 1.6T on 12 GB300](https://inferencex.semianalysis.com/inference/agentic/439873)

\[29\] [AgentX run: MiniMax M3 428B on 2 B300](https://inferencex.semianalysis.com/inference/agentic/439907)

\[30\] [AgentX run: Kimi K3 2.8T on 16 GB300](https://inferencex.semianalysis.com/inference/agentic/441066)

\[31\] [SemiAnalysis AgentX Dashboard](https://inferencex.semianalysis.com/inference?i_seq=agentic-traces&i_xmode=interactivity&g_runid=33418433573&i_best=0&i_active=b200_vllm%2Cb300_vllm%2Cgb200_dynamo-vllm%2Cgb300_dynamo-vllm&i_hc=1&i_advlabel=0&i_label=0)

\[32\] [SemiAnalysisAI/agentx-harness](https://github.com/SemiAnalysisAI/agentx-harness)

\[33\] [vLLM Ascend Docs: Dynamic chunked pipeline parallel](https://docs.vllm.ai/projects/ascend/en/latest/user_guide/feature_guide/dynamic_chunk_pipeline_parallel.html)

\[34\] [Inferact](https://inferact.ai/)
