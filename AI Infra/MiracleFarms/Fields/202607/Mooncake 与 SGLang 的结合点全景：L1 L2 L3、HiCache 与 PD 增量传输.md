---
title: "Mooncake 与 SGLang 的结合点全景：L1/L2/L3、HiCache 与 PD 增量传输"
source: "Fields"
category: "AI Infra"
group: "MiracleFarms"
url: "https://miraclefarms.github.io/notes/2026/07/17/sglang-mooncake-integration-panorama/"
published: 2026-07-17T11:00:00+08:00
saved: 2026-09-30T14:22:30+08:00
folo_key: "mf-fields::https://miraclefarms.github.io/notes/2026/07/17/sglang-mooncake-integration-panorama/"
tags:
  - "folo"
  - "AI_Infra"
  - "Fields"
---

# Mooncake 与 SGLang 的结合点全景：L1/L2/L3、HiCache 与 PD 增量传输

> [!info] Fields · AI Infra · 2026-07-17 11:00 · [原文](https://miraclefarms.github.io/notes/2026/07/17/sglang-mooncake-integration-panorama/)

![[Mooncake 与 SGLang 的结合点全景：L1 L2 L3、HiCach-01.png]]

> **版本声明**：本文分析基于 SGLang commit `7355e0cb`（2026-07-17）与 Mooncake commit `596363e3`（2026-07-17）；除非特别说明，以下描述均基于此版本。

昨天梳理 Mooncake 与 vLLM 的结合点时，结论是“两个正式席位 + 三条间接通道”：Mooncake 通过 connector 进入 PD 数据面和共享 KV 池，vLLM 自己掌握 L1/L2 的缓存状态与调度决策[\[14\]](https://miraclefarms.github.io/notes/2026/07/16/vllm-mooncake-integration-panorama/)。把同一套问题换到 SGLang，地图的画法明显不同。SGLang 从 HiCache 开始就把 GPU、host、distributed storage 明确定义为 L1/L2/L3，HiRadixTree、scheduler、cache controller 和 storage backend 共同执行一条分层命中链。Mooncake 不再只是调度器外部的 connector，而是进入了这条链的正式 L3 插槽。

这个“进入”需要准确理解。**Mooncake 在 SGLang 里接得更深，但没有接管 HiCache**：L1 的显存分配与 Radix 前缀状态、L2 的容量与淘汰、何时预取、等多久、写回哪些 page，仍由 SGLang 决定；Mooncake 提供 L3 Store、RDMA 零拷贝和 PD 点对点传输。最新主线还把 L3 实际命中长度反向带回 L2 预留，把 decode 已有的 L1/L2/L3 前缀反向带回 prefill 的发送范围。数据面第一次深度参与了调度链，却没有越过策略所有权的边界。

## 一、一张接入地图：一个 L3 backend，一套共享 Transfer Engine

SGLang 里与 Mooncake 有关的路径可以收敛成两个入口。第一个入口是 `--hicache-storage-backend mooncake`：`HiCacheStorage` 的 backend factory 创建 `MooncakeStore`，把它放在 L3；第二个入口是默认的 PD transfer backend `mooncake`：prefill 与 decode 注册 GPU buffer，通过 Mooncake Transfer Engine 做单边写。两条路径并非各起一套传输 runtime。只要 device name、metadata protocol 和 RDMA 配置一致，Store 会复用进程内已经初始化的 shared `MooncakeTransferEngine`[\[4\]](https://github.com/sgl-project/sglang/blob/7355e0cb875fe1240ab72642e1427e56aa63828a/python/sglang/srt/mem_cache/storage/mooncake_store/mooncake_store.py)[\[5\]](https://github.com/sgl-project/sglang/blob/7355e0cb875fe1240ab72642e1427e56aa63828a/python/sglang/srt/distributed/device_communicators/mooncake_transfer_engine.py)。

| 位置                     | SGLang 掌握的语义                     | Mooncake 提供的能力                            | 是否可替换                                |
| ---------------------- | -------------------------------- | ----------------------------------------- | ------------------------------------ |
| L1：GPU HBM             | Radix/HiRadix 命中、page 分配、引用与淘汰   | PD 时注册 GPU KV buffer；L3 数据最终经 L2 恢复到 L1   | L1 无 Mooncake backend 概念             |
| L2：host memory         | 容量、布局、分配、淘汰、L1↔L2 copy 与 overlap | 注册 host pool，作为 RDMA get/put 的零拷贝源和目的地    | L2 数据移动由 SGLang I/O backend 决定       |
| L3：distributed storage | 命中阈值、等待策略、write policy、连续前缀语义    | Mooncake Store 的对象寻址、分布式 DRAM/SSD、并行 RDMA | 是，和 file、3FS、NIXL、AIBrix 等同属 backend |
| PD：P→D                 | 请求握手、KV layout、传输范围、完成状态         | Transfer Engine 的内存注册与 batch write        | 是，PD backend 可切换                     |

这张表里最值得留意的是 L2。Mooncake 没有成为 L2 cache manager，却必须认识 L2 的物理内存，因为 L3 的 read/write 都以 L2 buffer 为落点。于是它在 SGLang 里形成一种少见的角色：**逻辑上属于 L3，物理上贴着 L2，传输上还与 PD 的 GPU 直写共用底座**。

![[Mooncake 与 SGLang 的结合点全景：L1 L2 L3、HiCach-02.png]]

*图 1：HiCache 的控制流由 SGLang 串起来：先匹配 L1/L2，再查询并预取 L3，随后把可复用前缀恢复到 L1，计算新增 suffix，最后异步写回。Mooncake 出现在 L3 query/get/put 与 RDMA 数据路径上，prefetch 截止条件和层间状态仍由 HiRadixTree 与 cache controller 管理。来源：LMSYS HiCache 官方博客。*

## 二、请求主路径：先问本地树，再问远端 Store

HiCache 不是三套彼此独立的 cache。它把一个请求的可复用前缀切成连续的三段：`L1 prefix + L2 prefix + L3 prefix`，剩余部分才是要计算的 suffix。HiRadixTree 精确记录 L1/L2 节点及其本地地址；为了避免全局元数据同步开销，它不持续镜像 L3 的对象位置，而是在请求到来时实时查询 backend[\[11\]](https://github.com/kvcache-ai/Mooncake/blob/596363e34f2a91c9b29436716998c3e10309f75a/docs/source/design/hicache-design.md)。

一条请求因此要经历四步。第一步，scheduler 沿 HiRadixTree 做 local match，直接拿到 L1 的 `device_indices` 与 L2 的 `host_indices`。第二步，对本地连续前缀之后的 page key 发起 L3 query；Mooncake 只回答哪些对象存在，连续命中在第一处 miss 截断。第三步，达到阈值的远端命中被异步读进 L2，再由 SGLang 的 loading-back 路径搬到 L1。第四步，模型只计算未命中的 suffix，新 KV 按 `write_through`、`write_through_selective` 或 `write_back` 策略进入下层[\[2\]](https://github.com/sgl-project/sglang/blob/7355e0cb875fe1240ab72642e1427e56aa63828a/python/sglang/srt/mem_cache/hiradix_cache.py)[\[3\]](https://github.com/sgl-project/sglang/blob/7355e0cb875fe1240ab72642e1427e56aa63828a/python/sglang/srt/managers/cache_controller.py)。

默认 256 token 的 prefetch threshold 是这条链上的成本闸门。L3 命中太短，网络 I/O 与调度开销可能高于重算；超过阈值才进入预取。进入之后还有 `best_effort`、`wait_complete`、`timeout` 三种停止策略，`timeout` 按基础时间加每 1024 token 的增量时间计算。Mooncake 决定一次 RDMA 是否成功、数据从哪个 Store 节点读，SGLang 决定这次读取值不值得等。

## 三、L1 接入：Mooncake 不管理显存，但 PD 和 L3 都以它为终点

L1 是 GPU 上的 KV pool，也是 RadixAttention 的命中执行层。page 的分配、树节点锁定、引用计数和 eviction 都在 SGLang 内部；`MooncakeStore` 不持有 device allocator，也不会修改 HiRadixTree。L3 命中最终能被 attention kernel 使用，必须先完成 `L3 → L2` 预取，再完成 `L2 → L1` loading-back。只要这两段恢复尚未完成，scheduler 就不能把这段前缀当成 device hit。

Mooncake 与 L1 的直接物理接触发生在另一条路径：PD 分离。Mooncake connector 会注册 prefill 和 decode 两侧的 GPU KV buffer，prefill 用 `batch_transfer_sync_write` 向 decode 暴露的目标地址做 P→D 单边写[\[6\]](https://github.com/sgl-project/sglang/blob/7355e0cb875fe1240ab72642e1427e56aa63828a/python/sglang/srt/disaggregation/mooncake/conn.py)。这里 Mooncake 能读写 L1 的物理地址，但仍不知道哪个请求该保留、哪个 page 该淘汰。**能访问显存和拥有显存缓存策略，是两件不同的事。**

这个边界对排障很重要。L3 query 有命中但 TTFT 没下降，要分别检查 Mooncake Store 读取、L2 落点分配、L2→L1 copy 以及 prefetch deadline；PD 传输完成但 decode 仍等待，则要检查目标 page 的 layout、receiver 状态和恢复提交。把所有问题统称为“Mooncake 慢”，会把四段不同的关键路径混在一起。

## 四、L2 接入：SGLang 管缓存，Mooncake 把 host pool 变成零拷贝 RDMA 区

L2 是两套系统结合最紧密的物理边界。SGLang 创建 host KV pool，决定 `hicache_ratio` 或显式容量，选择 `page_first` / `page_first_direct` 布局并管理 free slots；Mooncake 将这片 host memory 注册到 Transfer Engine，`batch_get` 直接把远端对象读进目标 pointer，`batch_put` 直接从同一 pointer 发出，不经过额外 staging copy[\[4\]](https://github.com/sgl-project/sglang/blob/7355e0cb875fe1240ab72642e1427e56aa63828a/python/sglang/srt/mem_cache/storage/mooncake_store/mooncake_store.py)。

embedded 模式下，每个 SGLang rank 可以把一部分 host memory 贡献给 Mooncake 分布式 segment；standalone 模式下，外部 Mooncake client 持有 Store 容量，SGLang 仍需本地 host buffer 作为 L3 I/O 落点。两种模式改变的是 L3 池的所有权，不改变 HiCache 的 L2 管理权。Mooncake Store 内部还可以启用 SSD offload，但从 SGLang 看仍然只有一个 L3 backend：Store 内部的 DRAM→SSD 冷热迁移不会凭空变成 HiCache 的 L4。

![[Mooncake 与 SGLang 的结合点全景：L1 L2 L3、HiCach-03.png]]

*图 2：`page_first_direct` 在 page 内按 layer 聚合连续 token，使 L2↔L1 能按 page-layer 合并传输，同时保留一页作为单个 L3 对象的连续物理空间。它连接了 GPU 的 layer-first 计算习惯与 Mooncake 偏好的批量零拷贝 I/O。来源：Mooncake x SGLang HiCache 设计文档。*

当前主线还有一条容易踩坑的配置规则：Mooncake backend 遇到 `layer_first` host layout 时，SGLang 会把它规范化为 `page_first` 或 `page_first_direct`。原因不在 API 偏好，而在对象边界——L3 要把一页 KV 当成连续对象做批量传输；若一页被拆散到所有 layer 的大数组里，就会把一次大 I/O 退化成许多小 I/O。HiCache 把 L3 batch 上限设为 128 pages，也是为了在并行吞吐和 `best_effort` / `timeout` 可中止性之间留出粒度[\[1\]](https://github.com/sgl-project/sglang/blob/7355e0cb875fe1240ab72642e1427e56aa63828a/docs_new/docs/advanced_features/hicache_design.mdx)。

## 五、L3 接入：Mooncake 的正式席位，策略仍在 HiCache

L3 是 Mooncake 在 SGLang 的正式主场。`MooncakeStore` 把 HiCache 的 page key 映射为分布式对象，query 返回存在性，get/put 执行批量数据搬运；只上传 Store 中不存在的对象，跨实例可共享相同前缀。多 TP 场景通过 `all_reduce(min)` 对命中页数和最终完成页数取一致值，任何一个 rank 少命中一页，整组请求都在同一连续边界截断，避免各 rank 对 prefix 长度产生分歧[\[11\]](https://github.com/kvcache-ai/Mooncake/blob/596363e34f2a91c9b29436716998c3e10309f75a/docs/source/design/hicache-design.md)。

这里可以清楚区分机制与策略：

- Mooncake 负责对象是否存在、对象位于哪个 segment、RDMA 读写是否完成，以及 Store 内部的副本和 SSD 分层。

- HiCache 负责查询哪段连续 prefix、命中多少 token 才值得取、host slots 是否足够、等待多久、取回多少已完成 page，以及何时把新 KV 写回。

这也是 SGLang 比昨天那张 vLLM 地图“接得更深”的准确含义。vLLM connector 通常只把外部命中长度交给 scheduler；SGLang 的 L3 命中会进入 host allocation、prefetch deadline、loading-back 和 write policy。Mooncake 得到更多框架语义配合，但 backend factory 仍允许 file、3FS、NIXL、AIBrix 或动态 backend 替代它[\[7\]](https://github.com/sgl-project/sglang/blob/7355e0cb875fe1240ab72642e1427e56aa63828a/python/sglang/srt/mem_cache/storage/backend_factory.py)。

## 六、最新主线变化：L3 先报实账，L2 再按命中页分配

截至本文对应的主线，最值得写进接入地图的是 2026 年 7 月 17 日合入的 PR #19320。旧路径在查询 L3 前会按“潜在可命中长度”预分配 L2 slots；真实 L3 命中更短或直接 miss 时，多占的 host pages 要随后释放。并发长上下文请求会因此制造瞬时 L2 压力，甚至让本来能服务的请求因过量预留而失败。

新路径把顺序倒了过来：先向 Mooncake query 真实连续命中页数，再只为这些页分配 L2；如果 host memory 仍不足，就把命中裁到 page 对齐的可分配前缀，裁完低于 prefetch threshold 则放弃本次远端恢复[\[8\]](https://github.com/sgl-project/sglang/pull/19320)。这不是简单的 allocation 优化，它改变了 L3 与 L2 的控制关系：**远端 Store 的实账决定本地 host 的预留，而不是本地先按上限押注。**

这条改动也给出一个更一般的工程判断。分层缓存里容量管理不能只盯静态大小；当低层命中带不确定性，预留时机本身就是容量。L3 query 越准确、返回越快，L2 的可用容量越接近物理容量；反过来，过早为“可能命中”锁住 host pages，会把网络不确定性放大成内存碎片和 admission failure。

## 七、PD 接入：decode 的 L1/L2/L3 命中反向削减 P→D 传输

PD 分离与 HiCache 原本是两条正交路径：前者把本次请求算出的 KV 从 prefill 交给 decode，后者复用历史请求留下的 KV。PR #26227 把两条路径真正接在了一起。decode 节点在接受请求时，不再只报告 L1 命中，而是匹配本地 L1、L2 和远端 L3 的连续前缀，得到完整的 `decode_prefix_len`；若有 L3 命中，先预取到 L2，再把 L2 段恢复到 L1。prefill 收到这个长度后，把 `start_send_idx` 前移到 decode 已有前缀的末端，只发送剩余 delta[\[9\]](https://github.com/sgl-project/sglang/pull/26227)[\[10\]](https://github.com/sgl-project/sglang/blob/7355e0cb875fe1240ab72642e1427e56aa63828a/python/sglang/srt/disaggregation/decode_hicache_mixin.py)。

用一个例子看最清楚：请求有 8K token 前缀，decode 的 L1 命中 2K、L2 命中 2K、Mooncake L3 再命中 3K，那么 decode 先把后 5K 恢复到 GPU，向 prefill 报告已有 7K；prefill 只需传最后 1K 的 KV。若只看 L1，这次 P→D 会多传 5K。**L3 命中在这里不只省 prefill 计算，还直接削减 PD 网络流量。**

完成条件同样由 SGLang 把关。decode 侧只有在 L3→L2 prefetch drain、L2→L1 restore 成功、恢复后的 device indices 覆盖预期 prefix 后，才把请求标为 ready；任一环节失败就回退，不会把“Mooncake get 已返回”误判为“attention 已可用”[\[10\]](https://github.com/sgl-project/sglang/blob/7355e0cb875fe1240ab72642e1427e56aa63828a/python/sglang/srt/disaggregation/decode_hicache_mixin.py)。这正是调度器内建分层状态的价值：数据面完成只是中间状态，能够安全计算才是最终状态。

## 八、对象组语义：一页 KV 已经不再只有 K 和 V

传统 MHA 模型里，一页 KV 通常可以拆成 K/V 两个物理对象；新模型的缓存形态已经复杂得多。SWA 混合注意力、Mamba state、DSA indexer、DeepSeek V4 side pools、speculative decoding 的 draft KV 都可能附着在同一段逻辑 prefix 上。若这些对象分别存取、分别淘汰，就可能出现主 KV 还在、sidecar 已丢，或部分组件写成功的生命周期裂缝。

2026 年 6 月合入的 PR #26574 在 SGLang 的 `MooncakeStore` 中加入 group semantics：属于同一逻辑 HiCache page 的物理对象使用 `sglang-hicache:{logical_key}` 生成共同 group id，K/V 与各类 sidecar 共享对象组生命周期[\[12\]](https://github.com/sgl-project/sglang/pull/26574)。代码还保留了对旧版 Mooncake Python binding 的兼容回退，只有 `ReplicateConfig` 支持 `group_ids` 时才启用。

这个细节说明 L3 backend 接口正在从“读写一块 bytes”走向“保存一组有共同生命周期的推理状态”。SGLang 仍定义哪些组件属于同一个 prefix page，Mooncake Store 开始承接跨对象的复制、淘汰和生命周期 hint。两边的边界仍清楚，但结合面已经从数据地址扩展到了对象语义。

## 九、性能边界：L3 的收益来自跨过本地容量，不来自层级数量

Mooncake 官方 HiCache 多轮 benchmark 很适合说明 L3 的收益边界。H800 测试使用 Qwen3-235B-A22B、80 个并发客户端、10 轮对话和 760GB Mooncake pool；图中到第 10 轮时，Mooncake L3 方案命中率约 90%、平均 TTFT 约 0.8 秒，本地 L2 方案的命中率已经降到约 8%、TTFT 升到约 4.2 秒，纯 GPU 约 4.5 秒[\[13\]](https://github.com/kvcache-ai/Mooncake/blob/596363e34f2a91c9b29436716998c3e10309f75a/docs/source/performance/sglang/sglang-hicache-benchmark-results-v1.md)。

![[Mooncake 与 SGLang 的结合点全景：L1 L2 L3、HiCach-04.png]]

*图 3：随着轮次增加，working set 越过本地 HBM/host 容量，GPU 与 L2 命中率持续下降；共享 Mooncake pool 仍能保留跨轮前缀。图中数据只代表文档给出的特定模型、硬件与并发配置，不能直接外推到其他部署。来源：Mooncake SGLang HiCache benchmark v1。*

这组数据不能读成“加 L3 一定更快”。单轮、短上下文、低复用，或者 working set 长期留在 L1/L2 内时，Mooncake query、RDMA 与同步就是额外开销；多轮会话、agentic trajectory、跨 replica 共享前缀且本地 host 容量开始见顶时，分布式 L3 才把容量优势转成命中率。LMSYS 汇总的测试给出 HiCache 最高 6 倍吞吐、TTFT 最多下降 80%，蚂蚁集团使用 Mooncake + HiCache 的生产评估里 cache hit 相对重算降低 84% TTFT[\[15\]](https://www.lmsys.org/blog/2025-09-10-sglang-hicache/)；这些数字都应与各自负载条件一起引用。

部署前可以用四个问题划边界：可复用前缀是否会跨实例；working set 是否经常越过本机 host 容量；单次连续命中是否大到足以覆盖网络成本；L2↔L1 带宽是否跟得上 L3→L2。最后一个问题尤其容易漏掉——远端 RDMA 再快，host 到 GPU 的恢复若成为瓶颈，TTFT 仍会卡在 L2/L1 边界。

## 十、结论

Mooncake 与 SGLang 的 L1/L2/L3 接入可以压缩成三句话。L1 由 SGLang 完整管理，Mooncake 只在 PD 时直接读写注册的 GPU buffer，或通过 L3→L2→L1 链路间接抵达；L2 仍是 SGLang 的 host cache，Mooncake 把它注册为零拷贝 RDMA 区，真实 L3 命中还会决定 L2 精确预留；L3 是 Mooncake 的正式 backend 席位，Store 提供共享对象池和传输机制，HiCache 保留命中阈值、prefetch deadline、write policy 与连续前缀规则。

和昨天的 vLLM 地图相比，SGLang 的区别不在于多了一个 Mooncake 组件，而在于框架对分层状态知道得更多。HiCache 可以把 L3 命中带回 admission，把 L1/L2/L3 的 decode prefix 带回 PD 发送范围，再用对象组覆盖 hybrid cache 的共同生命周期。这个结构让 Mooncake 的数据面能力更充分地参与运行时决策，也把耦合控制在 backend 与 Transfer Engine 接口内。

当前主线仍有三个值得继续追踪的问题。第一，decode-side HiCache + incremental PD 在异构 TP、混合缓存组件下的收益和失败回退是否稳定；第二，group semantics 会不会从兼容性可选项变成 hybrid model 的强制契约；第三，Mooncake TENT 的 priority、deadline 与 traffic class 何时向上接入 HiCache 的 foreground get、background prefetch 和 write-back——到那时，Mooncake 与 SGLang 的结合点会从“存储与传输机制”再向“跨层 I/O 调度”前进一步，但缓存策略归谁，依然会是判断架构边界的第一问。

- - -

## 参考资料

\[1\] [SGLang HiCache Design（commit 7355e0cb）](https://github.com/sgl-project/sglang/blob/7355e0cb875fe1240ab72642e1427e56aa63828a/docs_new/docs/advanced_features/hicache_design.mdx)

\[2\] [SGLang hiradix\_cache.py（commit 7355e0cb）](https://github.com/sgl-project/sglang/blob/7355e0cb875fe1240ab72642e1427e56aa63828a/python/sglang/srt/mem_cache/hiradix_cache.py)

\[3\] [SGLang cache\_controller.py（commit 7355e0cb）](https://github.com/sgl-project/sglang/blob/7355e0cb875fe1240ab72642e1427e56aa63828a/python/sglang/srt/managers/cache_controller.py)

\[4\] [SGLang MooncakeStore（commit 7355e0cb）](https://github.com/sgl-project/sglang/blob/7355e0cb875fe1240ab72642e1427e56aa63828a/python/sglang/srt/mem_cache/storage/mooncake_store/mooncake_store.py)

\[5\] [SGLang Shared Mooncake Transfer Engine（commit 7355e0cb）](https://github.com/sgl-project/sglang/blob/7355e0cb875fe1240ab72642e1427e56aa63828a/python/sglang/srt/distributed/device_communicators/mooncake_transfer_engine.py)

\[6\] [SGLang Mooncake PD Connector（commit 7355e0cb）](https://github.com/sgl-project/sglang/blob/7355e0cb875fe1240ab72642e1427e56aa63828a/python/sglang/srt/disaggregation/mooncake/conn.py)

\[7\] [SGLang HiCache Storage Backend Factory（commit 7355e0cb）](https://github.com/sgl-project/sglang/blob/7355e0cb875fe1240ab72642e1427e56aa63828a/python/sglang/srt/mem_cache/storage/backend_factory.py)

\[8\] [SGLang PR #19320：L3 miss 时优化 L2 内存分配](https://github.com/sgl-project/sglang/pull/19320)

\[9\] [SGLang PR #26227：decode-side HiCache prefetch 与 PD 增量传输](https://github.com/sgl-project/sglang/pull/26227)

\[10\] [SGLang decode\_hicache\_mixin.py（commit 7355e0cb）](https://github.com/sgl-project/sglang/blob/7355e0cb875fe1240ab72642e1427e56aa63828a/python/sglang/srt/disaggregation/decode_hicache_mixin.py)

\[11\] [Mooncake x SGLang HiCache System Design（commit 596363e3）](https://github.com/kvcache-ai/Mooncake/blob/596363e34f2a91c9b29436716998c3e10309f75a/docs/source/design/hicache-design.md)

\[12\] [SGLang PR #26574：Mooncake group semantics](https://github.com/sgl-project/sglang/pull/26574)

\[13\] [Mooncake x SGLang HiCache Benchmark Results v1（commit 596363e3）](https://github.com/kvcache-ai/Mooncake/blob/596363e34f2a91c9b29436716998c3e10309f75a/docs/source/performance/sglang/sglang-hicache-benchmark-results-v1.md)

\[14\] [前期调研：Mooncake 与 vLLM 的结合点全景](https://miraclefarms.github.io/notes/2026/07/16/vllm-mooncake-integration-panorama/)

\[15\] [LMSYS：SGLang HiCache—Supercharging Long-Context LLM Inference](https://www.lmsys.org/blog/2025-09-10-sglang-hicache/)

### 版本对齐信息

| 来源               | 版本                                  | 说明                                                            |
| ---------------- | ----------------------------------- | ------------------------------------------------------------- |
| SGLang 主线源码      | `main@7355e0cb`（2026-07-17）         | HiCache、Mooncake Store、PD connector 与 server args 均基于此 commit |
| Mooncake 主线源码    | `main@596363e3`（2026-07-17）         | HiCache 设计与 benchmark 文档基于此 commit                            |
| SGLang PR #19320 | merged 2026-07-17（merge `7cd55c68`） | L3 query 后按真实命中页延迟分配 L2                                       |
| SGLang PR #26227 | merged 2026-06-02（merge `3e993f61`） | decode-side L1/L2/L3 恢复与 PD incremental transfer              |
| SGLang PR #26574 | merged 2026-06-23（merge `62f7ffc4`） | Mooncake object group semantics                               |
