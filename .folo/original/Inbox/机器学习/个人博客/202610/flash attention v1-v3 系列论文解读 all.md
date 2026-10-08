---
title: "flash attention v1-v3 系列论文解读[all]"
source: "个人博客"
category: "Inbox"
group: "机器学习"
url: "https://l1cachedell.github.io/blog/hpc/flash-attention-series/"
published: 2026-10-08T09:36:46+08:00
saved: 2026-10-08T09:37:18+08:00
folo_key: "url::https://l1cachedell.github.io/blog/hpc/flash-attention-series"
tags:
  - "folo"
  - "Inbox"
  - "个人博客"
---

# flash attention v1-v3 系列论文解读[all]

> [!info] 个人博客 · Inbox · 2026-10-08 09:36 · [原文](https://l1cachedell.github.io/blog/hpc/flash-attention-series/)

# flash attention v1-v3 系列论文解读\[all\]

 更新于 2025/9/3

[FA-v1](https://arxiv.org/abs/2205.14135)[FA-v2](https://arxiv.org/abs/2307.08691)[FA-v3](https://arxiv.org/abs/2407.08608)

[Tri-Dao Offcial Code](https://github.com/Dao-AILab/flash-attention)

# Flash Attention v1-v3 Series Paper Walkthrough\[all\]

 更新于 2025/9/3

[FA-v1](https://arxiv.org/abs/2205.14135)[FA-v2](https://arxiv.org/abs/2307.08691)[FA-v3](https://arxiv.org/abs/2407.08608)

[Tri-Dao Official Code](https://github.com/Dao-AILab/flash-attention)

# 1\. 前言[](#1-前言)

> 为什么攀登珠峰？因为山就在那里。

从很早以前入门HPC开始，就听说过flash attention[1](#user-content-fn-1)，后来做research intern的时候刚好flash attention v2[2](#user-content-fn-2)崭新问世，但当时也仅仅只是直接`pip install`使用，看到论文里面的公式、演示图没有太过深究。直到再后来在paddle实习的时候有了机会，leader让我复现的量化算子与flash attention v3[3](#user-content-fn-3)对比一下效果看看孰强孰弱，于是又接触了v3算子，因为紧张的任务排期，这次依然是`pip install`，还是没有去了解原理。

为了不再当\*`pip install` boy\*，正好当前有一定时间可以沉淀沉淀内力，决定把flash attention从v1到v3全都读一遍，整理成文档方便日后阅读。同时，最近也卡在kernel开发的瓶颈期，好像什么都会，又好像什么都不会。不如看看在教科书级别的实践里面，大家是怎么看待数据与kernel的、怎么利用最新feature的，**知行合一**。

诚然，前人有关flash attention的解读已经有很多了，虽然串在一起梳理的并不多，但是归并结合阅读，总能从拼凑出完整的知识线。受DefTruth哥的文章[4](#user-content-fn-4)启发，确定学习与梳理顺序：online softmax → FA1 → FA2 → FA3。

# 2\. Prerequisite: Softmax\[all\][](#2-prerequisite-softmaxall)

## 2.1 Safe Softmax[](#21-safe-softmax)

Attention的计算形式化表达中，展示了softmax函数的用途，实际上在推理的时候由于softmax操作的memory-bound特性，众多研究人员提出了该公式的优化方式。

Softmax最naive的公式是：$softmax(x_i) = \frac{e^{x_i}}{\sum_{j=1}^N e^{x_j}}$。一般情况下在实现的时候我们会warp-block reduce求得一个sum，然后再来进行逐元素的softmax操作。

原softmax在工程上的问题是，用fp16做训推时$x_i$这个指数可能特别大，而$e^{x_i}$就很容易溢出。如果需要做一个量化估量，fp16最高能表达65536的量级，在$x_i \ge 11$的时候$e^{x_i}$就能够溢出65536了。在这种情况下，safe softmax的概念被提出了。（顺便link一个[safe softmax的清晰实现](https://github.com/xlite-dev/LeetCUDA/blob/main/kernels/softmax/softmax.cu)）

Safe Softmax可以被表示为：

$$
\frac{e^{x_i}}{\sum_{j=1}^N e^{x_j}} \rightarrow \frac{e^{x_i - m}}{\sum_{j=1}^N e^{x_j-m}}
$$

所有的项的指数都减去一个$m$，而$m$则是所有$x_i$当中的最大值：$m = max(x_i)$，这样做的目的是保证**指数最后小于0**，这样就实际上大大降低了数值溢出的风险。

## 2.2 Online Softmax[](#22-online-softmax)

实际上，假设我们现在在cpu上（书面语：在普通编程模型而不是SIMT）实现这一版softmax，我们会发现起码要迭代3轮才能完成这一步骤：

- 第一遍for循环，找出集合${x_i}$中的最大值，记为$m$
- 第二遍for循环，累加，最后算出来一个sum 也就是$\sum_{j=1}^N e^{x_j-m}$这一块，作为分母
- 第三遍for循环，逐元素挨个与第二步算出来的分母做除法，写回结果空间

下面我们看看怎么做online运算[5](#user-content-fn-5)。

上面的式子里面，三次for loop都不能彼此融合，因为这是前后依赖关系，第二步依赖第一步找出全局最大值$m$，第三步依赖第二步计算全局sum。如果我们要想尽可能压低复杂度，减少对global memory的访问，只能自己动手魔改公式了——寻找出一种online的办法，使之与原公式近乎等价。

Online softmax提出的方案是：把$m$从全局最大，替换为“当前最大” $m_i$：以当前扫过的区域的最大值，作为$m$。这个过程可以形式化表达为：

$$
\frac{e^{x_i}}{\sum_{j=1}^N e^{x_j}} \rightarrow \frac{e^{x_i - m}}{\sum_{j=1}^N e^{x_j-m}} \rightarrow \frac{e^{x_i - m_i}}{\sum_{j=1}^N e^{x_j-m_i}}
$$

（*由于本文突出**串讲**，所以干脆把之前的公式也纳入进来，以便直观体验这个公式的演化进程。*）

到了这里，分母我们可以单独拎出来进行分析，记$d_S = \sum_{j=1}^S e^{x_j-m_S}$，$S$表示当前运算到的下标，有如下演化过程：

$$
d_S = \sum_{j=1}^S e^{x_j-m_S} = \sum_{j=1}^{S-1} e^{x_{j-1}-m_S} + e^{x_j-m_S}
$$

这一步即：把最后一项单独拎出来。

$$
= (\sum_{j=1}^{S-1} e^{x_{j-1}-m_{S-1}}) \times e^{m_{S-1}-m_S} + e^{x_j-m_S}
$$

$$
= d_{S-1} \times e^{m_{S-1}-m_S} + e^{x_j-m_S}
$$

到这里我们完成了对于递推公式的推导，让每一$d_S$都与$d_{S-1}$建立了联系。

这一步有什么用呢？用处就在：**前面三次循环的操作，我们可以把第一、二步融合起来，现在仅用两次for循环就能完成softmax的计算**。

- 第一遍for循环，计算扫描到目前为止的最大值$m_S$（方法就是直接与$m_{S-1}$比较就行）；并且可以联动算出到当前为止的sum。
- 第二遍for循环，把算出来总的sum进行逐元素除法

好，有关online softmax的内容我们先放一放，作为预备知识，接下来我们进入flash attention v1的阅读。

# 3\. Flash Attention-v1[](#3-flash-attention-v1)

Flash Attention v1（下称FA-v1）旨在提出一种针对所有GPU设备的通用算子，所以不可能以NVIDIA GPU作为主视角陈述文章内容，因此用到HBM和SRAM两个名词。HBM是GPU high bandwidth memory的简称，如果在nv GPU里面找到代名词，就是我们常说的`global memory`，SRAM是**所有on-chip memory**的代称，这个集合里面有$\{ 'shared \_memory', 'registers', 'caches' \}$等内容，并不仅仅只是`shared memory`，所以在这里要做一个区分与声明，有一个nv论坛的discussion[6](#user-content-fn-6)讨论这一内容。

> FA-v1的优化，主要是针对Attention计算中的I/O方式进行优化，利用SRAM进行针对性优化。同时扩展Flash Attention为一种block-sparse attention的计算方式，比现有所有的attention加速方法都快（FA-v1提出时间是2022年）。

当时linear attention[7](#user-content-fn-7)还没提出，注意力计算的复杂度还是随着序列长度`sequence length`（记为$N$）指数增长的，即$N^2$，现有的加速方案确实都通过各种各样的近似方法，尽可能降低了attention计算的复杂度，但是都比较关注理论上的FLOP提升，这些与真实运行时间（论文中称为`wall-clock speedup`）的观测结果并不一致，并且这些工作都倾向于忽略访存上带来的开销。

背景部分我们先不讨论，默认都是掌握一定GPU & CUDA & HPC基础的，本文只讲FA-v1-v3的优化思路与贡献。

## 3.1 标准Attention的工程建模[](#31-标准attention的工程建模)

在进行attention计算的时候，我们一般会接触到这样的tensor shape: `[bsz, n_head, seq_len, head_dim]`，其中最后两个dim，sequence length和head dimension，我们分别形式化表示为$N$和$d$。FA-v1里面使用如下几个公式概括attn计算过程：

$$
S = QK^T \in \mathbb{R}^{N \times N}, P = softmax(S) \in \mathbb{R}^{N \times N}, O = PV \in \mathbb{R}^{N \times d}
$$

在一般情况下，`seq_len`都是远大于`head_dim`的，比如我们经常讨论一个32K长度的长序列，模型的head\_dim却只有128，这时候$32 \times 1024 >> 128$，也就是$N >> d$。所以这个过程实际上是IO主导的memory-bound。标准Attention的计算过程，文章中是用伪代码的形式展现的，在这里用朴实的中文来梳理：

1. 从`global memory`中加载$Q, K$到每一个thread独有的`local memory`或者`register`里面（threads均摊$QK$的各个元素），做一次$S=QK^T$的GEMM，然后把计算结果$S$写回`global memoey`。
2. 再从`global memory`中取得$S$，计算$P = softmax(S)$，再把计算结果$P$写回`global memory`
3. 把$P, V$再从`global memory`里面加载进来，做$O = PV$的GEMM，然后写回$O$到`global memory`
4. 输出$O$

> 看完这一段，我觉得FA-v1对于Attention这样的建模方式会不会太plain了，实际上一个稍微懂一点HPC的人都不会这样设计kernel。三进三出`global memory`开销太大。如果在FA-v1推出之前大家都是这样在计算attention的话，那确实是遭罪了。

FA-v1确立核心目标：**减少从`global memory`的访问**。最好的办法就是一次从global memory进行load，整体计算完之后，写回global memory。虽然最终并没有这么理想化的一次读一次写，但是还是大幅度降低了原假设中，对于global memory的访问次数。

为了实现这一目标，FA-v1主要提出两个techniques: **Tiling和Recomputation**。

## 3.2 FA-v1概念中的tiling[](#32-fa-v1概念中的tiling)

在接着往下讲之前，我想先梳理一下入行HPC以来，遇到的各种各样的`tiling`的概念，**以免混淆**。

**什么是`tile`——最朴素的定义**

我们在做kernel优化的时候，朴素的编程模型（CUDA Programming Guide）会告诉你，我们的编程模型是grid-block-thread。在PTX的编程模型会告诉你，我们的编程模型是CTA-cluster-grid。`Tile`的概念在数据切分方案低于block size的时候提出，为了做**缓存局部友好性，或者对register友好**而采用的。比如一块block的数据是$256 \times 256$，他就会被分为$8 \times 8$或者$16\times16$的小小tile，再递交计算。

`Tile`是一种最小数据切分单位，也是一种优化strategy，算不上任何official programming guide中编程模型的一部分。

> 如果要找一个关键词，我想用`slice strategy`来代替

**`PTX`里面的`tile`**

接触过`SM90`以上，或者亲眼看过PTX文档的朋友们应该清楚，在底层的PTX指令中，把`.m16n16k16`, `.m32n8k16`这样固定大小的数据块，也称为`tile`，在这样的语境下，tile又俨然变成了\*\*编程时规定的特定硬件规格，\*\*它更多是 **指令级数据片形状/寄存器碎片布局** 的术语，决定fragment 的映射和指令吞吐，不直接承载“把数据放到 shared/register、如何流水线”的程序级语义。我们做`mma`或者`wmma`的时候，都会时刻关注数据的切分形态，以便于适配这样的大小，从而安排我们自己的kernel逻辑。

> 如果要找一个关键词，我想用`fragment size`来代替

**`tile-lang`里面的`tile`**

关注`tile-lang`[8](#user-content-fn-8)有一段时间了（很优秀的仓库），这是一个基于TVM[9](#user-content-fn-9)思想开发的domain-specific-language编译器，tile被提升为一等公民，`tile`在`tile-lang`中既是数据分块的概念，也是一种程序抽象与调度单元。

> 如果要找一个关键词，我想用`scheduling unit`来代替

**正题：`flash attention`里面的`tile`**

![[flash attention v1-v3 系列论文解读 all-01.webp]]

在FA-v1里面，一次性把$Q,K,V$全部load进入shared memory显然不现实。最好的办法就是把他们切分为一个又一个小小的block，如果用形式化表示，橙色的小块就可以表示为：$Q_{block}, K_{block}, V_{block}$。在运算的时候，仅需要把这些小block 加载进入shared memory即可，最后算出来$O_{block}$，在加回对应位置之前，使用right normalization factor（右归一化因子）对$O_{block}$进行放缩，再进行加回操作，这样就能得到正确计算结果了。

对于$\frac{QK^T}{\sqrt{d}}$进行softmax是一个难题，尽管我们已经有了online softmax方法，但是现在的情况是：对于需要softmax的矩阵，一共有`seq_len`行，`head_dim`列，现在要针对每一行进行softmax。如果是挨个进行softmax的话，就得进行`seq_len`次online softmax。虽然我们现在已经做了block切分化，依然要对这个小小的block进行online softmax。

在前人[10](#user-content-fn-10)的工作中，已经提出了优化方式：假设现在一共就两行，`[2, head_dim]`，每一行可以被单独拎出来作为一个向量，文中记为$x^{(1)}, x^{(2)}$，结合之前的online softmax公式，并且保证数值的稳定性，文章提出如下计算方式：

$$
m(x) = m([x^{(1)} \; x^{(2)}])  \quad \quad 这里表示两个向量concat
$$

$$
=max(m(x^{(1)}), m(x^{(2)}))
$$

$$
f(x) = [e^{m(x^{(1)})-m(x)}f(x^{(1)}) \quad e^{m(x^{(2)})-m(x)}f(x^{(2)})]
$$

其中，$f(x)$在这里就表示两个元素，分别计算完online softmax之后，分子部分的concat。

记 $l(x)$为sum，有：

$$
l(x) = l([x^{(1)} \quad x^{(2)}]) = e^{m(x^{(1)})-m(x)}l(x^{(1)}) + e^{m(x^{(2)})-m(x)}l(x^{(2)})
$$

最后是softmax：

$$
softmax(x) = \frac{f(x)}{l(x)}
$$

其实在这里公式有表述不清楚的地方，如果觉得晦涩难懂，请参考原文[1](#user-content-fn-1)以及DefTruth的博客[4](#user-content-fn-4)。

迎面而来有几个问题：

1. 怎么切分QKV？众所周知QKV在进行attention计算时，有`[seq_len, head_dim]`两个维度，按照哪个维度切？切多大？
2. 怎么做$Q_{block}, K_{block}, V_{block}$的运算？

**切分size与伪代码解读**

在HPC领域，涉及访存问题的时候，一般习惯性使用$B$指代block size，使用$M$来指代memory size，这在很多年前的研究（cahe-aware，cache-oblivious）中就是这样使用的，在FA-v1中依然不例外。对于每一个sequence，在运算过程中的每一个注意力头，我们可以如下形式化表达：

- $Q,K,V \in \mathbb{R}^{N \times d}$, `shared memory`的**总**大小是$M$
- block size设置为： $B_c = \lceil \frac{M}{4d} \rceil$, $B_r = min(\lceil \frac{M}{4d} \rceil,d)$

这就是回答第一个问题：切KV切多大一块。从公式推断：$\frac{M}{4d}$，这个是$K/V$列块的大小。在每次处理的时候，都需要把这样的K和V列块（*注意是列主序哦*）都放进shared mem里面，\*\*为什么是这样的size？\*\*不妨让我们来做一个简短的计算，假设我们使用fp16推理：

1. 一个block占用：$B_c \times d \times 2\;bytes$

2. 一个K一个V都有一个block，一共占用：$2 \times B_c \times d \times 2\;bytes = 4B_cd \quad bytes$

假设我们的shared memory大小是$M$，为了保证不溢出，所以就是 $B_c \le \frac{M}{4d}$，实际上根据之前使用过Flash Attention的实战经验，向上取整之后，$B_c$通常取16或者32，以适配`mma.m?n?k?`的PTX指令。

切Q又切多大一块呢？$min(\lceil \frac{M}{4d} \rceil,d)$，我们后面再解释为什么是这样一个size，现在继续往下叙述FA-v1的运算过程：

- 在HBM上初始化$O$矩阵，大小为$N \times d$；$l(x)$这个sum向量，大小为$N$；$m(x)$这个暂存最大值的向量，大小为$N$。
- 把$Q$沿着`seq_len`维度切分（在这里就回答了第一个问题：沿什么维度切分），每一块大小为$B_r$，一共切分得到$T_r = \lceil \frac{N}{B_r} \rceil$份blocks: $Q_1, Q_2, ..., Q_{T_r}$，每一份的大小都是 $B_r \times d$ （也就是$B_r$个token x head\_dim的数据）
- 把$K, V$沿着`seq_len`维度切分，（不过需要注意的是，seq\_len在KV中是列向量了，所以我们现在实际上切分得到的是一个列块）一共得到 $T_c = \lceil \frac{N}{B_c} \rceil$块blocks：$K_1, K_2, ..., K_{T_c}$, $V_1, V_2, ..., V_{T_c}$，每一块大小是$B_c \times d$ （也就是$B_c$个token x head\_dim的数据）
- 切分$O$, 与$Q$同理切分方式，得到 $O_1, O_2, O_3, ..., O_{T_r}$，每一份的大小是$B_r \times d$.
- 切分$l, m$这两个向量，在这里原文用的是$divide \; into \; blocks$的说法，得到$l_1, l_2, .., l_{T_r}$，与 $m_1, m_2, ..., m_{T_r}$每一份大小是$B_r$
  - 个人觉得不太准确，本身就是行向量，用`blocks`听起来感觉像是二维矩阵，实际上，也就是切分得到数组的切片而已。**如果叫做$divide \; into \; slices$的说法可能会更好理解一点。**

- - -

现在完成了数据的切分逻辑梳理，进入计算逻辑：

- Outer loop (这个在图中明确被橙色标注出来，主要用于推进KV block的): 把KV blocks从global memory加载到shared memory
  - Inner loop (也在图中被蓝色标注出来，主要用于推进$Q$和$O$ block的): 把对应的$Q_i, O_i$ block从global memory加载到shared memory；
    - 利用shared memory计算attention score：$S_{ij} = Q_iK_i^T \in \mathbb{R}^{B_r \times B_c}$
    - 利用shared memory计算$m_{ij} = rowmax(S_{ij}) \in \mathbb{R}^{B_r}$, $P_{ij}=e^{S_{ij}-m_{ij}} \in \mathbb{R}^{B_r \times B_c}$, $l_{ij}=rowsum(P_{ij}) \in \mathbb{R}^{B_r}$
    - 利用shared memory计算 $m_i^{new}, l_i^{new}$, 在这里就是动态维护这个向量
    - 我们之前说的缩放因子：$diag(l_i^{new})^{-1}(diag(l_i)e^{m_i-m_i^{new}}O_i + e^{m_ij-m_i^{new}}P_{ij}V_j)$，算完了之后写回$O_i$在global memory上对应位置
    - 把最新维护的$l_i^{new}, m_i^{new}$写回global memory
  - end inner loop
- end outer loop

## 3.3 Recomputation[](#33-recomputation)

我们现在确实采用了沿`seq_len`维度切分的办法，迎面而来的下一个问题是：如果我们想要保存$P = softmax(S)$，随着序列增长，最终预估会有 $O(N^2)$的显存需求，还有附带的巨量显存读写成本( I/O overhead)。我们分块读取的方法很好，计算完当前block就丢，但是如果想要一次性得到输出：$O = PV$，通常需要在`shared memory`上保留较多中间变量。

Flash Attention的目标之一，就是**把中间需要保存的所有数据都尽可能变小，最好能够压缩到$O(N)$**的数量级，思想是**把昂贵的I/O换成多做一点计算** （注意：领悟思想才是最重要的，这些思想与经验这有助于我们在开发的时候做出更高效的kernel）。显然这是合理的，因为GPU设备计算的开销就是要比IO开销更划算，这样一来，也能提升一些`wall-clock speed`。

Recomputation的步骤，可以被形式化为如下公式：

把任务拆成两次扫描`c-block`，第一次只算归一化因子，第二次把权重乘以$V$:

1. Pass-1: 对于每一个c-block，有：
   1. $m_t = max(x^{(t)}), \ell_t = \sum{e^{x^{(t)}-m_t}}$用在线公式与全局 $(m, \ell)$合并即可。
   2. 之后，只用保存每一行的$m, \ell$, 大小都是$O(N)$数量级，不用保存$S, P$
2. Pass-2: 直接计算归一化之后的权重：
   1. $\omega^{(t)}=\frac{e^{x^{(t)}-m}}{\ell}$
   2. 与$V$做乘法：$r \larr r + \omega^{(t)}V^{(t)}$
   3. 扫完所有块之后，输出 $O_i = r$

## 3.4 I/O Analysis[](#34-io-analysis)

证明的内容，此处就忽略了，感兴趣的朋友直接移步论文，此处我们记录结论。

- 标准Attention的IO复杂度为：$\Theta(Nd+N^2)$, $d$是`head_dim`
- Flash-Attention的IO复杂度为：$\Theta(\frac{N^2d^2}{M})$, $M$是shared memory的大小。这直接解释了FA-v1的优势：**IO 的减少，决定了加速**。

有关“block-sparse FlashAttention”我们此处不再解读，因为并不是任务主线，感兴趣的朋友可以自行探索。

# 4\. Flash Attention-v2[](#4-flash-attention-v2)

FA-v1的性能仅仅只能达到理论最大FLOPS峰值的25-40%，背后的原因是**没有针对block、thread、warp进行充分优化的partitioning策略，造成了低占用率、不必要的shared memory读写。** 注意这句话是abstract里面高度浓缩、提炼出来的一句，我们会在后面的分析里面讲清楚到底是出了什么问题、以及做了什么优化。

FA-v2主要在A100上展开，为了做极致的优化，首先就要了解A100硬件的参数是多少，这样才方便我们了解硬件的极限在哪。

A100有81GB的显存（global memory），$1.5-2.0 \; TB/s$的带宽，一共$192KB$的shared memory，108个SM，$19 \; TB/s$的on-chip带宽。鉴于L2 cache没有办法通过编程形式操控，文章提出优化的重心，围绕还是围绕global memory和shared memory展开。

> 对于FA-v1，我们会展开大量的公式以及探讨kernel优化的具体步骤，而对于FA-v2开始的解读，我们主要以记录思路为主，因为有了之前的铺垫，探讨起优化方案就更加顺畅，不至于脑中无法形成概念。但是后文不会像介绍Flash Attention-v1一样细致。
> 
> 这也是DefTruth大佬在文章中指点过：梳理FA-v1的时候要细致，主要是为了掌握原理。
> 
> 因此，对于后续的FA-v2，FA-v3，我们把重心放在介绍优化思路上。

## 4.1 改进算法，省略空间[](#41-改进算法省略空间)

从FA-v2的角度审视，之前的FA-v1算法的计算思路，还是需要保留太多的东西在shared memory里面，并且也多了一些计算步骤，FA-v2做了这样的优化：

1. 从$O^{(2)}=diag(\frac{\ell^{(1)}}{\ell^{(2)}})^{-1}O^{(1)} + diag(\ell^{(2)})^{-1}e^{S^{(2)}-m^{(2)}}V^{(2)}$变成 $\tilde{O}^{(2)}=diag(\ell^{(1)})^{-1}O^{(1)}+e^{S^{(2)}-m^{(2)}}V^{(2)}$ .需要注意的是在新版本的公式中，$\tilde{O}^{(2)}$是一个`un-scaled`的数据，这样我们只需要在循环的最后再对最后一个$\tilde{O}^{(last)}$，通过 $diag(\ell^{(last)})^{-1}$ 进行scale一次，最后得到最终的output。
2. 与此同时，我们也并不需要同时保存 $m^{(j)}, \ell^{(j)}$来进行backward pass，取而代之的，保存一个 $L^{(j)}=m^{(j)}+log(\ell^{(j)})$ （logsumexp）变量即可。

通过这样两步变化，同时节约了**计算步骤**和**shared memory使用空间**。

**处理causal mask**

注意力推理过程都是需要进行causal mask的，也就是下三角矩阵。因为FA-v1的时候就已经把序列按照 行块`r`和列块`c`切成了网格。对于任意行`r`和列`c`:

1. 如果 $c > r$，即整个块都在上三角，这个时候直接跳过。
2. 这样对于一个长序列，能够跳过一半的blocks，理论上运算量就可以直接降低 $2 \times$，考虑到还有数据加载、同步、流水线等固定开销，实测能够提升 $1.7-1.8 \times$ 的加速。

同时，对于“对角块”，做逐元素的掩码，即：per-row only 1 block masked。

1. 对于 $c < r$，整个块都在下三角，正常计算
2. 对于 $c > r$，跳过
3. 对于 $c = r$，就是对角块，这个block里面确实有一些数据在上三角，有一些数据在下三角，只需要在这一块内部应用逐元素的causal-mask，把块内 $j>i$的那些位置全都置为 $-\infin$。这样一来**每一行只需要在1个块里面做mask**，其他块要么全算/全跳，避免了大量的分支，也消除了一些流水线bubble。

## 4.2 改变并行策略[](#42-改变并行策略)

在FA-v1中，并行运算应用在`batch-size`和`num_head`两个维度。我们使用1个cuda block来计算一个seq的一个注意力头，这就意味着一共会发射 $bsz \times n \_heads$ 个 cuda block。如果每一个block都运行在一个SM上，假设现在我们用的设备是A100，一共有108个SM，这样的调度方式，在batch\_size特别大的时候还算高效，比如batch\_size > 80，**如果是batch size比较小，且 seq\_len特别长的时候，或者num\_heads比较少的时候（GQA）**，这个时候这样的并行应用策略就不是那么高效了。

这一次在推理的时候，为了解决这个问题，决定把`seq_len`这一个维度，也作为并行计算的维度。换言之，同一个sequence在切分成不同blocks以后，直接递交到不同的cuda-block （也就是分配到不同的SM里面去）计算，因为他们这个过程不需要互相传递信息，同时也保留针对`batch_size`和`num_head`这两个维度的并行。这样一来，无论是大batch、大小num\_head，还是长短序列，都能保证kernel以最高效的形式在运行，最大程度用好GPU的硬件吞吐。

这样的设计要实现，主要是通过交换kernel内部的loop的顺序来办到的。outer-loop用来覆盖row-blocks，inner-loop用来覆盖col-blocks。

## 4.3 改变warp对于数据的分区[](#43-改变warp对于数据的分区)

![[flash attention v1-v3 系列论文解读 all-02.webp]]

FA-v1的warp对于数据的获取方法，就是左边这种形式。Warp 1-4都可以获取Q-block，而对于K和V，各个warp取各自的数据，然后进行QKV计算。这样一来，每一个warp都要负责计算一个$QK^T$，并且也要和对应的$V_{slice}$进行运算，最后还要求不同warp之间有communication才方便加起来，最后形成结果，文章称之为 “Split-K” 方案。

这个方案不高效的点就在于，所有warp都需要把计算结果存储到shared memory里面去，同步（也许是`__syncthreads()`），最后求和。针对shared memory大量的读写，以及其中需要的同步，就是FA-v1的痛点，FA-v2准备把这一步都省了。

FA-v2的warp对于数据的获取方法，如右图所示。warp 1-4各自负责获取Q的一份slice，这次让KV变成warp 1-4都能获取的数据。这成为 “split-Q”方案。

这样一来，只需要每一个slice各自进行 $QK^T$, 然后与V相乘，就能得到对应的output，不需要跨warp进行沟通，也就自然不涉及太多的shared memory读写，还省去了同步成本，这样一来直接speed up。

到这里，FA-v2的优化方法已经粗略整理完毕。注意backward-pass的内容没有太细致的梳理，感兴趣的朋友请直接移步原文。

# 5\. Flash Attention-v3[](#5-flash-attention-v3)

FA-v3的提出，在abstract里面也提及，就是因为FA-v2在H100上表现太低了，GPU利用率仅仅只有35%。

FA-v3的提出，相信对于绝大部分人来说也是老生常谈的话题了：利用SM90开始有的针对tensor-core与`.tensor`原生数据类型\[^11\]的异步特性，TMA特性、FP-8混合精度推理支持，做出优化。具体的成绩如下：

1. 在使用FP16推理时运算速率达到740 TFLOPS，75%的H100利用率。
2. FP8的推理运算达到1.2 PFLOPS。
3. FA-v3里面的FP8推理精度比baseline 的FP8 Attention计算结果提升$2.6 \times$

## 5.1 前言[](#51-前言)

对于H100 GPU来说，编程模型已经有了很大变化。在这里我们在继续往下之前，先做一个简单梳理，这一份梳理并不只是来自文章，也来自于本人之前的使用经验，以近乎最好理解的方式展现给大家：

> 注：H100就是SM90 arch的GPU

1. 在SM90之前的GPU，我们的主流GPU编程模型还是grid-block-thread，kernel按照block为单位发射在SM上，shared memory是SM独有的，一个SM里面的所有运行threads可以共享。跨SM无法共享shared memory。
2. 在SM90以上的GPU，优化已经深入到了PTX层面，PTX的编程模型自顶向下依次是：grid-cluster-CTA\[^12\]，在这里CTA 可以等价理解为我们之前编程模型里面的 `block`的概念。一组CTAs组成一个cluster，**shared memory可以在cluster的层面共享，也就是说，现在一组SM组成一个cluster，一组SM可以共享shared memory**。
3. `TMA`，这个总是被大家提起的名词，实际上是Hopper架构 GPU上的一个独有的硬件单元，主要就是用来做数据搬运工作的，可以用来做数据从global memory到shared memory的异步拷贝。
4. `.tensor`是PTX编程中的原生数据类型。的确，在SM90以下，SM80以上的设备，可以使用`cp.async`指令，但是有一个细节：\*\*只能搬运普通数据，无法搬运`.tensor`数据。\*\*到了SM90以上的GPU，就可以使用`cp.async.bulk.tensor.{...}`指令，异步搬运一整块tile进入shared memory，进而供运算逻辑使用，这就是异步拷贝在指令层面的区别。
5. `WGMMA`成为新的指令簇。在此之前，我们仅仅用到过`wmma`，或者`mma` （推荐各位去看DefTruth大哥的LeetCUDA，里面全都有详细演示），在SM90以上的GPU中，可以使用`warp-grouped mma`，不同于`wmma`和`mma`仅仅只有GEMM功能, **`wgmma`是可以异步的**，比如：`wgmma.mma_async`是可以使用的。
6. `FP8` 数据类型的原生运算支持。
7. 其他：例如`setmaxnreg`这样的reallocate register，以及SFU (Special Function Unit) 用来计算softmax之类的操作，在FA-v3里面也被使用了起来。

其实H100以上的GPU提供了这么多特性，最核心的用法就是**利用异步特性，打满指令流水，尽可能减少运行时kernel的bubble，进而提高kernel的吞吐**。

## 5.2 Warp-spec[](#52-warp-spec)

最近好多面试官都问我有关这个问题，确实是被拷打得有点麻。这一小节的关键词挺多，比如`warp-specialization`, `Producer-Consumer`, `pingpong-scheduling`之类的。在FA-v3的开发方案中，编程模型已经变成了CTA的`ptx`模型，因此下面的语境都是基于CTA展开的：

具体的称谓、形式化表达，在FA-v1里面已经陈述得很清楚，在没有做intra-consumer overlapping的CTA视角是这样的：

- 如果是在生产者warp中：
  - 释放先验设置的所有寄存器
  - 发射：把$Q_i$从global memory 加载到shared memory
  - 完成了之后，commit上面这个异步操作，通知其他warp：对于$Q_i$的异步加载已经完成
  - for loop:
    - 等待第 $j \; \% \; s$个stage被`[消费者warp]`计算
    - 在第$j \; \% \; s$ 阶段的同时，发射 $K_i, V_i$的加载指令
    - 完成了之后，通知其他warp：对于 $K_i, V_i$的加载已经完成
  - end loop.
- 如果是在消费者warp中：
  - 释放之前分配的寄存器
  - 在shared memory中，初始化 $O_i, \ell_i, m_i$这些变量。
  - 等待$Q_i$被加载到shared memory里面
  - for loop:
    - 等待$K_i$被加载到shared memory里面
    - 计算 $(S=QK^T )_{blocked}$，并且commit这一个异步操作（注意这里使用的是`wgmma.mma_async`，所以可以commit和wait）
    - 算出新的$m_i$，并且存放到shared memory中
    - 计算 $P=e^{S_i-m_i}$，$\ell_i = e^{m_i^{old}-m_i}\ell_i + rowsum(P_i^{(j)})$ 这两个东西
    - 等待$V_i$被加载到shared memory中
    - 计算 $O_i$，并且commit且等待 （使用`wgmma.mma_async`，所以可以commit和wait）
    - 释放计算需要的shared memory上的buffer，以便消费者warp可以做后续异步数据搬运
  - end loop.
- 写回

整体看起来，比FA-v1的要清晰多了。论文中的补充性文字在这里就不粘贴了，在上面的讲解中已经几乎覆盖。

实际上，这已然形成了`ping-pong scheduling`，生产者warp负责通过TMA把下一块 K/V tile（也包括偶尔的next Q tile）加载到shared memory的buffer区 （**一共有两块，我们称之为A/B buffer**）；而消费者warp负责使用`wgmma.mma_async`进行GEMM异步运算，紧跟着做softmax操作（注意这一操作已经被algorithm简化为了维护$m$和$\ell$这两个变量，当然最后存储的是$L$这一个变量）。

如果我们可以想象一个异步任务pipeline模型，会发现生产者写A，消费者算A-与此同时生产者写B，消费者算B-与此同时生产者写A……这样循环往复，这就是 `pingpong-scheduling`。他们做异步操作的wait，是通过`mbarrier`来完成的，也就是`bar.sync`之类的指令。

下面要说的一个问题，就是：**TMA以及数据搬运可以与计算异步，这个好理解，为什么GEMM还可以和softmax异步？** 我们如果从硬件角度来考虑，就可以解答这个问题：

- `wgmma.mma_async`使用的是Tensor Core进行运算
- `Softmax`虽然被简化为了algorithm这种形式，只需要维护几个变量，但是依然属于通用型运算，这个是放到Cuda Core上进行运算的（严谨来说，运算单元包括：Cuda Core、FP ALUs、SFU）
- Tensor Core与Cuda core的运算是可以overlap的，资源并不冲突，没什么问题

于是，就有了下面这个时序建模：

![[flash attention v1-v3 系列论文解读 all-03.webp]]

## 5.2 infra-warpgroup的overlap[](#52-infra-warpgroup的overlap)

实际上，我们在分析算法的**依赖关系**之后，还可以做更进一步的overlap优化。文章直接打破了这样的依赖关系，做更进一步的异步优化overlap，我们来看看是怎么做的：

- 首先第一阶段，先把$S_{cur} = Q_i K_0^T$，$\tilde{P}_{cur}, \ell_i$算好，重新缩放 $O_i$
- for loop: （**注意：等待异步loading的那些步骤在此就不写了，我们专注于看计算逻辑**）
  - 计算 $S_{next} = Q_iK_j^T$，这是在计算下一块的S，发射了之后不等待
  - 计算 $O_i = O_i + \tilde{P}_{cur}V_{j-1}$ 这是在计算当前的 O，因为$\tilde{P}_{cur}$是在上一阶段（或者是上一个loop、或者是外面的第一阶段）就算好了的
  - 基于 $S_{next}$算 $m_i, \tilde{P}_{next}, \ell_i$ 这一步就是softmax的计算过程
  - 计算 $\tilde{P}_{cur}V_{j-1}$，且等待，重新放缩 $O_i$
  - 释放buffer
  - 把 $S_{next}$拷贝到 $S_{cur}$ 的位置，方便下一个loop做计算
- end loop

所以实际上，一个循环里面在用Tensor Core算两块的内容：当前这一块的 $\tilde{P}_{cur}V_{j-1}$和下一块的 $S_{next} = Q_iK_j^T$。如果我们可以在流水上可视化这个时序任务图，就是这样的：

![[flash attention v1-v3 系列论文解读 all-04.webp]]

在 H100 上，`wgmma.mma_async`（张量核）与 softmax 里的 `exp`/缩放属于**不同功能单元**，吞吐/时延其实很不对称：例如一张典型配置里，`MUFU.EX2`（`exp2`）大约需要1500 cycles，而一次 WGMMA 的开销要大于这个数值 3072 cycles。**如果把 softmax 的那部分算进 WGMMA 的“影子里”，整体就更满**。[11](#user-content-fn-13)

之前通过ping-pong scheduling，我们把算力从580 TFLOPS提升到640 TFLPOS，再通过这样的infra-warpgroup重排，提升吞吐到了 670 TFLOPS。

## 5.3 FP8的使用[](#53-fp8的使用)

其实使用FP8，主要是因为算得更快，迎面而来就有两个问题与挑战：

1. 怎么才能把数据排布成FP8-wgmma需要的layout，来吃满tensor core？
2. FP8计算精度损失较大，如何尽可能压低？

**layout conformance**

在PTX的层面，`wgmma`对于accumulator指定FP32，FP8的最小tile块大小都做了强制性约束，实际我们计算的$Q,K,V$往往是在$head$维度连续的，直接投放进入`wgmma.xxx.mXnXkX`就会导致无法对齐，吞吐量上会打折。文章采用的方案是对layout做变换，让数据对齐：

1. 对于$V$，要求在shared memory中**按序列维度`seq_len`连续**。由于TMA没办法做转置+加载，FA-v3在kernel内部做transpose操作，把V tile从head-major变成seq-major，这一步在生产者warp中进行。
2. FA-v3在register或者shared memory内部做byte-permute，重排accumulator，以此适配第二次输入的时候需要的数据格式。

**Accuracy**

为了抑制精度损失，FA-v3提出两种方案：

1. block quantization: 以block为单位对$Q,K,V$分别求scale factor，块内部做FP8量化，这种方案比per-tensor的求scale更能够适应动态的局部范围，同时不引入额外的写操作，可以和RoPE等操作进行融合。
2. Incoherent Processing: 在量化到FP8之前，用一个随机正交矩阵 $M$ 乘到 $Q, K$的同一侧，量化后再乘一个$M^T$. 考虑到$M$是正交的，所以这样做不会改变注意力计算的最终结果，但是**能够把异常值在坐标层面给平摊**。

使用FP8之后，吞吐被推进到 1.2 PFLOPS，精度比baseline降低 $2.6 \times$

有关FP8 Attention，笔者推荐各位看看Sage Attention系列论文，这也是我在paddle时候做的reproduce与增强工作，整体上来说是一份比较不错的工程作品。

# Appendix: 面试快问快答[](#appendix-面试快问快答)

不用写结语，大家看到这里已经对于FA系列的优化心中有数。我们干脆整点有用的，如果面试官再来拷打Flash Attention，你能否回答下列问题？

- **FA-v1针对KV的切分方式是什么？**

答：列向切分KV，行向切分Q，同时在片上复用KV。

- **FA-v1对于softmax提出了什么优化，降低维护变量的成本？**

对于每一个query行，只需要维护三个变量，分别是 $m,l,r$

段内：$m_t=max(x^{(t)}), \; \ell_t = \sum e^{x^{(t)}-m_t}, \; r_t = \sum e^{x^{(t)}-m_t}V^{(t)}$

这样一来，不需要保留中间计算结果 $S,P$，这两个都是 $N^2$级别的数据，这样减少了IO次数，还降低了显存使用。

- **FA-v2做了哪些优化？**

**增加`seqlen`并行维度**：把计算切得更细、**更多独立 tile 实例**可并发，提升 SM 占用与流水线效率（尤其是head dim小、长序列时）。

**更好的并行策略**：优化片上数据复用，从warp访问KV，变成warp访问Q，共享KV，减少 shared memory往返与同步。

**根据causal mask跳过**：把 **上三角整块跳过**，只在对角块做逐元素掩码；大量块“全算/全跳”，分支更少。

**反向更省 I/O**：系统化应用 **recomputation**，在 bwd 里也不存 P或者S，按需重算 $QK^T$ 段并在线累计 $dQ,dK,dV$。

- **讲讲FA-v3的warp-spec**

**Producer**：用 **TMA** 异步把下一块 K/V（有时也含 Q 片）搬入 shared memory，设置两块shared memory作为buffer缓冲；

**Consumer**：在另一缓冲上跑 **WGMMA**（$QK^T$）、插空做 **softmax**，再 **WGMMA**（softmax·V）。

- **FA-v3的infra-warpgroup熟悉吗？讲一讲，尤其是相交原版的warp-spec做了什么？你有哪些启发？**

分析并发瓶颈之后，打破之前的算法依赖关系，同时计算当前的PV和下一块的QK，同时与softmax重叠，利用wgmma的异步+TMA的异步+Softmax的SFU进一步打碎流水并行粒度。

区别：3.1 解决的是 **Producer/Consumer 跨组并发**；**3.2 再在组内压缩气隙**，吃掉 WGMMA 与 softmax 的微小空洞。

启发：在遇到程序瓶颈，尤其是异步瓶颈时，要分析顺序依赖关系，并且灵活效仿FA-v3的异步技巧，交错不同的Tensor Core与Cuda Core的运算任务，进行更细粒度的优化。

- **FA-v3怎么做的FP8优化？怎么降低的精度损失？**

布局方面，让V tile在生产者warp中做转置，同时重排accumulator寄存器以适配FP8输入，让两次GEMM都能用上`wgmma`的FP8指令需求。

降低损失方面，做block scaling，同时也乘以正交矩阵 + 正交矩阵转置，虽然引入了额外计算量，但是均摊了精度损失，可以trade off。

# Reference[](#reference)

## Footnotes[](#footnote-label)

1.  Flash Attention: [https://arxiv.org/abs/2205.14135](https://arxiv.org/abs/2205.14135) [↩](#user-content-fnref-1) [↩2](#user-content-fnref-1-2)

2.  Flash Attention-v2: [https://arxiv.org/abs/2307.08691](https://arxiv.org/abs/2307.08691) [↩](#user-content-fnref-2)

3.  Flash Attention-v3: [https://arxiv.org/abs/2407.08608](https://arxiv.org/abs/2407.08608) [↩](#user-content-fnref-3)

4.  DefTruth图文并茂讲解Flash Attention：[https://zhuanlan.zhihu.com/p/668888063](https://zhuanlan.zhihu.com/p/668888063) [↩](#user-content-fnref-4) [↩2](#user-content-fnref-4-2)

5.  Online Softmax: [https://arxiv.org/pdf/1805.02867](https://arxiv.org/pdf/1805.02867) [↩](#user-content-fnref-5)

6.  GPU memory: DRAM or SRAM? [https://forums.developer.nvidia.com/t/gpu-memory-dram-or-sram/26742](https://forums.developer.nvidia.com/t/gpu-memory-dram-or-sram/26742) [↩](#user-content-fnref-6)

7.  Linear Attention: [https://arxiv.org/abs/2310.01082](https://arxiv.org/abs/2310.01082) [↩](#user-content-fnref-7)

8.  tile-lang: [https://github.com/tile-ai/tilelang](https://github.com/tile-ai/tilelang) [↩](#user-content-fnref-8)

9.  TVM: [https://tvm.apache.org/](https://tvm.apache.org/) [↩](#user-content-fnref-9)

10. Self-attention does not need $𝑂(n^2)$ memory: [https://arxiv.org/abs/2112.05682](https://arxiv.org/abs/2112.05682) [↩](#user-content-fnref-10)

11. Profile FA3: [https://static.rainfocus.com/nvidia/gtcs25/sess/1725825619590001romQ/FinalPresPDF/S71368\_FlashAttention-3-%20Fast%20and%20Accurate%20Attention%20With%20Asynchrony%20and%20Low%20Precision\_1742587734302001Zsa8.pdf?utm\_source=chatgpt.com](https://static.rainfocus.com/nvidia/gtcs25/sess/1725825619590001romQ/FinalPresPDF/S71368_FlashAttention-3-%20Fast%20and%20Accurate%20Attention%20With%20Asynchrony%20and%20Low%20Precision_1742587734302001Zsa8.pdf?utm_source=chatgpt.com) [↩](#user-content-fnref-13)

# 1\. Introduction[](#en-1-introduction)

> Why climb Everest? Because it’s there.

Ever since I first got into HPC a long time ago, I had heard of flash attention[1](#user-content-fn-1). Later, during my research internship, flash attention v2[2](#user-content-fn-2) had just come out, but back then I simply used it via `pip install`, and when I saw the formulas and illustrations in the paper, I didn’t dig too deeply. Later still, during my internship at Paddle, an opportunity came up: my leader asked me to compare the quantized operator I had reproduced against flash attention v3[3](#user-content-fn-3) to see which one was stronger. So I came into contact with the v3 operator, but because of a tight task schedule, it was once again just `pip install`, and I still didn’t go and understand the principles behind it.

To stop being a *`pip install` boy*, and since I happen to have some time right now to settle down and build up my inner strength, I decided to read through flash attention from v1 to v3 and organize it into a document for future reference. At the same time, I’ve recently been stuck at a bottleneck in kernel development: it feels like I know everything, and yet also like I know nothing. So why not look at how people think about data and kernels, and how they make use of the latest features, in textbook-level practice — **the unity of knowledge and action**.

Admittedly, there are already many existing write-ups on flash attention. Although few of them tie everything together into one thread, by reading them together and merging them, you can always piece together a complete line of knowledge. Inspired by DefTruth’s article[4](#user-content-fn-4), I settled on the order for studying and organizing: online softmax → FA1 → FA2 → FA3.

# 2\. Prerequisite: Softmax\[all\][](#en-2-prerequisite-softmaxall)

## 2.1 Safe Softmax[](#en-21-safe-softmax)

The formal expression of the attention computation shows where the softmax function is used. In practice, because the softmax operation is memory-bound during inference, many researchers have proposed ways to optimize this formula.

The most naive formula for softmax is: $softmax(x_i) = \frac{e^{x_i}}{\sum_{j=1}^N e^{x_j}}$. Typically, in an implementation we do a warp-block reduce to obtain a sum, and then perform the element-wise softmax operation.

The engineering problem with the original softmax is that, when training or running inference in fp16, the exponent $x_i$ can be very large, so $e^{x_i}$ overflows easily. To put a quantitative estimate on it: fp16 can represent values up to the order of 65536, and once $x_i \ge 11$, $e^{x_i}$ already overflows 65536. Under these circumstances, the concept of safe softmax was proposed. (By the way, here’s a link to [a clean implementation of safe softmax](https://github.com/xlite-dev/LeetCUDA/blob/main/kernels/softmax/softmax.cu))

Safe Softmax can be expressed as:

$$
\frac{e^{x_i}}{\sum_{j=1}^N e^{x_j}} \rightarrow \frac{e^{x_i - m}}{\sum_{j=1}^N e^{x_j-m}}
$$

Every exponent has $m$ subtracted from it, where $m$ is the maximum over all $x_i$: $m = max(x_i)$. The purpose of this is to guarantee that **the exponent ends up less than 0**, which in effect greatly reduces the risk of numerical overflow.

## 2.2 Online Softmax[](#en-22-online-softmax)

In fact, suppose we implement this version of softmax on a CPU (more formally: in an ordinary programming model rather than SIMT); we’ll find that it takes at least 3 passes to complete:

- In the first for loop, find the maximum value in the set ${x_i}$, denoted $m$
- In the second for loop, accumulate to finally compute a sum, i.e., the $\sum_{j=1}^N e^{x_j-m}$ part, which serves as the denominator
- In the third for loop, divide each element one by one by the denominator computed in the second step, and write the results back to the output space

Now let’s look at how to do the computation online[5](#user-content-fn-5).

In the scheme above, the three for loops cannot be fused with one another, because they have sequential dependencies: the second step depends on the first step finding the global maximum $m$, and the third step depends on the second step computing the global sum. If we want to push the complexity as low as possible and reduce accesses to global memory, the only option is to hack the formula ourselves — find an online method that is nearly equivalent to the original formula.

The approach proposed by online softmax is to replace $m$, the global maximum, with the “current maximum” $m_i$: the maximum over the region scanned so far is used as $m$. This process can be formally expressed as:

$$
\frac{e^{x_i}}{\sum_{j=1}^N e^{x_j}} \rightarrow \frac{e^{x_i - m}}{\sum_{j=1}^N e^{x_j-m}} \rightarrow \frac{e^{x_i - m_i}}{\sum_{j=1}^N e^{x_j-m_i}}
$$

(*Since this article emphasizes **connecting the dots**, I simply included the earlier formulas as well, so you can get an intuitive feel for how the formula evolves.*)

At this point, we can pull out the denominator and analyze it on its own. Let $d_S = \sum_{j=1}^S e^{x_j-m_S}$, where $S$ denotes the index we’ve computed up to so far; it evolves as follows:

$$
d_S = \sum_{j=1}^S e^{x_j-m_S} = \sum_{j=1}^{S-1} e^{x_{j-1}-m_S} + e^{x_j-m_S}
$$

This step simply pulls the last term out on its own.

$$
= (\sum_{j=1}^{S-1} e^{x_{j-1}-m_{S-1}}) \times e^{m_{S-1}-m_S} + e^{x_j-m_S}
$$

$$
= d_{S-1} \times e^{m_{S-1}-m_S} + e^{x_j-m_S}
$$

With this, we have completed the derivation of the recurrence, linking each $d_S$ to $d_{S-1}$.

What’s the use of this step? The point is: **of the three loops above, we can fuse the first and second steps, so softmax can now be computed with only two for loops**.

- In the first for loop, compute the running maximum $m_S$ scanned so far (just compare it directly with $m_{S-1}$); and, alongside it, compute the sum up to the current position.
- In the second for loop, perform the element-wise division by the total sum computed

OK, let’s set online softmax aside for now as prerequisite knowledge; next, we move on to reading flash attention v1.

# 3\. Flash Attention-v1[](#en-3-flash-attention-v1)

Flash Attention v1 (hereafter FA-v1) aims to provide a general-purpose operator for all GPU devices, so it can’t present its content primarily from the NVIDIA GPU perspective; hence it uses the two terms HBM and SRAM. HBM is short for GPU high bandwidth memory; its counterpart on NVIDIA GPUs is what we usually call `global memory`. SRAM is the umbrella term for **all on-chip memory**; this set includes $\{ 'shared \_memory', 'registers', 'caches' \}$ and so on, not just `shared memory`. So a distinction and clarification is needed here; there’s a discussion[6](#user-content-fn-6) on the NVIDIA forums about this.

> FA-v1’s optimizations mainly target the I/O pattern of the attention computation, making targeted use of SRAM. It also extends Flash Attention to a block-sparse attention computation, which is faster than all existing attention acceleration methods (FA-v1 was proposed in 2022).

At that time, linear attention[7](#user-content-fn-7) had not yet been proposed, and the complexity of attention still grew quadratically with the `sequence length` (denoted $N$), i.e., $N^2$. Existing acceleration schemes did all reduce the complexity of attention as much as possible through various approximation methods, but they mostly focused on theoretical FLOP reductions, which did not match the observed actual running time (called `wall-clock speedup` in the paper), and these works tended to ignore the overhead of memory access.

We’ll skip the background for now and assume everyone has some GPU & CUDA & HPC fundamentals; this article only covers the optimization ideas and contributions of FA-v1 through v3.

## 3.1 Engineering Model of Standard Attention[](#en-31-engineering-model-of-standard-attention)

When computing attention, we usually deal with tensor shapes like `[bsz, n_head, seq_len, head_dim]`, where the last two dims, sequence length and head dimension, are formally denoted $N$ and $d$ respectively. FA-v1 summarizes the attn computation with the following formulas:

$$
S = QK^T \in \mathbb{R}^{N \times N}, P = softmax(S) \in \mathbb{R}^{N \times N}, O = PV \in \mathbb{R}^{N \times d}
$$

In general, `seq_len` is far larger than `head_dim`. For example, we often talk about long sequences of length 32K, while the model’s head\_dim is only 128; here $32 \times 1024 >> 128$, i.e., $N >> d$. So this process is actually IO-dominated and memory-bound. The paper presents the standard attention computation as pseudocode; here I’ll lay it out in plain words:

1. Load $Q, K$ from `global memory` into each thread’s private `local memory` or `register` (the threads evenly share the elements of $QK$), do one GEMM $S=QK^T$, then write the result $S$ back to `global memory`.
2. Then fetch $S$ from `global memory`, compute $P = softmax(S)$, and write the result $P$ back to `global memory`
3. Load $P, V$ from `global memory` again, do the GEMM $O = PV$, then write $O$ back to `global memory`
4. Output $O$

> After reading this part, I wonder whether FA-v1’s way of modeling attention is a bit too plain; in reality, anyone who knows even a little HPC wouldn’t design a kernel like this. Going in and out of `global memory` three times is far too expensive. If this really is how everyone computed attention before FA-v1 came out, it must have been painful.

FA-v1 sets its core goal: **reduce accesses to `global memory`**. The ideal would be to load from global memory once, finish the entire computation, and then write back to global memory. Although it doesn’t ultimately achieve such an idealized single read and single write, it still drastically reduces the number of global memory accesses compared with the original setup.

To achieve this goal, FA-v1 mainly proposes two techniques: **Tiling and Recomputation**.

## 3.2 The Concept of Tiling in FA-v1[](#en-32-the-concept-of-tiling-in-fa-v1)

Before going further, I’d like to first sort out the various notions of `tiling` I’ve encountered since getting into HPC, **to avoid confusion**.

**What is a `tile` — the most basic definition**

When we do kernel optimization, the basic programming model (the CUDA Programming Guide) tells you that our programming model is grid-block-thread. The PTX programming model tells you that our programming model is CTA-cluster-grid. The concept of a `tile` comes up when the data partitioning scheme goes below the block size, and it is adopted for **cache-locality friendliness, or register friendliness**. For example, if a block’s data is $256 \times 256$, it will be split into small $8 \times 8$ or $16\times16$ tiles before being submitted for computation.

A `tile` is a minimal unit of data partitioning, and also an optimization strategy; it isn’t part of the programming model in any official programming guide.

> If I had to pick a keyword, I’d use `slice strategy` instead

**The `tile` in `PTX`**

Friends who have worked with `SM90` and above, or who have read the PTX documentation firsthand, should know that in low-level PTX instructions, fixed-size data blocks such as `.m16n16k16` and `.m32n8k16` are also called `tile`s. In this context, the tile effectively becomes **a specific hardware specification prescribed for programming;** it is more a term for **instruction-level fragment shape / register fragment layout**, which determines the fragment mapping and instruction throughput, and doesn’t directly carry the program-level semantics of “putting data into shared/register and how to pipeline it”. When we use `mma` or `wmma`, we always keep an eye on how the data is partitioned so that it fits these sizes, and arrange our own kernel logic accordingly.

> If I had to pick a keyword, I’d use `fragment size` instead

**The `tile` in `tile-lang`**

I’ve been following `tile-lang`[8](#user-content-fn-8) for a while (a really excellent repository). It’s a domain-specific-language compiler developed based on the ideas of TVM[9](#user-content-fn-9), in which the tile is elevated to a first-class citizen: in `tile-lang`, a `tile` is both a data-partitioning concept and a program abstraction and scheduling unit.

> If I had to pick a keyword, I’d use `scheduling unit` instead

**Back to the main topic: the `tile` in `flash attention`**

![[flash attention v1-v3 系列论文解读 all-05.webp]]

In FA-v1, loading all of $Q,K,V$ into shared memory at once is clearly unrealistic. The best approach is to split them into small blocks one after another; formally, the small orange blocks can be denoted $Q_{block}, K_{block}, V_{block}$. During computation, we only need to load these small blocks into shared memory and finally compute $O_{block}$; before adding it back to its corresponding position, we scale $O_{block}$ by the right normalization factor and then add it back, which yields the correct result.

Applying softmax to $\frac{QK^T}{\sqrt{d}}$ is a tricky problem. Although we already have the online softmax method, the situation now is: the matrix to be softmaxed has `seq_len` rows and `head_dim` columns, and softmax must be applied to each row. If we did softmax row by row, we’d have to run online softmax `seq_len` times. Even though we’ve now partitioned into blocks, we still need to run online softmax over each small block.

Prior work[10](#user-content-fn-10) has already proposed an optimization: suppose there are only two rows in total, `[2, head_dim]`; each row can be pulled out on its own as a vector, denoted $x^{(1)}, x^{(2)}$ in the paper. Combining this with the earlier online softmax formula while ensuring numerical stability, the paper proposes the following computation:

$$
m(x) = m([x^{(1)} \; x^{(2)}])  \quad \quad \text{here this denotes the concat of the two vectors}
$$

$$
=max(m(x^{(1)}), m(x^{(2)}))
$$

$$
f(x) = [e^{m(x^{(1)})-m(x)}f(x^{(1)}) \quad e^{m(x^{(2)})-m(x)}f(x^{(2)})]
$$

Here, $f(x)$ denotes the concat of the numerator parts of the two elements after online softmax has been computed on each.

Denoting the sum by $l(x)$, we have:

$$
l(x) = l([x^{(1)} \quad x^{(2)}]) = e^{m(x^{(1)})-m(x)}l(x^{(1)}) + e^{m(x^{(2)})-m(x)}l(x^{(2)})
$$

Finally, the softmax:

$$
softmax(x) = \frac{f(x)}{l(x)}
$$

Actually, the formulas here are not stated very clearly in places; if you find them obscure, please refer to the original paper[1](#user-content-fn-1) and DefTruth’s blog[4](#user-content-fn-4).

A few questions immediately arise:

1. How do we partition QKV? As we all know, during attention computation QKV have two dimensions, `[seq_len, head_dim]`; which dimension do we split along? How large should each piece be?
2. How do we compute with $Q_{block}, K_{block}, V_{block}$?

**Partition size and pseudocode walkthrough**

In the HPC field, when memory access is involved, it’s customary to use $B$ for block size and $M$ for memory size; this convention was already in use in research from many years ago (cache-aware, cache-oblivious), and FA-v1 is no exception. For each sequence, for each attention head during the computation, we can formalize as follows:

- $Q,K,V \in \mathbb{R}^{N \times d}$, the **total** size of `shared memory` is $M$
- The block sizes are set to: $B_c = \lceil \frac{M}{4d} \rceil$, $B_r = min(\lceil \frac{M}{4d} \rceil,d)$

This answers the first question: how big a chunk to cut KV into. From the formula: $\frac{M}{4d}$ is the size of the $K/V$ column blocks. Each time we process, we need to put such K and V column blocks (*note: column-major!*) into shared mem. **Why this size?** Let’s do a quick calculation, assuming fp16 inference:

1. One block occupies: $B_c \times d \times 2\;bytes$

2. K and V each have one block, occupying in total: $2 \times B_c \times d \times 2\;bytes = 4B_cd \quad bytes$

Assuming our shared memory size is $M$, to guarantee no overflow we need $B_c \le \frac{M}{4d}$. In practice, based on my hands-on experience using Flash Attention before, after rounding up, $B_c$ is usually 16 or 32, to fit the `mma.m?n?k?` PTX instructions.

And how big a chunk do we cut Q into? $min(\lceil \frac{M}{4d} \rceil,d)$. We’ll explain later why it’s this size; for now, let’s continue describing FA-v1’s computation process:

- Initialize on HBM the $O$ matrix of size $N \times d$; the sum vector $l(x)$ of size $N$; and the vector $m(x)$ that temporarily stores the maxima, of size $N$.
- Split $Q$ along the `seq_len` dimension (this answers the first question: which dimension to split along), each block of size $B_r$, yielding $T_r = \lceil \frac{N}{B_r} \rceil$ blocks in total: $Q_1, Q_2, ..., Q_{T_r}$, each of size $B_r \times d$ (i.e., the data of $B_r$ tokens x head\_dim)
- Split $K, V$ along the `seq_len` dimension (note, however, that seq\_len is a column vector in KV, so what we actually get is a column block), yielding $T_c = \lceil \frac{N}{B_c} \rceil$ blocks in total: $K_1, K_2, ..., K_{T_c}$, $V_1, V_2, ..., V_{T_c}$, each of size $B_c \times d$ (i.e., the data of $B_c$ tokens x head\_dim)
- Split $O$ in the same way as $Q$, obtaining $O_1, O_2, O_3, ..., O_{T_r}$, each of size $B_r \times d$.
- Split the two vectors $l, m$; here the original paper uses the phrase $divide \; into \; blocks$, obtaining $l_1, l_2, .., l_{T_r}$ and $m_1, m_2, ..., m_{T_r}$, each of size $B_r$
  - Personally, I find this not quite accurate: they are row vectors to begin with, and `blocks` sounds like a 2D matrix; in reality, it’s just slicing arrays. **Calling it $divide \; into \; slices$ might be a bit easier to understand.**

- - -

Now that we’ve sorted out the data partitioning logic, let’s move on to the computation logic:

- Outer loop (clearly marked in orange in the figure, mainly used to advance the KV blocks): load KV blocks from global memory into shared memory
  - Inner loop (marked in blue in the figure, mainly used to advance the $Q$ and $O$ blocks): load the corresponding $Q_i, O_i$ blocks from global memory into shared memory;
    - Compute the attention score in shared memory: $S_{ij} = Q_iK_i^T \in \mathbb{R}^{B_r \times B_c}$
    - Compute in shared memory $m_{ij} = rowmax(S_{ij}) \in \mathbb{R}^{B_r}$, $P_{ij}=e^{S_{ij}-m_{ij}} \in \mathbb{R}^{B_r \times B_c}$, $l_{ij}=rowsum(P_{ij}) \in \mathbb{R}^{B_r}$
    - Compute $m_i^{new}, l_i^{new}$ in shared memory; this is where these vectors are dynamically maintained
    - The scaling factor we mentioned earlier: $diag(l_i^{new})^{-1}(diag(l_i)e^{m_i-m_i^{new}}O_i + e^{m_ij-m_i^{new}}P_{ij}V_j)$; once computed, write it back to $O_i$‘s corresponding position in global memory
    - Write the newly maintained $l_i^{new}, m_i^{new}$ back to global memory
  - end inner loop
- end outer loop

## 3.3 Recomputation[](#en-33-recomputation)

We have indeed adopted the approach of splitting along the `seq_len` dimension. The next question that immediately arises is: if we want to store $P = softmax(S)$, then as the sequence grows we’d eventually need an estimated $O(N^2)$ of GPU memory, along with an enormous accompanying cost of memory reads and writes (I/O overhead). Our block-wise reading approach is nice — discard the current block once it’s computed — but if we want to get the output $O = PV$ in one go, we usually need to keep quite a few intermediate variables in `shared memory`.

One of Flash Attention’s goals is **to make all the intermediate data that needs to be stored as small as possible, ideally compressing it down to the order of $O(N)$**. The idea is **to trade expensive I/O for doing a bit more computation** (note: grasping the idea is what matters most; these ideas and experiences help us write more efficient kernels during development). This is clearly reasonable, because on GPU devices computation is simply cheaper than IO; this way, we can also improve the `wall-clock speed` somewhat.

The steps of recomputation can be formalized as follows:

Split the task into two scans over the `c-block`s: the first only computes the normalization factors, and the second multiplies the weights by $V$:

1. Pass-1: for each c-block:
   1. $m_t = max(x^{(t)}), \ell_t = \sum{e^{x^{(t)}-m_t}}$; merge them with the global $(m, \ell)$ using the online formula.
   2. Afterwards, we only need to store $m, \ell$ for each row, both on the order of $O(N)$ in size; there’s no need to store $S, P$
2. Pass-2: directly compute the normalized weights:
   1. $\omega^{(t)}=\frac{e^{x^{(t)}-m}}{\ell}$
   2. Multiply by $V$: $r \larr r + \omega^{(t)}V^{(t)}$
   3. After scanning all blocks, output $O_i = r$

## 3.4 I/O Analysis[](#en-34-io-analysis)

I’ll skip the proofs here; interested readers can head straight to the paper. Here we just record the conclusions.

- The IO complexity of standard attention is: $\Theta(Nd+N^2)$, where $d$ is `head_dim`
- The IO complexity of Flash-Attention is: $\Theta(\frac{N^2d^2}{M})$, where $M$ is the size of shared memory. This directly explains FA-v1’s advantage: **the reduction in IO is what determines the speedup**.

We won’t go into “block-sparse FlashAttention” here, since it isn’t part of the main thread; interested readers can explore it on their own.

# 4\. Flash Attention-v2[](#en-4-flash-attention-v2)

FA-v1’s performance reaches only 25-40% of the theoretical peak FLOPS. The reason behind this is **the lack of a partitioning strategy sufficiently optimized across blocks, threads, and warps, which results in low occupancy and unnecessary shared memory reads/writes.** Note that this sentence is a highly condensed, distilled one from the abstract; in the analysis below, we’ll make clear what exactly went wrong and what optimizations were made.

FA-v2 is mainly developed on the A100. To optimize to the extreme, you first need to know the A100’s hardware parameters, so that we can understand where the hardware’s limits lie.

The A100 has 81GB of GPU memory (global memory) with $1.5-2.0 \; TB/s$ of bandwidth, a total of $192KB$ of shared memory, 108 SMs, and $19 \; TB/s$ of on-chip bandwidth. Since the L2 cache can’t be controlled programmatically, the paper still centers its optimization efforts on global memory and shared memory.

> For FA-v1, we expanded on lots of formulas and discussed the concrete steps of kernel optimization. Starting from FA-v2, the walkthrough will mainly record the ideas: with the groundwork laid earlier, discussing the optimization schemes goes more smoothly, and you won’t be left unable to form a mental picture. But the rest of the article won’t be as detailed as the introduction to Flash Attention-v1.
> 
> This is also what the great DefTruth advised in his article: be meticulous when going through FA-v1, mainly in order to master the principles.
> 
> Therefore, for the subsequent FA-v2 and FA-v3, we focus on presenting the optimization ideas.

## 4.1 Improving the Algorithm to Save Space[](#en-41-improving-the-algorithm-to-save-space)

Viewed from FA-v2’s perspective, the computational approach of the earlier FA-v1 algorithm still needs to keep too much in shared memory, and it also has some extra computation steps. FA-v2 made the following optimizations:

1. Going from $O^{(2)}=diag(\frac{\ell^{(1)}}{\ell^{(2)}})^{-1}O^{(1)} + diag(\ell^{(2)})^{-1}e^{S^{(2)}-m^{(2)}}V^{(2)}$ to $\tilde{O}^{(2)}=diag(\ell^{(1)})^{-1}O^{(1)}+e^{S^{(2)}-m^{(2)}}V^{(2)}$. Note that in the new formula, $\tilde{O}^{(2)}$ is `un-scaled` data, so we only need to scale the last $\tilde{O}^{(last)}$ once by $diag(\ell^{(last)})^{-1}$ at the end of the loop to obtain the final output.
2. At the same time, we no longer need to store both $m^{(j)}, \ell^{(j)}$ for the backward pass; instead, storing a single variable $L^{(j)}=m^{(j)}+log(\ell^{(j)})$ (logsumexp) is enough.

With these two changes, we save both **computation steps** and **shared memory space**.

**Handling the causal mask**

Attention inference always requires a causal mask, i.e., a lower-triangular matrix. Since FA-v1 already cut the sequence into a grid of row blocks `r` and column blocks `c`, for any row `r` and column `c`:

1. If $c > r$, i.e., the entire block lies in the upper triangle, skip it directly.
2. This way, for a long sequence, half of the blocks can be skipped, which in theory directly cuts the amount of computation by $2 \times$; taking into account fixed overheads such as data loading, synchronization, and pipelining, the measured speedup is $1.7-1.8 \times$.

Meanwhile, for the “diagonal blocks”, element-wise masking is applied, i.e.: per-row only 1 block masked.

1. For $c < r$, the entire block lies in the lower triangle; compute it normally
2. For $c > r$, skip it
3. For $c = r$, i.e., the diagonal block, this block does have some data in the upper triangle and some in the lower triangle; we only need to apply an element-wise causal mask inside this block, setting all positions with $j>i$ within the block to $-\infin$. This way, **each row only needs masking in 1 block**, and every other block is either fully computed or fully skipped, which avoids a lot of branching and also eliminates some pipeline bubbles.

## 4.2 Changing the Parallelization Strategy[](#en-42-changing-the-parallelization-strategy)

In FA-v1, parallelism is applied over two dimensions: `batch-size` and `num_head`. We use 1 cuda block to compute one attention head of one sequence, which means a total of $bsz \times n \_heads$ cuda blocks are launched. If each block runs on one SM — say the device we’re using is an A100 with 108 SMs in total — this scheduling scheme is reasonably efficient when batch\_size is very large, e.g., batch\_size > 80. **But when the batch size is relatively small and seq\_len is very long, or when num\_heads is small (GQA)**, this parallelization strategy isn’t that efficient.

This time, to solve this problem during inference, it was decided to also make the `seq_len` dimension a dimension of parallel computation. In other words, after the same sequence is split into different blocks, the blocks are submitted directly to different cuda-blocks (i.e., distributed to different SMs) for computation, since this process doesn’t require them to pass information to one another; meanwhile, parallelism over the `batch_size` and `num_head` dimensions is retained as well. This way, whether it’s a large batch, large or small num\_head, or long or short sequences, the kernel is guaranteed to run in its most efficient form, making the most of the GPU’s hardware throughput.

This design is implemented mainly by swapping the order of the loops inside the kernel: the outer-loop covers row-blocks, and the inner-loop covers col-blocks.

## 4.3 Changing How Data Is Partitioned Across Warps[](#en-43-changing-how-data-is-partitioned-across-warps)

![[flash attention v1-v3 系列论文解读 all-06.webp]]

The way warps access data in FA-v1 is shown on the left. Warps 1-4 can all access the Q-block, while for K and V, each warp fetches its own portion of the data and then performs the QKV computation. As a result, each warp is responsible for computing a $QK^T$ and also for computing with its corresponding $V_{slice}$, and in the end communication between the different warps is required to add everything up and form the result. The paper calls this the “Split-K” scheme.

What makes this scheme inefficient is that all warps need to store their results into shared memory, synchronize (perhaps via `__syncthreads()`), and finally sum. The heavy shared memory reads/writes, along with the synchronization they require, are FA-v1’s pain point, and FA-v2 sets out to eliminate this step entirely.

The way warps access data in FA-v2 is shown on the right. Warps 1-4 are each responsible for fetching one slice of Q, and this time KV becomes data that warps 1-4 can all access. This is called the “split-Q” scheme.

This way, each slice only needs to compute its own $QK^T$ and then multiply by V to get the corresponding output, with no cross-warp communication needed. Naturally, this doesn’t involve much shared memory reading/writing and also saves the synchronization cost, so it directly speeds things up.

At this point, FA-v2’s optimization methods have been roughly covered. Note that the backward-pass content hasn’t been gone through in much detail; interested readers please head straight to the original paper.

# 5\. Flash Attention-v3[](#en-5-flash-attention-v3)

As its abstract also mentions, FA-v3 was proposed precisely because FA-v2 performed too poorly on the H100, with GPU utilization of only 35%.

FA-v3, I believe, is also a well-worn topic for most people: it optimizes by exploiting the asynchronous features for tensor cores and the native `.tensor` data type\[^11\] available starting with SM90, the TMA feature, and FP-8 mixed-precision inference support. The concrete results are as follows:

1. It reaches 740 TFLOPS with FP16 inference, i.e., 75% H100 utilization.
2. FP8 inference reaches 1.2 PFLOPS.
3. The FP8 inference accuracy in FA-v3 is $2.6 \times$ better than the baseline FP8 Attention results.

## 5.1 Introduction[](#en-51-introduction)

For the H100 GPU, the programming model has changed a lot. Before going further, let’s do a brief overview here. This overview comes not only from the paper but also from my own prior experience, presented to everyone in what I hope is the easiest-to-understand way:

> Note: the H100 is an SM90-arch GPU

1. On GPUs before SM90, the mainstream GPU programming model was still grid-block-thread: kernels are launched onto SMs in units of blocks, and shared memory is private to an SM, shared by all threads running within that SM. Shared memory can’t be shared across SMs.
2. On GPUs at SM90 and above, optimization has gone deep down to the PTX level. The PTX programming model, from top to bottom, is: grid-cluster-CTA\[^12\], where a CTA can be understood as equivalent to the `block` concept in our earlier programming model. A group of CTAs forms a cluster, and **shared memory can be shared at the cluster level; that is, a group of SMs now forms a cluster, and that group of SMs can share shared memory**.
3. `TMA`, a term everyone keeps bringing up, is actually a dedicated hardware unit on Hopper-architecture GPUs, mainly used for data movement; it can perform asynchronous copies of data from global memory to shared memory.
4. `.tensor` is a native data type in PTX programming. Indeed, on devices below SM90 and at or above SM80, you can use the `cp.async` instruction, but there’s a catch: **it can only move ordinary data, not `.tensor` data.** On GPUs at SM90 and above, you can use the `cp.async.bulk.tensor.{...}` instruction to asynchronously move an entire tile into shared memory for the compute logic to use. That’s the difference in asynchronous copy at the instruction level.
5. `WGMMA` becomes a new instruction family. Before this, we had only used `wmma` or `mma` (I recommend checking out big brother DefTruth’s LeetCUDA, which has detailed demos of all of them). On GPUs at SM90 and above, you can use `warp-grouped mma`; unlike `wmma` and `mma`, which only provide GEMM functionality, **`wgmma` can be asynchronous** — for example, `wgmma.mma_async` is available.
6. Native compute support for the `FP8` data type.
7. Others: e.g., register reallocation such as `setmaxnreg`, and the SFU (Special Function Unit) used to compute operations such as softmax, are also put to use in FA-v3.

Really, of all the features that H100-and-above GPUs provide, the core usage is: **exploit asynchrony to saturate the instruction pipeline, minimize runtime bubbles in the kernel, and thereby raise kernel throughput**.

## 5.2 Warp-spec[](#en-52-warp-spec)

Lately, lots of interviewers have asked me about this, and honestly I’ve been grilled on it to the point of going a bit numb. This subsection has quite a few keywords, such as `warp-specialization`, `Producer-Consumer`, `pingpong-scheduling`, and so on. In FA-v3’s development approach, the programming model has become the CTA-based `ptx` model, so the following discussion is all framed in terms of CTAs:

The specific naming and formal notation have already been laid out clearly in FA-v1. From the CTA’s perspective, without intra-consumer overlapping, it looks like this:

- In a producer warp:
  - Release all the pre-allocated registers
  - Issue: load $Q_i$ from global memory to shared memory
  - Once done, commit the above asynchronous operation and notify the other warps: the asynchronous load of $Q_i$ has completed
  - for loop:
    - Wait for stage $j \; \% \; s$ to be computed by the `[consumer warp]`
    - While at stage $j \; \% \; s$, issue the load instructions for $K_i, V_i$
    - Once done, notify the other warps: the loads of $K_i, V_i$ have completed
  - end loop.
- In a consumer warp:
  - Release the previously allocated registers
  - Initialize the variables $O_i, \ell_i, m_i$ in shared memory.
  - Wait for $Q_i$ to be loaded into shared memory
  - for loop:
    - Wait for $K_i$ to be loaded into shared memory
    - Compute $(S=QK^T )_{blocked}$ and commit this asynchronous operation (note that `wgmma.mma_async` is used here, so we can commit and wait)
    - Compute the new $m_i$ and store it in shared memory
    - Compute both $P=e^{S_i-m_i}$ and $\ell_i = e^{m_i^{old}-m_i}\ell_i + rowsum(P_i^{(j)})$
    - Wait for $V_i$ to be loaded into shared memory
    - Compute $O_i$, then commit and wait (using `wgmma.mma_async`, so we can commit and wait)
    - Release the shared memory buffer needed for the computation, so that the consumer warp can perform subsequent asynchronous data movement
  - end loop.
- Write back

Overall, this looks much clearer than FA-v1’s. I won’t paste the supplementary text from the paper here; it’s almost entirely covered in the explanation above.

In fact, this already forms `ping-pong scheduling`: the producer warp is responsible for loading the next K/V tile (and occasionally the next Q tile) via TMA into the buffer region of shared memory (**there are two buffers in total, which we call the A/B buffers**), while the consumer warp is responsible for performing the GEMM asynchronously with `wgmma.mma_async`, followed immediately by the softmax operation (note that the algorithm has already simplified this operation to maintaining the two variables $m$ and $\ell$; of course, what’s ultimately stored is the single variable $L$).

If we picture an asynchronous task pipeline model, we’ll see: the producer writes A; the consumer computes A while the producer writes B; the consumer computes B while the producer writes A… and so on, round and round. This is `pingpong-scheduling`. The waits for their asynchronous operations are implemented with `mbarrier`, i.e., instructions like `bar.sync`.

The next question to address is: **it’s easy to understand that TMA and data movement can be asynchronous with computation, but why can GEMM also be asynchronous with softmax?** If we think about it from the hardware perspective, we can answer this:

- `wgmma.mma_async` uses the Tensor Cores for computation
- Although `Softmax` has been simplified by the algorithm into this form where only a few variables need to be maintained, it’s still a general-purpose computation, which runs on the Cuda Cores (strictly speaking, the compute units include: Cuda Cores, FP ALUs, SFU)
- Tensor Core and Cuda Core computations can overlap; they don’t contend for resources, so there’s no problem

Hence, we get the following timing model:

![[flash attention v1-v3 系列论文解读 all-07.webp]]

## 5.2 intra-warpgroup overlap[](#en-52-intra-warpgroup-overlap)

In fact, after analyzing the algorithm’s **dependencies**, we can go a step further with overlap optimization. The paper directly breaks these dependencies to achieve further asynchronous overlap; let’s see how it’s done:

- First, in the initial stage, compute $S_{cur} = Q_i K_0^T$ and $\tilde{P}_{cur}, \ell_i$, and rescale $O_i$
- for loop: (**note: the steps that wait on asynchronous loading are omitted here; we focus on the computation logic**)
  - Compute $S_{next} = Q_iK_j^T$ — this computes the S of the next block; issue it without waiting
  - Compute $O_i = O_i + \tilde{P}_{cur}V_{j-1}$ — this computes the current O, since $\tilde{P}_{cur}$ was already computed in the previous stage (either the previous loop iteration or the initial stage outside the loop)
  - Based on $S_{next}$, compute $m_i, \tilde{P}_{next}, \ell_i$ — this step is the softmax computation
  - Compute $\tilde{P}_{cur}V_{j-1}$ and wait, then rescale $O_i$
  - Release the buffer
  - Copy $S_{next}$ into $S_{cur}$‘s position, ready for the next loop iteration’s computation
- end loop

So in effect, within one loop iteration the Tensor Cores are computing two blocks’ worth of work: the current block’s $\tilde{P}_{cur}V_{j-1}$ and the next block’s $S_{next} = Q_iK_j^T$. If we visualize this timing task graph on the pipeline, it looks like this:

![[flash attention v1-v3 系列论文解读 all-08.webp]]

On the H100, `wgmma.mma_async` (Tensor Cores) and the `exp`/scaling in softmax belong to **different functional units**, and their throughput/latency is actually very asymmetric: for example, in a typical configuration, `MUFU.EX2` (`exp2`) takes about 1500 cycles, while one WGMMA costs more than that, at 3072 cycles. **If the softmax part is tucked into WGMMA’s “shadow”, the whole pipeline stays fuller**.[11](#user-content-fn-13)

Earlier, through ping-pong scheduling, we raised throughput from 580 TFLOPS to 640 TFLOPS; then, with this intra-warpgroup reordering, throughput goes up to 670 TFLOPS.

## 5.3 Using FP8[](#en-53-using-fp8)

The main reason to use FP8 is really that it computes faster, but two problems and challenges immediately arise:

1. How do we arrange the data into the layout required by FP8 wgmma, to saturate the tensor cores?
2. FP8 computation loses a lot of precision; how do we keep that loss as low as possible?

**layout conformance**

At the PTX level, `wgmma` imposes mandatory constraints: the accumulator is specified as FP32, and FP8 has a minimum tile size. In practice, the $Q,K,V$ we compute with are usually contiguous along the $head$ dimension, so feeding them directly into `wgmma.xxx.mXnXkX` leads to misalignment and discounted throughput. The paper’s solution is to transform the layout so that the data lines up:

1. For $V$, it must be **contiguous along the sequence dimension `seq_len`** in shared memory. Since TMA can’t do transpose + load, FA-v3 performs the transpose inside the kernel, converting the V tile from head-major to seq-major; this step is done in the producer warp.
2. FA-v3 does a byte-permute inside registers or shared memory to reorder the accumulator, so that it matches the data format required when it is fed in as input the second time.

**Accuracy**

To curb the precision loss, FA-v3 proposes two schemes:

1. block quantization: compute the scale factors for $Q,K,V$ separately on a per-block basis and perform FP8 quantization within each block. Compared with per-tensor scaling, this scheme adapts better to dynamic local ranges, introduces no extra write operations, and can be fused with operations such as RoPE.
2. Incoherent Processing: before quantizing to FP8, multiply the same side of $Q, K$ by a random orthogonal matrix $M$, and multiply by $M^T$ after quantization. Since $M$ is orthogonal, this doesn’t change the final result of the attention computation, but it **spreads the outliers out across coordinates**.

With FP8, throughput is pushed to 1.2 PFLOPS, and the error is $2.6 \times$ lower than the baseline.

On FP8 Attention, I recommend checking out the Sage Attention series of papers; this is also what I reproduced and enhanced during my time at Paddle, and overall it’s a pretty solid piece of engineering work.

# Appendix: Interview Rapid-Fire Q&A[](#en-appendix-interview-rapid-fire-qa)

No need to write a conclusion — if you’ve read this far, you already have a good grasp of the optimizations in the FA series. Let’s do something useful instead: if an interviewer grills you on Flash Attention again, can you answer the following questions?

- **What is FA-v1’s partitioning scheme for KV?**

A: Split KV column-wise and Q row-wise, while reusing KV on-chip.

- **What optimization did FA-v1 propose for softmax to reduce the cost of maintaining variables?**

For each query row, only three variables need to be maintained: $m,l,r$

Within a segment: $m_t=max(x^{(t)}), \; \ell_t = \sum e^{x^{(t)}-m_t}, \; r_t = \sum e^{x^{(t)}-m_t}V^{(t)}$

This way, there’s no need to keep the intermediate results $S,P$, both of which are $N^2$-sized data; this reduces the number of IOs and also lowers GPU memory usage.

- **What optimizations did FA-v2 make?**

**Adding `seqlen` as a parallel dimension**: slice the computation more finely so that **more independent tile instances** can run concurrently, improving SM occupancy and pipeline efficiency (especially with a small head dim and long sequences).

**A better parallelization strategy**: optimize on-chip data reuse, going from warps accessing KV to warps accessing Q and sharing KV, which reduces shared memory round trips and synchronization.

**Skipping based on the causal mask**: **skip whole blocks in the upper triangle**, and apply element-wise masking only on the diagonal blocks; most blocks are either “fully computed / fully skipped”, so there’s less branching.

**A more I/O-efficient backward pass**: apply **recomputation** systematically; the bwd pass doesn’t store P or S either, but recomputes the $QK^T$ segments on demand and accumulates $dQ,dK,dV$ online.

- **Talk about FA-v3’s warp-spec**

**Producer**: use **TMA** to asynchronously move the next K/V block (sometimes also a Q slice) into shared memory, with two shared memory regions set up as buffers;

**Consumer**: run **WGMMA** ($QK^T$) on the other buffer, squeeze **softmax** into the gaps, then run **WGMMA** again (softmax × V).

- **Are you familiar with FA-v3’s intra-warpgroup overlapping? Talk about it — in particular, what does it do compared with the original warp-spec? What insights did you take from it?**

After analyzing the concurrency bottleneck, it breaks the earlier algorithmic dependencies, computing the current PV and the next block’s QK at the same time while overlapping them with softmax, and uses wgmma’s asynchrony + TMA’s asynchrony + the SFU for softmax to further break down the granularity of pipeline parallelism.

Difference: 3.1 addresses **cross-group Producer/Consumer concurrency**; **3.2 then squeezes out the air gaps within a group**, eating up the tiny holes between WGMMA and softmax.

Insight: when you hit a program bottleneck, especially an asynchrony bottleneck, analyze the sequential dependencies, flexibly emulate FA-v3’s asynchronous techniques, and interleave the different Tensor Core and Cuda Core compute tasks for finer-grained optimization.

- **How does FA-v3 do its FP8 optimization? How does it reduce the precision loss?**

On the layout side: transpose the V tile in the producer warp, and at the same time reorder the accumulator registers to fit the FP8 input, so that both GEMMs can meet the requirements of `wgmma`’s FP8 instructions.

On the loss-reduction side: do block scaling, and also multiply by an orthogonal matrix + its transpose. Although this introduces extra computation, it spreads out the precision loss, which makes it a worthwhile trade-off.

# Reference[](#en-reference)

## Footnotes[](#en-footnote-label)

1.  Flash Attention: [https://arxiv.org/abs/2205.14135](https://arxiv.org/abs/2205.14135) [↩](#user-content-fnref-1) [↩2](#user-content-fnref-1-2)

2.  Flash Attention-v2: [https://arxiv.org/abs/2307.08691](https://arxiv.org/abs/2307.08691) [↩](#user-content-fnref-2)

3.  Flash Attention-v3: [https://arxiv.org/abs/2407.08608](https://arxiv.org/abs/2407.08608) [↩](#user-content-fnref-3)

4.  DefTruth’s illustrated explanation of Flash Attention: [https://zhuanlan.zhihu.com/p/668888063](https://zhuanlan.zhihu.com/p/668888063) [↩](#user-content-fnref-4) [↩2](#user-content-fnref-4-2)

5.  Online Softmax: [https://arxiv.org/pdf/1805.02867](https://arxiv.org/pdf/1805.02867) [↩](#user-content-fnref-5)

6.  GPU memory: DRAM or SRAM? [https://forums.developer.nvidia.com/t/gpu-memory-dram-or-sram/26742](https://forums.developer.nvidia.com/t/gpu-memory-dram-or-sram/26742) [↩](#user-content-fnref-6)

7.  Linear Attention: [https://arxiv.org/abs/2310.01082](https://arxiv.org/abs/2310.01082) [↩](#user-content-fnref-7)

8.  tile-lang: [https://github.com/tile-ai/tilelang](https://github.com/tile-ai/tilelang) [↩](#user-content-fnref-8)

9.  TVM: [https://tvm.apache.org/](https://tvm.apache.org/) [↩](#user-content-fnref-9)

10. Self-attention does not need $𝑂(n^2)$ memory: [https://arxiv.org/abs/2112.05682](https://arxiv.org/abs/2112.05682) [↩](#user-content-fnref-10)

11. Profile FA3: [https://static.rainfocus.com/nvidia/gtcs25/sess/1725825619590001romQ/FinalPresPDF/S71368\_FlashAttention-3-%20Fast%20and%20Accurate%20Attention%20With%20Asynchrony%20and%20Low%20Precision\_1742587734302001Zsa8.pdf?utm\_source=chatgpt.com](https://static.rainfocus.com/nvidia/gtcs25/sess/1725825619590001romQ/FinalPresPDF/S71368_FlashAttention-3-%20Fast%20and%20Accurate%20Attention%20With%20Asynchrony%20and%20Low%20Precision_1742587734302001Zsa8.pdf?utm_source=chatgpt.com) [↩](#user-content-fnref-13)
