---
title: "大模型 GPU 通信（一）从 PCIe、NVLink 到 RDMA"
source: "个人博客"
category: "Inbox"
group: "机器学习"
author: "韦子扬"
url: "https://weiziyang.wiki/posts/gpu-networking/gpu_communication1"
published: 2026-08-21T08:53:11+08:00
saved: 2026-10-10T09:28:26+08:00
folo_key: "url::https://weiziyang.wiki/posts/gpu-networking/gpu_communication1"
tags:
  - "folo"
  - "Inbox"
  - "个人博客"
---

# 大模型 GPU 通信（一）从 PCIe、NVLink 到 RDMA

> [!info] 个人博客 · Inbox · 2026-08-21 08:53 · [原文](https://weiziyang.wiki/posts/gpu-networking/gpu_communication1)

> 写这篇文章主要是因为之前做模型训练的时候经常会碰到模型训练慢卡在通信的问题，配置项和日志里也经常会有nccl相关日志，如果不懂通信， 这块排查起来会很抓瞎，而且前沿的deepEP之类的论文也都高度依赖这些知识， 所以这块的基础一定要补上， 加深对ai infra通信的理解；

# 为什么要关心通信？

随着大模型越训越大， 训练需要的gpu也随之增加， 从以前的单卡到多卡单机到多机多卡， 训练的通信时间占到总时间的比重是逐渐增加的，甚至会到总时间的一半以上；可想而知，想把模型训好，高效的gpu通信越来越重要；

![[大模型 GPU 通信（一）从 PCIe、NVLink 到 RDMA-01.png]]

# 单机通信拓扑

![[大模型 GPU 通信（一）从 PCIe、NVLink 到 RDMA-02.png]]

**GPU**

这个就是核心的运算单元，在典型 HGX/DGX 大模型训练节点中，常见配置是 8 GPU， gpu里的数据存HBM上；

**PCIe**

PCIe是CPU、GPU、NIC、SSD 等设备之间的通用高速 I/O 总线；

**NIC**

nic是网卡，本机往外走都需要经过 NIC这层才行

## 同一台机器的GPU怎么通信

可以找台机器用命令：

`nvidia-smi topo -m`

看一下结果如下（这是我h200的结果，参考），这是为了引出后面的图解

TEXT

```text
      GPU0  GPU1  GPU2  GPU3  GPU4  GPU5  GPU6  GPU7  NIC0  NIC1  NIC2  NIC3  NIC4  NIC5  NIC6  NIC7  NIC8  CPU Affinity   NUMA  GPU NUMA
-------------------------------------------------------------------------------------------------------------------------------------------
GPU0   X    NV18  NV18  NV18  NV18  NV18  NV18  NV18  PIX   NODE  NODE  NODE  SYS   SYS   SYS   SYS   NODE  0,2-47,96-143  0     N/A
GPU1  NV18   X    NV18  NV18  NV18  NV18  NV18  NV18  NODE  PIX   NODE  NODE  SYS   SYS   SYS   SYS   NODE  0,2-47,96-143  0     N/A
GPU2  NV18  NV18   X    NV18  NV18  NV18  NV18  NV18  NODE  NODE  PIX   NODE  SYS   SYS   SYS   SYS   NODE  0,2-47,96-143  0     N/A
GPU3  NV18  NV18  NV18   X    NV18  NV18  NV18  NV18  NODE  NODE  NODE  PIX   SYS   SYS   SYS   SYS   NODE  0,2-47,96-143  0     N/A
GPU4  NV18  NV18  NV18  NV18   X    NV18  NV18  NV18  SYS   SYS   SYS   SYS   PIX   NODE  NODE  NODE  SYS   48-95,144-191  1     N/A
GPU5  NV18  NV18  NV18  NV18  NV18   X    NV18  NV18  SYS   SYS   SYS   SYS   NODE  PIX   NODE  NODE  SYS   48-95,144-191  1     N/A
GPU6  NV18  NV18  NV18  NV18  NV18  NV18   X    NV18  SYS   SYS   SYS   SYS   NODE  NODE  PIX   NODE  SYS   48-95,144-191  1     N/A
GPU7  NV18  NV18  NV18  NV18  NV18  NV18  NV18   X    SYS   SYS   SYS   SYS   NODE  NODE  NODE  PIX   SYS   48-95,144-191  1     N/A
NIC0  PIX   NODE  NODE  NODE  SYS   SYS   SYS   SYS    X    NODE  NODE  NODE  SYS   SYS   SYS   SYS   NODE
NIC1  NODE  PIX   NODE  NODE  SYS   SYS   SYS   SYS   NODE   X    NODE  NODE  SYS   SYS   SYS   SYS   NODE
NIC2  NODE  NODE  PIX   NODE  SYS   SYS   SYS   SYS   NODE  NODE   X    NODE  SYS   SYS   SYS   SYS   NODE
NIC3  NODE  NODE  NODE  PIX   SYS   SYS   SYS   SYS   NODE  NODE  NODE   X    SYS   SYS   SYS   SYS   NODE
NIC4  SYS   SYS   SYS   SYS   PIX   NODE  NODE  NODE  SYS   SYS   SYS   SYS    X    NODE  NODE  NODE  SYS
NIC5  SYS   SYS   SYS   SYS   NODE  PIX   NODE  NODE  SYS   SYS   SYS   SYS   NODE   X    NODE  NODE  SYS
NIC6  SYS   SYS   SYS   SYS   NODE  NODE  PIX   NODE  SYS   SYS   SYS   SYS   NODE  NODE   X    NODE  SYS
NIC7  SYS   SYS   SYS   SYS   NODE  NODE  NODE  PIX   SYS   SYS   SYS   SYS   NODE  NODE  NODE   X    SYS
NIC8  NODE  NODE  NODE  NODE  SYS   SYS   SYS   SYS   NODE  NODE  NODE  NODE  SYS   SYS   SYS   SYS    X

Legend:

X    = Self
       自身（对角线）
NV#  = Connection traversing a bonded set of # NVLinks
       经由 # 条聚合 NVLink 相连，GPU 间最快路径（此处 NV18 = 18 条 NVLink 聚合）
PIX  = Connection traversing at most a single PCIe bridge
       最多经过一个 PCIe 桥，同一 PCIe switch 下，GPU-NIC 最优亲和
PXB  = Connection traversing multiple PCIe bridges (without traversing the PCIe Host Bridge)
       跨多个 PCIe 桥，但不上到 Host Bridge（CPU root complex）
PHB  = Connection traversing PCIe as well as a PCIe Host Bridge (typically the CPU)
       需经过 PCIe Host Bridge（通常即 CPU root complex）
NODE = Connection traversing PCIe as well as the interconnect between PCIe Host Bridges within a NUMA node
       同一 NUMA node 内、跨 Host Bridge 互联，比 PIX 慢
SYS  = Connection traversing PCIe as well as the SMP interconnect between NUMA nodes (e.g., QPI/UPI)
       跨 NUMA node，需走 CPU 间 SMP 互联（QPI/UPI），最慢

NIC Legend:

  NIC0: mlx5_0
  NIC1: mlx5_3
  NIC2: mlx5_4
  NIC3: mlx5_5
  NIC4: mlx5_6
  NIC5: mlx5_7
  NIC6: mlx5_8
  NIC7: mlx5_9
  NIC8: mlx5_bond_0
```

### PCIe GPU通信

在没有nvlink的时候，gpu 之间也可以通过PCIe做p2p通信，而且这个通信也并不需要经过cpu和内存，PCIe switch负责转发， cpu不参与数据搬运；

![[大模型 GPU 通信（一）从 PCIe、NVLink 到 RDMA-03.png]]

这里面分几个情况：

#### **PIX**

最理想的情况， GPU 在**同一个 PCIe Switch** 下，通常就是最标准的 PCIe P2P 场景。NCCL 把 `PIX` 定义为 same PCI switch，并明确允许这种距离使用 P2P。

TEXT

```text
情况 1：PIX，最理想

        PCIe Switch
        /         \
     GPU0         GPU1

GPU0 ───────────▶ GPU1
      PCIe P2P
```

#### **PXB**

表示中间经过多个 PCIe bridge/switch。它**仍然可以支持 P2P**，只是路径更远。NCCL 也允许 P2P 跨多个 PCIe switch。

TEXT

```text
情况 2：PXB

      PCIe Switch
          |
      PCIe Switch
       /       \
    GPU0       GPU1
```

#### **PHB**

此时通信要经过 **CPU PCIe host bridge / root complex**，而不是在 PCIe switch 内部直接转发。PHB 拓扑下 NCCL 可以配置使用 P2P

TEXT

```text
情况 3：PHB

GPU0
  |
PCIe Root Port
  |
CPU / Host Bridge
  |
PCIe Root Port
  |
GPU1
```

#### **SYS**

这时候就是不仅跨 PCIe Root Complex，甚至要跨 NUMA socket。它不一定支持p2p；

PYTHON

```python
GPU0
  ↓
Root Complex 0
  ↓
CPU0
  ↓
UPI / QPI / SMP interconnect
  ↓
CPU1
  ↓
Root Complex 1
  ↓
GPU1
```

> NUMA（Non-Uniform Memory Access，非一致内存访问）是多路 CPU 服务器中的一种架构。每个 CPU 都有自己更“近”的本地内存和 PCIe 设备；如果访问另一个 CPU 所属的内存或设备，就需要经过 CPU 间互联，因此延迟更高、带宽也可能受到影响。

### NVLink

NVLink 是 NVIDIA 专门面向 GPU 等设备设计的高速互联链路,它比走总线的PCIe快很多，以常用的h100卡举例：

| 互联            | 总双向带宽    |
| ------------- | -------- |
| PCIe Gen5 x16 | 128 GB/s |
| NVLink 4      | 900 GB/s |
| 倍数            | 约 7×     |

![[大模型 GPU 通信（一）从 PCIe、NVLink 到 RDMA-04.png]]

如果只有nvlink的话，假设我们有八张卡，想要让他们互联， 就需要拉28根线，拓展性很差，为了解决这个问题，我们就需要引入NVswitch

### NVSwitch

NVSwitch类似于是一种交换机的形式，很显然，引入nvswitch以后，GPU 之间通常都能通过 NVLink fabric 高带宽互访，路径差异小很多，更容易扩展到多卡，集体通信性能更稳定。

![[大模型 GPU 通信（一）从 PCIe、NVLink 到 RDMA-05.png]]

总结如下：

![[大模型 GPU 通信（一）从 PCIe、NVLink 到 RDMA-06.png]]

# 多机通信拓扑

![[大模型 GPU 通信（一）从 PCIe、NVLink 到 RDMA-07.png]]

多机视角下分为三种情况：

**NIC + TCP/IP:** 这种就是用传统的以太网走TCP IP 网络去进行互联，软件栈开销和 CPU 参与较大，通常 latency 和有效 GPU-GPU throughput 不如 RDMA， ，但成本比较低；

**RoCE** : 这也是基于以太网的，但是它支持RDMA, 所以它性能能接近IB, 且成本更低；

**IB**: 这种是专业的比较高级的专用交换机，延迟最低， 带宽最高；

## NIC

![[大模型 GPU 通信（一）从 PCIe、NVLink 到 RDMA-08.png]]

NIC 一般是插在服务器的 **PCIe 插槽**上的扩展卡，例如 NVIDIA/Mellanox ConnectX-6、ConnectX-7。

也可能直接集成在主板上，但本质仍然是 NIC。**NVSwitch 通常管机内，NIC 主要管跨机。**

### AI 集群里的 NIC 有什么特殊之处？

其实家里的个人pc也有NIC千兆网卡，但那个是给个人用的，性能很差， 走的tcp， AI 集群关注的不是普通 TCP 网卡，而通常是**高性能 NIC**。比如ConnectX-7

它会支持：

- RDMA
- GPUDirect RDMA
- RoCEv2
- InfiniBand
- 多队列
- 高吞吐
- 低延迟

最关键的是 **RDMA**。

## RoCE(RDMA over Converged Ethernet)

在以太网上实现 RDMA 的一套协议,它解决的核心问题是：**机器之间传输数据时，尽量绕开 CPU 和内核网络协议栈，让网卡直接把数据搬到对端内存/GPU 显存里。 它和TCP/IP 是对应概念。**

**普通TCP的路径**

TEXT

```text
Application
    ↓
Socket
    ↓
Linux Kernel
    ↓
TCP/IP stack
    ↓
NIC
    ↓
Ethernet
```

数据到另一台机器以后再反过来走一遍。

这意味着 CPU 要参与：

- syscall
- TCP 协议处理
- packet processing
- buffer copy
- context switch

RoCE 是

TEXT

```text
Application
    ↓
RDMA Verbs
    ↓
RNIC
    ↓
Ethernet
    ↓
Remote RNIC
    ↓
Remote Memory
```

### RoCE为什么怕丢包

TCP 是为有损网络设计的：

TEXT

```text
丢包
↓
TCP detection
↓
重传
↓
拥塞控制
```

所以 Ethernet 丢一点包无所谓。

但 RDMA 最初是按一种非常低丢包、高性能网络思路设计的。

RoCE 在 Ethernet 上跑之后：

TEXT

```text
Ethernet 本来可能丢包
+
RDMA 对丢包很敏感
```

RoCE 网络必须非常认真地做拥塞控制。

### PFC(Priority Flow Control)

交换机 buffer 快满时：让上游暂时别发数据。

它是以太网 Layer 2 的机制。

作用：**防止交换机 buffer overflow 导致丢包。**

### ECN(Explicit Congestion Notification)

当交换机发现Queue 快拥塞了,不是直接丢包，而是在 packet 上打一个标记：ECN = congestion,

接收端发现以后反馈给发送端：你发慢点；

### RoCE为什么更通用

从性能上看RoCE往往不是最优的， 但是它一定是性价比最高的，最通用的，因为数据中心大多数都是

TEXT

```text
Ethernet NIC
Ethernet Switch
IP network
```

可以复用很多现有技术体系。

而且Switch 厂商多，IB 很大程度是 NVIDIA/Mellanox 生态， 但Ethernet 厂商更多

## RDMA

这里对RDMA不做太深入的展开,这里只简单介绍一下， 如果没有RDMA，数据需要经过一层内存拷贝，需要CPU参与， 而RDMA 的核心思想，是让一台机器上的设备能够更直接地访问远端内存，减少 CPU 和内核参与。RDMA 是 remote direct memory access， 它可以实现 kernel bypass、减少 CPU involvement、减少 memory copy，zero-copy 是其中可能获得的效果

![[大模型 GPU 通信（一）从 PCIe、NVLink 到 RDMA-09.png]]

## GPUDirect RDMA

普通的RDMA，可以绕过CPU和内核，直接访问远端内存， 而GPUDirectRDMA则更进一步， NIC/HCA 可以直接 DMA GPU memory，避免 GPU↔Host DRAM staging

# 总结：一个 Tensor 是如何从 GPU0 到 GPU3 的

一图以蔽：

![[大模型 GPU 通信（一）从 PCIe、NVLink 到 RDMA-10.png]]

# 这些拓扑最终是怎么被 NCCL 使用的？

前面介绍的 NVLink、PCIe、NIC 等硬件拓扑，最终都会被 NCCL 感知并用于通信路径选择。

NCCL 初始化时会识别 GPU、CPU、PCIe、NVLink/NVSwitch 和 NIC 之间的拓扑关系。例如对于同机 GPU，NCCL 会优先选择更近的 P2P 路径；对于跨机通信，则会结合 GPU 与 NIC 的亲和关系选择合适的网卡，并在条件允许时使用 GPUDirect RDMA。

因此我们在 `nvidia-smi topo -m` 中看到的 `NVL / PIX / PXB / PHB / SYS` 并不只是硬件信息，它们会直接影响 NCCL 对通信路径的判断。

可以简单理解为：

![[大模型 GPU 通信（一）从 PCIe、NVLink 到 RDMA-11.png]]

所以在排查 NCCL 通信性能问题时，`nvidia-smi topo -m` 往往是最先需要确认的信息之一。

至于 NCCL 在这些物理路径之上如何实现 **AllReduce、AllGather、ReduceScatter、AllToAll**，以及 Ring、Tree 等通信算法，就属于下一篇要讨论的内容了。

# 参考官方文档

[https://docs.nvidia.com/deploy/nvidia-smi/index.html](https://docs.nvidia.com/deploy/nvidia-smi/index.html) 官方直接定义了 `SYS / NODE / PHB / PXB / PIX / NV#`，包括 SYS 跨 NUMA/QPI/UPI 的含义。

[https://www.nvidia.com/en-us/data-center/h100/](https://www.nvidia.com/en-us/data-center/h100/) h100官方文档

[https://docs.nvidia.com/cuda/gpudirect-rdma/?utm\\\_source=chatgpt.com](https://docs.nvidia.com/cuda/gpudirect-rdma/?utm%5C_source=chatgpt.com) rdma官方文档

[https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/index.html?utm\\\_source=chatgpt.com](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/index.html?utm%5C_source=chatgpt.com) nccl官方文档
