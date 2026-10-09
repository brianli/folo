---
title: "Mooncake 到底在推理集群里干什么？算力网工必须看懂的 KV Cache 新架构"
source: "算力网络架构手记"
category: "Inbox"
group: "AI基础设施"
author: "疯狂的兔子tommy"
url: "https://mp.weixin.qq.com/s/WLfYdJVLHsG2An4C7-M6Hw"
published: 2026-10-09T21:54:43+08:00
saved: 2026-10-09T21:54:50+08:00
folo_key: "url::https://mp.weixin.qq.com/s/WLfYdJVLHsG2An4C7-M6Hw"
tags:
  - "folo"
  - "Inbox"
  - "算力网络架构手记"
---

# Mooncake 到底在推理集群里干什么？算力网工必须看懂的 KV Cache 新架构

> [!info] 算力网络架构手记 · Inbox · 2026-10-09 21:54 · [原文](https://mp.weixin.qq.com/s/WLfYdJVLHsG2An4C7-M6Hw)

这里是「算力网络架构手记」北京

- **专注 AI 集群网络架构与性能优化**

- **深入 GPU×RoCE×NCCL×K8s 跨层瓶颈**

- **只拆真实问题，不写概念科普**

- - -

👇 试听内容

- - -

00

Mooncake 不是“又一个推理框架”，而是在重构 KV Cache 的位置关系

很多算力网工第一次听到 Mooncake，容易把它理解成：

- 一个推理框架？

- 一个缓存系统？

- 一个 RDMA 传输库？

- 一个 P/D 分离组件？

- 一个给 vLLM 或 SGLang 用的插件？

这些说法都沾边。  但都不够准确。

更工程化地说：

> Mooncake 的核心价值，是把 KV Cache 从“某个推理实例里的临时状态”，变成“可以被集群管理、复用、迁移和调度的资源”。

这句话非常关键。

过去我们理解 KV Cache，往往是站在单个推理实例里看。

用户请求进来。  Prefill 生成 KV。  Decode 继续用 KV。   

请求结束，KV 可能释放。  这个 KV 基本跟当前实例绑定。

但在长上下文、多轮对话、Agent、RAG、P/D 分离、高并发推理里，这种做法会遇到几个问题：

同一个前缀被反复计算。  请求换了实例后，原来的 KV 不能复用。  Prefill 和 Decode 分开后，KV 必须高效搬运。  GPU HBM 很贵，放不下越来越多的上下文状态。  长上下文请求会把显存、网络、调度一起拖重。

Mooncake 要解决的，就是这类问题。

它不是简单“加一个缓存”。  而是把 KV Cache 提升为推理集群里的一级资源。

- 

```
Mooncake 官方介绍也明确说，它采用 KV cache-centric disaggregated architecture，分离 prefill 和 decode 集群，并利用 GPU 集群里未充分使用的 CPU、DRAM、SSD 资源构建 disaggregated KV cache pool。
```

所以：

- KV Cache 从哪里产生？

- 存到哪里？

- 谁负责找？

- 谁负责搬？

- 走哪张网卡？

- 跨不跨 Rack？

- 会不会形成热点？

- ECN/CNP/PFC 会不会被牵动？

这才是网络视角真正该看的东西。

![[Mooncake 到底在推理集群里干什么？算力网工必须看懂的 KV Cache-01.png]]

01

为什么推理集群突然需要这种 KV Cache 新架构？

原因很简单：

推理的瓶颈变了。

以前大家讨论大模型系统，最容易想到训练。

- NCCL

- AllReduce

- 参数同步

- 大规模 GPU 集群

- 东西向大流量

这些当然重要。

但到了推理场景，尤其是长上下文和 Agent 场景，问题开始变成另一种样子。

用户不是只问一句短问题。  用户会上传长文档。  会多轮对话。  会反复调用工具。  会在同一类系统 prompt 下不断发起请求。  会让 Agent 一步一步规划、检索、生成、再检索。

这时候，系统里会产生大量可以复用的 KV Cache。

如果这些 KV 只能留在本实例里，一旦请求被路由到别的实例，就可能重新 Prefill。  

如果这些 KV 不能被高效迁移，P/D 分离就会被 KV 传输拖慢。  

如果这些 KV 只能放在 GPU HBM 里，容量又会受限。

- 

```
vLLM 与 Mooncake 的集成文章也提到，在 Agentic workloads 里，不能再把推理服务看成一组彼此孤立的 vLLM 副本，实例之间需要共享分布式 KV cache pool，用来获得更大的聚合容量和跨实例 cache hit。
```

这就是 Mooncake 出现的背景。

不是为了炫技。  而是因为推理系统已经从“单实例服务”走向“集群级状态管理”。

> 当请求可以跨实例流动，KV Cache 就不能再只属于某一个实例。

这句话，就是理解 Mooncake 的入口。

对算力网工来说，这意味着：

网络承载的不再只是模型并行通信。  还会承载 KV Cache 的跨实例读写、迁移和复用。

它可能不像 NCCL AllReduce 那样整齐。  但它同样会影响 TTFT、TPOT、P99。

![[Mooncake 到底在推理集群里干什么？算力网工必须看懂的 KV Cache-02.png]]

02

Mooncake 里最重要的两层：Transfer Engine 和 Mooncake Store

算力网工理解 Mooncake，先抓两层就够了。

第一层：**Transfer Engine，简称 TE。**  第二层：**Mooncake Store。**

可以先用一句话理解：

> Transfer Engine 负责“怎么高效搬数据”，Mooncake Store 负责“KV Cache 放在哪里、怎么找、怎么管”。

- 

```
Mooncake 官方文档把 Transfer Engine 描述为高性能、zero-copy data transfer library，围绕 Segment 和 BatchTransfer 两个核心抽象；它支持在 DRAM、VRAM、NVMe-oF 等 Segment 之间做批量读写，并支持 TCP、RDMA、EFA、NVMe-oF、NVLink 等多种传输后端。
```

这对网络工程师很好理解。

TE 就像一个专门给 KV、Tensor、Embedding、权重这类大对象搬运设计的数据面。

它关心的是：

- 从哪里读

- 写到哪里

- 数据有多大

- 用哪种传输协议

- 走哪张网卡

- 要不要多 NIC 聚合

- 失败后能不能换路

- GPU 内存能不能直接参与 RDMA

Mooncake Store 则是另一层。

- 

```
Mooncake Store 官方文档说，它是面向 LLM inference 的高性能分布式 KV cache storage engine；它提供 Put、Get、Remove 等对象级 API，支持多副本、强一致、zero-copy 传输、多 NIC 带宽聚合、动态扩缩容、多层存储等能力。
```

这就不是单纯“搬数据”了。

它还要管理对象。

- 哪个 KV object 存在哪些节点？

- 有没有副本？

- 生命周期怎么管？

- 缓存满了怎么淘汰？

- 节点加入和离开怎么办？

- 某个 KV 被访问时，应该从哪里读？

所以 Mooncake 不是只有一个“传输库”。

它更像：

底层有 TE 负责高速搬运。  上层有 Store 负责分布式 KV 管理。  再往上接 vLLM、SGLang、TensorRT-LLM 等推理框架。

![[Mooncake 到底在推理集群里干什么？算力网工必须看懂的 KV Cache-03.png]]

03

Master、Client、TE：到底谁干什么？

很多人会把 Mooncake Store 里的角色搞混。

尤其是 Master、Client、Transfer Engine。

我们拆开讲。

### 1）Master：管元数据，不搬数据

### 

- 

```
Mooncake Store 文档里提到，架构中有两个关键组件：Master Service 和 Client。Master Service 负责整个集群逻辑存储空间池的编排，管理节点加入和离开，负责对象空间分配和元数据维护。
```

所以 Master 很像“调度员”和“登记员”。

它知道：

- 有哪些 Client 节点。

- 哪些节点贡献了多少存储空间。

- 某个 KV object 放在哪里。

- 副本有几份。

- 哪些节点健康。

- 对象生命周期怎么维护。

但它不是数据搬运工。

### 

### 2）Client：既是调用方，也是存储贡献方

- 

```
Mooncake Store 文档里有一个很关键的点：Client 有两个角色。它既可以被上层应用调用，发起 Put/Get 等请求；也可以作为 store server，贡献一段连续内存 segment，向其他 Client 提供分布式 KV cache 存储能力。实际数据传输是 Client 到 Client，绕过 Master Service。
```

这句话对算力网工非常重要。

也就是说，Mooncake Store 不是所有数据都从 Master 过一遍。  Master 主要管元数据。   

真正的数据，是 Client 之间通过 TE 搬。

### 

### 3）TE：负责真正的数据通道

### 

当一个 Client 要从另一个 Client 读 KV，真正走数据面的，是 Transfer Engine。

- 它负责建立连接

- 注册内存

- 发起 RDMA Read/Write

- 选择 NIC

- 切片传输

- 处理失败重试

- 尽量做到 zero-copy 和高带宽

所以，最简化的理解是：

**Master 告诉你“数据在哪里”。**  **Client 负责“我要存/我要取”。**  **TE 负责“把数据搬过去”。**

> Master 不在数据面，Client 才是数据面的参与者，TE 是数据搬运的发动机。

这也是为什么算力网工排查 Mooncake 时，不能只看 Master 是否正常。   

Master 正常，只说明元数据服务可能正常。  真正影响性能的，往往在 Client 到 Client 的数据路径上。

比如：

- RDMA 网卡选错

- NUMA 亲和不对

- 某条 Rail 热点

- CNP 增加

- PFC 触发

- TE fallback 到 TCP

- 多 NIC 没有聚合起来

- 某个 KV Owner 被集中访问

这些都不一定能从 Master 进程状态里看出来。

![[Mooncake 到底在推理集群里干什么？算力网工必须看懂的 KV Cache-04.png]]

04

Mooncake 在 P/D 分离里到底干了什么？

P/D 分离，是理解 Mooncake 的关键场景。

P/D 分离就是：

Prefill 和 Decode 分开跑。  Prefill 处理 prompt，生成 KV Cache。  Decode 使用 KV Cache，逐 token 生成输出。

这种架构的好处是明显的。

Prefill 偏计算密集。  Decode 偏访存和持续生成。   

把它们拆开，可以让资源更专业、更灵活。

但问题也随之出现：

Prefill 生成的 KV，Decode 要用。   

如果它们不在同一张 GPU、同一台服务器，KV 就要搬。

这一步如果做不好，P/D 分离的收益会被抵消。

Mooncake 在这里的作用，就是为 KV Cache 搬运和共享提供一条更高效的路径。

- 

```
vLLM 与 Mooncake 的文章提到，vLLM 已经通过 MooncakeConnector 在 P/D disaggregation 中采用 Mooncake，用 Mooncake 的 transfer engine 在 GPU 之间移动 KV cache；进一步的 Mooncake Store 方案，则构建了分布式 KV cache pool。
```

这就说明，Mooncake 不是在替代 Prefill 或 Decode 的计算。  它是在接管中间那段很关键的 KV 数据路径。

流程大概是：

- Prefill 生成 KV。

- KV 被写入本地或远端 Mooncake 管理的存储空间。

- Master 维护 KV object 的元数据。

- Decode 需要时查询并读取 KV。

- Client 之间通过 TE 传输 KV。

- TE 尽量用 RDMA、GPUDirect RDMA、多 NIC、拓扑感知路径把数据搬过去。

> P/D 分离真正难的不是“把 Prefill 和 Decode 拆开”，而是“拆开之后 KV 怎么高效、稳定、低延迟地过去”。

Mooncake 就是在解决这条链。

![[Mooncake 到底在推理集群里干什么？算力网工必须看懂的 KV Cache-05.png]]

05

Mooncake Store 不是 Redis：它管理的是大对象、热对象、GPU 相关对象

很多人听到 KV Store，会下意识想到 Redis、Memcached。

但 Mooncake Store 不能按这个思路理解。

Redis 里的 key/value，更多是通用缓存对象。  Mooncake Store 面向的是 LLM inference 里的 KV Cache、模型权重、hidden states 这类大对象和张量对象。

- 

```
Mooncake Store 文档也强调，它不是通用 caching system，而是 distributed KV cache；它提供底层对象存储和管理能力，并专门设计来加速 LLM inference performance。
```

这意味着它更关心：

- 大对象如何切片。

- 如何多副本。

- 如何多 NIC 并行。

- 如何 zero-copy。

- 如何 DRAM/SSD 分层。

- 如何跟推理 Scheduler 协同。

- 如何避免热点。

- 如何让 GPU HBM、CPU DRAM、SSD 形成一个可用的缓存层级。

普通缓存系统不一定要理解 GPU。  Mooncake 这类系统必须理解 GPU 相关数据路径。

因为 KV Cache 的价值就在推理链路里。

如果它取回来太慢，TTFT 变差。  如果它迁移太慢，P/D 分离受损。  如果它热点太集中，RoCE 队列突刺。  如果它路径选错，NUMA/PCIe/NIC 亲和都会出问题。

> Mooncake Store 不是“把 KV 存起来”这么简单，而是要让 KV Cache 以推理系统可接受的方式被复用。

这也是算力网工必须参与的原因。

因为这里面有大量网络和硬件路径问题。

![[Mooncake 到底在推理集群里干什么？算力网工必须看懂的 KV Cache-06.png]]

06

Mooncake 对网络意味着什么？

从网络角度看，Mooncake 最大的变化是：

KV Cache 不再只是 GPU 内部状态。  它会变成集群内可移动的数据流。

这意味着网络上会出现新的流量类型：

- Prefill 到 Decode 的 KV 传输

- Decode 从远端 Store 拉 KV

- 多个实例共享同一个前缀 KV

- KV 从 HBM 写回 DRAM/SSD

- KV 从 DRAM/SSD 预取回 GPU

- 热点 KV 的多副本读

- 失效、重试、故障切换后的重传

这些流量不一定像 NCCL AllReduce 那样周期整齐。   

它可能更像请求驱动型流量。

请求来了才有。  命中远端才有。  长上下文才大。  热点前缀才集中。  Scheduler 同步释放时才爆。

所以它更容易表现为：

- 短时突刺

- 局部热点

- 某条 Rail 不均衡

- 某个 KV Owner 被打爆

- ECN/CNP 增量突然上升

- P99 抖动

- 

```
Mooncake 的 Transfer Engine 虽然支持多 RDMA NIC 聚合、拓扑感知路径选择、故障自动换路等能力，官方文档也提到它会根据 CPU/GPU 与 RDMA 设备的拓扑关系生成矩阵，优先选择本地 NUMA 或 GPU Direct RDMA 更合适的 NIC；大请求还可以切成多个 slice 使用不同路径。
```

但这并不意味着网络工程师可以不管。

正好相反。

因为 TE 越强，越能把 KV 流量打出来。  如果调度不合理，或者网络规划不合理，KV 流量就会非常真实地压到 RoCE Fabric 上。

> Mooncake 不是让网络问题消失，而是把 KV Cache 流量从隐性状态变成显性数据面。

这对算力网工是机会，也是挑战。

机会是：你终于能观测、压测、治理这条 KV 数据路径。  挑战是：你必须理解它为什么会出现、什么时候出现、打向哪里。

![[Mooncake 到底在推理集群里干什么？算力网工必须看懂的 KV Cache-07.png]]

07

Mooncake 能不能解决 KV Cache 热点？

很多人看到 Mooncake Store 支持多副本、分布式缓存、多 NIC、zero-copy，就会觉得：

那 KV 热点是不是就解决了？

不能这么想。

Mooncake 确实提供了缓解热点的基础能力。

比如 Mooncake Store 文档提到，它支持 multi-replica，同一个对象可以存多份副本，从而缓解访问压力热点；还支持 striping、parallel I/O 和多 NIC 聚合等能力。

这当然很有价值。

但热点治理不是存储系统单方面能完成的。

因为热点的来源在上层。

- Router 是否把请求打向同一个 KV Owner？

- Scheduler 是否在同一时间窗口释放大量请求？

- 某个系统 prompt 是否被大量复用？

- 长上下文请求是否被集中处理？

- P/D 映射是否跨 Rack？

- 副本位置是否和访问位置匹配？

- 网络是否某条 Rail 已经热了？

如果这些不改，Mooncake 只能帮你更快地搬、更合理地放。  它不能自动保证上层不会制造 Incast。

所以，Mooncake 更像是给推理系统提供了一套 KV Cache 数据底座。

它提供能力。   

但能力怎么用，仍然要靠 Router、Scheduler、KV 管理策略和网络规划。

> Mooncake 能降低 KV Cache 复用和迁移的成本，但不能替上层调度消灭错误的访问关系。

这句话要冷静。

不要把 Mooncake 神化成“用了就没网络问题”。  真正成熟的系统，需要 Mooncake + 调度策略 + 拓扑感知 + 网络反馈闭环。

![[Mooncake 到底在推理集群里干什么？算力网工必须看懂的 KV Cache-08.png]]

08

Mooncake 和 NCCL 是什么关系？

算力网工很容易把各种 GPU 通信组件混在一起。

NCCL 是干什么的？  Mooncake 是干什么的？  两者是不是替代关系？

不是。

NCCL 主要面向 GPU 间 collective communication，比如训练里的 AllReduce，也包括推理 TP/EP 场景里的某些 AllGather、AllReduce、All-to-All 类通信。

Mooncake 的主线不是 collective communication。  它的主线是 KV Cache、Tensor、Embedding、权重等对象的数据移动和分布式管理。

当然，Mooncake 项目现在也不止 Store 和 TE。官方 README 提到 Mooncake EP 和 Mooncake PG 扩展了 MoE 推理里的容错分布式执行能力，Mooncake PG 可以注册为 torch.distributed 后端，支持类似 all\_gather 的标准 collective API。

但对算力网工而言，理解边界很重要：

NCCL 更像“多个 GPU 按 collective 规则一起通信”。  Mooncake TE 更像“把某个对象从 A 搬到 B，或者从多个位置读写”。  Mooncake Store 更像“管理这些可复用对象在哪里”。

所以排障时也要分开看。

- 如果是 TP/EP 同步慢，你可能要看 NCCL、Rank、Channel、Rail、GPU-NIC 亲和。

- 如果是 KV 迁移慢，你要看 Mooncake Client、TE、RDMA 路径、KV Owner、Master 元数据、Store 副本和对象位置。

- 如果是 MoE expert dispatch/combine，还要看对应 EP 通信栈和网络热点。

> NCCL 看的是 collective 通信链，Mooncake 看的是 KV/Tensor 对象的数据链；两条链都会打网络，但证据完全不同。

这句话非常适合排障时贴在白板上。

不要看到 RoCE CNP 就只怀疑 NCCL。  也不要看到 P99 抖动就只怀疑 Mooncake。   

要把通信类型分清楚。

![[Mooncake 到底在推理集群里干什么？算力网工必须看懂的 KV Cache-09.png]]

09

算力网工排查 Mooncake，要看哪些证据？

如果集群上了 Mooncake，算力网工不能只问“服务是不是 alive”。

要看证据链。

### 

### 第一层：控制面和元数据

- Master 是否正常。

- Client 是否注册。

- 对象元数据是否可查。

- KV object 是否有正确位置。

- 副本数是否符合预期。

- 节点健康状态是否变化。

- TE metadata service 是否可用。

- 

```
注意，Mooncake Store 的 Master Service 不包括 Transfer Engine 所需的 metadata service；TE 可能依赖 etcd、Redis、HTTP 等元数据服务，需要单独部署和排查。
```

### 第二层：数据面路径

- Client 到 Client 是否走 RDMA。

- 有没有 fallback 到 TCP。

- 是否启用了多 NIC。

- TE 是否选到了正确 NIC。

- GPU/NUMA/NIC 亲和是否合理。

- 大对象是否被切片并行传输。

- 路径是否跨 Rack、跨 Pod。

### 

### 第三层：网络拥塞信号

- 每条 Rail 的流量是否均衡。

- CNP 是否增加。

- ECN marked packets 是否抬升。

- PFC pause 是否触发。

- 是否有某个 KV Owner 对应端口成为热点。

- 是否有某个 RoCE priority / TC 队列突刺。

### 

### 第四层：推理业务结果

- TTFT 是否变好。

- TPOT 是否稳定。

- P99/P999 是否下降。

- 长上下文请求是否改善。

- 缓存命中后是否真的减少 Prefill。

- 跨实例请求是否不再重复计算。

这四层要对齐看。

不要只看 Mooncake 单点性能。  也不要只看 RoCE 计数器。

> Mooncake 排障的核心，是把“KV object 位置”和“网络实际路径”对上账。

这就是算力网工的价值。

你要能回答：

- 这个 KV 在哪里？

- 谁在读它？

- 从哪张 NIC 出去？

- 走哪条 Rail？

- 有没有 ECN/CNP？

- 对 TTFT/P99 有什么影响？

能回答这些，才算真正看懂 Mooncake。

![[Mooncake 到底在推理集群里干什么？算力网工必须看懂的 KV Cache-10.png]]

10

Mooncake 会倒逼网络规划发生什么变化？

Mooncake 这类架构一旦进入生产，网络规划就不能只按“训练 AllReduce”思路来做。

因为推理集群里多了一类重要流量：

KV Cache 数据流。

这类流量有几个特点。

第一，它是请求驱动的  不是固定周期通信。

第二，它可能高度不均匀  热点 prompt、热点会话、热点 KV Owner 都会影响流量分布。

第三，它和缓存命中相关  命中率越高，不代表网络越轻；还要看命中在哪里。

第四，它对尾延迟敏感  KV 拉慢了，TTFT 就可能变差。

第五，它依赖拓扑  同机、同 Rack、跨 Rack、跨 Pod，代价完全不同。

所以网络规划要多考虑：

- P/D 集群怎么摆放。

- Prefill、Decode、Store 是否跨 Rack。

- KV Store 节点是否靠近访问方。

- RoCE Rail 是否均衡。

- 每台机器 GPU-NIC 亲和是否正确。

- DRAM/SSD 层级是否会导致跨网回源。

- ECN/PFC 阈值是否能承受 KV burst。

- 监控是否能按 KV object 和 Request ID 追踪。

> Mooncake 不是一个“应用小组件”，它会改变推理集群的数据面流量模型。

这就是算力网工必须提前看懂它的原因。

如果你等到 P99 抖动、CNP 增加、PFC 偶发触发时再看，就已经很被动了。

![[Mooncake 到底在推理集群里干什么？算力网工必须看懂的 KV Cache-11.png]]

11

Mooncake 真正带来的，是 KV Cache 视角的系统重构

回到标题：

Mooncake 到底在推理集群里干什么？

它不是简单替代某个推理框架。  不是单纯做缓存。  也不是只有一个 RDMA 传输库。

它真正带来的是一种新架构：

以 KV Cache 为中心，把 Prefill、Decode、存储、调度、网络连接起来。

过去，KV Cache 是某个请求、某个实例、某张 GPU 上的临时状态。  现在，它开始变成集群级资源。

- 它可以被复用

- 可以被迁移

- 可以被放到 DRAM/SSD

- 可以跨实例共享

- 可以由 Master 管元数据

- 可以由 Client 贡献存储

- 可以由 TE 通过 RDMA、多 NIC、拓扑感知路径高速搬运

但这也意味着：

- KV Cache 也会制造网络流量

- 也会形成热点

- 也会导致 Incast

- 也会触发 ECN/CNP/PFC

- 也会影响 TTFT、TPOT、P99

![[Mooncake 到底在推理集群里干什么？算力网工必须看懂的 KV Cache-12.png]]

> Mooncake 的主线不是“多一个缓存”，而是“让 KV Cache 变成可管理、可复用、可迁移的集群资源”。

> Master 管元数据，Client 参与存储和访问，Transfer Engine 负责真正的数据搬运。

> 对算力网工来说，Mooncake 的重点不是会不会安装，而是能不能看懂 KV Cache 从哪里来、到哪里去、走哪张网卡、压到哪条队列。

未来推理集群的网络瓶颈，可能不再只是 NCCL。  也不再只是端口带宽。

它可能来自一个被很多 Decode 同时访问的 KV Owner。  来自一次跨 Rack 的 KV 迁移。  来自一次错误的 Router 选择。  来自 Scheduler 同一时间窗口释放的一批长上下文请求。  来自 TE 在某条 Rail 上打出的短时突刺。

这就是 Mooncake 给算力网工上的课：

**不要只看包。**  **要看状态。**  **不要只看链路。**  **要看 KV Cache 的生命周期。**

当你能把 KV object、Client、TE、RDMA NIC、交换机队列、TTFT/P99 串成一条证据链时，你才真正看懂了 Mooncake。

-
-
-
-
- 

```
# 技术参考
``````
## 本文关于 Mooncake 的总体定位、KV cache-centric disaggregated architecture、Prefill/Decode 分离、CPU/DRAM/SSD 构建 disaggregated KV cache pool 的描述，参考 Mooncake 官方 README 和 Mooncake 论文摘要。
``````
## 本文关于 Transfer Engine 的 Segment、BatchTransfer、zero-copy、多协议传输、RDMA、NVMe-oF、拓扑感知路径选择、多 NIC 与故障换路的描述，参考 Mooncake Transfer Engine 官方设计文档。
``````
## 本文关于 Mooncake Store 的 Master Service、Client、对象级 API、多副本、Client-to-Client 数据传输、DRAM/SSD 分层、Master 不承载实际数据面的描述，参考 Mooncake Store 官方设计文档。
``````
## 本文关于 vLLM 通过 MooncakeConnector 用 Transfer Engine 在 GPU 之间移动 KV cache，以及通过 Mooncake Store 构建分布式 KV cache pool 的描述，参考 vLLM 官方博客。
```

## 

- - -

👉 AI 训练网络全路径拆解 → **私信：****AI网络**

👉 AI 推理网络全路径拆解 → **私信：****推理**

**👉 AI 算力网络架构系统（真机实验环境） → 私信：系统**

**👉** AI 算力网络架构专家（真机实验环境） **→ 私信：专家**

**👉** AI 网络架构工程指南手册 **→ **私信：******工程指南**

👉 日常工作1对1答疑 → **私信：****答疑**

- - -

> **如果你也在做 AI 集群架构**  
> 
> **欢迎关注「算力网络架构手记」**  
> 
> **长期拆解真实算力网络问题**
