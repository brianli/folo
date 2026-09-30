---
title: "Mooncake 与 vLLM 的结合点全景：L1/L2/L3 卸载与 PD 分离的接入地图"
source: "Fields"
category: "AI Infra"
group: "MiracleFarms"
url: "https://miraclefarms.github.io/notes/2026/07/16/vllm-mooncake-integration-panorama/"
published: 2026-07-16T09:10:00+08:00
saved: 2026-09-30T14:46:37+08:00
folo_key: "mf-fields::https://miraclefarms.github.io/notes/2026/07/16/vllm-mooncake-integration-panorama/"
tags:
  - "folo"
  - "AI_Infra"
  - "Fields"
---

# Mooncake 与 vLLM 的结合点全景：L1/L2/L3 卸载与 PD 分离的接入地图

> [!info] Fields · AI Infra · 2026-07-16 09:10 · [原文](https://miraclefarms.github.io/notes/2026/07/16/vllm-mooncake-integration-panorama/)

> **版本声明**：本文分析基于 vLLM commit `0885b519`（2026-07-15）与 Mooncake commit `7755a146`（2026-07-16）；除非特别说明，以下描述均基于此版本。

打算在 vLLM 里启用 Mooncake 的工程师，第一个困惑通常长这样：文档里有 MooncakeConnector 和 MooncakeStoreConnector 两个名字，社区讨论里又有 L1/L2/L3 分层卸载和 PD 分离两套叙事，它们到底是什么对什么？把源码翻开后答案很直接：**vLLM 没有给 Mooncake 留出”逐层插入”的接口，L1/L2/L3 这套分层是 vLLM 自己的内建体系，Mooncake 以 connector 身份进入，正式席位只有两个**——PD 分离的点对点数据面（MooncakeConnector），和跨实例共享 KV 池（MooncakeStoreConnector）。前者回答”prefill 算好的 KV 怎么交给 decode”，后者回答”这段前缀是不是集群里别人已经算过”。

这个结构性事实决定了本文的读法。与其问”Mooncake 怎么做 vLLM 的 L2”，先要弄清 vLLM 自己的分层长什么样、留了哪些缝隙，再看 Mooncake 从哪条缝进去、进去之后拿到了多少调度语义。前期调研文章已经分别拆过 vLLM 的前缀缓存与卸载管线[\[13\]](https://miraclefarms.github.io/notes/2026/06/16/vllm-prefix-cache-offload/)和 Mooncake 的源码架构[\[14\]](https://miraclefarms.github.io/notes/2026/06/16/mooncake-source-architecture-inference-data-plane-os/)；本文把两边的接口对到一起，给出一张截至 2026 年 7 月主线代码的结合点地图。

## 一、先数清楚结合点：两个正式席位，三条间接通道

vLLM 的 `KVConnectorFactory` 注册了十几种 connector，Mooncake 占了两个正式席位。`MooncakeConnector` 自 2025 年 12 月起直接进入 vLLM v1 主线，做 PD 分离下 prefill 到 decode 的点对点 KV 传输[\[1\]](https://docs.vllm.ai/en/latest/features/mooncake_connector_usage/)；`MooncakeStoreConnector` 经 PR #40900 于 2026 年 5 月 12 日合入，把 MooncakeDistributedStore 接成集群级共享 KV 池[\[4\]](https://github.com/vllm-project/vllm/pull/40900)。两者可以通过 `MultiConnector` 组合在同一个部署里。

正式席位之外还有三条间接通道，容易被忽略但覆盖了不少生产部署。第一条是 LMCache：vLLM 挂 LMCacheConnector 时，LMCache 可以用 `mooncakestore://` 协议把 Mooncake Store 当 remote backend[\[11\]](https://kvcache-ai.github.io/Mooncake/getting_started/examples/lmcache-integration.html)，Mooncake 退到存储层，缓存策略归 LMCache。第二条是 NIXL：Mooncake Transfer Engine 自 2025 年 5 月起是 NIXL 的 backend plugin[\[10\]](https://github.com/ai-dynamo/nixl/blob/main/src/plugins/mooncake/README.md)，vLLM 用 NixlConnector 做 PD 传输时，底层搬数据的可以是 Mooncake TE。第三条最新：2026 年 7 月 12 日合入的 Mooncake Conductor（PR #2833）以独立 C++ 服务的形态订阅 vLLM 的 BlockStored/BlockRemoved KV 事件流，维护跨实例前缀命中索引，向路由器暴露查询端点[\[12\]](https://github.com/kvcache-ai/Mooncake/pull/2833)——这条通道不搬数据，搬的是元数据，Mooncake 第一次把手伸向了 vLLM 引擎之上的路由层。

![[Mooncake 与 vLLM 的结合点全景：L1 L2 L3 卸载与 PD 分-01.png]]

*图 1：Mooncake 的原始架构以 KV cache 为中心组织 prefill/decode 池与分布式 KV 池。对 vLLM 而言，图中 Transfer Engine 对应 MooncakeConnector 的点对点传输，分布式 KVCache Pool 对应 MooncakeStoreConnector 的共享池，Conductor 对应新落地的 KV 事件索引服务。来源：Mooncake README。*

平台变体沿用同一套模式：vLLM-Ascend 用 Mooncake TE 做 NPU 上的 disaggregated prefill、用 Store 做 KV 池（前期调研文章拆过这条 KV pool connector 路径[\[16\]](https://miraclefarms.github.io/notes/2026/05/17/vllm-ascend-kv-pool-connector/)）；vLLM-Omni 用 `MooncakeTransferEngineConnector` 和 `MooncakeStoreConnector` 做多模态 pipeline 的跨阶段数据交换[\[7\]](https://github.com/kvcache-ai/Mooncake/blob/7755a1468faa44cab1d4a199090864031f88bd0f/README.md)。数一遍会发现规律：**Mooncake 进入 vLLM 生态的每一条路，走的都是数据面和元数据面，从来不碰调度决策**。

## 二、L1：Mooncake 进不去，也不需要进去

vLLM 的 L1 是 GPU HBM 上的前缀缓存，由 BlockPool 的 block 状态机和 KVCacheManager 在调度主路径上管理。这一层没有任何外部系统的位置——block 的分配、hash 索引、引用计数、LRU 驱逐全部是调度决策的组成部分，连 vLLM 自己的 OffloadingConnector 都只能站在外面。

那 Mooncake 和 L1 的关系是什么？是一条语义契约：block hash。Store 的对象 key（PoolKey）由 model name、TP/PP/DCP rank 加 block hash 拼成，外部命中查询沿着从 token 0 开始的连续 block hash 链做前缀增长，中间断一块，后面全部作废——这条规则由 L1 的 full-block 匹配语义规定，任何外部层只能遵守。

源码里有个细节把这层关系暴露得很彻底。Store connector 的调度侧协调器里有一个 `ExternalCachedBlockPool`，注释直白地写着 “Duck-typed BlockPool backed by a (group\_id, hash) exists set”[\[6\]](https://github.com/vllm-project/vllm/blob/0885b51981d09089e7a067276046b04a20e5b59d/vllm/distributed/kv_transfer/kv_connector/v1/mooncake/store/coordinator.py)——它把”远端 Store 里存在哪些 block”伪装成一个假的 BlockPool，喂给每个 KV cache group 各自的 manager 去跑原生的命中匹配逻辑。这样 sliding window、hybrid attention 这些 spec 各自的窗口语义不需要在 connector 里重写一遍。换个说法：**外部缓存想被 vLLM 理解，就得把自己打扮成 BlockPool 的样子**。这就是 L1 结合点的形态：一场语义模仿——外部层复用 vLLM 的匹配规则，而 block 的所有权和状态机始终留在 vLLM 手里。

## 三、L2：一条缝隙里挤着两条路线

CPU 内存这一层是地图上最拥挤的地方，因为 vLLM 给了接口但没有指定实现，两条路线在同一条缝隙里竞争。

第一条是 vLLM 原生路线。OffloadingConnector 的 `CPUOffloadingSpec` 把算完的 GPU block 用 `cudaMemcpyAsync` DMA 到 pinned host memory，异步执行、不占用计算关键路径[\[3\]](https://docs.vllm.ai/en/latest/features/kv_offloading_usage/)。这条路线是单实例私有的：CPU tier 只服务本机 GPU，跨实例不可见。它的延伸是 `TieringOffloadingSpec`——CPU 主层之下挂文件系统、对象存储等二级层，所有 GPU 与二级层的传输经 CPU 中转。前期调研文章拆过这套三 hook 松耦合设计[\[13\]](https://miraclefarms.github.io/notes/2026/06/16/vllm-prefix-cache-offload/)，这里只补一个观察：这套原生栈在 2026 年上半年长出了多级 tiering、per-request 策略钩子和 self-describing KV 事件，vLLM 对分层卸载的态度已经从”接口外包”转向”实现内建”。

第二条是 Mooncake Store 的 embedded 模式。配置里 `mode: "embedded"` 时，每个 vLLM rank 把 `global_segment_size` 大小的 CPU 内存贡献进分布式池[\[2\]](https://docs.vllm.ai/en/latest/features/mooncake_store_connector_usage/)——同一块 CPU DRAM，在原生路线里是本机私有的 L2，在 Mooncake 路线里同时是集群共享池的一个 segment，本机和邻居都能通过 RDMA 读它。

两条路线的取舍界线可以说得很具体。单实例、working set 没有溢出本机 CPU 容量时，原生 DMA 路线胜出：没有网络栈、没有 master 依赖、没有对象封装开销，而且前期实测显示 CPU tier 的收益本身要等 working set 超过约 250K–300K token 才转正，此前 connector 开销是净损失。一旦负载出现跨实例前缀共享——agentic 多轮会话被负载均衡打散到不同 replica 是典型场景——私有 L2 就失效了，每台机器各存一份、彼此 miss，此时 embedded Store 用同样的 CPU 内存换来全集群去重和共享。**L2 的选型问题实质上是”这批前缀会不会被第二台机器用到”的负载判断，参数只是执行**。

## 四、L3：Mooncake 的主场，以及它的语义边界

L3 层 Mooncake 的位置最稳固。原生 TieringOffloadingSpec 的 fs/obj 二级层走的是”CPU 中转 + 文件/对象写入”路径，定位是单实例容量扩展；Mooncake Store 给出的是另一种形态：DRAM/SSD 资源池化、RDMA 可达、跨实例寻址。vLLM 博客给出的 610 条 Codex/SWE-bench Pro agentic trace 数据是这个形态的核心证据：跨实例共享把 KV 命中率从 1.7% 拉到 92.2%，吞吐 3.8 倍，P50 TTFT 46 倍，端到端延迟 8.6 倍[\[9\]](https://vllm.ai/blog/mooncake-store)。前期调研文章逐段拆过这条数据路径[\[15\]](https://miraclefarms.github.io/notes/2026/05/07/vllm-mooncake-store-distributed-kv-cache/)；这里挑两处合入后仍在演进的机制。

![[Mooncake 与 vLLM 的结合点全景：L1 L2 L3 卸载与 PD 分-02.svg]]

*图 2：多个 vLLM 实例嵌入 Mooncake client，共享由 master 管理元数据的 DRAM/SSD 池；scheduler 先查前缀命中，worker 再经 RDMA 搬运 KV block。这张图对应 embedded 模式；standalone-store 模式把池的所有权移交给外部 client 进程。来源：vLLM 官方博客。*

第一处是拓扑选项的分化。PR #40900 的基线是 embedded 模式；当前文档新增了 `standalone-store` 模式——vLLM rank 变成纯请求方（`global_segment_size` 置 0），CPU 池和可选的 SSD 层由外部 `mooncake_client` 进程持有[\[2\]](https://docs.vllm.ai/en/latest/features/mooncake_store_connector_usage/)。配合 `enable_offload` 打开的 DirectIO staging buffer（防止大 prefill 冲垮 SSD 写入预算），Store 在 vLLM 侧的形态从”实例互借内存”走向”独立存储服务”。这个方向和 Mooncake 自身的多租户配额、SSD 分层演进是同一条线。

第二处是执行时机。Store connector 的 `start_load_kv()` 和 `wait_for_save()` 都是 no-op，真正的 I/O 在 `get_finished()` 里排队，在模型 compute 发射到 compute stream 之后才发出；保存路径记录 CUDA event，后台线程 put 前同步，保证不读到未写完的 GPU KV。数据搬运由 Mooncake 的 RDMA/GPUDirect 承担，GPU SM 不执行 copy kernel。**这条 SM-free 路径是 Store 对原生 fs/obj tier 最实质的性能差异来源**：后者的二级层 I/O 要经 CPU 中转两跳。

边界也要说清楚。Store 的甜区是”大块、连续、前缀语义”的 KV 复用；把访问模式换成稀疏注意力的逐层 demand fetch，RDMA 池的消息协议和散落小请求会成为结构性短板——前期调研的 CXL 对比实验里，Mooncake RDMA 方案在 16-token 原生 block 粒度下 TTFT 恶化到 76.8 秒、比重算更差[\[17\]](https://miraclefarms.github.io/notes/2026/05/13/beluga-cxl-kvcache-memory-pool/)。问题的根源在访问语义与传输原语的错配：Mooncake 为吞吐型批量搬运设计，细粒度随机访问是 CXL load/store 这类内存语义的领地，实现层面再怎么调参也跨不过去。在 vLLM 里启用 Store 前值得自问的是负载的复用粒度，而这个问题文档不会替你回答。

## 五、PD 分离：点对点数据面，和把两个席位拼起来的组合器

PD 分离场景下 Mooncake 的接入点是 MooncakeConnector，机制与 Store 完全不同：没有池、没有 master，prefill 和 decode 实例直连。控制面上，prefill 侧起一个 bootstrap server（默认端口 8998），decode 侧通过 ZMQ side channel 发起握手和 pull 请求；数据面上方向相反——源码注释明确写着这是 **P-push 架构**：decode 只发元数据请求，实际搬运由 prefill 调用 `batch_transfer_sync_write` 向 decode 已注册的 KV 内存做单边 RDMA 写[\[5\]](https://github.com/vllm-project/vllm/blob/0885b51981d09089e7a067276046b04a20e5b59d/vllm/distributed/kv_transfer/kv_connector/v1/mooncake/mooncake_connector.py)。这个方向选择有个可观测的副作用：成功传输的延迟和字节数指标只在 P 侧 worker 上，D 侧只记录失败——监控面板要去 prefill 节点找传输指标。

工程细节里能看出这条路径吃过生产的亏。每个 prefill worker 默认开 10 线程的传输线程池；请求被 abort 而 decode 未通知 prefill 时，prefill 侧按 480 秒超时自动释放该请求的 KV block，避免无限期占用；异构 TP 有一套显式的 transfer plan 计算，处理 producer 和 consumer TP 度不同、MLA 模型 KV 全副本（`producer_cache_replicated`）等布局差异，PP 布局不一致时还会按 layer 名做区域对齐校验[\[5\]](https://github.com/vllm-project/vllm/blob/0885b51981d09089e7a067276046b04a20e5b59d/vllm/distributed/kv_transfer/kv_connector/v1/mooncake/mooncake_connector.py)。这些逻辑对使用者不可见，但它们意味着 1P1D 之外的 XpYd、异构并行度组合是被一等公民对待的。

![[Mooncake 与 vLLM 的结合点全景：L1 L2 L3 卸载与 PD 分-03.gif]]

*图 3：MultiConnector 让 MooncakeConnector 的 P→D 点对点传输和 MooncakeStoreConnector 的共享前缀池同时生效：prefill 先从 Store 拉命中前缀、只算增量，再把完整 KV 推给 decode，两侧都可回写 Store。来源：vLLM 官方博客。*

两个席位拼起来才是完整的 PD 部署形态。官方推荐的 XpYd 配置里，prefill 和 decode 各自挂 `MultiConnector`，内层同时列出 MooncakeConnector（P 侧 `kv_producer` / D 侧 `kv_consumer`）和 MooncakeStoreConnector（两侧都是 `kv_both`）[\[2\]](https://docs.vllm.ai/en/latest/features/mooncake_store_connector_usage/)。分工是正交的：Store 解决”这段前缀历史上算过没有”（跨请求、跨实例的时间维度复用），P2P connector 解决”这次请求的 KV 怎么从 P 到 D”（单请求的空间维度交接）。**PD 分离与 L3 池在 vLLM 里由两个独立 connector 承担、经组合器叠加，这个设计让两者可以独立开关、独立换供应商**——比如 P2P 换 NixlConnector、池换 LMCache，Mooncake 在每个位置都面对竞争。

## 六、对照 SGLang HiCache：同一个 Mooncake，两种接入哲学

把镜头转向 SGLang 能看清 vLLM 这套设计的独特之处。SGLang 的 HiCache 把 L1（GPU）、L2（host）、L3（分布式存储）组织成显式层级，元数据统一在 HiRadixTree 里——每个节点记录 KV 在哪几层、本地层精确到地址；prefetch 有 best\_effort/wait\_complete/timeout 三种终止策略，timeout 还带按 token 量动态计算的公式；write-back 有 write\_through/selective/write\_back 三档[\[8\]](https://github.com/kvcache-ai/Mooncake/blob/7755a1468faa44cab1d4a199090864031f88bd0f/docs/source/design/hicache-design.md)。Mooncake 在其中是 L3 的一个 backend，L3 命中超过 256 token 阈值才触发预取，数据经 RDMA 零拷贝直达 L2。

![[Mooncake 与 vLLM 的结合点全景：L1 L2 L3 卸载与 PD 分-04.png]]

*图 4：HiCache 仿照 CPU 缓存层级，L1/L2 私有、L3 共享，分层逻辑内嵌在调度器的 HiRadixTree 里。对照 vLLM 的 connector 模式，同一个 Mooncake 集群拿到的调度语义深度完全不同。来源：Mooncake HiCache 设计文档。*

同一个 Mooncake 集群，在两个框架里的角色一致——L3 存储 substrate 加 PD 传输引擎——但拿到的调度语义深度差别很大。SGLang 把分层做进调度器，Mooncake 因此获得 prefetch 超时、write-back 策略、层间提升这些细粒度语义的配合；vLLM 把分层挡在 connector 接口外，调度器只看到”外部还有 N 个 token 可用”一个数字，Mooncake 拿不到延迟特性、优先级这些上下文，但换来的是可替换性和主线演进不被绑定。这两种哲学没有对错，判断标准是谁承担系统复杂度：HiCache 模式复杂度进框架、换 backend 便宜；vLLM 模式复杂度进外部系统、框架保持中立。而两边正在向对方的方向渗透——vLLM 原生 offload 栈在内建化多级 tiering，Mooncake Conductor 在通过 KV 事件流爬向路由层。**缝隙两侧的领土都在扩张，connector 接口这条边界线未来几个季度可能是 vLLM 生态里争夺最激烈的位置**。

## 七、结论

回到开篇的问题：Mooncake 与 vLLM 的结合点，数下来是”两个正式席位 + 三条间接通道”，而 L1/L2/L3 的分层叙事要按层拆开读——L1 是纯语义契约（block hash 与前缀连续性，外部系统只能模仿 BlockPool 来对话）；L2 是两条路线的竞争地带（原生 DMA 私有卸载 vs embedded Store 共享池，界线在负载是否跨实例复用前缀）；L3 是 Mooncake 的主场（RDMA 池化 + SM-free 数据路径，甜区是大块连续前缀，稀疏细粒度访问是明确的语义边界）；PD 分离则由独立的 P-push 点对点 connector 承担，与 Store 经 MultiConnector 正交组合。

这张地图的适用条件也要交代。它基于 vLLM `main@0885b519` 与 Mooncake `main@7755a146`，其中 standalone-store 模式、Conductor 服务都还在快速变化期，不能当作稳定 API 依赖。悬而未决的问题有两个：一是 Conductor 的前缀命中索引会不会成为 vLLM 集群路由的标准控制面输入——如果成，Mooncake 就从数据面第一次进入了调度决策链路；二是 vLLM 原生 tiering 栈持续内建化之后，Store 在单集群场景的差异化会收窄到什么程度——跨实例共享和 RDMA 数据面是它守得住的阵地，而”给单实例扩容”这个早期卖点正在被原生栈蚕食。这两个问题的答案，大概率决定下一版地图的画法。

- - -

## 参考资料

\[1\] [vLLM MooncakeConnector Usage Guide](https://docs.vllm.ai/en/latest/features/mooncake_connector_usage/)

\[2\] [vLLM MooncakeStoreConnector Usage Guide](https://docs.vllm.ai/en/latest/features/mooncake_store_connector_usage/)

\[3\] [vLLM KV Offloading Usage Guide](https://docs.vllm.ai/en/latest/features/kv_offloading_usage/)

\[4\] [vLLM PR #40900: Add MooncakeStoreConnector](https://github.com/vllm-project/vllm/pull/40900)

\[5\] [vLLM mooncake\_connector.py（commit 0885b519）](https://github.com/vllm-project/vllm/blob/0885b51981d09089e7a067276046b04a20e5b59d/vllm/distributed/kv_transfer/kv_connector/v1/mooncake/mooncake_connector.py)

\[6\] [vLLM mooncake store coordinator.py（commit 0885b519）](https://github.com/vllm-project/vllm/blob/0885b51981d09089e7a067276046b04a20e5b59d/vllm/distributed/kv_transfer/kv_connector/v1/mooncake/store/coordinator.py)

\[7\] [Mooncake README（commit 7755a146）](https://github.com/kvcache-ai/Mooncake/blob/7755a1468faa44cab1d4a199090864031f88bd0f/README.md)

\[8\] [Mooncake x SGLang HiCache System Design](https://github.com/kvcache-ai/Mooncake/blob/7755a1468faa44cab1d4a199090864031f88bd0f/docs/source/design/hicache-design.md)

\[9\] [vLLM 官方博客：Mooncake Store](https://vllm.ai/blog/mooncake-store)

\[10\] [NIXL Mooncake Backend Plugin](https://github.com/ai-dynamo/nixl/blob/main/src/plugins/mooncake/README.md)

\[11\] [Mooncake x LMCache Integration Guide](https://kvcache-ai.github.io/Mooncake/getting_started/examples/lmcache-integration.html)

\[12\] [Mooncake PR #2833: Conductor KV event indexer](https://github.com/kvcache-ai/Mooncake/pull/2833)

\[13\] [前期调研：vLLM 前缀缓存与分层卸载——一条被刻意解耦的两段式管线](https://miraclefarms.github.io/notes/2026/06/16/vllm-prefix-cache-offload/)

\[14\] [前期调研：Mooncake 源码架构分析——从 KV Cache Store 到推理数据面操作系统](https://miraclefarms.github.io/notes/2026/06/16/mooncake-source-architecture-inference-data-plane-os/)

\[15\] [前期调研：vLLM x Mooncake Store——Agentic 推理为什么需要分布式 KV Cache 池](https://miraclefarms.github.io/notes/2026/05/07/vllm-mooncake-store-distributed-kv-cache/)

\[16\] [前期调研：昇腾 KV Pool 阅读——vLLM Ascend 如何把 Mooncake 接成硬件亲和的数据面](https://miraclefarms.github.io/notes/2026/05/17/vllm-ascend-kv-pool-connector/)

\[17\] [前期调研：Beluga——CXL 内存池为什么会改变 KV Cache Offload 的工程重心](https://miraclefarms.github.io/notes/2026/05/13/beluga-cxl-kvcache-memory-pool/)

### 版本对齐信息

| 来源                | 版本                                  | 说明                                          |
| ----------------- | ----------------------------------- | ------------------------------------------- |
| vLLM 主线源码         | `main@0885b519`（2026-07-15）         | connector 源码与 docs/features 文档引用均基于此 commit |
| Mooncake 主线源码     | `main@7755a146`（2026-07-16）         | README、HiCache 设计文档基于此 commit               |
| vLLM PR #40900    | merged 2026-05-12                   | MooncakeStoreConnector 合入主线                 |
| Mooncake PR #2833 | merged 2026-07-12（merge `d3ebf384`） | Conductor KV 事件索引服务落地                       |
