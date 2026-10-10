---
title: "大模型 GPU 通信（二）：从 AllReduce 到 AllToAll，大模型训练到底在通信什么？"
source: "算力网络架构手记"
category: "Inbox"
group: "AI基础设施"
author: "韦子扬"
url: "https://weiziyang.wiki/posts/gpu-networking/gpu_communication_2"
published: 2026-08-28T17:36:01+08:00
saved: 2026-10-10T09:28:56+08:00
folo_key: "url::https://weiziyang.wiki/posts/gpu-networking/gpu_communication_2"
tags:
  - "folo"
  - "Inbox"
  - "算力网络架构手记"
---

# 大模型 GPU 通信（二）：从 AllReduce 到 AllToAll，大模型训练到底在通信什么？

> [!info] 算力网络架构手记 · Inbox · 2026-08-28 17:36 · [原文](https://weiziyang.wiki/posts/gpu-networking/gpu_communication_2)

> 上一篇主要介绍了 GPU 通信的硬件基础：数据在单机内可以经过 PCIe、NVLink 和 NVSwitch，在多机之间则可能经过 NIC、IB、RoCE 和 RDMA , 它解决的是从一个机器的gpu到另一个机器的gpu会经过哪些链路。但是真实的需求往往是很多gpu一起干一件事儿然后进行汇总、派发等。所以这篇文章往上游探究NCCL在这些硬件上提供的**AllReduce、ReduceScatter、AllGather 和 AllToAll**，以及为什么这些会出现在 TP、SP 和 EP 中。
> 
> 这篇内容有一些其实已经在《初识 LLM 与Sglang》交代过， 但那篇更多是推理视角，这篇更多是训练视角。

# 为什么要通信

其实这些通信原语说复杂也不复杂，核心就是解释好着三点：

1. 哪些gpu需要参与这次通信？
2. 通信中是否需要对这些数据进行求和/平均？
3. 通信结束以后， 完整的tensor是出现在一张卡还是所有卡还是继续分片保存？

MARKDOWN

```markdown

AllReduce / ReduceScatter / AllGather / AllToAll
                 │
                 │ 描述“通信完成后张量应该长什么样”
                 ▼
           Ring / Tree / Pairwise
                 │
                 │ 描述“GPU 按什么顺序交换数据”
                 ▼
      NVLink / PCIe / RDMA / InfiniBand
                 │
                 │ 描述“字节实际经过哪条硬件路径”
                 ▼
              GPU HBM
```

总而言之,如图所示， all reduce ,reduce scatter , all gather 这些都是**通信语义**， ring tree都是**通信算法**， nvlin这些是**通信链路**，在不同层次上

![[大模型 GPU 通信（二）：从 AllReduce 到 AllToAll，大模型-01.png]]

# 前置名词解释

## Rank

通常一张卡就是对应一个rank，这个非常好理解

## Process Group

process group决定哪些rank参与通信， 同一组gpu也可以同时属于不同的通信组；

- TP Group：共同切分同一层权重；
- DP Group：保存相同模型分片、处理不同数据；
- EP Group：共同承载一组不同专家；
- PP Group：分别承载不同流水线 Stage。

## World Size

总共有多少张卡就有多少word size

# 集合通信的本质——张量布局变换

为了把四种原语串起来，可以把 Tensor 在多张 GPU 上的状态分成三类：

- **Partial（局部结果）**：每张卡有相同形状的 Tensor，但每份只是局部计算结果，最终需要求和；
- **Sharded（分片状态）**：完整 Tensor 被切开，每张卡只保存其中一部分；
- **Replicated（复制状态）**：每张卡都拥有完整且相同的 Tensor

四种通信原语，其实就是四种张量布局变换：

| 通信原语          | 输入布局      | 输出布局       | 是否做 Reduce | 一句话理解                       |
| ------------- | --------- | ---------- | ---------- | --------------------------- |
| AllReduce     | Partial   | Replicated | 是          | 把局部答案相加，并让所有 GPU 得到完整答案     |
| ReduceScatter | Partial   | Sharded    | 是          | 把局部答案相加，但每张 GPU 只保留一片       |
| AllGather     | Sharded   | Replicated | 否          | 收集所有分片，让每张 GPU 都看到完整 Tensor |
| AllToAll      | Sharded A | Sharded B  | 否          | 不求和，按照目标 GPU 重新分发数据         |

![[大模型 GPU 通信（二）：从 AllReduce 到 AllToAll，大模型-02.png]]

# AllReduce

AllReduce 做两件事：

1. 对所有 Rank 的输入执行 Reduce，例如 Sum、Max、Min；
2. 把 Reduce 后的结果交给每一个 Rank。

最典型的场景就是dp的反向传播场景， 假设4张gpu分别处理不同的mini batch, 并在反向传播以后得到各自局部的梯度；

```
GPU0: g0
GPU1: g1
GPU2: g2
GPU3: g3
```

昨晚一次sum all reduce以后

TEXT

```text
GPU0: g0 + g1 + g2 + g3
GPU1: g0 + g1 + g2 + g3
GPU2: g0 + g1 + g2 + g3
GPU3: g0 + g1 + g2 + g3
```

写成公式就是：

$$
g=\sum_{r=0}^{P-1}g_r
$$

如果训练框架需要的是平均梯度，还会再除以数据并行度 $P$。有些框架在通信前缩放，有些在通信后缩放，但数学结果相同。

**为什么需要同步梯度？因为不同步每张卡上的权重就不同步了， 总不能一次训练训出多个不同版本的权重出来吧**

因此，AllReduce 完成的是：

$$
\text{Partial} \rightarrow \text{Replicated}
$$

![[大模型 GPU 通信（二）：从 AllReduce 到 AllToAll，大模型-03.png]]

## Ring All-Reduce

最容易想到的 AllReduce 实现是：先把数据都发送给一个根节点做 Reduce，再把结果 Broadcast 给其他 GPU。

TEXT

```text
GPU1 ─┐
GPU2 ─┼─> GPU0 Reduce ─> Broadcast
GPU3 ─┘
```

它的语义没有问题，但 GPU0 会成为中心瓶颈，其他 GPU 的链路也无法始终并行工作。Ring AllReduce 的思路是

**把大 Tensor 切成 **$P$** 块，让所有 GPU 在环上同时发送、接收和归约不同的数据块。**

假设 4 张 GPU 上的梯度都被切成 4 块：

TEXT

```text
GPU0: [A0, B0, C0, D0]
GPU1: [A1, B1, C1, D1]
GPU2: [A2, B2, C2, D2]
GPU3: [A3, B3, C3, D3]
```

下标表示数据来自哪张 GPU，字母表示它属于完整 Tensor 的哪一个分片；

注意这里从数学上看不是4个tensor，而是一个tensor， 它先是被行切到4个gpu上， 然后为了实现ring all reduce所以在内部又切了4份出来；

### 具体过程

#### 第一阶段： Reduce Scatter

4张gpu构成一个逻辑环；

TEXT

```text
GPU0 → GPU1 → GPU2 → GPU3 → GPU0
```

每一轮，每张 GPU 都向右发送一个数据块，同时从左边接收一个数据块，并把收到的数据累加到对应分片中。经过 $P-1$ 轮后：

TEXT

```text
GPU0: A0 + A1 + A2 + A3
GPU1: B0 + B1 + B2 + B3
GPU2: C0 + C1 + C2 + C3
GPU3: D0 + D1 + D2 + D3
```

此时 Reduce 已经完成，但每张 GPU 只拥有最终结果的一部分，这就是 ReduceScatter。

#### 第二阶段：AllGather

接下来继续沿着环传递已经归约完成的分片。再经过 $P-1$ 轮后，每张 GPU 都收齐 A、B、C、D 四个最终分片：

TEXT

```text
GPU0: [A, B, C, D]
GPU1: [A, B, C, D]
GPU2: [A, B, C, D]
GPU3: [A, B, C, D]
```

要注意一下， 这里all-gather的细节，并不是每次都把自己持有的所有数据都往下一个发， 这样会导致每一轮通信量都比上一轮大一个分片；而是每次只往下传自己多的那一块，通信量不变；

初始状态：

TEXT

```text
GPU0: A
GPU1: B
GPU2: C
GPU3: D
```

AllGather 第 1 轮

TEXT

```text
GPU0: [A, D]
GPU1: [A, B]
GPU2: [B, C]
GPU3: [C, D]
```

第二轮：

TEXT

```text
GPU0: [A, C, D]
GPU1: [A, B, D]
GPU2: [A, B, C]
GPU3: [B, C, D]
```

第三轮：

TEXT

```text
GPU0: [A, B, C, D]
GPU1: [A, B, C, D]
GPU2: [A, B, C, D]
GPU3: [A, B, C, D]
```

通信完成。  
 综上所述， 不难发现：

$$
\text{Ring AllReduce}
=
\text{ReduceScatter}
+
\text{AllGather}
$$

图示如下：

![[大模型 GPU 通信（二）：从 AllReduce 到 AllToAll，大模型-04.png]]

### Ring all reduce 的通信量

设：

- Rank 数量为 $P$；
- 每张 GPU 的输入 Tensor 大小为 $S$ Byte；
- Tensor 被均匀切成 $P$ 份，每份大小为 $S/P$。

ReduceScatter 有 $P-1$ 轮，每轮每张 GPU 发送 $S/P$：

$$
V_{RS}=\frac{P-1}{P}S
$$

AllGather 同样有 $P-1$ 轮：

$$
V_{AG}=\frac{P-1}{P}S
$$

所以 Ring AllReduce 中，每张 GPU 理论上发送的数据量为：

$$
V_{AR}=2\frac{P-1}{P}S
$$

当 $P$ 很大时，它接近 $2S$，而不是随着 GPU 数量线性增长到 $PS$。更重要的是，环上的所有 GPU 可以同时发送和接收，因此适合充分利用大消息带宽。

如果使用简单的 $\alpha$-$\beta$ 模型，其中 $\alpha$ 表示每轮通信的固定延迟，$B$ 表示有效链路带宽，那么 Ring AllReduce 的时间可以近似写成：

$$
T_{ring}\approx 2(P-1)\alpha+2\frac{P-1}{P}\frac{S}{B}
$$

这也暴露了 Ring 的特点：数据量很大时第二项占主导，Ring 的带宽利用率很好；消息很小时，$2(P-1)$ 轮带来的延迟会变得明显。

## Tree All-Reduce

Ring 的问题是轮数随 Rank 数线性增长。Tree AllReduce 换了个方向：不再让所有 GPU 排成一条链，而是把它们组织成一棵树，先沿树从叶子向根逐层 Reduce，再从根向叶子 Broadcast。

### 具体方法

#### 第一步：先把树建出来

以 8 张 GPU 为例，NCCL 会建出一棵每个节点最多两个孩子的树：

TEXT

```text
              GPU0                  ← 根 (root)
            /      \
        GPU1        GPU2
       /    \      /    \
    GPU3   GPU4  GPU5   GPU6
    /
 GPU7
```

树的深度是 3。注意这棵树只是**逻辑拓扑**，边并不要求是物理直连，NCCL 会尽量让树边落在带宽高的链路上（同机内 NVLink、跨机走网卡）。

#### 第二步：Reduce 阶段，自底向上逐层求和

规则只有一条：**每个节点等收齐所有孩子发来的数据，把它们累加到自己的 buffer 上，再整块发给父节点。**

和 Ring 最大的区别在这里：Ring 每轮只传 $S/P$ 的一个分片，而 Tree 每条边传的是 完整的 $S$，只是传的次数少。

逐轮展开：

TEXT

```text
Round 1:  GPU7 ──g7──> GPU3
          GPU3: g3 + g7
          (GPU4/5/6 是叶子，本轮无事可做)

Round 2:  GPU3 ──(g3+g7)──> GPU1     GPU5 ──g5──> GPU2
          GPU4 ──g4────────> GPU1     GPU6 ──g6──> GPU2
          GPU1: g1 + (g3+g7) + g4
          GPU2: g2 + g5 + g6

Round 3:  GPU1 ──(g1+g3+g4+g7)──> GPU0
          GPU2 ──(g2+g5+g6)────> GPU0
          GPU0: g0 + g1 + g2 + g3 + g4 + g5 + g6 + g7   ← 全局和
```

到这里，全局和只存在于根节点 GPU0 上。这一步等价于 NCCL 的 `Reduce`，而**不是** ReduceScatter——Ring 的第一阶段结束时每张卡各持有 $1/P$ 的最终结果，Tree 的第一阶段结束时是一张卡持有 $100\%$ 的最终结果，其余卡手上都是没用的中间和。

#### 第三步：Broadcast 阶段，自顶向下逐层下发

第二阶段沿完全相同的树边、相反的方向走，只是这次不做加法，只做复制：

TEXT

```text
Round 4:  GPU0 ──g──> GPU1,  GPU0 ──g──> GPU2
Round 5:  GPU1 ──g──> GPU3,  GPU1 ──g──> GPU4
          GPU2 ──g──> GPU5,  GPU2 ──g──> GPU6
Round 6:  GPU3 ──g──> GPU7
```

6 轮之后 8 张卡都拿到了同一个 $g=\sum_i g_i$，AllReduce 语义完成。

$$
\text{Tree AllReduce}
=
\text{Reduce}
+
\text{Broadcast}
$$

### 分块流水线

看上面的流程会有个疑问：Round 1 只有一条边在动，其他 GPU 全在等，这不是白白空转吗？

答案是**分块流水线**。NCCL 不会等整个 Tensor 走完一层再走下一层，而是把 Tensor 切成很多小 chunk，一个 chunk 发出去之后立刻发下一个。于是树的每一层都在同时处理不同的 chunk：

TEXT

```text
时间 ──────────────────────────────────────>

GPU3 → GPU1 :  c0   c1   c2   c3   c4  ...
GPU1 → GPU0 :       c0   c1   c2   c3  ...
GPU0 → GPU1 :            c0   c1   c2  ...
```

流水线一旦填满，所有层的链路都在同时工作，树的深度只贡献一次**填充/排空**的延迟，。这正是下面时间模型里 $\log_2 P$ 只出现在 $\alpha$ 项、$S/B$ 项前面系数是常数 2 的原因。

### 时间模型

当 Rank 数量为 $P$ 时，平衡树的深度约为 $\lceil\log_2P\rceil$。因此树形 Reduce + Broadcast 的通信轮数约为：

$$
2\lceil\log_2P\rceil
$$

而 Ring 需要 $2(P-1)$ 轮。GPU 数量很多、消息较小时，Tree 更容易降低延迟。

可以用一个直觉化的近似式理解它：

$$
T_{tree}\approx 2\lceil\log_2P\rceil\alpha+2\frac{S}{B}
$$

### Double Binary Tree

单棵二叉树有一个明显的不公平：叶子节点在 Reduce 阶段只发送 $S$，一个有两个孩子的中间节点却要接收 $2S$、再发送 $S$。在 8 卡的例子里，GPU4/5/6 几乎全程闲着，GPU1/GPU2 的链路被压满。而在一棵满二叉树里，接近一半的节点是叶子——也就是接近一半的算力和链路被浪费。

NCCL 的做法是同时构建**两棵互补的树**：让每个 Rank 在第一棵树里是叶子，在第二棵树里就是中间节点。然后把 Tensor 一分为二，前一半走树 A，后一半走树 B。

TEXT

```text
树 A 中：GPU4 是叶子（轻负载）
树 B 中：GPU4 是中间节点（重负载）
两棵树各跑一半数据 → 每个 Rank 的总负载被抹平
```

这样每个 Rank 的收发量都接近 $2S$，既保住了 $\log_2 P$ 的轮数，又拿回了满带宽。

## Ring 和 Tree 对比？

| 维度     | Ring          | Tree                    |
| ------ | ------------- | ----------------------- |
| 通信轮数   | $2(P-1)$ | 约 $2\log_2P$         |
| 主要优势   | 大消息峰值带宽利用率高   | 中小消息和大规模 Rank 下延迟更低     |
| 数据流    | 所有 Rank 构成环   | 分层 Reduce，再分层 Broadcast |
| 常见适用场景 | 大梯度、大激活值      | 对延迟敏感的中小消息              |
| 潜在问题   | Rank 增多后轮数增加  | 树上不同节点角色和链路压力不同         |

不能简单认为 Ring 一定比 Tree 快。NVIDIA 对 NCCL 调优的说明中也强调：Ring 通常具有更好的峰值带宽利用率，而 Tree 的复杂度是对数级，在中等消息大小下表现很好。默认情况下应让 NCCL 根据拓扑、Rank 数和消息大小自动选择，而不是看到某个算法在一台机器上更快，就在所有环境中强制指定。

![[大模型 GPU 通信（二）：从 AllReduce 到 AllToAll，大模型-05.png]]

# Reduce-Scatter

理解了All-Reduce， Reduce-Scatter就更好理解了，因为它只做了all reduce的前半段：

TEXT

```text
通信前：

GPU0: [A0, B0, C0, D0]
GPU1: [A1, B1, C1, D1]
GPU2: [A2, B2, C2, D2]
GPU3: [A3, B3, C3, D3]

通信后：

GPU0: [A0 + A1 + A2 + A3]
GPU1: [B0 + B1 + B2 + B3]
GPU2: [C0 + C1 + C2 + C3]
GPU3: [D0 + D1 + D2 + D3]
```

它完成的是：

$$
\text{Partial}\rightarrow\text{Sharded}
$$

如果完整输入大小为 $S$，那么每张 GPU 的输出只有 $S/P$，Ring ReduceScatter 的每卡发送量为：

$$
V_{RS}=\frac{P-1}{P}S
$$

# AllGather：用通信换取临时完整视图

AllGather 不做加法。它只是收集所有 Rank 的分片，按 Rank 顺序拼接，并让每个 Rank 都得到完整 Tensor。

TEXT

```text
通信前：

GPU0: [A]
GPU1: [B]
GPU2: [C]
GPU3: [D]

通信后：

GPU0: [A, B, C, D]
GPU1: [A, B, C, D]
GPU2: [A, B, C, D]
GPU3: [A, B, C, D]
```

它完成的是：

$$
\text{Sharded}\rightarrow\text{Replicated}
$$

如果收集后的完整 Tensor 大小为 $S$，那么通信前每张 GPU 只有 $S/P$，Ring AllGather 的每卡发送量为：

$$
V_{AG}=\frac{P-1}{P}S
$$

## AllGather 和 Gather 有什么区别？

- Gather：只有 Root Rank 得到完整结果；
- AllGather：每个 Rank 都得到完整结果。

## FSDP 的all-gather 和 reduce-scatter

在前向列切的情况下，megatron是不会进行all-gather的（其实从效率角度来说也没必要gather） ，但是fsdp是要恢复完整的参数的(因为fsdp没有列切行切的，就是直接flatten以后直接切） 所以在前向的时候需要进行all-gather ； all-gather的反向传播的语义对应是reduce-scatter；

![[大模型 GPU 通信（二）：从 AllReduce 到 AllToAll，大模型-06.png]]

# AllToAll：不求和、不复制，只把数据送到正确的 GPU

前三种通信都在回答“如何得到一个完整结果”。AllToAll 的问题不同：

**每张 GPU 上都有一些属于其他 GPU 的数据，如何一次性完成重新分发？**

假设每个 Rank 都把本地输入分成 $P$ 份，$x_{ij}$ 表示“当前位于 Rank $i$，目标是 Rank $j$”的数据：

TEXT

```text
通信前：

GPU0: [x00, x01, x02, x03]
GPU1: [x10, x11, x12, x13]
GPU2: [x20, x21, x22, x23]
GPU3: [x30, x31, x32, x33]

通信后：

GPU0: [x00, x10, x20, x30]
GPU1: [x01, x11, x21, x31]
GPU2: [x02, x12, x22, x32]
GPU3: [x03, x13, x23, x33]
```

AllToAll 不做 Reduce，也不会让每张 GPU 得到相同结果。它只是把数据从“按来源切分”变成“按目的地切分”：

$$
\text{Sharded}{source}\rightarrow\text{Sharded}{destination}
$$

所以它其实最大的应用就是在moe上，在moe的时候，token位于每个ep分组上， 此时形状是 \[local\_token, hidden\_size\]， 然后根据算出来的moe router信息，派发到其他ep组上进行信息交换，这就是典型的**从“按来源切分”变成“按目的地切分”**。

## AllToAll 的通信量

设每张 GPU 的总发送缓冲区大小为 $S$，理想情况下平均分成 $P$ 份。发给自己的 $S/P$ 不需要经过链路，因此每张 GPU 的跨 Rank 发送量约为：

$$
V_{A2A}=\frac{P-1}{P}S
$$

从式子上看这个结果看起来和 AllGather、ReduceScatter 一样，但性能特征很不一样， AllGather 和 ReduceScatter 的块大小通常规则且一致，是非常均匀的，但是all2all会存在很多通信对，moe也不保证专家非常均匀， 最差情况下退化到all\_to\_one,另外MoE还会出现排序、打包、反排列等额外开销。

## MoE 和 AllToAll

回顾一下， MoE 有多个 Expert，Router 会为每个 token 选择 Top-K 个专家。

假设 Expert0～Expert3 分布在 4 张 GPU 上：

TEXT

```text
GPU0 上的 token：A → Expert2，B → Expert0
GPU1 上的 token：C → Expert3，D → Expert2
GPU2 上的 token：E → Expert0，F → Expert1
GPU3 上的 token：G → Expert1，H → Expert3
```

Router 只负责决定目标，真正把 token hidden states 送到专家所在 GPU 的是 AllToAll。

一次 MoE 前向通常包含两次 AllToAll：

TEXT

```text
原始 token 顺序
      ↓
Router + Top-K
      ↓
本地 Permute / Pack
      ↓
AllToAll Dispatch
      ↓
token 到达对应 Expert GPU
      ↓
Expert GEMM
      ↓
AllToAll Combine
      ↓
本地 Unpermute
      ↓
恢复原始 token 顺序并按路由权重合并
```

反向传播会沿相反方向传递梯度，因此通常又有两次对应的 AllToAll。也就是说，一个标准 MoE Layer 的一次完整前向加反向，通常会看到 4 次 token 相关的 AllToAll：

| 阶段              | 通信目的                              |
| --------------- | --------------------------------- |
| 前向 Dispatch     | 把 token hidden states 送到目标 Expert |
| 前向 Combine      | 把 Expert 输出送回 token 原始 Rank       |
| 反向 Combine 的梯度  | 把输出梯度送回 Expert Rank               |
| 反向 Dispatch 的梯度 | 把输入梯度送回 token 原始 Rank             |

Megatron Core 的 MoE 文档将 NCCL AllToAll 作为标准 EP>1 的 token dispatcher；DeepEP 一类通信库则继续优化跨节点 Dispatch/Combine、低精度传输、流量不均衡以及通信与计算重叠。

![[大模型 GPU 通信（二）：从 AllReduce 到 AllToAll，大模型-07.png]]

# 通信带宽到底应该怎么看？

集合通信中的“带宽”经常有三种不同含义。

## 1\. 链路理论带宽

例如上一篇提到的 PCIe、NVLink 或 InfiniBand 标称带宽。它描述硬件链路的能力，但不等于应用一定能达到的速度。

## 2\. Algorithm Bandwidth

`nccl-tests` 中的 `algbw` 使用最直观的计算方式：

$$
BW_{alg}=\frac{S}{T}
$$

其中 $S$ 是这次操作定义下的逻辑 Tensor 大小，$T$ 是执行时间。它适合回答：

**一个大小为 **$S$** 的训练 Tensor，执行这次集合通信需要多久？**

## 3\. Bus Bandwidth

不同集合通信为了完成一个大小为 $S$ 的逻辑操作，真实经过链路的字节数不同，不能直接拿 `algbw` 与硬件峰值比较。因此 `nccl-tests` 用一个与原语相关的修正系数计算 `busbw`。

设 Rank 数为 $P$，在理想均匀通信模型下：

| 原语            | 单卡逻辑输入/输出                             | 单卡理论链路发送量                | \`busbw / algbw\` |
| ------------- | ------------------------------------- | ------------------------ | ----------------- |
| AllReduce     | 输入 $S$，输出 $S$     | $2(P-1)S/P$            | $2(P-1)/P$     |
| ReduceScatter | 输入 $S$，输出 $S/P$     | $(P-1)S/P$            | $(P-1)/P$     |
| AllGather     | 输入 $S/P$，输出 $S$     | $(P-1)S/P$            | $(P-1)/P$     |
| AllToAll      | 每卡总发送 $S$，总接收 $S$ | 跨 Rank 部分约 $(P-1)S/P$ | $(P-1)/P$     |

以 8 张 GPU 为例：

TEXT

```text
AllReduce 修正系数：2 × 7 / 8 = 1.75
AllGather 修正系数：7 / 8 = 0.875
ReduceScatter 修正系数：7 / 8 = 0.875
AllToAll 修正系数：7 / 8 = 0.875
```

所以出现 `busbw` 大于 `algbw` 并不意味着带宽凭空增加了，而是两者衡量的对象不同。

## 相同通信量，为什么 AllToAll 可能更难跑满？

虽然 AllGather、ReduceScatter 和 AllToAll 的理论修正系数都是 $(P-1)/P$，但 AllToAll 往往包含更多通信对，而且每一对的数据量可能更小、更不均匀。

因此判断通信性能不能只看总字节数，还要同时看：

- Message Size：单个消息是否太小；
- Number of Peers：同一时刻要和多少 Rank 通信；
- Load Balance：各 Rank 收到的数据是否均匀；
- Locality：流量停留在 NVLink 域内，还是经过跨节点网络；
- Overlap：通信能否与 GEMM 重叠；
- Contention：不同通信组是否争抢同一张 NIC。

# 从通信到训练

之前有说过，其实这些集合通信原语都是在做tensor的布局转化，可以理解为**沿某个维度切分参数、激活值或 token，然后在算子边界把布局转换成下一步计算需要的样子。**

| 并行方式 | 切分对象                | 主要通信数据              | 典型通信原语                                |
| ---- | ------------------- | ------------------- | ------------------------------------- |
| DP   | Batch               | 梯度、参数分片             | AllReduce；或 ReduceScatter + AllGather |
| TP   | Hidden、Head、Vocab   | 激活值及其梯度             | AllReduce、AllGather、ReduceScatter     |
| SP   | Sequence/Token 维激活值 | 激活值及其梯度             | AllGather、ReduceScatter               |
| EP   | Expert              | token hidden states | AllToAll / AllToAllv 类通信              |
| PP   | Layer               | micro-batch 激活值及梯度  | Send/Recv，通常不是集合通信                    |

为了避免不同框架实现差异造成歧义，本文采用 **Megatron 风格的一维 Tensor Parallel、与 TP 配套的 Sequence Parallel，以及标准 MoE Expert Parallel** 作为基准。这部分内容之前在大模型基础里也讲过一些，但当时侧重前向，没说反向；

## TP(Tensor Parallel)

一个线性层可以写成：

$$
Y=XW
$$

Tensor Parallel 不复制整份 $W$，而是把权重矩阵切到多个 GPU 上。最常见的是 Column Parallel 和 Row Parallel。

### 列切 Column Parallel Linear

Column Parallel 沿输出维切分权重：

$$
W=[W_0,W_1,\ldots,W_{P-1}]
$$

每张 GPU 都使用完整输入 $X$，只计算一部分输出：

$$
Y_r=XW_r
$$

于是输出天然沿 Hidden 维分片：

TEXT

```text
GPU0: Y0 = XW0
GPU1: Y1 = XW1
GPU2: Y2 = XW2
GPU3: Y3 = XW3

完整输出 Y = concat(Y0, Y1, Y2, Y3)
```

如果下一个算子能够直接消费分片输出，就不需要立刻 AllGather。Megatron 的 \`ColumnParallelLinear\` 默认 \`gather\_output=False\`，正是为了避免不必要的通信。

但在反向传播中，每张 GPU 只能算出输入梯度的一部分：

$$
dX_r=dY_rW_r^T
$$

完整输入梯度需要把所有部分相加：

$$
dX=\sum_{r=0}^{P-1}dX_r
$$

因此在**未开启 Sequence Parallel** 的经典 TP 中：

TEXT

```text
ColumnParallelLinear
前向：通常无通信，输出保持 Hidden 分片
反向：对 dX 执行 AllReduce
```

### Row Parallel Linear（行切）

Row Parallel 沿输入维切分权重，并让输入 $X$ 以相同方式切分：

$$
X=[X_0,X_1,\ldots,X_{P-1}]
$$

每张 GPU 计算一个完整形状的局部结果：

$$
Y_r=X_rW_r
$$

但这些只是 Partial Result，完整输出需要相加：

$$
Y=\sum_{r=0}^{P-1}Y_r
$$

因此：

TEXT

```text
RowParallelLinear
前向：对局部输出 Y_r 执行 AllReduce
反向：dX_r 可以在本地计算，通常无需 TP 集合通信
```

Column Parallel 和 Row Parallel 恰好互补：

| 线性层切法           | 前向           | 反向           |
| --------------- | ------------ | ------------ |
| Column Parallel | 通常无通信        | dX AllReduce |
| Row Parallel    | 输出 AllReduce | 通常无通信        |

Megatron TP 的一个 Transformer Layer 中：

| 模块/算子                  | 权重切分            | 前向通信         | 反向通信         |
| ---------------------- | --------------- | ------------ | ------------ |
| LayerNorm / RMSNorm    | 通常复制            | 无            | 无            |
| QKV Projection         | Column Parallel | 无            | dX AllReduce |
| Attention Core         | 按 Head 本地计算     | 无            | 无            |
| Output Projection      | Row Parallel    | 输出 AllReduce | 无            |
| LayerNorm / RMSNorm    | 通常复制            | 无            | 无            |
| MLP Up/Gate Projection | Column Parallel | 无            | dX AllReduce |
| GeLU / SwiGLU          | 本地              | 无            | 无            |
| MLP Down Projection    | Row Parallel    | 输出 AllReduce | 无            |
| Residual / Dropout     | 本地              | 无            | 无            |

其实我们不需要记那么多，我们只要记住， 如果激活值是\[seq, k\*dim\]那就是行切，如果是\[seq, dim\] 那就是列切；

行切只有两种情况：Attention 大O矩阵 、 MLP的w2矩阵；

![[大模型 GPU 通信（二）：从 AllReduce 到 AllToAll，大模型-08.png]]

## SP(Sequence Parallel)

只有开了tp才能开sp， TP 切分了线性层权重和部分中间激活值结果， 但经典 TP 中 LayerNorm、Dropout、Residual 等区域的激活值仍在每个 TP Rank 上复制。Sequence Parallel 进一步沿 Sequence/Token 维切分这些激活值， 唯独有个例外： attention计算的时候， sp仍然要all-gather全序列，否则语义不对；

这部分涉及到我未来在文章里做的一些tp/sp改造工作， 所以印象特别深刻

TEXT

```text
完整激活形状：[Tokens, Hidden]

GPU0: [Tokens 0   ... T/P-1, Hidden]
GPU1: [Tokens T/P ... 2T/P-1, Hidden]
GPU2: [Tokens 2T/P ... 3T/P-1, Hidden]
GPU3: [Tokens 3T/P ... T-1, Hidden]
```

### SP 如何改写 Column Parallel？

Column Parallel 需要每张 GPU 使用完整的输入 token 集合执行本地权重分片 GEMM。因此：

TEXT

```text
前向：Sequence-Sharded X
         ↓ AllGather
      完整 Sequence X
         ↓ Column Parallel GEMM
      Hidden-Sharded Y
```

反向则是它的共轭操作：各 Rank 产生局部 $dX_r$，但最终不需要让所有 Rank 都保存完整 $dX$，而是直接 ReduceScatter 回 Sequence-Sharded 状态：

TEXT

```text
反向：局部完整形状 dX_r
         ↓ ReduceScatter
      Sequence-Sharded dX
```

所以 Column Parallel 从：**无 SP：前向无通信，反向 AllReduce** 变成：**有 SP：前向 AllGather，反向 ReduceScatter**

### SP 如何改写 Row Parallel？

Row Parallel 的每张 GPU 会产生完整形状的 Partial Output。没有 SP 时，需要 AllReduce，让每个 Rank 都得到完整输出；开启 SP 后，可以直接 ReduceScatter，让不同 Rank 只得到属于自己的 token 分片：

TEXT

```text
前向：Partial Output
         ↓ ReduceScatter
      Sequence-Sharded Output
```

反向是对应的 AllGather：

TEXT

```text
反向：Sequence-Sharded dY
         ↓ AllGather
      完整 Sequence dY
         ↓ Row Parallel Backward
      Hidden-Sharded dX
```

所以 Row Parallel 从：**无 SP：前向 AllReduce，反向无通信** 变成：**有 SP：前向 AllGather，反向 ReduceScatter**

**总之就记住一点，开了sp，前行就得all-gather， 反向就得reduce scatter；**

### **SP有没有增加通信量**

从链路数据量的理想模型看，一对 ReduceScatter + AllGather 与一次 Ring AllReduce 相同：

$$
\frac{P-1}{P}S+\frac{P-1}{P}S
=
2\frac{P-1}{P}S
$$

**但实际上并非严格零开销，原因有2点：**

1. collective call 的位置变了。虽然理论通信量相等， 但原来的 AllReduce 可能是一个 NCCL collective，现在变成两个不同位置的 collective，所以kernel launch、同步点、通信 overlap 的行为都会改变。
2. 前面也提到过，现代 NCCL 的 AllReduce 未必是简单 Ring。可能使用 Tree、NVLS、CollNet 等算法

## EP （Expert Parallel)

TP 和 SP 主要是在规则的矩阵维度上切分 Tensor，通信量通常可以提前计算。EP 则根据 Router 的动态选择切分 token，通信模式由当前 batch 的路由结果决定。

在不进一步对单个 Expert 做 Tensor Parallel 的基本 EP 中，各算子的通信可以概括为：

| MoE 阶段                    | 前向通信               | 反向通信                                  | 说明                                     |
| ------------------------- | ------------------ | ------------------------------------- | -------------------------------------- |
| Router Projection / Top-K | EP 维通常无 token 集合通信 | 路由器参数梯度按其复制组做 AllReduce/ReduceScatter | Router 在本地 token 上计算所有 Expert 分数       |
| Permute / Pack            | 无，GPU 本地重排         | 无                                     | 按目标 Expert 整理发送缓冲区                     |
| Token Dispatch            | AllToAll           | 反向 AllToAll                           | token hidden states 发往 Expert 所在 Rank  |
| Expert Up/Gate GEMM       | 无                  | 无                                     | Expert 参数和 token 已在本地；若启用 ETP，另有 TP 通信 |
| Expert Activation         | 无                  | 无                                     | 本地计算                                   |
| Expert Down GEMM          | 无                  | 无                                     | 同上                                     |
| Token Combine             | AllToAll           | 反向 AllToAll                           | Expert 输出回到 token 原始 Rank              |
| Unpermute + Weighted Sum  | 无                  | 无                                     | 恢复 token 顺序并合并 Top-K 输出                |

### ETP(Expert tensor Parallel)

从2025年开始megatron就支持并行折叠了，这是跟sglang推理最大的不同，sglang是总rank=tp \* ep , 但megatron里ep是分开算的，开了tp不会作用到moe上， 如果想作用到moe上就得开etp； 这时候如果开了ETP，就会同时存在两类通信：

**EP Group**：AllToAll，把 token 送到正确 Expert

**ETP Group：**AllReduce 或 AllGather/ReduceScatter，完成 Expert 内部的分片 GEMM

## 表格汇总

| 并行模式  | 关键算子/边界                     | 前向通信             | 反向通信             | 通信对象                     |
| ----- | --------------------------- | ---------------- | ---------------- | ------------------------ |
| TP    | Column Parallel Linear      | 通常无              | AllReduce dX     | 激活值梯度                    |
| TP    | Row Parallel Linear         | AllReduce 输出     | 通常无              | 局部矩阵乘结果                  |
| TP    | Attention Core / Activation | 无                | 无                | 每卡本地 Head 或 Hidden 分片    |
| TP+SP | Column Parallel Linear      | AllGather 输入     | ReduceScatter dX | Sequence-Sharded 激活值     |
| TP+SP | Row Parallel Linear         | ReduceScatter 输出 | AllGather dY     | Sequence-Sharded 激活值     |
| SP    | Norm / Dropout / Residual   | 无                | 无                | 本地 token 分片              |
| EP    | Token Dispatch              | AllToAll         | AllToAll         | token hidden states 及其梯度 |
| EP    | Expert GEMM（无 ETP）          | 无                | 无                | 已路由到本地的 token            |
| EP    | Token Combine               | AllToAll         | AllToAll         | Expert 输出及其梯度            |
| ETP   | Expert Column/Row Linear    | 与普通 TP 类似        | 与普通 TP 类似        | Expert 内部激活值             |

# 总结

总而言之， **通信原语就是张量布局的转换器**

- 局部结果需要求和，并且所有人都要完整结果 就是 all-reduce
- 局部结果需要求和，但每个人只保留一部分 就是 reduce-scatter
- Tensor 已经分片，但下一步计算需要完整视图 就是all gather
- 数据不需要求和，只需要按照目标 GPU 重新分发, all to all

# 参考资料

- [NVIDIA NCCL：Collective Operations](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/usage/collectives.html)
- [NVIDIA nccl-tests：Performance reported by NCCL tests](https://github.com/NVIDIA/nccl-tests/blob/master/doc/PERFORMANCE.md)
- [NVIDIA Technical Blog：Understanding NCCL Tuning to Accelerate GPU-to-GPU Communication](https://developer.nvidia.com/blog/understanding-nccl-tuning-to-accelerate-gpu-to-gpu-communication/)
- [Megatron Core：Tensor Parallel Mappings](https://docs.nvidia.com/megatron-core/developer-guide/latest/apidocs/core/core.tensor_parallel.mappings.html)
- [Megatron Core：Tensor Parallel Layers](https://docs.nvidia.com/megatron-core/developer-guide/latest/apidocs/core/core.tensor_parallel.layers.html)
- [Megatron Core：Mixture of Experts](https://docs.nvidia.com/megatron-core/developer-guide/latest/user-guide/features/moe.html)
- [Megatron-LM: Training Multi-Billion Parameter Language Models Using Model Parallelism](https://arxiv.org/abs/1909.08053)
- [Reducing Activation Recomputation in Large Transformer Models](https://arxiv.org/abs/2205.05198)
