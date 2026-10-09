---
title: "一文看懂 Attention 演进史：从 MHA、GQA、MLA 到 DeltaNet、KDA 与 Hybrid Attention"
source: "AINLP"
category: "Inbox"
group: "机器学习"
author: "寒江雪"
url: "https://mp.weixin.qq.com/s/RBsvDYoGdgE1abo78llvmw"
published: 2026-10-09T21:29:24+08:00
saved: 2026-10-09T21:30:41+08:00
folo_key: "url::https://mp.weixin.qq.com/s/RBsvDYoGdgE1abo78llvmw"
tags:
  - "folo"
  - "Inbox"
  - "AINLP"
---

# 一文看懂 Attention 演进史：从 MHA、GQA、MLA 到 DeltaNet、KDA 与 Hybrid Attention

> [!info] AINLP · Inbox · 2026-10-09 21:29 · [原文](https://mp.weixin.qq.com/s/RBsvDYoGdgE1abo78llvmw)

> **Attention 的本质不是一个公式，而是一套 Context Memory Architecture。**

### 为什么 Attention 已经不只是 Attention？

从 2017 年 Transformer 横空出世，到今天的 DeepSeek、Qwen、Kimi，Attention 已经经历了非常大的变化。

我们先后见到了：

> MHA、MQA、GQA、MLA、FlashAttention、Linear Attention、RetNet、Mamba、GLA、DeltaNet、Gated DeltaNet、KDA、Gated Attention、MoBA、NSA、DSA、CSA、HCA……

如果只是一个个记名字，很容易越学越乱。

但如果换一个角度：

> **把 Attention 看成“模型如何保存、更新、检索上下文记忆”的机制。**

整个技术演进会突然变得非常清晰。

因为过去十几年 Attention 一直在解决同一个问题：

如 何 让 模 型 用 尽 可 能 低 的 成 本 ， 保 存 并 读 取 尽 可 能 多 的 上 下 文 信 息 ？

于是今天的 Attention 已经逐渐从：

演变成：

这也是理解 MLA、DeltaNet、KDA、DSA、CSA/HCA 的最好方式。

- - -

# 一、Attention 最本质的问题：模型到底该去哪里找信息？

先抛开所有复杂公式。

假设当前 Token 是：

> “他把订单号发给了客服，然后客服帮他查询……”

模型生成“查询”时，需要知道：

> “订单号”是什么？

这其实是一个信息检索问题。

当前 Token：

历史信息：

模型要做：

于是 Attention 本质可以理解成：

其中：

- Q：我要找什么？

- K：哪个位置包含相关信息？

- V：找到之后拿什么信息？

这就是 Attention 最核心的思想。

- - -

# 二、从 2014 到 2017：Attention 为什么诞生？

## 1\. Bahdanau Attention：解决“固定长度记忆”的瓶颈

在早期机器翻译中，Encoder 往往需要把整个输入序列压缩成一个固定长度向量，再交给 Decoder。

问题非常明显：

```
一句话  

Token1 Token2 Token3 ... TokenN  
             │  
             ▼  
       一个固定向量  
             │  
             ▼  
          Decoder  
```

输入越长，固定向量的信息瓶颈越明显。

Bahdanau 等人在 2014 年提出 Attention，让 Decoder 在生成每一个 Token 时动态访问 Encoder 的不同位置，而不是只依赖一个固定长度向量。(\[DOI\]\[1\])

因此 Attention 最早解决的问题其实是：

- - -

# 三、2017：Transformer——Attention 成为整个模型的发动机

2017 年，《Attention Is All You Need》发表。

Transformer 最重要的事情，并不是第一次发明 Attention，而是：

> **直接把 Attention 变成了整个序列建模架构的核心。**

论文中，Transformer 放弃 RNN/CNN，完全依赖 Attention 进行序列建模，并获得了更好的并行训练能力。

标准 Scaled Dot-Product Attention：

- - -

# 四、为什么需要 Multi-Head Attention？

如果只有一个 Attention Head，那么模型只能在一个表示空间中建模关系。

但是语言中的关系太多了。

例如：

```
Head 1：主谓关系  

Head 2：指代关系  

Head 3：变量依赖  

Head 4：局部语法  

Head 5：长距离依赖  

Head 6：语义关联  
```

所以：

最后：

这就是：

Multi-Head Attention。

这个阶段 Attention 的核心目标是：

> **尽可能增强表示能力。**

但随后，一个新的问题越来越严重：

> **Attention 太贵了。**

- - -

# 五、LLM 时代真正让 Attention 头疼的东西：KV Cache

训练的时候，Attention 最大的问题通常是：

因为：

会产生：

的注意力矩阵。

但是在 LLM Decode 阶段，还有一个更加现实的问题：

# KV Cache。

假设模型已经生成：

```
Token1  
Token2  
Token3  
...  
TokenN  
```

下一步只来了一个新 Token。

新的 Query：

需要和所有历史：

计算。

所以模型把过去的：

缓存下来。

于是：

```
               历史 Token  
                   │  
                K / V  
                   │  
                   ▼  
              KV Cache  
                   │  
                   │  
新 Token ───────→ Q  
                   │  
                   ▼  
               Attention  
```

问题是：

上下文越长，Cache 越大。

而且 Decode 时不断读取这些 KV，所以瓶颈往往不仅是显存容量，还包括：

也就是：

> GPU 能不能足够快地把历史 KV 读出来。

- - -

# 六、第一条优化路线：先减少“每个 Token 的 KV”

于是出现了 MQA 和 GQA。

- - -

## 1\. MQA：所有 Query 共用 KV

MHA：

```
Q1 → K1,V1  
Q2 → K2,V2  
Q3 → K3,V3  
Q4 → K4,V4  
```

MQA：

```
Q1 ─┐  
Q2 ─┤  
Q3 ─┼──→ K,V  
Q4 ─┘  
```

Query 还是很多个，但：

因此 Cache 大幅下降。

Shazeer 的 MQA 工作核心就是减少增量解码中 K/V Tensor 的内存带宽开销。(\[NeurIPS 会议录\]\[3\])

- - -

# 七、GQA：MHA 与 MQA 之间的折中

MQA 太激进。

所以后来出现 GQA：

```
Q1 Q2 Q3 Q4 → KV1  

Q5 Q6 Q7 Q8 → KV2  

Q9 Q10 Q11 Q12 → KV3  

...  
```

比如：

于是：

| 方法  | Q Heads | KV Heads |
| --- | ------- | -------- |
| MHA | 32      | 32       |
| GQA | 32      | 8        |
| MQA | 32      | 1        |

GQA 的目标就是：

> **尽量少存 KV，同时不要牺牲太多表达能力。**

GQA 工作显示，在合理分组下可以取得接近 MHA 的质量，同时获得接近 MQA 的效率。

于是出现第一条路线：

- - -

# 八、第二条路线：能不能干脆不保存每个 Token？

这是 Attention 发展史真正的分水岭。

前面的思路都是：

> 历史 Token 还是保存，只不过少存一点。

Linear Attention 则开始问：

> **为什么一定要逐 Token 保存？**

- - -

# 九、Linear Attention：把整个历史变成一个 State

标准 Attention：

如果去掉 Softmax，采用可结合的 Kernel Attention，可以得到：

定义：

那么：

于是：

这意味着：

```
Token1 ─┐  
Token2 ─┤  
Token3 ─┤  
...     ├──→ S  
TokenN ─┘  
```

最后只需要保存：

而不需要：

因此：

相对于 sequence length (L) 来说，不再增长。

Katharopoulos 等人在 2020 年的 Linear Transformer 工作中系统展示了如何利用矩阵结合律，把 Attention 改写成线性复杂度并得到递归形式。

- - -

# 十、这里出现了 Attention 世界第一个真正的大矛盾

现在有两种 Memory。

## Softmax Attention

```
Token1 ─→ K1,V1  
Token2 ─→ K2,V2  
Token3 ─→ K3,V3  
...  
TokenN ─→ KN,VN  
```

优点：

> 每一个历史 Token 都可以直接访问。

缺点：

- - -

## Linear Attention

```
Token1 ─┐  
Token2 ─┤  
Token3 ─┤  
...     ├──→ S  
TokenN ─┘  
```

优点：

Memory。

缺点：

> 所有历史都必须写进同一个固定大小状态。

于是：

成为现代 Attention 架构最核心的矛盾之一。

- - -

# 十一、RetNet：把“Attention”和“RNN”连接起来

2023 年 RetNet 提出了一个很有意思的思路：

> **同一个机制既可以用 Parallel Attention 形式训练，也可以用 Recurrent 形式推理。**

RetNet 同时支持：

- Parallel

- Recurrent

- Chunkwise recurrent

其中 recurrent 形式能够让推理状态大小相对于序列长度保持 (O(1))，而训练仍然可以利用并行计算。

这揭示了一个非常重要的事实：

> **Attention 与 RNN 并不一定是完全割裂的两个世界。**

它们在某些数学形式上可以相互转换。

- - -

# 十二、GLA：Linear Attention 开始学会“遗忘”

Linear Attention 有一个天然问题：

它基本上是在：

> 不断往 Memory 里面写。

但现实记忆不是这样的。

有些信息：

> 应该长期保存。

有些信息：

> 应该快速遗忘。

于是出现：

# Gated Linear Attention

加入：

这样的衰减机制：

这样：

```
旧 Memory  
   │  
   ▼  
 Gate  
   │  
   ├── 保留  
   └── 遗忘  
```

GLA 工作系统研究了 data-dependent gating，并提出了硬件高效训练算法。

这一步非常重要：

- - -

# 十三、Mamba：另一条“不要 Attention”的路线

与此同时，SSM 体系快速发展。

Mamba 的关键思想是：

> 让状态空间模型的参数依赖当前输入，从而能够根据 Token 内容选择传播或遗忘信息。

Mamba 将传统 SSM 的固定动态变成 input-dependent selective state space，从而增强了处理离散语言的 content-based reasoning 能力，并实现线性序列复杂度和高吞吐推理。(\[DOI\]\[8\])

所以：

```
Transformer  
     │  
     └── Softmax Attention  

  
SSM  
     │  
     └── State Transition  

  
Mamba  
     │  
     └── Input-dependent State Transition  
```

- - -

# 十四、Mamba-2：Attention 与 SSM 开始真正“数学握手”

Mamba-2 更进一步。

论文《Transformers are SSMs》指出：

> SSM 与 Transformer Attention 在某些形式下存在深层数学联系。

通过 State Space Duality，可以把这两类模型放进一个统一框架中。Mamba-2 的核心层相比 Mamba 的 selective SSM 在论文实验中实现了显著的速度提升，同时保持具有竞争力的语言建模能力。

这是一个非常关键的认知：

并不是两个完全独立的世界。

- - -

# 十五、DeltaNet：Linear Attention 开始真正解决“如何修改记忆”

普通 Linear Attention：

它有一个很严重的问题。

假设之前已经记住：

现在又出现：

普通累加：

结果可能变成：

> 老信息和新信息叠在一起。

而 Delta Rule 的思想是：

> **先看看这个 Key 当前在 memory 中对应什么，再根据误差去修正。**

先计算：

再得到：

最后：

这其实非常像：

> 在线做一次“小型梯度下降”，把当前 Key 对应的 Value 修正到正确结果。

DeltaNet 工作指出，Delta Rule 相比简单 additive update 更适合 associative recall；同时通过硬件友好的并行算法解决了原始 DeltaNet 难以沿 sequence length 并行训练的问题。

所以：

本质就是：

> **从“累加记忆”升级成“可纠错记忆”。**

- - -

# 十六、Gated DeltaNet：再给 DeltaNet 加一套“遗忘机制”

2024 年，Gated Delta Networks 出现。

它发现：

> Gating 和 Delta Rule 是互补的。

Delta Rule 擅长：

Gating 擅长：

于是得到：

可以简单理解成：

```
                    Memory  
                      │  
             ┌────────┴────────┐  
             │                 │  
          Delta              Gate  
             │                 │  
       精确修改内容          遗忘旧信息  
             │                 │  
             └────────┬────────┘  
                      ▼  
                   State S_t  
```

Gated DeltaNet 论文报告，它在语言建模、in-context retrieval、长度外推和长上下文理解等多个任务上优于此前的 Mamba2/DeltaNet 基线，并验证了与其他注意力机制做 Hybrid 的价值。

到这里：

- - -

# 十七、但 Gated DeltaNet 仍然没有解决最根本的问题

它拥有：

这样一个固定大小的 State。

因此：

这是巨大的优势。

但问题也在这里：

> **无论上下文是 10K、100K 还是 1M Token，最后都是一个固定大小的状态。**

比如：

```
用户姓名 = 张三  

订单号 = A123456  

退款金额 = 999  

地址 = 上海浦东……  

大量上下文  
```

最终都要：

这意味着：

> 无限历史被压进有限状态。

因此必然出现：

这也是为什么 Gated DeltaNet 的论文以及后续工作一直把 retrieval、associative recall、long-context memory 作为重点问题。

- - -

# 十八、这一点特别重要：为什么不能说 DeltaNet “全面碾压 MLA”？

因为两者解决的问题不一样。

- - -

## MLA

它保留：

> **每一个历史位置。**

只是把每个位置压缩得更小。

- - -

## DeltaNet

它保留：

> **一个汇总后的 Memory State。**

所以：

```
MLA：  

T1 T2 T3 T4 T5 ... TN  
│  │  │  │  │      │  
↓  ↓  ↓  ↓  ↓      ↓  
c1 c2 c3 c4 c5 ... cN  

  
DeltaNet：  

T1 T2 T3 T4 ... TN  
 │  │  │  │      │  
 └──┴──┴──┴──────┘  
          ↓  
          S  
```

因此：

而：

- - -

# 十九、2024：MLA——DeepSeek 选择“压缩显式记忆”

DeepSeek-V2 提出了 Multi-Head Latent Attention。

它没有选择：

> 把全部历史压成一个 State。

而是选择：

> **仍然保留每个 Token，但把 K/V 压缩到低维 Latent。**

标准 MHA：

MLA：

缓存：

需要时：

因此：

```
Hidden  
  │  
  ▼  
KV Encoder  
  │  
  ▼  
cKV  
  │  
 ┌┴───────┐  
 ▼        ▼  
WUK      WUV  
 │        │  
 ▼        ▼  
 K        V  
```

DeepSeek-V2 的论文报告，MLA 将 KV Cache 压缩到 Latent 表示，在相较 DeepSeek 67B 的配置对比中把 KV Cache 降低了 93.3%。

- - -

# 二十、MLA 真正漂亮的地方：可以直接做 Weight Absorption

假设：

那么：

可以写成：

于是可以把：

吸收到 Query 侧。

这样：

> 推理时不需要把完整 K 真正恢复出来。

Value 侧也是类似的思想。

所以 MLA 不是：

```
压缩  
↓  
解压  
↓  
Attention  
```

而更接近：

```
Compressed latent  
       │  
       ├──────────────┐  
       │              │  
       ▼              ▼  
  Query absorption   Output absorption  
       │              │  
       └──────┬───────┘  
              ▼  
          Attention  
```

这才是 MLA 真正厉害的地方。

- - -

# 二十一、但 MLA 为什么还不够？

因为：

它只是把：

压小。

假设一个 Token 只需要 576 个 cache elements。

那么：

Token：

个元素。

但是：

Token：

个元素。

所以：

而不是：

这就是后来 DeepSeek 继续搞 DSA、CSA、HCA 的真正原因。

- - -

# 二十二、这里可以建立一个非常重要的“四维 KV Cache 公式”

粗略地说：

其中：

- (L)：Token 数

- (D)：每 Token 的 Cache 维度

- (N\_{layer})：需要保存多少层

- (B)：每元素多少 Byte

于是：

### GQA / MQA

主要降低：

- - -

### MLA

进一步降低：

- - -

### DSA / MoBA / CSA

开始攻击：

或者：

- - -

### Cross-Layer Reuse

攻击：

- - -

### FP8 / FP4

攻击：

而：

### DeltaNet / KDA

更加激进：

这就是现代 Attention 为什么看起来越来越复杂：

> **大家其实是在攻击不同的成本维度。**

- - -

# 二十三、FlashAttention 又是什么？

FlashAttention 经常和 MLA、GQA 混在一起，其实这是一个巨大的误区。

FlashAttention 不是：

> “换了一种 Attention 公式。”

它更接近：

> **如何让原本的 Attention 在 GPU 上更高效地计算。**

标准 Attention：

FlashAttention 使用 tiling，将中间计算尽量留在 GPU SRAM 中，减少 HBM ↔ SRAM 的数据搬运。论文将其定义为 IO-aware exact attention。

所以：

而：

这两个可以同时使用。

- - -

# 二十四、长上下文还有第三条路线：Sparse Attention

如果：

一个 Query 真的需要看 100 万 Token 吗？

答案通常不是。

所以出现：

- - -

# 二十五、从 Longformer 到 BigBird：最早的稀疏路线

例如 Longformer：

```
当前 Token  
 │  
 ├── 最近窗口  
 ├── 少量 Global Token  
 └── 不再看全部历史  
```

Longformer 将滑动窗口 Attention 与任务相关的 Global Attention 结合，把复杂度降低到线性规模。

BigBird 则结合：

- Local

- Random

- Global

等稀疏模式，把标准 Attention 的二次依赖降低为线性规模，同时分析了其表达能力。

不过早期 Sparse Attention 的问题也很明显：

> **稀疏模式往往依赖人工设计。**

例如：

```
每个 Token 看前 512 个  
```

这种规则当然便宜，但可能把真正重要的信息挡在门外。

- - -

# 二十六、MoBA：让模型自己决定应该看哪些 Block

2025 年 Kimi 提出了：

# MoBA —— Mixture of Block Attention

核心思想是：

> **把 Context 切成 Block，再像 MoE 一样让 Router 选择最相关的 Block。**

```
                  Query  
                    │  
                    ▼  
                 Router  
                    │  
       ┌────────────┼────────────┐  
       ▼            ▼            ▼  
     Block1       Block8       Block21  
       ✕            ✓            ✓  
                    │  
                    ▼  
              Sparse Attention  
```

这就把：

变成：

MoBA 本质上是：

Kimi 报告其在百万 Token 长上下文中可以显著降低 Attention 计算，并且还能在 Full / Sparse Attention 之间切换。

- - -

# 二十七、MInference：不改模型，只优化长上下文 Prefill

还有一个容易被忽略的路线：

# MInference

它并没有试图重新设计模型结构，而是观察：

> 长上下文 Attention 本身就具有明显稀疏模式。

MInference 识别了包括 A-shape、Vertical-Slash、Block-Sparse 等模式，并通过动态稀疏索引和 GPU Kernel 加速 Prefill。其论文报告，在 1M Token prompt 上，部分设置可实现最高约 10× 的 Prefill 加速。

所以这里又出现一个非常重要的分类：

和：

并不一样。

- - -

# 二十八、NSA：把 Sparse Attention 变成原生可训练模块

2025 年 DeepSeek 还提出：

# Native Sparse Attention

NSA 不只是推理阶段“偷偷跳过一些计算”。

它直接把稀疏 Attention 做进训练过程。

NSA 的设计可以理解为：

```
                   History  
                      │  
          ┌───────────┼───────────┐  
          │           │           │  
          ▼           ▼           ▼  
      Compression   Selection   Sliding Window  
          │           │           │  
          └───────────┼───────────┘  
                      ▼  
                Sparse Attention  
```

即：

- 粗粒度 Compression

- 细粒度 Selection

- Local Sliding Window

并且原生参与训练。NSA 论文报告，它在长序列上的 Forward、Backward 和 Decode 都能获得明显收益。

- - -

# 二十九、DSA：DeepSeek 把 Sparse Attention 真正带到主模型

DeepSeek-V3.2 的核心 Attention 创新之一是：

# DeepSeek Sparse Attention

DSA 使用轻量 Indexer：

然后真正的 Attention 只访问 Top-K 历史 Token。

```
1,000,000 Tokens  
        │  
        ▼  
   Lightning Indexer  
        │  
        ▼  
   Top-K Tokens  
        │  
        ▼  
  Real Attention  
```

所以：

而：

这是两个完全不同的维度。

DeepSeek-V3.2 正式把 DSA 作为核心技术之一，用于显著降低长上下文 Attention 的计算复杂度。

- - -

# 三十、到这里，MLA 与 Sparse Attention 已经可以同时理解

假设：

- - -

## 没优化

```
1M Token  
×  
完整 KV  
×  
全部 Attention  
```

- - -

## MLA

```
1M Token  
×  
压缩 KV  
×  
全部 Attention  
```

解决：

- - -

## DSA

```
1M Token  
×  
压缩 KV  
×  
只访问 Top-K  
```

解决：

同时：

- - -

# 三十一、DeepSeek V4：开始直接压缩“Sequence Dimension”

如果说 MLA 是：

> 压缩一个 Token。

那么 DeepSeek V4 的 CSA/HCA 开始做：

> **压缩一整段 Token。**

DeepSeek V4 的公开模型资料把 Hybrid Attention 定义为：

其中：

### CSA

Sequence-wise 压缩 KV，再通过稀疏 Indexer 选择相关压缩块。

### HCA

更加激进地压缩 Sequence，然后在压缩后的表示上进行 Dense Attention。

官方 Hugging Face 文档给出的实现细节包括：

- CSA 使用低压缩率、重叠窗口 + Indexer Top-k

- HCA 使用更高压缩率、非重叠窗口

- HCA 不再使用 Indexer，而直接对压缩后的历史做 Dense Attention

- 两者还保留局部 Sliding Window 分支。(\[Hugging Face\]\[19\])

这其实意味着：

进一步：

- - -

# 三十二、一个非常直观的例子

假设：

- - -

### MLA

```
T1   T2   T3   T4 ... T1M  
 │    │    │    │       │  
 ▼    ▼    ▼    ▼       ▼  
c1   c2   c3   c4 ... c1M  
```

还是：

个 Entry。

- - -

### CSA

例如每：

个 Token 做一个压缩块：

```
T1 T2 T3 T4     → C1  

T5 T6 T7 T8     → C2  

...  

1M Token  
  ↓  
250K Compressed Entries  
```

然后：

- - -

### HCA

更加激进：

```
128 Tokens  
     ↓  
1 Compressed Entry  
```

于是：

级别的 compressed history。

当然，真实模型的具体压缩比例、分支配置和实现更复杂，这里只是为了理解思想。

- - -

# 三十三、与此同时，另一条路线已经发展出了 Gated Attention

现在我们重新回到 Softmax Attention。

有一个问题：

> Attention 已经算出来了，但模型是否真的应该让这些信息全部流出去？

于是产生：

# Gated Attention

它仍然计算：

但再增加：

最后：

可以理解为：

```
Q K V  
 │  
 ▼  
Attention  
 │  
 ▼  
Attention Output  
 │  
 ×  
 │  
Gate  
 │  
 ▼  
Final Output  
```

2025 年的 Gated Attention 研究系统比较了大量 gating 位置，发现简单的 head-specific sigmoid gate 放在 SDPA 输出之后就可以有效提升性能，同时增强非线性、query-dependent sparsity，并减轻 attention sink。

注意：

虽然都有“Gated”，但作用完全不同。

- - -

# 三十四、Gated Attention 和 Gated DeltaNet 到底差在哪里？

这是最容易混淆的问题。

## Gated Attention

Gate 控制：

> **Attention 输出多少信息。**

- - -

## Gated DeltaNet

Gate 控制：

> **Memory 保留多少、忘掉多少。**

- - -

所以：

```
Gated Attention  
      ↓  
控制 Output  

  
Gated DeltaNet  
      ↓  
控制 Memory  

  
MLA  
      ↓  
控制 Memory Representation  
```

三个 Gate/Attention 其实处于不同位置。

- - -

# 三十五、Qwen3-Next：一个非常典型的 Hybrid

Qwen3-Next 采用：

具体采用 48 层结构：

也就是：

```
DeltaNet  
DeltaNet  
DeltaNet  
Attention  

DeltaNet  
DeltaNet  
DeltaNet  
Attention  

...  
```

官方模型资料明确给出了这一 3:1 Hybrid Layout，并同时给出了 Gated Attention 的 Q/KV Head 配置以及 Gated DeltaNet 的状态维度。(\[Hugging Face\]\[21\])

为什么？

因为它其实在做：

也就是：

### Gated DeltaNet

负责：

> 便宜地维护上下文状态。

### Gated Attention

负责：

> 偶尔做精确的全局检索。

这不是“谁淘汰谁”。

而是：

> **不同 Memory，各自负责擅长的工作。**

- - -

# 三十六、KDA：你上一版最重要的遗漏

KDA 就是：

# Kimi Delta Attention

它是 Kimi Linear 的核心机制之一。

最重要的一句话：

它不是和 Gated DeltaNet 平级、完全独立的一条路线。

而是：

Kimi Linear 论文明确说，KDA 是 Gated DeltaNet 的扩展，通过更细粒度的 gating 来更有效地使用有限的 finite-state RNN memory。

- - -

# 三十七、Gated DeltaNet 与 KDA 最大的区别是什么？

这是 KDA 最应该记住的一点。

可以简化理解：

### Gated DeltaNet

Memory decay：

通常是比较粗粒度的。

可以理解为：

> “这个 Head 整体该忘多少？”

- - -

### KDA

进一步变成：

也就是：

> **不同 channel 可以有不同的 decay。**

因此：

```
Gated DeltaNet：  

Head  
████████████████  
   ↑  
一个 gate  

  
KDA：  

Head  
█ █ ██ █ ███ █  
↑ ↑ ↑  ↑ ↑  
不同 channel  
不同 decay  
```

Kimi Linear 官方资料把 KDA 描述为给每个 key channel 提供自己的 forget gate，使 recurrent state 的衰减从 per-head 变成 per-channel。(\[GitHub\]\[23\])

这一步本质上是在解决：

> **有限容量 Memory 怎样分配给不同维度的信息？**

- - -

# 三十八、所以 KDA 的逻辑非常漂亮

如果把 DeltaNet 的 Memory 看成一个矩阵：

那么 Gated DeltaNet 是：

> 对这个 Memory 做比较粗粒度的整体衰减。

KDA 则开始说：

> 不同 Key channel 的重要性可能完全不一样。

于是：

这就是 KDA 的核心创新。

- - -

# 三十九、Kimi Linear 为什么不是“全 KDA”？

因为 KDA 虽然很强，但它仍然是：

依然存在：

> 精确长距离 Retrieval Capacity

问题。

所以 Kimi Linear 的架构不是：

```
全部 KDA  
```

而是：

也就是：

```
KDA  
KDA  
KDA  
MLA  

KDA  
KDA  
KDA  
MLA  
...  
```

Kimi Linear 论文明确采用这种 layer-wise Hybrid Architecture，并报告相比全 MLA 在长上下文下最高可减少 75% KV Cache，并实现最高约 6× 的 Decode Throughput。

所以：

- - -

# 四十、现在把 Qwen3-Next 和 Kimi Linear 放一起看

突然非常有意思。

## Qwen3-Next

它的 Hybrid：

```
3 × Recurrent Memory  
+  
1 × Full Attention  
```

- - -

## Kimi Linear

它的 Hybrid：

```
3 × Improved Recurrent Memory  
+  
1 × Compressed Explicit Memory  
```

- - -

这两条路线其实高度相似：

> **绝大多数层用便宜的 recurrent memory，少部分层用精确 retrieval。**

差别只在于：

> “那一小部分高精度 Memory 应该使用什么？”

Qwen：

Kimi：

- - -

# 四十一、所以 Hybrid Attention 的真正含义不是“两个 Attention 混一起”

而是：

例如：

```
                         Context Memory  
                               │  
              ┌────────────────┼────────────────┐  
              │                │                │  
              ▼                ▼                ▼  
         Recurrent         Compressed        Sparse  
          Memory             Memory          Retrieval  
              │                │                │  
         GDN / KDA            MLA          DSA / CSA  
              │                │                │  
              └────────────────┼────────────────┘  
                               ▼  
                        Cross-Layer Reuse  
                               │  
                               ▼  
                         Quantized Cache  
```

这已经越来越像数据库 / 计算机系统里的：

> 多级 Memory Hierarchy。

- - -

# 四十二、2026：Gated DeltaNet-2 又在解决什么？

GDN 还有一个很有意思的问题。

在传统 Gated DeltaNet 中：

> erase 和 write 之间存在一定耦合。

也就是说：

> “我要擦掉多少旧信息”和“我要写入多少新信息”，并不完全独立。

于是 2026 年提出：

# Gated DeltaNet-2

核心思想：

它进一步把：

- erase gate

- write gate

分开。

并且都可以做到 channel-wise：

```
Old Memory  
    │  
    ▼  
Erase Gate  
    │  
    ▼  
剩余 Memory  
    +  
Write Gate  
    │  
    ▼  
New Information  
```

GDN-2 论文把它描述为 Gated DeltaNet 与 KDA 的进一步推广：KDA 是其在 gates 退化到共同标量等条件下的特例，而 GDN-2 可以分别控制 channel-wise erase 与 channel-wise write。

于是这条路线继续变成：

- - -

# 四十三、这个演进特别有意思

你可以把它类比成优化器。

最开始：

后来：

开始根据 error 定向修改。

然后：

控制遗忘。

再后来：

让不同维度有不同记忆能力。

最后：

于是越来越接近：

> **一个真正可编程的神经 Memory。**

- - -

# 四十四、2026 的另一个方向：甚至开始给 Memory 建模“置信度”

这就是近期非常有意思的 Kalman Delta Networks。

KDN 的核心观察：

> DeltaNet 每次看到一个新 Token，都必须决定要不要覆盖旧记忆。

但它并不知道：

> 当前 Memory 对这个位置到底有多确定？

所以 KDN 把 recurrent associative memory 写成线性高斯状态空间模型，引入 Kalman 式的不确定性估计，让更新强度依赖 accumulated evidence 和 observation reliability。

从思想上来说：

```
传统 DeltaNet：  

New Token  
   ↓  
我要写多少？  

  
KDN：  

New Token  
   ↓  
我现在有多确定？  
   ↓  
Memory 对此信息有多可信？  
   ↓  
我应该写多少？  
```

这说明 Recurrent Memory 研究已经开始从：

> “怎么更新？”

继续进入：

> **“我对当前记忆到底有多确定？”**

这是很值得关注的方向。

- - -

# 四十五、2026 还有 FG²-GDN：连 Write Gate 都开始细粒度化

还有一个非常新的方向：

# FG²-GDN

其核心思想是：

KDA 已经把 decay 做成 channel-wise。

但：

这个 update strength 仍然比较粗。

于是 FG²-GDN 继续：

甚至进一步把：

> erase

和：

> write

继续解耦。

这条研究线说明一个非常明确的趋势：

而不再只是“怎么做到 O(1)”。

- - -

# 四十六、于是现在 Attention 其实出现了两条非常明显的路线

## 路线 A：Compressed Explicit Memory

代表：

特点：

> 历史还在，只不过不断变小。

核心能力：

代价：

- - -

## 路线 B：Recurrent Memory

代表：

特点：

> 历史被压成一个动态状态。

核心能力：

代价：

> 固定 Memory Capacity。

- - -

# 四十七、这两个路线为什么最终开始融合？

答案非常简单：

### Recurrent Memory

擅长：

> 低成本保存大量长期状态。

### Explicit Attention

擅长：

> 精确找到某个历史信息。

所以：

比：

更合理。

- - -

# 四十八、这其实就是 Hybrid Attention 的第一性原理

你可以把整个模型想象成：

```
               Context  
                  │  
        ┌─────────┴─────────┐  
        │                   │  
        ▼                   ▼  
  Recurrent Memory     Explicit Memory  
        │                   │  
     KDA/GDN             MLA  
        │                   │  
        ▼                   ▼  
   长期状态维护         精确检索  
        │                   │  
        └─────────┬─────────┘  
                  ▼  
               Output  
```

这就是为什么现在越来越多模型开始 Hybrid。

实际上，DeltaNet 自己的论文在实验中就已经验证了把 DeltaNet 层与 Sliding Window Attention 或少量 Global Attention 混合能够获得更好的整体表现。(\[NeurIPS 会议录\]\[27\])

这并不是 2025 年才突然出现的想法。

而是从 Linear Attention 研究开始就一直存在：

> **Fixed State + Attention**

只是到了 Qwen3-Next、Kimi Linear，这个想法真正进入了更大规模语言模型。

- - -

# 四十九、所以 DeepSeek 为什么没有“直接拥抱 DeltaNet”？

现在答案就非常明确了。

DeepSeek 的长期路线更偏向：

于是它不断做：

而不是：

DeepSeek 选择的是：

> **我不要放弃可显式寻址的历史，我要把这种历史记忆压缩到极致。**

这就是 DeepSeek 为什么一直在“折腾 KV Cache”。

- - -

# 五十、到了 DeepSeek V4.1-Flash，这个思路已经走得非常远

2026 年 DeepSeek-V4.1-Flash 明确把 KV Cache 进一步优化到：

论文报告其全局 KV Cache footprint 达到：

约为 DeepSeek-V4-Flash 的：

同时持久化 KV Cache footprint 约降低到：

官方发布信息也明确强调，V4.1-Flash 相比上一代需要约 1/4 的 HBM 和 1/8 的 SSD KV Cache 存储。

这恰好证明：

> **MLA 从来不是 KV Cache 优化的终点。**

- - -

# 五十一、为什么 DeepSeek 还要继续压？

因为 Cache 实际是：

MLA 只是把：

压了很多。

但：

还在。

于是：

### DSA

减少访问的 Token。

### CSA/HCA

进一步压缩 Sequence。

### CSA2 / Cross-Layer Reuse

减少层间重复存储。

### FP4

降低每个元素的 Byte。

DeepSeek-V4.1-Flash 的论文把这种 KV 优化进一步推向跨层复用与低精度缓存。

所以 DeepSeek 路线可以理解成：

逐维攻击。

- - -

# 五十二、再回头看 KDA，就更容易理解了

KDA 并不是在回答：

> “怎样把 KV Cache 压小？”

它回答的是：

> **“既然我要把历史压缩到固定大小 State 中，怎样把这个 State 用得更聪明？”**

于是：

### DeltaNet

- - -

### Gated DeltaNet

- - -

### KDA

- - -

### GDN-2

- - -

### KDN

这一整条路线都在解决：

- - -

# 五十三、所以 MLA 与 KDA 其实是在优化两个完全不同的 Memory

可以直接这样记：

|           | MLA                | KDA                        |
| --------- | ------------------ | -------------------------- |
| 路线        | Softmax Attention  | Linear/Recurrent Attention |
| Memory    | 显式历史               | 固定状态                       |
| State     | 每个 Token 一个 latent | 所有 Token 共用一个 state        |
| Cache     | (O(L))             | (O(1))                     |
| Retrieval | 强                  | 受限                         |
| 重点        | Compression        | Memory Update              |
| 核心创新      | Latent KV          | Fine-grained Gating        |
| 代表模型      | DeepSeek           | Kimi Linear                |

所以：

而：

- - -

# 五十四、把所有技术放到一张“完整进化地图”

现在可以把整个 Attention 领域真正串起来：

```
                          Attention  
                              │  
             ┌────────────────┴────────────────┐  
             │                                 │  
       Softmax Attention                 Linear / Recurrent  
             │                                 │  
      ┌──────┼──────┐                 ┌────────┼────────┐  
      │      │      │                 │        │        │  
     MHA    GQA    MQA              Linear   RetNet   Mamba  
      │      │      │                 │        │        │  
      └──────┴──────┘                 │        │        │  
             │                        └────┬───┴────────┘  
             ▼                             │  
            MLA                           GLA  
             │                             │  
             │                          DeltaNet  
             │                             │  
             │                       Gated DeltaNet  
             │                             │  
             │                             ▼  
             │                            KDA  
             │                             │  
             │                        GDN-2 / FG²  
             │  
             ├──── DSA  
             │  
             ├──── MoBA  
             │  
             ├──── NSA  
             │  
             └──── CSA / HCA  
                         │  
                         ▼  
                  Cross-Layer Reuse  
                         │  
                         ▼  
                     FP8 / FP4  

  
       ┌─────────────────────────────────────────┐  
       │                                         │  
       │            Hybrid Attention             │  
       │                                         │  
       │   Recurrent Memory + Explicit Retrieval │  
       │                                         │  
       └─────────────────────────────────────────┘  
```

- - -

# 五十五、还有几个名字必须知道，但不要和架构混为一谈

到这里还需要补充几个容易被混淆的东西。

- - -

## 1\. FlashAttention

不是新的 Attention 语义。

它主要是：

优化计算和 HBM 访问。

- - -

## 2\. MInference

不是新的训练架构。

它主要是：

尤其优化长上下文 Prefill。(\[微软\]\[29\])

- - -

## 3\. Ring Attention

也是偏系统层面的长上下文方法：

> 把不同 Sequence Block 分布到不同 GPU，再通过 Ring 交换 KV。

它解决的主要是：

而不是改变 Attention 的数学定义。

- - -

# 五十六、于是今天讨论 Attention，至少要分成四层

这是最容易让人学乱的地方。

## 第一层：Attention Semantics

例如：

- MHA

- GQA

- MQA

- MLA

- Gated Attention

回答：

> **Attention 本身怎么算？**

- - -

## 第二层：Memory Architecture

例如：

- Linear Attention

- RetNet

- DeltaNet

- Gated DeltaNet

- KDA

回答：

> **历史状态怎么保存？**

- - -

## 第三层：Sparse Retrieval

例如：

- Longformer

- BigBird

- MoBA

- NSA

- DSA

- CSA/HCA

回答：

> **到底需要看哪些历史？**

- - -

## 第四层：Systems / Kernel

例如：

- FlashAttention

- FlashLinearAttention

- MInference

- Ring Attention

回答：

> **怎么在 GPU / 多 GPU 上更快地算？**

这四层千万不要混在一起。

- - -

# 五十七、真正理解 Attention 演进后，可以得到一个非常漂亮的统一公式

对于一个 LLM 来说，Context Memory 的成本可以粗略理解成：

同时还有一个另外的维度：

即：

> 每一个 Query 到底访问多少历史。

于是各种技术在攻击不同因素：

而：

这张公式基本可以解释绝大多数现代 Attention 优化。

- - -

# 五十八、为什么“固定 State”仍然不是终极答案？

因为它牺牲了：

MLA：

```
Query  
 ↓  
c1 c2 c3 c4 c5 ... cN  
 ↓  
找相关位置  
```

DeltaNet：

```
Query  
 ↓  
一个 State  
```

于是：

### MLA

更像：

> 数据库 + 索引。

### DeltaNet

更像：

> 一个高度压缩的工作记忆。

所以未来最可能的方向，不是：

而是：

- - -

# 五十九、我认为未来 Attention 最可能出现的架构形态

可能不是：

```
所有层 = MLA  
```

也不是：

```
所有层 = KDA  
```

而更可能是：

```
                 Token Stream  
                      │  
     ┌────────────────┼────────────────┐  
     │                │                │  
     ▼                ▼                ▼  
 Recurrent         Compressed        Sparse  
  Memory             Memory         Retrieval  
     │                │                │  
   KDA/              MLA           DSA/CSA  
   GDN  
     │                │                │  
     └────────────────┼────────────────┘  
                      ▼  
                Cross-Layer Memory  
                      │  
                      ▼  
                  FP4 Cache  
```

甚至进一步：

> 不同层自己学习自己应该属于哪一种 Memory。

也就是说：

本身都可以成为模型学习的一部分。

- - -

# 六十、甚至“Memory”本身也可能有多级

未来很可能出现：

### Level 1：Working Memory

例如：

> KDA / GDN

- - -

### Level 2：Compressed Explicit Memory

例如：

> MLA / CSA

- - -

### Level 3：Sparse Long-Term Memory

只保留最重要的：

> DSA / Sparse Retrieval

- - -

### Level 4：Persistent Memory

存到：

> Host / SSD

- - -

于是整个 LLM 的 Context 可能越来越像：

```
                 Context Memory  
                       │  
       ┌───────────────┼────────────────┐  
       │               │                │  
       ▼               ▼                ▼  
 Working Memory   Explicit Memory   Persistent Memory  
       │               │                │  
      KDA             MLA/CSA          SSD Cache  
       │               │                │  
       └───────────────┼────────────────┘  
                       ▼  
                 Retrieval System  
```

这时候：

> Attention 已经真的变成 Memory Architecture 了。

- - -

# 六十一、把整个 Attention 历史浓缩成一句话

如果从 2014 年到今天，只记住一条主线：

动 态 读 取 多 头 表 达 共 享 压 缩 固 定 状 态 记 忆 可 学 习 遗 忘 可 纠 错 记 忆 细 粒 度 记 忆 稀 疏 检 索 序 列 压 缩 跨 层 复 用 低 精 度

对应的核心技术就是：

另一条线：

再另一条线：

最后：

开始把这些路线重新组合起来。

- - -

# 六十二、最终真正值得记住的是：Attention 正在从“算子”变成“记忆系统”

过去我们会问：

> Q、K、V 怎么算？

现在应该问：

> 历史信息怎么存？

再进一步：

> 哪些信息应该长期保存？

再进一步：

> 哪些信息应该忘掉？

再进一步：

> 哪些信息需要精确检索？

再进一步：

> 每层是否需要保存完整历史？

再进一步：

> 这些状态能不能压缩？

再进一步：

> 能不能用 FP4 存？

最终：

这才是今天重新理解 Attention 的正确方式。

- - -

# 六十三、最后用一张表，把所有核心技术彻底记住

| 技术               | 核心问题           | Memory 形式           | 相对 (L) 的特点        |
| ---------------- | -------------- | ------------------- | ----------------- |
| MHA              | 表达能力           | 每 Head 独立 KV        | Cache (O(L))      |
| MQA              | KV 太多          | 共享 KV               | Cache (O(L))，常数更小 |
| GQA              | MQA 掉点         | 分组共享 KV             | Cache (O(L))      |
| MLA              | KV 还是太大        | Latent KV           | Cache (O(L))，常数很小 |
| FlashAttention   | GPU IO 太贵      | 不改变语义               | 优化计算/IO           |
| Linear Attention | 二次 Attention   | Fixed State         | State (O(1))      |
| RetNet           | 固定状态 + 并行训练    | Recurrent State     | (O(1))            |
| GLA              | 不会遗忘           | Gated State         | (O(1))            |
| DeltaNet         | Memory 只能累加    | Delta State         | (O(1))            |
| Gated DeltaNet   | 遗忘 + 更新        | Gated State         | (O(1))            |
| KDA              | Gate 太粗        | Channel-wise State  | (O(1))            |
| GDN-2            | Erase/Write 耦合 | Channel-wise Memory | (O(1))            |
| Gated Attention  | Attention 输出控制 | Full KV             | (O(L))            |
| MoBA             | 访问太多 Block     | Sparse KV Access    | 降访问量              |
| NSA              | 稀疏需要训练         | Sparse Attention    | 降 Attention 计算    |
| DSA              | 长上下文太多 Token   | Top-K Retrieval     | 降访问量              |
| CSA              | Token 太多       | Compressed Sequence | 降 Sequence        |
| HCA              | Sequence 还是太长  | Heavy Compression   | 更激进压缩             |
| Cross-Layer      | 每层重复存储         | Shared KV           | 降 Layer           |
| FP8/FP4          | Cache 元素太贵     | Quantized KV        | 降 Byte            |
| Hybrid           | 单一 Memory 不够   | 多种 Memory           | 综合优化              |

- - -

# 六十四、结语：真正的下一代 Attention，可能根本不再叫 Attention

到这里，你会发现一个非常有意思的事情。

最早的 Attention 是：

> **我该看哪里？**

MHA：

> **我能从多个角度看。**

GQA/MQA：

> **能不能少存一点？**

MLA：

> **能不能把每个历史 Token 压得更小？**

Linear Attention：

> **能不能不保存每个 Token？**

DeltaNet：

> **能不能主动修改 Memory？**

Gated DeltaNet：

> **能不能主动忘记？**

KDA：

> **能不能让不同维度分别决定忘多少？**

GDN-2：

> **能不能把擦除和写入彻底分开？**

DSA/MoBA：

> **能不能只看真正相关的历史？**

CSA/HCA：

> **能不能先把整个历史压短？**

Cross-Layer：

> **能不能不同层之间共享 Memory？**

FP4：

> **能不能让 Memory 更便宜？**

最终这些问题汇聚成了一个更加底层的问题：

所以我越来越倾向于把今天的 Attention 理解成：

> **不是一种计算公式，而是一种 Context Memory Architecture。**

而 MHA、GQA、MLA、DeltaNet、KDA、DSA、CSA/HCA，并不是互相割裂的“新名词”。

它们其实都在围绕同一个目标优化：

从这个角度再去看 DeepSeek、Qwen、Kimi，你会发现：

**DeepSeek 在压缩显式 Memory。**

**Qwen 在引入 Recurrent Memory。**

**Kimi 在把两者组合起来。**

而整个 Attention 的下一阶段，很可能不是：

> “谁把谁淘汰？”

而是：

真正的下一代 Transformer，可能最终不会再有一个单一的“Attention 模块”。

它更可能拥有一整套：

# **Memory System。**
