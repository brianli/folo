---
title: "DeepInfer 读后：DeepSeek V4.1 Flash 如何组织计算、缓存与故障域"
source: "Readings"
category: "AI Infra"
group: "MiracleFarms"
url: "https://miraclefarms.github.io/notes/2026/10/20/deepinfer-v41-flash-serving/"
published: 2026-10-20T22:30:00+08:00
saved: 2026-10-21T21:57:00+08:00
folo_key: "mf-readings::https://miraclefarms.github.io/notes/2026/10/20/deepinfer-v41-flash-serving/"
tags:
  - "folo"
  - "AI_Infra"
  - "Readings"
---

# DeepInfer 读后：DeepSeek V4.1 Flash 如何组织计算、缓存与故障域

> [!info] Readings · AI Infra · 2026-10-20 22:30 · [原文](https://miraclefarms.github.io/notes/2026/10/20/deepinfer-v41-flash-serving/)

Decode 当前不受 KV 容量约束，因此不用 Hi-Sparse；前缀可以全局命中，因此不做亲和路由。从这些出人意料的部署取舍出发，追问它们如何成立，又把成本转移到了哪里。

![[DeepInfer 读后：DeepSeek V4.1 Flash 如何组织计算、-01.png]]

> **版本声明**：源码核对基于 DeepInfer `774ce244b8`（2026-10-20）、DeepGEMM-Ascend `3a35326271`（2026-10-14）与 DeepSeek-V4.1-Flash-1017 配置；截至 2026-10-20，前两个仓库的主分支与上述提交一致。本文是公开材料与源码阅读，未执行集群性能复现。

> “Decode is not bound by cache capacity, so Hi-Sparse is not used either.”[\[2\]](https://github.com/deepseek-ai/DeepInfer/blob/774ce244b87cc0eb8d4b94ab9f5de2ab22049f01/doc/inference_deployment_qa.md)
> 
> 译文：Decode 未受缓存容量约束，因此也未使用 Hi-Sparse。

读到这里，很容易停一下。讨论长上下文推理，我们常常从显存还能容纳多少 KV 开始，随后才谈压缩、卸载和搬运。DeepSeek 却在部署 Q&A 里给出了一个干脆的回答：在它当前的 Decode 工作点上，缓存容量没有成为约束，因此没有启用 KV offload，也没有使用 Hi-Sparse。

《DeepSeek V4.1 Flash 大规模推理部署实践》以及刚刚开源的 DeepInfer 文档里，还有不少这样的表述：任意 Prefill 引擎都能命中任意前缀，因此不做前缀亲和路由；全局 KV 可以保存数天，SWA 却只留在生成机器的内存里，丢了就重放；同一个模型，Prefill 尽量用小 EP，Decode 却使用 64—128 卡的 Wide-EP。[\[1\]](https://zhuanlan.zhihu.com/p/2095560295930508953)[\[2\]](https://github.com/deepseek-ai/DeepInfer/blob/774ce244b87cc0eb8d4b94ab9f5de2ab22049f01/doc/inference_deployment_qa.md) 把这些短句放在一起，许多熟悉的部署经验似乎都需要重新问一遍。

**这些选择成立的条件，藏在模型结构、流量形状和硬件组织的相互配合里。** 当全模型的全局 KV 已经压到每个原始 token 890 字节，Decode 是否需要卸载，就要重新看它的实际并发和上下文分布；当历史状态能从共享存储取回，路由也获得了摆脱缓存位置约束的余地。省下的计算和容量，又会把带宽、调度与故障恢复推到前面。脱离这些条件，单独搬走一项“最佳实践”，结果可能完全不同。

前期调研文章《DeepSeek-V4.1-Flash 读后：把 KV cache 压到 890 字节之后，瓶颈换了个位置》拆过模型的压缩机制。[\[24\]](https://miraclefarms.github.io/notes/2026/09/10/deepseek-v41-flash-kv-cache-compression/) 这次沿着官方原话继续往下读：一条请求如何穿过这些模块，状态在哪里产生、保存和恢复，收益依赖什么，代价又落到谁身上。下文会分别交代官方生产描述、公开代码和本文估算，沿调用链核对实现，也说明哪些问题还缺少公开证据。

## 一、任意 P 命中任意前缀，路由就能只看计算吗？

> “由于任何引擎都能命中任何前缀，不做前缀亲和路由”——官方部署文章，Prefill Load Balancer，原文节选。[\[1\]](https://zhuanlan.zhihu.com/p/2095560295930508953)

字面上，这句话回答的是 P 引擎怎样选任务。过去，前缀驻留在哪台机器，常常就把下一轮请求拴在哪台机器；DeepSeek 把全局 KV 放到独立的 3FS 存储集群，任意 P 都能查询并加载，因此可以按剩余计算预算拉取任务。[\[2\]](https://github.com/deepseek-ai/DeepInfer/blob/774ce244b87cc0eb8d4b94ab9f5de2ab22049f01/doc/inference_deployment_qa.md)

![[DeepInfer 读后：DeepSeek V4.1 Flash 如何组织计算、-02.png]]

*图 1：蓝色数据路径把外部 KV 存储、P 和 D 接起来，黑色路径负责请求与控制。任意 P 可加载前缀，依赖的是这条共享数据路径；它没有消除加载本身。来源：DeepInfer 部署 Q&A 官方原图。*

设一条请求已有 128K 历史，本轮新增 512 token。Entrypoint 先用哈希链匹配元数据，得到可复用长度，再把请求交给计算预算足够的 P；假如 SWA 快照在上一轮 D 的主机内存里，新 P 还要向那里取回快照。**执行位置可以变化，状态位置仍须被目录找到。** 命中长度必须在路由前确定，官方也明确承认，这部分开销无法靠后续计算掩盖。

用 128K＝131,072 token、890 B/token 估算，完整历史全局 KV 约 116.65 MB，即 111.25 MiB；512 token 的增量只有约 456 KB。即便忽略尾部保留区，前者仍是后者的 256 倍。新算的 token 很少，历史搬运并不会按相同比例缩小。

这也回应了此前缓存亲和路由的经验：聚合式服务里，会话留在原卡上可能省掉重取与预取占用；共享存储架构则主动把“到处可取”作为基础能力。两种选择对应不同的状态布局。对架构师而言，取消亲和的前提应包括缓存可达性、加载延迟和网络余量，单看“命中率相同”还不够。公开仓库没有完整的生产路由器，无法据离线引擎重现这一决策的全部代价。

## 二、Decoder 的 KV 来自 Encoder，究竟省掉了什么？

> “CED 架构下 decoder 层的 Block KVCache 由 encoder 末层 hidden state 投影得到”——官方部署文章，SWA Bounded Replay，原文节选。[\[1\]](https://zhuanlan.zhihu.com/p/2095560295930508953)

这里的 Encoder/Decoder 是模型内部的因果编码器与解码器，和部署侧 Prefill/Decode 是两个维度。**Prefill 会经过 Encoder，也要为 Decoder 准备状态；Decode 每一步仍要走完模型主干。** 把二者混成同一件事，会误算计算量和 P→D 传输量。

模型主干是 40 层，前 20 层为因果 Encoder，后 20 层为 Decoder。以下从第 1 层开始编号；配置里的 source id 从 0 开始。[\[3\]](https://huggingface.co/deepseek-ai/DeepSeek-V4.1-Flash/blob/314113f8a7dadb388215396ba5fbee2c610a04d6/DeepSeek_V41_Tech_Report.pdf)[\[4\]](https://huggingface.co/deepseek-ai/DeepSeek-V4.1-Flash-1017/blob/1bae0e1088993b9b82de08f59de5be5a34e42170/config.json)

| 层范围   | 全局 KV 来源及压缩                   | 索引选择                          |
| ----- | ----------------------------- | ----------------------------- |
| 1—2   | 纯 SWA，无全局 KV                  | 无全局索引                         |
| 3—8   | 第 3 层生成一份 C2，其后五层复用           | 复用第 3 层 Top-K                 |
| 9—14  | 第 9 层生成一份 C2，其后五层复用           | 复用第 9 层 Top-K                 |
| 15—20 | 第 15 层生成一份 C2，其后五层复用          | 复用第 15 层 Top-K                |
| 21—40 | 第 21 层路径从 Encoder 末端表示生成一份 C1 | 21、25、29、33、37 层产出选择，各供随后三层复用 |

![[DeepInfer 读后：DeepSeek V4.1 Flash 如何组织计算、-03.png]]

*图 2：Encoder 有三组 C2，Decoder 共用一份 C1。第一层 Decoder 同时建立后续 Reindex 的候选池；这张图描述模型结构，不能把左右两半直接读成部署侧 P 与 D。来源：V4.1 技术报告 Figure 3。*

代码里的依赖链可以顺着读：`deepseek.py` 逐层执行，在 consumer 边界切换完整与截断后的 metadata；`decoder_layer.py` 保存边界处经 attention norm 处理的完整隐藏表示；`attn.py::_store_yoco_full_kv()` 对完整表示做共享 KV 投影与写入，后续主计算只处理保留的尾段。Decoder 全局 KV 因而不必等待每一层都对完整 prompt 做完 attention 和 MoE。[\[5\]](https://github.com/deepseek-ai/DeepInfer/blob/774ce244b87cc0eb8d4b94ab9f5de2ab22049f01/deepinfer/model_executor/models/deepseek.py)[\[6\]](https://github.com/deepseek-ai/DeepInfer/blob/774ce244b87cc0eb8d4b94ab9f5de2ab22049f01/deepinfer/model_executor/models/decoder_layer.py)[\[7\]](https://github.com/deepseek-ai/DeepInfer/blob/774ce244b87cc0eb8d4b94ab9f5de2ab22049f01/deepinfer/model_executor/models/v4_modules/attn.py)

CSA2 又把“历史向量”与“本次选择”拆开。Indexer K 是持久的检索向量；Top-K 是当前 Query 算出的选择结果。Reindex 保留历史向量，用自己的 indexer Q 重选；Reuse 连选择一起复用。每层仍有自己的 main Q 和输出变换，所以相同 Top-K 不会强制相同 attention 权重。第 21 层还建立最多 2,048×8＝16,384 个候选位置的池，后续 Reindex 在池内选 Top-512，不再各扫一遍全部历史。[\[3\]](https://huggingface.co/deepseek-ai/DeepSeek-V4.1-Flash/blob/314113f8a7dadb388215396ba5fbee2c610a04d6/DeepSeek_V41_Tech_Report.pdf)[\[4\]](https://huggingface.co/deepseek-ai/DeepSeek-V4.1-Flash-1017/blob/1bae0e1088993b9b82de08f59de5be5a34e42170/config.json)[\[8\]](https://github.com/deepseek-ai/DeepInfer/blob/774ce244b87cc0eb8d4b94ab9f5de2ab22049f01/deepinfer/model_executor/models/v4_modules/cross_layer.py)

**共享全局 KV 与共享局部 SWA 输入必须分开核对。** `attn.py` 确有 `yoco_swa_share_hidden` 可选分支，但本次公开 Hugging Face 适配器在 `_GPT_DEFAULTS` 中明确设为 `False`，默认加载路径没有打开它。因此，不能把这条备用分支写成 Flash-1017 的默认行为；默认仍是各层从本层隐藏状态生成 SWA，与报告的逐层局部计算描述一致。[\[9\]](https://github.com/deepseek-ai/DeepInfer/blob/774ce244b87cc0eb8d4b94ab9f5de2ab22049f01/deepinfer/hugging_face/config.py)

对长 prompt，主要收益是 Decoder 不再为所有历史 token 重跑完整层计算。用报告的复杂度口径，工作从约 (NL) 变为 (NL/2+W L/2)。当 (N\\gg W=128) 时接近减半；投影、Encoder 重放和其他开销仍在，短后缀请求不能直接套用“端到端快一倍”。

## 三、890 B/token，应该怎样复算？

> “存储 4 层的 main KV 和 indexer K。”——官方部署文章，Block KVCache，原文节选。[\[1\]](https://zhuanlan.zhihu.com/p/2095560295930508953)

四份缓存的形状已经确定：三份 C2、一份 C1。C2 将相邻两个 token 压成一个记录，C1 不合并 token。压缩也有具体含义：对隐藏表示投影得到 latent 和 score，然后在同一个通道内，对这两个位置做 softmax 加权求和。用说明性记号表示：

\[ u\_t=W\_ux\_t,\\quad s\_t=W\_sx\_t,\\qquad z\_{j,c}=\\sum\_{t\\in{2j,2j+1}} \\operatorname{softmax}*{t}(s*{t,c})u\_{t,c}. \]

这组记号用于解释机制，实际实现还包括归一化、位置处理与量化。`tilelang_ops/compress.py` 的参考函数可直接看到沿 token 维的 softmax 和加权求和；它不是隔一个 token 丢一个，也不是固定平均。[\[10\]](https://github.com/deepseek-ai/DeepInfer/blob/774ce244b87cc0eb8d4b94ab9f5de2ab22049f01/deepinfer/model_executor/layers/tilelang_ops/compress.py)

按 `cache_common.py` 的实际格式，Main KV 是 512 个 FP4 数值，每 16 个数值带一个单字节 scale；Indexer K 是 128 个 FP4 数值，每 32 个带一个单字节 scale。因此：[\[11\]](https://github.com/deepseek-ai/DeepInfer/blob/774ce244b87cc0eb8d4b94ab9f5de2ab22049f01/deepinfer/model_executor/layers/tilelang_ops/cache_common.py)

\[ B\_{\\rm Main}=512/2+512/16=288\\ {\\rm B},\\qquad B\_{\\rm Index}=128/2+128/32=68\\ {\\rm B}. \]

\[ B\_{\\rm global/token} =3\\times\\frac{288+68}{2}+(288+68) =890\\ {\\rm B}. \]

**890 B 已经是全模型每个原始 token 的全局缓存口径，无须再乘 40 层或 K/V 的 2。** 其中 Main 为 720 B，Indexer 为 170 B。128 个原始 token 对应 113,920 B，约 111.25 KiB，与部署文章所说的约 112 KiB block 相符；分页填充和存储协议仍可能带来额外开销。

V4-Flash 的配置使用 C4/C128，V4.1 改成 C2/C1，单个来源层压得更少，却依靠跨层共享与 FP4，把整体全局 KV 降到前代约 1/4。[\[3\]](https://huggingface.co/deepseek-ai/DeepSeek-V4.1-Flash/blob/314113f8a7dadb388215396ba5fbee2c610a04d6/DeepSeek_V41_Tech_Report.pdf)[\[12\]](https://huggingface.co/deepseek-ai/DeepSeek-V4-Flash/blob/60d8d70770c6776ff598c94bb586a859a38244f1/config.json) 这组结果支持的是联合设计，不能拿它反推“C128 因效果失败被删除”：模型、训练与共享拓扑同时变化，缺少隔离变量的因果证据。

对芯片和互联预算，890 B 是很有用的逻辑数据量；对 HBM 容量预算，它只是一项。权重、Engram、SWA、压缩中间态、推测分支、激活和连续解量化缓冲，都要另算。

## 四、SWA 留在生成机器，丢了怎么办？

> “SWA KV 不进持久化缓存，而是留在产生它的那台机器的内存里”——官方部署文章，SWA KVCache，原文节选。[\[1\]](https://zhuanlan.zhihu.com/p/2095560295930508953)

“生成机器”可以是 P，也可以是上一轮 D。逻辑统一、按 key 分片在 Entrypoint 上的目录记录快照在哪台主机；新 P 即使按计算负载被路由到别处，也可以跨机拉取。官方报告这一内存池约 180 秒 TTL、命中率超过 98%，对应活跃会话的分钟级复用。[\[1\]](https://zhuanlan.zhihu.com/p/2095560295930508953)

这里保存的是对齐边界上的快照。前 20 层各保留 128 token 窗口与 6 个回退位置，每条记录 528 B：

\[ 20\\times134\\times528=1,415,040\\ {\\rm B}\\approx1.35\\ {\\rm MiB}. \]

Prefill 只在最后一个 chunk 结束时保存；Decode 跨过 128-token 边界时，先在设备内复制到副本区，请求结束后取最近一次完成的副本。**DDR 快照与活跃 HBM 的更新节奏不同，不能把它理解成实时镜像。**

| 状态                | 活跃执行     | 跨请求保留            |
| ----------------- | -------- | ---------------- |
| Main KV＋Indexer K | HBM 分页缓存 | 3FS              |
| Encoder SWA       | HBM 环形窗口 | 生成机器 DDR 快照      |
| Decoder SWA       | HBM 环形窗口 | 不长期保存，重放恢复       |
| C2 压缩边界状态         | HBM 中间态  | 保存位置对齐，避免跨边界恢复依赖 |

`cache_engine.py` 把 state 分成 stored region 和 dropped region。前者包含 Encoder SWA；后者包含 Decoder SWA 与压缩状态。名称里的 `state_size_to_disk` 只限定取出的字节范围，不能据变量名断言 SWA 真写进了 3FS。更关键的是，P→D 传完整 inference state，两部分都要传；1.35 MiB 仅是跨请求快照，不能当成完整 PD state 大小。[\[13\]](https://github.com/deepseek-ai/DeepInfer/blob/774ce244b87cc0eb8d4b94ab9f5de2ab22049f01/deepinfer/worker/cache_engine.py)

如果主机下线或快照被驱逐，全局 KV 仍可能命中。系统退回一个 128-token 块，重放末尾窗口；从位置 (s) 开始，位置 (i) 只看 (\\max(s,i-127)) 到 (i)。前面窗口缺失的历史不再补齐，得到的是近似 SWA，已有全局 KV 保持复用。[\[2\]](https://github.com/deepseek-ai/DeepInfer/blob/774ce244b87cc0eb8d4b94ab9f5de2ab22049f01/doc/inference_deployment_qa.md)[\[3\]](https://huggingface.co/deepseek-ai/DeepSeek-V4.1-Flash/blob/314113f8a7dadb388215396ba5fbee2c610a04d6/DeepSeek_V41_Tech_Report.pdf)

这里的 128 是算法重放窗口。公开 README 的默认 Prefill 保留尾段为 140 token，需覆盖 SWA 窗口和最大回退长度，不能把运行时保留长度与算法窗口直接等同。[\[22\]](https://github.com/deepseek-ai/DeepInfer/blob/774ce244b87cc0eb8d4b94ab9f5de2ab22049f01/README.md)

官方把重放成本形容为与长度和层数无关的常数。准确的工程读法是：**重放的 token 范围不再扩成“窗口×层数”，执行这些 token 仍然需要经过相关层。** 它以模型允许的近似换掉长期 SWA 持久化，让计算节点故障主要损失短期快照，而非整个前缀。换成未按此机制训练与验证的模型，不能直接照搬这一恢复策略。

## 五、D 不做 KV offload，SSD 为什么仍然很大？

> “Decode is not bound by cache capacity, so Hi-Sparse is not used either.”——部署 Q&A，Block KV cache。[\[2\]](https://github.com/deepseek-ai/DeepInfer/blob/774ce244b87cc0eb8d4b94ab9f5de2ab22049f01/doc/inference_deployment_qa.md)
> 
> 译文：Decode 未受缓存容量约束，因此也未使用 Hi-Sparse。

这里描述的是当前生产工作点：活跃 Decode KV 留在 HBM；生成的 KV 仍写入磁盘，供未来轮次复用。把活跃 KV 从 HBM 卸出去，和把结果写给长期缓存，是两条生命周期不同的路径。

按每 rank 60 条序列、各序列平均长度相同、仅计算 global KV 的假设，可得：

| 平均上下文        | 每 rank 全局 KV 估算，十进制 GB |
| ------------ | ---------------------- |
| 32K＝32,768   | 1.75 GB                |
| 128K＝131,072 | 7.00 GB                |
| 1M＝1,048,576 | 55.99 GB               |

若把 K/M 按 1,000/1,000,000 计，同三行则是 1.71、6.84、53.40 GB。本文显式写出 token 数，避免混用两套单位。**“目前未受容量约束”是运行条件，不能外推为百万上下文任意并发都放得下。** 有的工作点先碰 TBT，有的会先碰 HBM。

P 的峰值还可能更复杂。`attn.py::_prepare_prefill_kv()` 为当前来源组建立连续 BF16 缓冲，同组消费层复用，到下一来源组时替换；一百万个 C1 记录仅 512 通道 BF16 数据就约 1.07 GB，尚不含其他张量。减小本轮 chunk 能减少部分激活，却不会自动缩短已经命中的历史。临时 gather 缓冲释放，也不等于底层分页 KV 释放。D 不卸载活跃 KV 的选择也不限定 P 的回收策略；当前资料尚不足以确认生产 P 在各来源组之后是否逐组回收底层分页 KV。[\[7\]](https://github.com/deepseek-ai/DeepInfer/blob/774ce244b87cc0eb8d4b94ab9f5de2ab22049f01/deepinfer/model_executor/models/v4_modules/attn.py)

长期存储容量则由跨轮次工作集决定。官方给 Agent 三天、Chatbot 七天的保留窗口；后者少量热点前缀的长尾复用更重。它解释了为什么一个“不做 Decode offload”的系统仍需要大 SSD 池：SSD 在买未来的计算复用，HBM 在服务此刻的执行。

## 六、用 NIC 做 H2D，究竟绕过了什么瓶颈？

> “换入使用 GDR 实现，用 RDMA 网卡就能打满 PCIe 带宽，并且不占 SM。”——官方部署文章，Block KVCache。[\[1\]](https://zhuanlan.zhihu.com/p/2095560295930508953)

这句话包括本机共享内存到 HBM 的搬运。NIC 在这里充当 DMA 发起者；目标是提高细粒度 KV block 拷贝效率，并给模型计算留下 SM。路径分三段：入口元数据匹配，独立进程异步做 3FS→shm，engine 内线程做 shm→HBM，争取与逐层 forward 重叠。[\[2\]](https://github.com/deepseek-ai/DeepInfer/blob/774ce244b87cc0eb8d4b94ab9f5de2ab22049f01/doc/inference_deployment_qa.md)

代码里的 `load_paged_kv_cache()` 容易让人误判。它接收已经在设备上的分页缓存与分段映射，做 HBM 内 gather，并按格式解量化或整理布局；它没有实现生产的 3FS→DDR→HBM 服务。离线推理直接在设备生成、保留 KV，也不能用来证明外部 swap-in 的 ready 协议。[\[14\]](https://github.com/deepseek-ai/DeepInfer/blob/774ce244b87cc0eb8d4b94ab9f5de2ab22049f01/deepinfer/model_executor/layers/tilelang_ops/load_paged_kv_cache.py)

延续 128K 示例，116.65 MB 在理想 400 Gbit/s、即 50 GB/s 的单链路上，仅串行化下界约 2.33 ms；若可用带宽只有假设的 25 GB/s，下界变为 4.67 ms。存储随机读、元数据、PCIe 拓扑、排队和多个请求竞争都还没算。因此，**前向能够覆盖某段传输时间，才有重叠收益；异步本身不保证传输全部隐藏。**

DualPath 提供了一条相关但需区分的路线。P-read 先把历史读到 P DDR，再供 P HBM 使用，并交给 D；D-read 把历史先读到 D DDR，P 从那里读取历史进行计算，主要回传新增 KV。这样可以借用 D 闲置的存储入口带宽。它还把本机 H2D/D2H 纳入 NIC 流量管理，用 QoS 保护亚毫秒级的计算通信。[\[15\]](https://arxiv.org/html/2602.21548v1)

V4.1 Q&A 明确引用了 DualPath 的组内调度；目前可据公开材料确认的关联到这里为止，不能把完整双路径数据面直接写成 V4.1 的已验证生产实现。同理，“算第 3—8 层时预取第 9 层”是可讨论的流水方案，尚缺生产时序与 ready/wait 证据。

若历史最终通过 P→D 路径完整交接，一条基础带宽预算是：

\[ R\_{\\rm PD,global}\\approx \\lambda\_{\\rm handoff}\\times890\\times\\mathbb{E}\[L\]. \]

其中长度按原始 token、速率按每秒交接请求计，再加完整执行状态与协议开销。若每秒 100 次、平均 128K，global KV 就约 11.67 GB/s；不同数据面可能让历史绕行或在 D 预置，所以公式是指定路径的基线。**高命中节省 Prefill 计算，不保证同比例减少 PD 流量。**

## 七、Prefill 用小 EP，Decode 用 Wide-EP，为什么合理？

> “Decode: EP64~128/DP64~128 (Wide-EP); 在显卡可用的情况下用尽可能大的 EP/DP”——官方部署文章，并行策略。[\[1\]](https://zhuanlan.zhihu.com/p/2095560295930508953)

同一模型，P 推荐 EP2—8，D 推荐 EP64—128。Attention DP 的数量等于 MoE EP 的数量，它们作用在同一组 rank 的不同子层，不能把 DP64×EP64 算成 4,096 卡。

用均匀路由作一个简化估算：共有 384 个 routed experts，每 token 选 6 个；EP 规模为 (g)，每 rank 本步提交 (b) 个 token，则平均每专家收到：

\[ m\_{\\rm expert}=\\frac{gb\\times6}{384}=\\frac{gb}{64}. \]

固定每卡负载 (b=60)，EP8 平均每专家 7.5 个 token，EP64 则为 60 个；不含副本时，每卡专家数量同时从 48 降至 6。均匀情况下，每卡 token-expert 任务仍约为 (6b=360)。**Wide-EP 改善的是每份权重服务的 token 数和 GEMM 形状，每卡 FLOPs 未必下降。**

![[DeepInfer 读后：DeepSeek V4.1 Flash 如何组织计算、-04.png]]

*图 3：官方离线曲线的横轴为单用户速度，纵轴为每 GPU 输出吞吐；标注数字为每 rank 请求数。上下文为 131,072，固定 nextn＝3，假设逐 token 接受率 75%。EP64 配 384＋64 专家槽，EP8 配 384＋32，因此它比较的是两套完整配置，不能把差异全部归因于 EP 数量。来源：DeepInfer Q&A 原图，条件来自图内标注。*

P 本来就能把大量 prompt token 聚成较大的专家 batch，继续扩大 EP 的收益可能抵不过通信与同步；D 的小步执行更需要跨 rank 汇聚 token，前提是流量足以维持每卡并发，互联也位于合适的 scale-up 域。DSpark 会改变每步验证 token 数，所以最佳 EP 还与推测策略联动。

EPLB 为此付出额外容量：EP64 从基础 6 专家/卡增加到约 7 个，routed-expert 槽位从 384 到 448，增加约 16.7%，不含共享专家。副本让逻辑专家可映射到多个物理位置，统计与权重迁移则产生额外工作。[\[2\]](https://github.com/deepseek-ai/DeepInfer/blob/774ce244b87cc0eb8d4b94ab9f5de2ab22049f01/doc/inference_deployment_qa.md)

此前知识库中“某些模型依靠训练均衡，不需要运行时 EPLB”的经验仍有适用范围；DeepInfer 明确选择了动态复制与重排，生产需求不能由训练目标代替。中文文章的 P 侧 `moe_lb` 为 EP2 的 1.0320、EP4 的 1.0618，当前英文 Q&A 则为 1.045、1.074；两者 D 侧均为 1.090。本文保留差异，不合成一个精确基准，也不据此推断改进幅度。

## 八、MegaMoE 融合之后，互联需要保证什么？

> “（超）节点内 MoE 部分专家并行（EP）：增大 batch size，提高 GPU 矩阵乘法效率，从而提高吞吐。”——官方部署文章，推理系统设计概览。[\[1\]](https://zhuanlan.zhihu.com/p/2095560295930508953)

这句把 EP 的目标落在矩阵效率上，但专家 batch 只有经过 dispatch、计算和 combine 才会变成输出。一个 rank 的 token 送往远端专家，算完还要回到原来的 token 位置；边界过粗，通信和计算会排成串行阶段。

DeepInfer 的 `mlp.py` 选择 MegaMoE 路径；昇腾后端公开代码进一步暴露了细粒度执行：先建立 dispatch 元数据并 pull 输入，GEMM、SwiGLU 和第二次 GEMM 在 AIC/AIV 与 MTE、L1/L0 流水之间推进，结果 push 回目的端，按 ready 状态归并。[\[16\]](https://github.com/deepseek-ai/DeepInfer/blob/774ce244b87cc0eb8d4b94ab9f5de2ab22049f01/deepinfer/model_executor/models/v4_modules/mlp.py)[\[17\]](https://github.com/deepseek-ai/DeepGEMM-Ascend/blob/3a35326271b069b8b5bf1a754435acd6cc91575b/deep_gemm/include/deep_gemm/mega_moe.hpp) 这给系统提供的是局部完成即推进的机会，HBM 缓冲、原子计数、barrier 与尾部排空依然存在。

设一组数据块的通信时间为 (T\_c)，计算为 (T\_m)。完全串行要 (T\_c+T\_m)；理想流水稳态最多逼近 (\\max(T\_c,T\_m))，还要加启动与排空。若两者恰好相等，这个两阶段示意模型的上限是 2 倍；如果计算已远大于通信，可隐藏的时间比例自然缩小。这是解释重叠边界的模型，**不代表 MegaMoE 的实测加速比。**

再往下看，comm 代码用原子计数发布进度，配合 `asc_threadfence()`、数据 barrier；主 kernel 显式使用 cache mode，包括 write-through-share 和特定加载策略。[\[17\]](https://github.com/deepseek-ai/DeepGEMM-Ascend/blob/3a35326271b069b8b5bf1a754435acd6cc91575b/deep_gemm/include/deep_gemm/mega_moe.hpp)[\[18\]](https://github.com/deepseek-ai/DeepGEMM-Ascend/blob/3a35326271b069b8b5bf1a754435acd6cc91575b/deep_gemm/include/deep_gemm/comm/mega_moe.hpp) 消费端看到 ready 后，必须能读到对应数据，不能拿到陈旧缓存。

**远端 load/store 证明了访问能力，发布—消费协议决定结果何时可用。** 仅凭源码出现远端地址，无法推出系统依赖完整的硬件 Cache Coherency；只保证消息顺序也不够，仍要处理可见性。对互联设计，应该逐项审阅远端访问、原子、缓存策略与完成语义，再谈融合是否成立。`.cuda()` 等 API 名称同样不能替代平台判断，DeepInfer 的 Ascend 兼容层还会映射相关接口。[\[19\]](https://github.com/deepseek-ai/DeepInfer/blob/774ce244b87cc0eb8d4b94ab9f5de2ab22049f01/deepinfer/platform.py)

## 九、Confidence 分配验证算力，CPU 为什么可以晚一步知道？

> “调度时把验证算力分配给边际收益最高的 token。”——官方部署文章，DSpark 投机解码，原文节选。[\[1\]](https://zhuanlan.zhihu.com/p/2095560295930508953)

线上 draft 一次生成 5 个候选，但并非所有候选都应验证。对某请求，第 (j) 个候选只有前面都被接受才贡献输出，因此要比较的是前缀存活概率；`schedule_mtp_varlen.py` 用 confidence 累乘、排序与预算分配，选出各请求的验证长度，最终接受仍由主模型决定。[\[20\]](https://github.com/deepseek-ai/DeepInfer/blob/774ce244b87cc0eb8d4b94ab9f5de2ab22049f01/deepinfer/model_executor/layers/tilelang_ops/schedule_mtp_varlen.py)

假设两个请求的前缀存活概率分别为 ((0.95,0.90,0.85)) 和 ((0.90,0.30,0.10))，只有四个额外验证位置时，按此简化模型选择前三个加第二条的第一个，期望贡献为 3.60；平均各给两个则是 3.05。示例只说明分配逻辑，实际预算还要使用硬件吞吐曲线，优化“有效输出/耗时”。

官方中文文章报告变长验证相对固定 5 个候选，生产吞吐约提升 6%；英文 Q&A 给了可复算的工作点：EP64/DP64、每 rank 60 请求，估计吞吐 13,730→14,466 tok/s/rank，约 +5.4%，平均验证长度 5→3.58，接受长度 3.30→3.21。相同表中，每 rank 16 请求时却是 −1.5%。[\[1\]](https://zhuanlan.zhihu.com/p/2095560295930508953)[\[2\]](https://github.com/deepseek-ai/DeepInfer/blob/774ce244b87cc0eb8d4b94ab9f5de2ab22049f01/doc/inference_deployment_qa.md) **少做验证和更高接受率都只是中间量，最终要看同一延迟要求下的有效吞吐。**

异步调度又提出一个依赖问题：CPU 调度 N＋1 时，N 的 confidence 尚未回传，怎么决定长度？公开路径是 CPU 先确定总预算，GPU 使用上一批的 confidence 和 remap 重新分配逐请求长度，再将结果回写 pinned buffer。`worker.py` 发射当前 forward 后处理上一步结果；采样前必要的长度修正、事件等待仍然存在。[\[20\]](https://github.com/deepseek-ai/DeepInfer/blob/774ce244b87cc0eb8d4b94ab9f5de2ab22049f01/deepinfer/model_executor/layers/tilelang_ops/schedule_mtp_varlen.py)[\[21\]](https://github.com/deepseek-ai/DeepInfer/blob/774ce244b87cc0eb8d4b94ab9f5de2ab22049f01/deepinfer/worker/worker.py)

因此 CPU 可以晚获知结果，GPU 所需的数据依赖仍须按时满足。“Zero-overhead”描述要消掉的调度空转，不能解释成无同步。停止条件、语法约束与回退更新也必须沿各自的依赖完成。变长验证、Wide-EP 和异步执行共享同一个 batch 与时间预算，收益不能直接相乘。

## 十、大 EP 的故障域，必须进入容量预算

> “任意一张卡故障即导致整个实例不可用。”——官方部署文章，自动扩缩容，原文节选。[\[1\]](https://zhuanlan.zhihu.com/p/2095560295930508953)

64 张卡共同执行专家通信，也共同承担一次实例失效。若每卡独立故障强度为 (r)，忽略相关故障，则 EP 组暴露出的故障强度近似为 (gr)。这只是说明规模效应的模型，电源、交换机和软件故障会破坏独立假设，不能由它推算线上可用率。

官方的处理是冗余配额加请求重路由，实例重启为分钟级；重启后按健康卡数重新确定 EP。**这里没有声称在原执行过程中无缝摘掉故障卡。** 全局 KV 留在外部存储，短期 SWA 可近似恢复，给请求级恢复提供了状态基础，但仍要支付重新分配、加载与计算成本。[\[2\]](https://github.com/deepseek-ai/DeepInfer/blob/774ce244b87cc0eb8d4b94ab9f5de2ab22049f01/doc/inference_deployment_qa.md)

缩容也付同一笔状态迁移费。D 收到下线通知后等约一分钟，再把仍未完成的长尾请求及其 KV 一次性交给其他实例。如果 100 条剩余请求各有 128K 历史，仅 global KV 就约 11.67 GB；理想独占 400G 链路的串行化下界约 0.23 秒，真实耗时还受并发链路、完整 state 与接收端空闲容量约束。

P 扩缩容以未命中 token 需求为信号，D 以运行请求数为信号；P 内部按总长分桶并用 (cost(q,c)) 反解时间预算，D 放置同时看请求数和总 KV 长度。[\[1\]](https://zhuanlan.zhihu.com/p/2095560295930508953)[\[2\]](https://github.com/deepseek-ai/DeepInfer/blob/774ce244b87cc0eb8d4b94ab9f5de2ab22049f01/doc/inference_deployment_qa.md) 两条计数都只是需求信号，实例容量还受 HBM、存储入口、PD 带宽与 TBT 限制。配额利用率高于 0.8 禁止缩容，是一条控制规则，不能机械推导成所有容量公式都除以 0.8。

对架构师，Wide-EP 的验收单位应该同时包含计算域、通信域、调度域和故障域。每卡吞吐提高之后，还要留下故障时能接住请求的资源；把全部卡都填满，可能会吃掉此前在算子上省出的成本。

## 十一、读 11 取 10，把什么代价换成了尾延迟？

> “读取时同时向组内 11 台 Storage 节点发起请求，任意 10 个响应即可解码返回。”——官方部署文章，3FS 分布式文件系统。[\[1\]](https://zhuanlan.zhihu.com/p/2095560295930508953)

官方同时给出 10 数据分片＋2 校验分片的纠删码配置。逻辑上，一个条带是一组分片，许多条带复用存储节点；条带数量不能再乘分片数，算成服务器台数。同一条带的分片可并行，不同条带也可同时读。

等待 10 个必要分片时，慢的那一个决定完成时间；多发一个，就可能用额外响应提前结束。比如已有 9 个数据片与 1 个校验片，恢复缺失数据还需要解码，应比较“现在解码”与“继续等原始数据片”的时间。设单片为 (S)，最少读取 10S；如果第 11 片也完整返回，载荷就是 11S，约多 10%。是否及时取消慢请求，会影响实际放大，但公开材料没有交代取消策略与解码位置。

**额外读取能削弱独立慢节点的尾部影响，共享出口已经饱和时，它还可能增加排队。** Q&A 的“10＋2”与“向 11 个节点读”之间的精确映射也未讲透，不能自行补全成特定条带放置或校验选择算法。[\[2\]](https://github.com/deepseek-ai/DeepInfer/blob/774ce244b87cc0eb8d4b94ab9f5de2ab22049f01/doc/inference_deployment_qa.md)

官方建议每存储服务器配置 20×30.73 TB SSD，至少 40 GB/s 读、5 GB/s 写。400G 的理论字节带宽为 50 GB/s，单路这样的出口若承担 40 GB/s 读取，协议与并发余量已有限；但文档未说明实际 NIC 端口数，不能据此替它定配置。40 GB/s 若已经是物理读取口径，就不应再重复乘 1.1；若是应用逻辑吞吐目标，才需要另计读放大。

存储保护的对象同样是推理 SLO。为了让 P 在截止时间前拿到状态，系统愿意付一些额外读流量；是否划算，得和计算网络、PCIe 与存储队列一起看。

## 十二、离线引擎与生产服务之间，还隔着哪些条件？

> “This open-source release provides offline inference with the core features needed for serving at scale.”——DeepInfer README。[\[22\]](https://github.com/deepseek-ai/DeepInfer/blob/774ce244b87cc0eb8d4b94ab9f5de2ab22049f01/README.md)
> 
> 译文：本次开源提供离线推理，以及支撑大规模服务所需的核心能力。

README 明确列出尚未包含的完整服务组件：KV cache 系统、E/P/D 分离部署，以及跨 DP 副本的 P/D 负载均衡。这和代码中已经存在的 state 导出、跨层复用、变长验证并不冲突。引擎为外部组件留好了能力，完整控制面与生产数据面仍要接上。

deepseek-recipe 补的是另一段真实开销：请求协议转成统一对话格式，再做模型模板与 token 编码；用户输入里看似特殊 token 的字符串按普通文本编码，控制 token 由模板显式插入；输出端跨 chunk 保留解析状态，尽早送出正文或工具参数。它能帮助第三方对齐 API 行为，也说明单看模型 forward 覆盖不了用户感知时延。[\[23\]](https://github.com/deepseek-ai/deepseek-recipe/tree/935817c66f5d6e64838f419692545f06787e485f)

可把一次请求的 TTFT 写成一个用于定位瓶颈的分解：编码、元数据匹配、排队、状态加载、Prefill、交接和首轮 Decode。部分项可以重叠，所以测量关键路径时不能简单求和。如果某段仅占端到端时间的 20%，即便它加速 5%，串行 Amdahl 模型中的总加速也只有 (1/(0.8+0.2/1.05)\\approx1.0096)。这是示意假设，提醒我们先确认优化覆盖了哪一段。

Q&A 关于内部/开源 kernel 的比较，提到相关端到端性能差距至多约 5%；它出现在算子实现问答的上下文，缺少一份完整线上服务同负载对照，不能把它当成“下载仓库就能达到线上 95% 效率”的证明。[\[2\]](https://github.com/deepseek-ai/DeepInfer/blob/774ce244b87cc0eb8d4b94ab9f5de2ab22049f01/doc/inference_deployment_qa.md) 155 万亿 token/天和每卡小时收入也是官方生产披露，缺少缓存计费占比、总卡数与负载分布，无法反推出可复现的每卡生成速度或利润。

**DeepInfer 展示了一组相互依赖的成立条件：模型压小全局状态，生命周期决定哪些状态长期保存，数据路径保障状态按时到达，专家批次与负载均衡提高计算效率，冗余承担执行域扩大的风险。** 收益落在这些条件共同满足的工作点上。

公开材料还没有回答外部 KV 的完整 ready 协议、P 侧分页缓存回收时序、V4.1 是否采用 DualPath 完整数据面，以及不同硬件上整套服务的同口径复现结果。部署评估可以从这几处开始：让缓存命中后的 TTFT、完整 state 大小、加载流量和故障恢复进入同一份测量。模型已经把历史压小了，系统接下来要证明，这份历史能在正确的时间、正确的位置被使用。

- - -

## 参考资料

中文引句按官方部署文章 2026-10-20 版本的完整文本核对；在线页本轮访问受限，生产机制与可访问的固定版 Q&A 交叉核验。本文未使用缺失的知乎图片，正文配图来自官方仓库与已核对的报告。

\[1\] [DeepSeek：V4.1 Flash 大规模推理系统部署文章（中文）](https://zhuanlan.zhihu.com/p/2095560295930508953)

\[2\] [DeepInfer：Inference Deployment Q&A](https://github.com/deepseek-ai/DeepInfer/blob/774ce244b87cc0eb8d4b94ab9f5de2ab22049f01/doc/inference_deployment_qa.md)

\[3\] [DeepSeek-V4.1-Flash Technical Report](https://huggingface.co/deepseek-ai/DeepSeek-V4.1-Flash/blob/314113f8a7dadb388215396ba5fbee2c610a04d6/DeepSeek_V41_Tech_Report.pdf)

\[4\] [DeepSeek-V4.1-Flash-1017：config.json](https://huggingface.co/deepseek-ai/DeepSeek-V4.1-Flash-1017/blob/1bae0e1088993b9b82de08f59de5be5a34e42170/config.json)

\[5\] [DeepInfer：主干执行与 CED 边界](https://github.com/deepseek-ai/DeepInfer/blob/774ce244b87cc0eb8d4b94ab9f5de2ab22049f01/deepinfer/model_executor/models/deepseek.py)

\[6\] [DeepInfer：Decoder layer 与完整隐藏状态捕获](https://github.com/deepseek-ai/DeepInfer/blob/774ce244b87cc0eb8d4b94ab9f5de2ab22049f01/deepinfer/model_executor/models/decoder_layer.py)

\[7\] [DeepInfer：Attention、共享 KV 与 Prefill 连续缓冲](https://github.com/deepseek-ai/DeepInfer/blob/774ce244b87cc0eb8d4b94ab9f5de2ab22049f01/deepinfer/model_executor/models/v4_modules/attn.py)

\[8\] [DeepInfer：跨层 KV、Indexer K、Top-K 与候选池拓扑](https://github.com/deepseek-ai/DeepInfer/blob/774ce244b87cc0eb8d4b94ab9f5de2ab22049f01/deepinfer/model_executor/models/v4_modules/cross_layer.py)

\[9\] [DeepInfer：Hugging Face 配置适配器与 SWA 默认开关](https://github.com/deepseek-ai/DeepInfer/blob/774ce244b87cc0eb8d4b94ab9f5de2ab22049f01/deepinfer/hugging_face/config.py)

\[10\] [DeepInfer：压缩参考公式与状态更新](https://github.com/deepseek-ai/DeepInfer/blob/774ce244b87cc0eb8d4b94ab9f5de2ab22049f01/deepinfer/model_executor/layers/tilelang_ops/compress.py)

\[11\] [DeepInfer：KV cache 字节格式](https://github.com/deepseek-ai/DeepInfer/blob/774ce244b87cc0eb8d4b94ab9f5de2ab22049f01/deepinfer/model_executor/layers/tilelang_ops/cache_common.py)

\[12\] [DeepSeek-V4-Flash：config.json](https://huggingface.co/deepseek-ai/DeepSeek-V4-Flash/blob/60d8d70770c6776ff598c94bb586a859a38244f1/config.json)

\[13\] [DeepInfer：StateBlock、stored/dropped 分区与 P→D 导出](https://github.com/deepseek-ai/DeepInfer/blob/774ce244b87cc0eb8d4b94ab9f5de2ab22049f01/deepinfer/worker/cache_engine.py)

\[14\] [DeepInfer：设备端分页 KV gather](https://github.com/deepseek-ai/DeepInfer/blob/774ce244b87cc0eb8d4b94ab9f5de2ab22049f01/deepinfer/model_executor/layers/tilelang_ops/load_paged_kv_cache.py)

\[15\] [DualPath: Breaking the Storage Bandwidth Bottleneck in Agentic LLM Inference（v1）](https://arxiv.org/html/2602.21548v1)

\[16\] [DeepInfer：MoE 执行路径](https://github.com/deepseek-ai/DeepInfer/blob/774ce244b87cc0eb8d4b94ab9f5de2ab22049f01/deepinfer/model_executor/models/v4_modules/mlp.py)

\[17\] [DeepGEMM-Ascend：MegaMoE 计算与通信流水](https://github.com/deepseek-ai/DeepGEMM-Ascend/blob/3a35326271b069b8b5bf1a754435acd6cc91575b/deep_gemm/include/deep_gemm/mega_moe.hpp)

\[18\] [DeepGEMM-Ascend：MegaMoE 进度发布与通信同步](https://github.com/deepseek-ai/DeepGEMM-Ascend/blob/3a35326271b069b8b5bf1a754435acd6cc91575b/deep_gemm/include/deep_gemm/comm/mega_moe.hpp)

\[19\] [DeepInfer：NVIDIA 与 Ascend 平台兼容层](https://github.com/deepseek-ai/DeepInfer/blob/774ce244b87cc0eb8d4b94ab9f5de2ab22049f01/deepinfer/platform.py)

\[20\] [DeepInfer：Confidence 累乘与变长验证预算](https://github.com/deepseek-ai/DeepInfer/blob/774ce244b87cc0eb8d4b94ab9f5de2ab22049f01/deepinfer/model_executor/layers/tilelang_ops/schedule_mtp_varlen.py)

\[21\] [DeepInfer：Worker 异步执行与事件等待](https://github.com/deepseek-ai/DeepInfer/blob/774ce244b87cc0eb8d4b94ab9f5de2ab22049f01/deepinfer/worker/worker.py)

\[22\] [DeepInfer：README 与开源边界](https://github.com/deepseek-ai/DeepInfer/blob/774ce244b87cc0eb8d4b94ab9f5de2ab22049f01/README.md)

\[23\] [DeepSeek Recipe：API 协议与 token 编解码](https://github.com/deepseek-ai/deepseek-recipe/tree/935817c66f5d6e64838f419692545f06787e485f)

\[24\] [前期调研：DeepSeek-V4.1-Flash 的 KV cache 压缩](https://miraclefarms.github.io/notes/2026/09/10/deepseek-v41-flash-kv-cache-compression/)

### 版本对齐信息

| 材料                 | 本文固定版本                  | 核对结果与适用边界                            |
| ------------------ | ----------------------- | ------------------------------------ |
| DeepInfer 及部署 Q&A  | `774ce244b8`，2026-10-20 | 2026-10-20 查询主分支，仍为此提交；源码链接固定到该版     |
| DeepGEMM-Ascend    | `3a35326271`，2026-10-14 | 当前主分支与附件指定版一致；分析的是 Ascend 实现         |
| V4.1-Flash-1017 配置 | `1bae0e1088`，2026-10-17 | 层号、压缩比与来源层以此 checkpoint 为准           |
| V4.1 技术报告 HF 仓库    | `314113f8a7`，2026-10-16 | 读取其中 PDF；仓库修订日期不等于论文首发日期             |
| V4-Flash 配置        | `60d8d70770`，2026-06-22 | 仅比较 C4/C128 与 C2/C1 编排，不推断隔离因果       |
| DeepSeek Recipe    | `935817c66f`，2026-10-19 | 固定版 README 与部署文档交叉核对，不作吞吐复现          |
| DualPath           | arXiv `2602.21548v1`    | 用于解释双路径与 NIC/QoS；不将整套数据面视为 V4.1 上线事实 |

文中估算统一区分十进制 GB 与二进制 MiB；每 rank/global KV 的计算不含权重、SWA 与临时缓冲。Prefill `moe_lb` 的中英文数值差异已在第七节保留，未混作同一组测量。
