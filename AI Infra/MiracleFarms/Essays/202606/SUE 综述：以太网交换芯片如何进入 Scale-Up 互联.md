---
title: "SUE 综述：以太网交换芯片如何进入 Scale-Up 互联"
source: "Essays"
category: "AI Infra"
group: "MiracleFarms"
url: "https://miraclefarms.github.io/notes/2026/06/21/sue-scale-up-ethernet-fabric/"
published: 2026-06-21T15:40:00+08:00
saved: 2026-09-27T16:28:45+08:00
folo_key: "mf-essays::https://miraclefarms.github.io/notes/2026/06/21/sue-scale-up-ethernet-fabric/"
tags:
  - "folo"
  - "AI_Infra"
  - "Essays"
---

# SUE 综述：以太网交换芯片如何进入 Scale-Up 互联

> [!info] Essays · AI Infra · 2026-06-21 15:40 · [原文](https://miraclefarms.github.io/notes/2026/06/21/sue-scale-up-ethernet-fabric/)

> **版本历史**
> 
> | 版本   | 日期         | 说明                                                                                                   |
> | ---- | ---------- | ---------------------------------------------------------------------------------------------------- |
> | v1.0 | 2026-06-21 | 初稿，基于 OCP《Scale Up Ethernet (SUE)》v1.0.0、Broadcom Tomahawk Ultra 与 OCP ESUN 资料，梳理 SUE 的协议栈、交换芯片和生态边界 |
> | v1.1 | 2026-06-21 | 增补 UALink/SUE/UEC 的 PHY/PL/DL/TL 分层差异、UALink 交换芯片进展，以及 Broadcom 在 Ethernet 与 UALink 路线上的商业取舍         |
> | v1.2 | 2026-06-23 | 增补 Broadcom 交换产品对比表，并加入 SUE 信息图作为题图                                                                  |

![[SUE 综述：以太网交换芯片如何进入 Scale-Up 互联-01.jpg]]

*题图：SUE 把 XPU 端点事务、SUE/SUE Lite、AFH/LLR/CBFC 和 Broadcom Ethernet 交换产品线连成一条开放 scale-up 路径。*

SUE 最容易被误读成“用以太网替代 NVLink”。这个说法抓住了商业冲突，却漏掉了技术主线。SUE 要做的事情更具体：把 Ethernet 的 MAC/PHY、交换芯片、线缆生态和运维经验压缩到一个 rack-scale scale-up 域里，让 XPU 之间可以跑 load / store / atomic 这类一侧内存事务。**它的赌注是：scale-up fabric 可以长在以太网生态上，只要把端点接口、封装、可靠性、流控和交换芯片时延一起改到位。**

这让 SUE 在南向互联版图里占了一个很微妙的位置。NVLink 和华为 UB 更像垂直闭环：硬件、互联、通信库、运行时一起设计，软件能吃到更强的专有语义。UALink 走开放专用协议路线：用 Ethernet 物理层生态，但在协议栈上重新定义 memory-semantic fabric。SUE 更贴近交换芯片厂商的思路：保留 Ethernet 合规性，用更小的 AI Fabric Header、链路层重传、credit-based flow control 和低时延交换芯片，把以太网短路径改造成 scale-up pod。**如果 NVLink 的核心资产是闭环优化，SUE 的核心资产就是交换芯片供应链和以太网运维生态。**

前期超节点南向互联调研已经把这些路线放进同一张地图：NVLink/UB 代表单厂商闭环，UALink 代表开放专用总线，SUE/ESUN 则代表 Ethernet 交换生态向 rack-scale 内存事务域下沉[\[10\]](https://miraclefarms.github.io/notes/2026/06/21/supernode-southbound-interconnect-survey/)。本文的重点，是沿着 SUE 这条分支继续拆交换层和协议分层。

**（v1.1 更新）** 把这场竞争拆到协议分层后，争议会更清楚。多数开放路线都想复用同一代 100G/200G PAM4、以太网物理层、光模块和测试生态；真正的分野发生在 PCS/RS、data link、transaction 和 memory semantics。SUE 的策略是把 Ethernet data link 改到足够低时延、lossless、可转发内存事务；UALink 的策略是借用 Ethernet 物理层经验，但从 data link 和 transaction layer 开始定义专用 accelerator fabric。**底层物理生态趋同，上层事务语义分裂，是这一代 scale-up 标准之争的主线。**

## 一、SUE 的第一性问题：64 个 XPU 怎么看起来像一个低延迟域

> **本节论点**：SUE 白皮书的起点是单跳 switched deployment；它要把 XPU 间通信压进一个已知端点、短距离、多平面的低延迟域，任意规模数据中心路由留给 scale-out 网络处理。 **本节核心参考**：
> 
> - ★★★ **OCP《Scale Up Ethernet (SUE)》v1.0.0**[\[1\]](https://www.opencompute.org/documents/ocp-sue-spec-final-pdf-1) — SUE 部署模型、需求表和协议栈的一手规范
> - ★★ OCP ESUN 公告[\[3\]](https://www.opencompute.org/blog/introducing-esun-advancing-ethernet-for-scale-up-ai-infrastructure-at-ocp) — 说明 ESUN 与 SUE-T 在 OCP 中的分工

SUE 白皮书把 scope 写得很窄：它定义的是 XPU 之间 load / store / single-sided memory operations over Ethernet 的 transport[\[1\]](https://www.opencompute.org/documents/ocp-sue-spec-final-pdf-1)。这个范围很重要，因为它主动放弃了通用 NIC 的许多职责。SUE 不定义 XPU 之间的高级互操作协议，不负责内存注册和保护域管理，也不承诺不同厂商 XPU 自动互通。它处理的是更底层的一段：XPU 已经知道要向哪个远端 XPU 发命令，SUE 负责把这条命令低时延、可靠、可调度地送过去。

白皮书的主部署模型是 single-hop switched deployment。示例里有 64 个 XPU，每个 XPU 集成 12 个 800G SUE instance，后面接 12 个 64 口 Ethernet switch；在全交换连接下，任意两 XPU 之间最高有 9.6Tbps，也就是 12 × 800G 的带宽[\[1\]](https://www.opencompute.org/documents/ocp-sue-spec-final-pdf-1)。这个指标描述的是 scale-up 域的物理形态：端点集合已知，路径层级很浅，所有连接可以在启动时静态建立，运行时重心转向 line-rate 起步、incast 处理、按 VC 调度和故障绕行。

![[SUE 综述：以太网交换芯片如何进入 Scale-Up 互联-02.png]]

*图 1：SUE 白皮书的 64 XPU 单跳交换部署示例。每个 XPU 通过多个 SUE instance 接入多平面交换芯片，任意两 XPU 的带宽来自多个 800G 平面叠加。来源：OCP SUE 白皮书 Figure 1。*

这个模型把 SUE 和 scale-out 网络分开了。Scale-out 网络要处理更大规模、更长路径、动态拥塞和多租户路由，packet spraying、选择性重传、复杂拥塞控制都很有价值。SUE 白皮书的判断更直接：scale-up 网络规模更小、RTT 更低、带宽高一个数量级、端点在启动时已知，因而可以用单平面 single-path transport 和 lossless 链路把状态压小。**SUE 的工程哲学是用拓扑约束换实现简化：既然 rack-scale XPU 域可以被设计成短路径、少层级，就不要把 scale-out transport 的全部复杂度搬进端点。**

需求表也给了边界：最多 1024 个 XPU、单跳低延迟、RTT 小于 2us、每个 SUE instance 800Gbps 并可扩到 1.6Tbps、200Gbps SerDes、最多 4 个 virtual channel、线缆从 SUE 到 switch 最长 10m[\[1\]](https://www.opencompute.org/documents/ocp-sue-spec-final-pdf-1)。这些数字说明 SUE 的目标集中在 rack / multi-rack 的短距离内：把以太网压到内存事务能接受的时延区间。

## 二、协议栈的核心动作：把内存事务塞进 Ethernet PDU

> **本节论点**：SUE 的关键层次是 mapping / packing / transport / Ethernet datalink；XPU 语义保持 opaque，SUE 只按 destination 和 VC 把事务打包、排序、确认和转发。 **本节核心参考**：
> 
> - ★★★ **OCP《Scale Up Ethernet (SUE)》v1.0.0**[\[1\]](https://www.opencompute.org/documents/ocp-sue-spec-final-pdf-1) — SUE Stack、接口和封装规则的一手规范
> - ★ Synopsys ESUN/SUE/UALink 概览[\[5\]](https://www.synopsys.com/articles/ethernet-standards-scale-up-ai.html) — 生态视角下的三套开放 scale-up 分层对比

SUE stack 的好处是边界干净。XPU NOC 通过 signal/FIFO 或 AXI 4 接口把 command 交给 SUE；command 的操作码、读写语义、是否映射成 put/get/atomic，由 XPU 自己定义。SUE 只需要看见 remote XPU identifier、VC、控制长度和数据长度。随后 mapping and packing layer 把同一个 destination / VC 的多个 command opportunistically 打包进一个 SUE PDU；transport 层给 PDU 加 reliability header 和 R-CRC；network layer 再生成标准 Ethernet/IP/UDP 头，或更小的 AFH Gen 1 / Gen 2 头[\[1\]](https://www.opencompute.org/documents/ocp-sue-spec-final-pdf-1)。

![[SUE 综述：以太网交换芯片如何进入 Scale-Up 互联-03.png]]

*图 2：SUE stack 把 XPU 事务映射到 Ethernet packet：XPU 语义在上层保持 opaque，SUE 负责 packing、transport reliability、AFH/Ethernet 封装与链路层 lossless。来源：OCP SUE 白皮书 Figure 3。*

这种设计把“开放”和“互操作”切开了。SUE 是开放规范，落在 OCP 的 OWFa/FSA 框架下；但它不试图让不同 XPU 的内存模型自动一致。XPU 可以通过 AXI 4，把 read request 映射到 VC0，把 read data response 映射到 VC1；也可以用更开放的 signal interface。SUE 传输 control/data，少量字段被用于寻址、长度、VC 和调度，其余控制位大多作为 opaque payload 送到远端。**SUE 开放 fabric 承载方式，XPU 微架构仍由各家自己掌握。**

这也是它和 UALink 的一条关键分界。UALink 白皮书把自己描述为四层功能架构：Protocol、Transaction、Data Link、Physical，直接面向 memory semantics、57-bit 物理地址空间、最高 1024 accelerator 的 switched pod，并给出 93% effective bandwidth target[\[6\]](https://ualinkconsortium.org/wp-content/uploads/2026/01/UALink_White_Paper_Publication_FINAL_UPDATED.pdf)。SUE 则更像可嵌入 XPU 的 transport + datalink 框架：上层内存服务由 XPU 定义，SUE 负责把命令搬过以太网。

这里没有谁天然更优，只有集成点不同。UALink 把更多语义纳入标准化协议栈，未来互操作目标更强；SUE 把更多语义留给 XPU 厂商和系统集成商，换来 IP 面积、功耗和实现自由度。对于已经有自家 XPU NOC / memory model / cache coherence 设计的厂商，SUE 这种 opaque command channel 反而更容易接进去。

## 三、SUE Lite 是端点面积与可靠性位置的取舍

> **本节论点**：SUE Lite 的本质是把端到端可靠 transport 从端点里拿掉，依赖 LLR 和 lossless 链路承接可靠性，从而换取更小 IP 面积和更低端点开销。 **本节核心参考**：
> 
> - ★★★ **OCP《Scale Up Ethernet (SUE)》v1.0.0**[\[1\]](https://www.opencompute.org/documents/ocp-sue-spec-final-pdf-1) — SUE Lite profile、封装和处理流程的一手规范
> - ★★ Broadcom Tomahawk Ultra 发布资料[\[2\]](https://investors.broadcom.com/news-releases/news-release-details/broadcom-ships-tomahawk-ultra-reimagining-ethernet-switch-hpc) — 产品层面把 SUE Lite 定位为低面积/低功耗集成路径

SUE Lite 是白皮书里最能暴露工程取舍的一段。完整版 SUE 有 reliable transport layer，提供端到端可靠性、固定窗口拥塞控制、partition feature、AXI 4 或 signal interface，transaction packing 可以更大。SUE Lite 则移除 reliable transport layer，只保留 signal-based interface，packing size 限制到 1KB，端到端可靠性和 congestion control 都被拿掉，可靠性主要交给 hop-by-hop LLR 和 lossless 链路[\[1\]](https://www.opencompute.org/documents/ocp-sue-spec-final-pdf-1)。白皮书直接说，这样可以把 SUE IP size 降到最多 50%。

![[SUE 综述：以太网交换芯片如何进入 Scale-Up 互联-04.png]]

*图 3：SUE Lite 去掉 reliable transport layer 和 R-CRC，只用 AFH Gen 2 6B 头里的 destination、source XPU address 与 VC 完成转发，最多打包 1KB PDU。来源：OCP SUE 白皮书 Figure 10。*

这件事不能只按“精简版”理解。它真正改变的是可靠性责任放在哪里。完整版 SUE 在端点 transport 层有 PSN、ACK/NACK、GoBackN 和 R-CRC；SUE Lite 则假设单跳交换域、lossless class、LLR 和有序平面足够稳定，端点无需承担完整重传状态。**在 XPU package 里放 8 个或 16 个 SUE instance 时，transport state 的面积和功耗会变成硬约束；SUE Lite 让厂商可以把这部分成本从端点 IP 转移到交换芯片和链路层能力上。**

Broadcom 的 Tomahawk Ultra 发布资料也沿着这条线讲：SUE Lite 面向 power and area-sensitive accelerator applications，保留低延迟和 lossless 特性，同时进一步降低 XPU/CPU 上 Ethernet interface 的 silicon footprint 与 power consumption[\[2\]](https://investors.broadcom.com/news-releases/news-release-details/broadcom-ships-tomahawk-ultra-reimagining-ethernet-switch-hpc)。这是芯片 floorplan 里的选择：端点多放一层 transport，就少一点算力、缓存、SerDes 或封装预算。

这条线也解释了为什么 SUE 对交换芯片要求高。SUE Lite 把可靠性和拥塞隔离更多交给网络，交换芯片就不能只是“能转包”。它必须在极低时延下支持 LLR、CBFC、lossless class、AFH lookup、短头部转发和故障状态反馈。换句话说，SUE Lite 看起来减了端点，实际上把一部分复杂度推进了 fabric。

## 四、AFH、LLR、PFC/CBFC：以太网被压成 scale-up 形态

> **本节论点**：SUE 对 Ethernet 的改造集中在三件事：头部变小、链路错误就地恢复、流控从粗粒度 PFC 走向更细的 CBFC。 **本节核心参考**：
> 
> - ★★★ **OCP《Scale Up Ethernet (SUE)》v1.0.0**[\[1\]](https://www.opencompute.org/documents/ocp-sue-spec-final-pdf-1) — AFH Gen 1/Gen 2、FEC、LLR、PFC 与 CBFC 说明
> - ★★ Broadcom Tomahawk Ultra 发布资料[\[2\]](https://investors.broadcom.com/news-releases/news-release-details/broadcom-ships-tomahawk-ultra-reimagining-ethernet-switch-hpc) — 10B 优化头、lossless fabric 和 64B line-rate 产品口径
> - ★ UEC 1.0 白皮书[\[7\]](https://ultraethernet.org/wp-content/uploads/sites/20/2025/06/UEC1.0Whitepaper.pdf) — CBFC/LLR 所在的 Ultra Ethernet 生态背景

传统 Ethernet 头部对 AI scale-up 负载太重。SUE 白皮书给了三种网络封装：标准 Ethernet/IP/UDP；AFH Gen 1，保留标准 MAC destination/source address 的形态，但交换硬件只看 16-32 bit 做 forwarding lookup；AFH Gen 2，用 IEEE 802.c-2017 的 Structured Local Address Plan，把 forwarding information 压到 6B 或 12B[\[1\]](https://www.opencompute.org/documents/ocp-sue-spec-final-pdf-1)。Broadcom 在 Tomahawk Ultra 发布稿里把产品口径进一步压成一句话：header overhead 可以从 46B 降到最低 10B，同时保持 Ethernet compliance[\[2\]](https://investors.broadcom.com/news-releases/news-release-details/broadcom-ships-tomahawk-ultra-reimagining-ethernet-switch-hpc)。

AI scale-up 里，小包和控制事务不只是边角料。MoE dispatch/combine、barrier、atomic、cacheline-sized memory transaction、ACK/NACK 都会被头部成本放大。如果 payload 只有几十到几百字节，传统 IP/UDP/Ethernet 头部会直接吃掉有效带宽。AFH 的意义就在这里：**它把“以太网兼容”从“照搬完整通用头部”改成“在标准允许的局部地址空间里重写低延迟转发格式”。**

可靠性路径同样体现 scale-up 约束。FEC 可以纠错，但 FEC block 越大、交织越重，时延越高；白皮书明确提到 RS-272 这类较小 FEC block 可用更低纠错能力换更低时延。LLR 则在链路层就地重传单个损坏包，避免等端点 transport 发现丢包或损坏[\[1\]](https://www.opencompute.org/documents/ocp-sue-spec-final-pdf-1)。这和 scale-out 里常见的端到端恢复思路不同：single-hop scale-up 域里，链路错误越早修越便宜。

流控是更难的部分。PFC 可以按 priority 暂停，SUE 要求至少两个 lossless PFC class，一个给 request，一个给 ack，避免 request 阻塞 ack 引发 deadlock。CBFC 则是更适合 SUE 的方向：PFC 只有 8 个 class，CBFC 支持 32 个 class；发送端知道每个 VC 的 credit usage，可以把这些信息用于 scheduling 和 load balancing；同等 buffer 条件下，CBFC 对链路长度/延迟所需 buffering 更小[\[1\]](https://www.opencompute.org/documents/ocp-sue-spec-final-pdf-1)。这就是以太网进入 scale-up 的现实代价：想要 lossless 和低尾延迟，就必须把“网络会丢包、端点慢慢恢复”的旧假设往前改。

## 五、Tomahawk Ultra 是 SUE 的产品化辅线

> **本节论点**：SUE 白皮书回答协议和端点集成，Tomahawk Ultra 回答交换芯片能否在时延、吞吐、小包、lossless 和 collective offload 上承接这套协议。 **本节核心参考**：
> 
> - ★★★ **Broadcom Tomahawk Ultra 发布资料**[\[2\]](https://investors.broadcom.com/news-releases/news-release-details/broadcom-ships-tomahawk-ultra-reimagining-ethernet-switch-hpc) — 51.2Tbps、250ns、64B line-rate、LLR/CBFC、in-network collectives 的产品口径
> - ★★★ **OCP《Scale Up Ethernet (SUE)》v1.0.0**[\[1\]](https://www.opencompute.org/documents/ocp-sue-spec-final-pdf-1) — 单向延迟预算和 SUE processing flow

Tomahawk Ultra 是理解 SUE 的最好产品注脚。Broadcom 给出的核心参数很硬：51.2Tbps 交换吞吐、满吞吐下 250ns switch latency、64B minimum packet 线速、最高 77B packets/s、LLR + CBFC lossless fabric、optimized Ethernet header、in-network collectives[\[2\]](https://investors.broadcom.com/news-releases/news-release-details/broadcom-ships-tomahawk-ultra-reimagining-ethernet-switch-hpc)。这些参数逐项对应 SUE 的痛点：小包效率、单跳时延、链路错误恢复、buffer 溢出、collective 热路径。

![[SUE 综述：以太网交换芯片如何进入 Scale-Up 互联-05.png]]

*图 4：SUE processing flow。端点发出 command，SUE 按 destination/VC 打包、调度、加 reliability header，再经过 Ethernet header 转发到目标 XPU。来源：OCP SUE 白皮书 Figure 18。*

白皮书的单向时延预算把这件事算得很直观。SUE 到 Ethernet bridge 的 Tx+Rx 小于 100ns，Ethernet link/PHY 的 Tx+Rx 小于 100ns，switch Tx+Rx 小于 250ns；线缆传播时延按长度和介质叠加。3m twinax copper 的总单向预算是 477.6ns；5m hollow core fiber 是 496ns；10m hollow core fiber 是 520ns；30m single-mode fiber 是 747.6ns[\[1\]](https://www.opencompute.org/documents/ocp-sue-spec-final-pdf-1)。Broadcom 发布稿里的 sub-400ns XPU-to-XPU communication latency（包含 switch transit time）则是更偏产品营销的口径，需要结合具体线缆、端点和系统实现理解[\[2\]](https://investors.broadcom.com/news-releases/news-release-details/broadcom-ships-tomahawk-ultra-reimagining-ethernet-switch-hpc)。

![[SUE 综述：以太网交换芯片如何进入 Scale-Up 互联-06.png]]

*图 5：SUE 单向延迟预算把端点 bridge、PHY、switch 和线缆逐项拆开。SUE 的低延迟来自端点 IP、交换芯片、线缆距离和 FEC/LLR 策略的共同压缩，单个芯片数字只覆盖其中一段。来源：OCP SUE 白皮书 Figure 22。*

In-network collectives 是 Tomahawk Ultra 里需要继续追的一条辅线。发布稿称它可以在交换芯片内执行 AllReduce、Broadcast、AllGather，减轻 XPU 负担并改善 job completion time[\[2\]](https://investors.broadcom.com/news-releases/news-release-details/broadcom-ships-tomahawk-ultra-reimagining-ethernet-switch-hpc)。这和 NVIDIA NVSwitch SHARP、Hopper 之后 fabric 内 reduction 的方向相通。差别在于 Broadcom 试图把它放进 open Ethernet switch 叙事里，让 endpoint-agnostic 成为卖点。

边界也要写清楚。Tomahawk Ultra 已经 shipping，但公开资料主要是产品发布和 partner quote，缺少真实 MoE / TP / EP / KV 场景下的端到端 benchmark。51.2Tbps 和 250ns 是交换芯片层面的必要条件；scale-up 系统是否成立，还要看 XPU 端点里的 SUE/SUE Lite 实现、collective library 如何使用 AFH/VC/CBFC、调度器能否识别 SUE domain，以及故障或 incast 下 p99 是否稳住。**交换芯片能让以太网进入赛场，完整软件栈决定它能不能留下来。**

## 六、SUE、ESUN、UALink、NVLink、UB 的边界在哪里

> **本节论点**：SUE 属于 Ethernet-derived scale-up 路线的一块拼图；ESUN 负责开放交换/帧格式协作，SUE-T 负责端点 transport 工作流，UALink 定义更完整的 memory-semantic fabric，NVLink/UB 则代表厂商闭环的高性能路径。 **本节核心参考**：
> 
> - ★★★ **OCP ESUN 公告**[\[3\]](https://www.opencompute.org/blog/introducing-esun-advancing-ethernet-for-scale-up-ai-infrastructure-at-ocp) — ESUN 与 SUE-T 范围划分的一手说明
> - ★★ Arista ESUN 博客[\[4\]](https://blogs.arista.com/blog/the-sun-rises-on-scale-up-ethernet) — 生态方对 ESUN/SUE-T 分层的解释
> - ★★ UALink 2026 白皮书[\[6\]](https://ualinkconsortium.org/wp-content/uploads/2026/01/UALink_White_Paper_Publication_FINAL_UPDATED.pdf) — UALink 与 Ethernet-derived approaches 的比较
> - ★ 前期调研文章《NVLink 演进史》[\[9\]](https://miraclefarms.github.io/notes/2026/06/21/nvlink-evolution-from-p100-to-rubin-feynman/) — NVLink 作为垂直闭环 scale-up fabric 的参照

OCP 的 ESUN 公告帮我们把几个名字分开了。ESUN 是 Ethernet for Scale-Up Networking，是 OCP Networking Project 里的新 workstream，初始参与方包括 AMD、Arista、ARM、Broadcom、Cisco、HPE Networking、Marvell、Meta、Microsoft、NVIDIA、OpenAI、Oracle。它的关注点是 scale-up 网络里的 L2/L3 Ethernet framing and switching、lossless data transfer、error handling、interoperability，并明确排除 host-side stacks、non-Ethernet protocols、application-layer solutions 和 proprietary technologies[\[3\]](https://www.opencompute.org/blog/introducing-esun-advancing-ethernet-for-scale-up-ai-infrastructure-at-ocp)。

SUE-T 则是端点侧 workstream。OCP 公告写得很清楚：XPU-endpoint domain 依赖 workload partitioning、memory ordering、load balancing，常常和 XPU 架构紧耦合；此前的 SUE workstream 已改名并澄清为 SUE-Transport（SUE-T），会继承 Broadcom 贡献的 SUE v1.0 部分工作[\[3\]](https://www.opencompute.org/blog/introducing-esun-advancing-ethernet-for-scale-up-ai-infrastructure-at-ocp)。Arista 的解释更像工程分层：ESUN 提供 Ethernet headers、data link layer、PHY 这些公共底座，SUE-T 可以作为 upper layer transport 之一，处理 reliability scheduling、load balancing、transaction packing[\[4\]](https://blogs.arista.com/blog/the-sun-rises-on-scale-up-ethernet)。

UALink 的定位更完整。它同样借用 Ethernet 物理层组件，但协议上定义 purpose-built fabric、memory semantics、read/write/atomic、最高 1024 accelerators per pod、sub-microsecond RTT 和 93% effective bandwidth target[\[6\]](https://ualinkconsortium.org/wp-content/uploads/2026/01/UALink_White_Paper_Publication_FINAL_UPDATED.pdf)。Synopsys 的概览把三者关系总结得比较中性：ESUN 协调开放 Ethernet 行为，SUE 提供 pod-level encapsulation + reliability，UALink 引入 shared-memory workload 所需的 memory semantics[\[5\]](https://www.synopsys.com/articles/ethernet-standards-scale-up-ai.html)。这三者并不一定互斥，未来系统完全可能在物理层、交换层、端点 transport、内存语义层混合采用不同规范。

NVLink 和 UB 给了另一类参照。前期 NVLink 分析已经梳理过，NVLink 从 P100 的点对点链路走到 GB200/Rubin 的 rack-scale fabric，软件侧由 NCCL、NVSHMEM、CUDA VMM、TensorRT-LLM one-sided AllToAll 和调度器一起消费 NVLink domain[\[9\]](https://miraclefarms.github.io/notes/2026/06/21/nvlink-evolution-from-p100-to-rubin-feynman/)。华为 CloudMatrix384 则通过 UB plane 把 384 个 Ascend 910C NPU 和 192 个 Kunpeng CPU 组织成超节点，论文给出的 UB 跨节点 NPU-NPU read 带宽退化低于 3%，512B read 延迟跨节点 1.9us、节点内 1.2us，并把 EMS 分布式内存池放进 UB 访问路径[\[8\]](https://arxiv.org/html/2506.12708v1)。这两条路线都更像单厂商的深度闭环，性能和软件语义可能更强，生态自由度更低。

把这几条路线放在同一张表里，比较对象就清楚了：

| 路线                 | 主要控制点                                             | 强项                                               | 风险                                |
| ------------------ | ------------------------------------------------- | ------------------------------------------------ | --------------------------------- |
| NVLink / NVSwitch  | NVIDIA GPU、switch、CUDA/NCCL/NVSHMEM/runtime       | 最高性能闭环已经产品化，NVL72/NVL576/Rubin 路线清晰              | 生态绑定强，第三方接入依赖 NVLink Fusion 等授权路径 |
| Huawei UB          | Ascend/Kunpeng/UB/CANN/HCCL/CloudMatrix 系统        | CPU/NPU/内存池一体化，生产论文已有 KV/EMS 案例                  | 公开生态和跨厂商互操作有限，很多指标仍依赖厂商材料         |
| UALink             | 联盟规范，Ethernet PHY + purpose-built memory fabric   | 规范层直接定义 memory semantics，目标 1024 accelerator pod | 硬件与多厂商互操作仍在 2026 验证期              |
| SUE / SUE-T / ESUN | OCP workstream + Ethernet switch / endpoint IP 生态 | 复用 Ethernet 交换芯片、运维和供应链，Tomahawk Ultra 已产品化      | 上层 XPU 内存语义不统一，端到端软件栈成熟度待验证       |

我的判断是：**SUE 短期内最适合成为“开放 scale-up Ethernet 的交换芯片路径”；跨厂商共享内存语义仍要等待更完整的软件和端点生态。**它能把低延迟 Ethernet switch、AFH、LLR、CBFC、SUE Lite 和 in-network collectives 先送到系统厂商手里；至于上层是否形成和 NVLink/NVSHMEM、UALink memory semantics 同等级的软件体验，还要等端点 IP、通信库、调度器和真实 workload benchmark 共同补齐。

## 七、分层看分野：PHY 趋同，DL/TL 开始分裂

> **本节论点**：开放 scale-up 标准很少从 SerDes 层完全分道扬镳；真正决定系统形态的是 data link 的可靠性/流控模型和 transaction layer 的内存语义。 **本节核心参考**：
> 
> - ★★★ **UALink 200G 1.0 规范概览**[\[11\]](https://ualinkconsortium.org/blog/ualink-200g-1-0-specification-overview-802/) — 明确 UALink 的 TL/DL/PHY 分层与 IEEE P802.3dj PHY 基础
> - ★★★ **OCP《Scale Up Ethernet (SUE)》v1.0.0**[\[1\]](https://www.opencompute.org/documents/ocp-sue-spec-final-pdf-1) — SUE stack、AFH、LLR、PFC/CBFC 与 SUE Lite 的一手规范
> - ★★ UALink Specifications 页面[\[12\]](https://ualinkconsortium.org/specification/) — UALink 200G 1.0、Common 2.0、DL/PL 2.0 与 Chiplet 规格的公开状态

**（v1.1 新增）** 如果把南向互联粗略拆成 PHY/PL/DL/TL 四层，SUE 和 UALink 的差异不是“谁用不用以太网物理层”这么简单。UALink 公开概览写得很直白：Transaction Layer 管理 credits、TL flit packing/unpacking；Data Link Layer 负责 TL 和 Physical Layer 之间的数据搬运；Physical Layer 包含 PCS，并基于 IEEE P802.3dj draft，以便复用 Ethernet 技术再按 UALink 需要做调整[\[11\]](https://ualinkconsortium.org/blog/ualink-200g-1-0-specification-overview-802/)。换言之，UALink 在 PMD/PMA/SerDes 这层并不想另起炉灶，它从 DL/TL 开始把通用 packet network 改成 accelerator memory-semantic fabric。

SUE 的分层路径不同。SUE 仍然把每个 PDU 映射成一个 Ethernet packet，网络层可以使用标准 Ethernet header，也可以使用 AFH Gen 1/Gen 2 优化头；链路侧用 FEC、LLR、PFC/CBFC 提供 lossless 行为；transport 层再处理 per-destination/VC packing、strict ordering / unordered mode 和可靠传输[\[1\]](https://www.opencompute.org/documents/ocp-sue-spec-final-pdf-1)。**SUE 的分野发生在 Ethernet 内部：它保留 Ethernet 交换芯片和运维生态，但把 header、flow control、link retry 和 transport scheduling 都向 scale-up 事务压缩。**

这件事可以用一张分层表看得更清楚：

| 路线          | PHY / PMD / PMA                          | PL / PCS / RS                            | DL                                       | TL / Protocol                                                   |
| ----------- | ---------------------------------------- | ---------------------------------------- | ---------------------------------------- | --------------------------------------------------------------- |
| SUE / ESUN  | Ethernet PHY / 200G SerDes 生态            | Ethernet-compatible，配合 AFH/低时延 FEC       | LLR、PFC/CBFC、lossless class              | SUE-T / SUE transport，XPU command 多数 opaque                     |
| UEC         | Ethernet PHY                             | Ethernet PHY/PCS 演进                      | 面向 AI/HPC 的 congestion signaling、CBFC 等  | UET / advanced RDMA、multipath、out-of-order、selective retransmit |
| UALink      | Ethernet-derived PHY，基于 IEEE P802.3dj 方向 | 为 UALink flit/codeword 对齐和低时延 replay 做适配 | UALink flit、CRC、link-level replay、credit | UPLI/TL，load/store/atomic 与 memory semantics                    |
| CXL         | PCIe PHY                                 | PCIe PL 之上复用/扩展                          | CXL.io 与 CXL.cache/mem link              | CXL.io、CXL.cache、CXL.mem                                        |
| NVLink / UB | 厂商自定义                                    | 厂商自定义                                    | 厂商自定义                                    | 厂商自定义 memory fabric、collective/runtime 协同                       |

这张表解释了为什么“南向互联总线标准”会看起来都像四层栈，却不在同一层竞争。PHY 层竞争的是 lane rate、reach、功耗、BER、封装和光电转换；DL 层竞争的是 replay 粒度、credit、deadlock avoidance、lossless class；TL 层竞争的是 load/store/atomic、memory ordering、collective、地址空间和 endpoint/switch 协同。**同一代 AI rack 里，最底层可能都在用相似的 SerDes 和光模块，系统行为却会因为 DL/TL 语义完全不同。**

OCI MSA 把这个趋势推得更底层。它关注的是开放 optical scale-up interconnect 规格，目标是让 AI scale-up 从 copper 逐步走向 silicon-centric optics，并让 XPU 和 switch 可以匹配多供应商光技术[\[17\]](https://investors.broadcom.com/news-releases/news-release-details/broadcom-showcases-industry-leading-solutions-scaling-ai)。OCI 不替上层选择 UALink、SUE、NVLink-like 或私有协议，它试图把未来最贵、最难量产的 optical PHY 层先标准化。由此可见，后续标准竞争大概率是“双层战场”：底层光电接口尽量开放，上层 memory fabric 继续分化。

## 八、交换芯片路线：Broadcom 主押 Ethernet，UALink 是保留期权

> **本节论点**：截至 2026 年中，UALink 交换芯片仍处在早期出货/评估爬坡；Broadcom 产品公开路线集中在 Tomahawk Ultra / Tomahawk 6 / Thor Ultra / Jericho4 这一套 Ethernet AI networking portfolio。 **本节核心参考**：
> 
> - ★★★ **Broadcom Tomahawk 6 发布资料**[\[15\]](https://investors.broadcom.com/news-releases/news-release-details/broadcom-ships-tomahawk-6-worlds-first-1024-tbps-switch) — 102.4Tbps、scale-up/scale-out、SUE Framework 与 UEC compliance 的官方口径
> - ★★★ **Astera Scorpio X-Series 发布资料**[\[13\]](https://www.asteralabs.com/news/astera-labs-extends-leadership-in-open-ai-scale-up-networking-with-new-320-lane-scorpio-x-series-smart-fabric-switch/) — 320-lane memory-semantic fabric switch 与 2H 2026 ramp
> - ★★ Marvell 收购 XConn 公告[\[14\]](https://investor.marvell.com/news-events/press-releases/detail/1004/marvell-to-acquire-xconn-technologies-expanding-leadership-in-ai-data-center-connectivity) — Marvell 明确扩充 UALink scale-up switch team
> - ★★ Broadcom Thor Ultra 发布资料[\[16\]](https://investors.broadcom.com/news-releases/news-release-details/broadcom-introduces-industrys-first-800g-ai-ethernet-nic) — 800G UEC-compliant NIC 与 Broadcom Ethernet AI portfolio

**（v1.1 新增）** UALink 的标准进度很快，但交换芯片生态仍在从规范走向硬件。UALink 官网已经列出 200G 1.0、Common 2.0、128G DL/PL 1.0、200G DL/PL 2.0 和 Chiplet 1.01 等规格；其中 Common 2.0 加入 In-Network Compute，意图把 collective 这类计算/通信融合能力纳入 fabric[\[12\]](https://ualinkconsortium.org/specification/)。UALink 路线图说 member companies 正在开发 IP、accelerators、switches、test platforms 和 management solutions，目标是在 2026 和 2027 推动商业部署[\[18\]](https://ualinkconsortium.org/blog/ualink-roadmap-insights-accelerating-open-scalable-ai-networking-1296/)。

公开进展里，Astera 和 Marvell 比 Broadcom 更像 UALink switch 的直接进攻方。Astera 在 2026 年 5 月发布 Scorpio X-Series 320 Lane Smart Fabric Switch，称其是 open, memory-semantic fabric switch，已经 shipping to leading hyperscalers，生产爬坡目标在 2026 下半年；它还强调 Hypercast 和 In-Network Compute engines，用于降低 collective 和 token routing 的通信开销[\[13\]](https://www.asteralabs.com/news/astera-labs-extends-leadership-in-open-ai-scale-up-networking-with-new-320-lane-scorpio-x-series-smart-fabric-switch/)。Marvell 2026 年收购 XConn 时也明说，交易会扩充自己的 UALink scale-up switch team，并把 PCIe/CXL switching IP 与 UALink 机会连接起来[\[14\]](https://investor.marvell.com/news-events/press-releases/detail/1004/marvell-to-acquire-xconn-technologies-expanding-leadership-in-ai-data-center-connectivity)。

Broadcom 的公开产品叙事则明显压在 Ethernet 侧。Tomahawk 6 是 102.4Tbps Ethernet switch，官方同时写 scale-up 和 scale-out，给出 512 XPU scale-up cluster、100,000+ XPU two-tier scale-out、100G/200G SerDes、CPO option、Cognitive Routing 2.0，并强调 SUE Framework 会被共享给 OCP 等开放标准组织[\[15\]](https://investors.broadcom.com/news-releases/news-release-details/broadcom-ships-tomahawk-6-worlds-first-1024-tbps-switch)。Tomahawk Ultra 则把 51.2Tbps、250ns、64B line-rate、LLR/CBFC 和 in-network collectives 做成低时延 scale-up SKU[\[2\]](https://investors.broadcom.com/news-releases/news-release-details/broadcom-ships-tomahawk-ultra-reimagining-ethernet-switch-hpc)。Thor Ultra 又补上 UEC-compliant 800G NIC：packet-level multipathing、out-of-order delivery、selective retransmission、programmable congestion control、PCIe Gen6 x16 host interface，一起服务大规模 Ethernet AI fabric[\[16\]](https://investors.broadcom.com/news-releases/news-release-details/broadcom-introduces-industrys-first-800g-ai-ethernet-nic)。

这组产品线回答了“后续交换芯片是否会分成 scale-up 和 scale-out 两条线”的问题。Tomahawk 6 可以覆盖一部分 scale-up 和主流/高端 scale-out，因为它的优势是总带宽、radix、routing、CPO 和统一运营栈；Tomahawk Ultra 更像低延迟特化分支，优势是 250ns、小包线速、lossless、optimized header 和 collective offload。**Broadcom 统一的是 Ethernet 生态和工具链，分化的是 ASIC 微架构：高带宽 Tomahawk 主线服务 scale-out 并兼顾部分 scale-up，Ultra 分支把极致 scale-up 的延迟和小包路径单独拉出来。**

**（v1.2 新增）** 把 Broadcom 公开交换产品放进同一张表，分工会更清楚：

| 产品                        | 公开定位                                           | 关键公开指标                                                                                                                                                                                                                                                        | 在 AI fabric 里的角色                                                         | 主要边界                                                          |
| ------------------------- | ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------ | ------------------------------------------------------------- |
| Tomahawk Ultra / BCM78920 | 低时延 HPC / AI scale-up Ethernet switch          | 51.2Tbps、250ns full-throughput latency、64B line-rate、约 77B packets/s、LLR/CBFC、optimized header、in-network collectives[\[2\]](https://investors.broadcom.com/news-releases/news-release-details/broadcom-ships-tomahawk-ultra-reimagining-ethernet-switch-hpc) | 最贴近 SUE 的交换芯片路径：承接小包、lossless、VC/credit 和 collective 热路径                 | 公开资料仍缺端到端 MoE / TP / KV workload benchmark                    |
| Tomahawk 6 / BCM78910     | 高带宽统一 Ethernet switch，覆盖 scale-up 与 scale-out  | 102.4Tbps、100G/200G SerDes、CPO option、512 XPU scale-up cluster、100K+ XPU two-tier scale-out、Cognitive Routing 2.0[\[15\]](https://investors.broadcom.com/news-releases/news-release-details/broadcom-ships-tomahawk-6-worlds-first-1024-tbps-switch)          | 把同一套 Ethernet 运维和交换主线推到更大 radix / 更大 AI cluster                          | 延迟和小包路径不如 Ultra 特化，更多是通用高带宽平台                                 |
| Jericho4                  | 跨机房 / 区域级分布式 AI fabric router                  | 面向 1M+ XPU，多 DC interconnect；单系统可扩到 36,000 个 3.2Tbps HyperPort，deep buffer、MACsec、RoCE over 100km+、UEC compliant[\[19\]](https://investors.broadcom.com/news-releases/news-release-details/broadcom-ships-jericho4-enabling-distributed-ai-computing-across)  | 当 AI 集群越过单园区电力和机房边界后，用长距离 lossless Ethernet 把多个 data center 拼成一个训练/推理资源池 | 它解决的是 scale-out / DCI，不是 XPU pod 内的 sub-us memory transaction |
| Ramon3 / BCM88920         | StrataDNX fabric device，用于模块化 scheduled fabric | 51.2Tbps cell-based self-routing fabric element，可连接多个 Jericho 设备，产品页称平台可扩展到 50,000 个 400GbE 或 25,000 个 800GbE 端口[\[20\]](https://www.broadcom.com/products/ethernet-connectivity/switching/stratadnx/bcm88920)                                                | 给 Jericho/DDC 这类模块化 fabric 做内部交换背板，把端口规模继续往上堆                            | 更偏模块化 chassis / fabric building block，不直接定义 SUE 端点语义          |

这张表的工程含义是：**Broadcom 并没有把“AI networking”压成一颗万能芯片，而是把 AI fabric 切成四个物理问题。**Rack 内低延迟小包交给 Tomahawk Ultra，高 radix 和统一 Ethernet 主线交给 Tomahawk 6，跨机房长距离 lossless 交给 Jericho4，模块化 fabric 的内部扩展交给 Ramon3。SUE 在这里不是孤立协议，而是这套产品组合里最靠近 XPU pod 的一层系统契约。

商业上，Broadcom 没有理由完全放弃 UALink。它仍在 UALink 会员名单里，也具备 SerDes、retimer、switch ASIC、custom silicon 和 chiplet/IP 能力；如果 hyperscaler 或 AMD/Intel/custom XPU 阵营把 UALink 变成明确采购条件，Broadcom 可以用专用 ASIC、dual-protocol switch、retimer/PHY/IP 或定制封装切入。但当前公开信号指向另一个优先级：**Broadcom 把 UALink 当作期权，把 Ethernet scale-up 当作主线产品。**这也符合它的商业激励。SUE/ESUN/UEC 能最大化复用 Tomahawk、Jericho、Thor、optics、SONiC/OEM/ODM 生态；全力押 UALink 会削弱“Ethernet 已经足够进入 scale-up”的产品叙事。

这个判断有一个边界。若 UALink 在 2026-2027 年通过 AMD Helios、自研 XPU 或 hyperscaler rack-scale 平台形成真实出货，并且软件能稳定消费 memory-semantic fabric，Broadcom 的期权价值会上升。若系统厂商更关心统一运维、可采购性、交换芯片现货和 Ethernet 工具链，Tomahawk Ultra / Tomahawk 6 / Thor Ultra 这条线会先吃到规模化部署。**短期看，SUE 是 Broadcom 的进攻路线；UALink 是它不能忽视、但暂时不愿替主线让路的开放标准。**

## 九、结论：SUE 的验收指标应该怎么写

SUE 这条路线最有用的地方，是把“开放南向互联”从抽象价值观落到了可验收的工程指标。写项目材料或技术评审时，不应只写“支持 Scale-Up Ethernet”。更准确的指标至少要拆成五层。

第一层是端点能力：每个 XPU 支持多少个 SUE/SUE Lite instance，每个 instance 是 800G 还是 1.6T，XPU-to-SUE 接口采用 signal/FIFO 还是 AXI 4，是否支持 strict ordering / unordered mode，VC 数量如何映射到请求、响应和不同 traffic class。

第二层是封装效率：是否支持 AFH Gen 2 6B/12B，是否仍可回退标准 Ethernet/IP/UDP，64B 小包是否线速，packing 上限和 opportunistic packing 策略是否可配置。这里要测有效 payload bandwidth，SerDes 速率只回答了物理层问题。

第三层是可靠性与流控：FEC 模式、LLR 是否启用、PFC/CBFC class 数量、request/ack 是否分离、incast 下是否触发 head-of-line blocking，链路故障时 unordered 多端口是否能把 work 转移到其他 port/plane。**（v1.1 更新）** 如果项目声称同时支持 UALink 或其他专用 scale-up fabric，还要把 DL replay 粒度、credit model、flit size、transaction ordering 和 memory semantics 单独列出；只写“复用 200G Ethernet PHY”不能证明上层 fabric 语义一致。

第四层是交换芯片：switch latency、full-throughput latency、64B packet rate、in-network collectives 覆盖哪些 primitive，Dragonfly / Mesh / Torus 等 topology-aware routing 是否真正被系统软件使用。Tomahawk Ultra 给出了 51.2Tbps、250ns、77B packets/s、LLR/CBFC 和 in-network collectives 的产品基线[\[2\]](https://investors.broadcom.com/news-releases/news-release-details/broadcom-ships-tomahawk-ultra-reimagining-ethernet-switch-hpc)，Tomahawk 6 则给出了 102.4Tbps、512 XPU scale-up 与 100K+ XPU scale-out 的高带宽平台基线[\[15\]](https://investors.broadcom.com/news-releases/news-release-details/broadcom-ships-tomahawk-6-worlds-first-1024-tbps-switch)。后续竞品不该只报总 Tbps，也要同时报小包线速、延迟、lossless class、collective offload 和端点语义。

第五层是 workload：MoE dispatch/combine 的 p50/p99 latency，TP/EP collective 的有效带宽，KV cache Put/Get 和 P/D transfer 的尾延迟，故障注入下 job completion time 的变化，以及多租户 partition / QoS 是否能保护关键请求。**Scale-up fabric 的最后验收不在链路表格里，而在训练 step time、推理 TPOT 和故障恢复窗口里。**

SUE 对南向互联版图的意义，正是把以太网的成熟生态推到这组指标面前。它不会让 Ethernet 自动拥有 NVLink 的闭环软件优势，也不会替代 UALink 对 memory-semantic pod 的标准化野心；但它已经证明了一件具体的事：以太网交换芯片可以被重新设计成低时延、lossless、小头部、支持 collective offload 的 scale-up fabric。对 NVIDIA 之外的 XPU 生态来说，这是一条现实得多的路：先让开放交换芯片进入 rack 内同步域，再让上层软件一点点学会消费这个域。**（v1.1 更新）** 后续真正要追的不是哪一个标准名字赢，而是哪一层先形成稳定商品：PHY/optics 会最早趋同，switch ASIC 会在 Tomahawk Ultra/Tomahawk 6 与 Scorpio/Marvell 这类路线之间竞争，端点 TL 和软件 runtime 决定最终能否把 XPU pod 当成一台机器使用。

- - -

## 参考资料

\[1\] [OCP: Scale Up Ethernet (SUE) v1.0.0](https://www.opencompute.org/documents/ocp-sue-spec-final-pdf-1)

\[2\] [Broadcom Ships Tomahawk Ultra: Reimagining the Ethernet Switch for HPC and AI Scale-up](https://investors.broadcom.com/news-releases/news-release-details/broadcom-ships-tomahawk-ultra-reimagining-ethernet-switch-hpc)

\[3\] [OCP: Introducing ESUN, Advancing Ethernet for Scale-Up AI Infrastructure](https://www.opencompute.org/blog/introducing-esun-advancing-ethernet-for-scale-up-ai-infrastructure-at-ocp)

\[4\] [Arista: The Sun Rises on Scale-Up Ethernet](https://blogs.arista.com/blog/the-sun-rises-on-scale-up-ethernet)

\[5\] [Synopsys: Ethernet Standards for Scale-Up AI: ESUN, SUE, and UALink](https://www.synopsys.com/articles/ethernet-standards-scale-up-ai.html)

\[6\] [UALink: An Open, High-Efficiency Scale-Up Interconnect for AI](https://ualinkconsortium.org/wp-content/uploads/2026/01/UALink_White_Paper_Publication_FINAL_UPDATED.pdf)

\[7\] [Ultra Ethernet Consortium 1.0 White Paper](https://ultraethernet.org/wp-content/uploads/sites/20/2025/06/UEC1.0Whitepaper.pdf)

\[8\] [Serving Large Language Models on Huawei CloudMatrix384](https://arxiv.org/html/2506.12708v1)

\[9\] [前期调研：NVLink 演进史：从 P100 四条链路到 Rubin/Feynman 的机架级 fabric](https://miraclefarms.github.io/notes/2026/06/21/nvlink-evolution-from-p100-to-rubin-feynman/)

\[10\] [前期调研：超节点南向互联：从单卡算力到单一逻辑计算机](https://miraclefarms.github.io/notes/2026/06/21/supernode-southbound-interconnect-survey/)

\[11\] [UALink 200G 1.0 Specification Overview](https://ualinkconsortium.org/blog/ualink-200g-1-0-specification-overview-802/)

\[12\] [UALink Specifications](https://ualinkconsortium.org/specification/)

\[13\] [Astera Labs Extends Leadership in Open, AI Scale-Up Networking with New 320 Lane Scorpio X-Series Smart Fabric Switch](https://www.asteralabs.com/news/astera-labs-extends-leadership-in-open-ai-scale-up-networking-with-new-320-lane-scorpio-x-series-smart-fabric-switch/)

\[14\] [Marvell to Acquire XConn Technologies, Expanding Leadership in AI Data Center Connectivity](https://investor.marvell.com/news-events/press-releases/detail/1004/marvell-to-acquire-xconn-technologies-expanding-leadership-in-ai-data-center-connectivity)

\[15\] [Broadcom Ships Tomahawk 6: World’s First 102.4 Tbps Switch](https://investors.broadcom.com/news-releases/news-release-details/broadcom-ships-tomahawk-6-worlds-first-1024-tbps-switch)

\[16\] [Broadcom Introduces Industry’s First 800G AI Ethernet NIC](https://investors.broadcom.com/news-releases/news-release-details/broadcom-introduces-industrys-first-800g-ai-ethernet-nic)

\[17\] [Broadcom Showcases Industry-Leading Solutions for Scaling AI Infrastructure at OFC 2026](https://investors.broadcom.com/news-releases/news-release-details/broadcom-showcases-industry-leading-solutions-scaling-ai)

\[18\] [UALink Roadmap Insights: Accelerating Open, Scalable AI Networking](https://ualinkconsortium.org/blog/ualink-roadmap-insights-accelerating-open-scalable-ai-networking-1296/)

\[19\] [Broadcom Ships Jericho4, Enabling Distributed AI Computing Across Data Centers](https://investors.broadcom.com/news-releases/news-release-details/broadcom-ships-jericho4-enabling-distributed-ai-computing-across)

\[20\] [Broadcom BCM88920: 51.2 Tb/s StrataDNX Ramon3 Fabric Device](https://www.broadcom.com/products/ethernet-connectivity/switching/stratadnx/bcm88920)

### 版本对齐信息

- SUE：基于 OCP《Scale Up Ethernet (SUE)》Version 1.0.0，Date Approved 2025-09-05；文中需求表、SUE/SUE Lite 差异、AFH、LLR/PFC/CBFC、processing flow 与 latency budget 均取自该版本。
- Tomahawk Ultra：基于 Broadcom 2025-07-15 发布稿与 BCM78920 产品页元信息；公开资料确认 51.2Tbps、250ns、64B line-rate、77B packets/s、LLR/CBFC、in-network collectives 与 shipping 状态。
- ESUN / SUE-T：基于 OCP 2025-10-13 ESUN workstream 公告与 Arista 生态说明；本文按 OCP 口径区分 ESUN 的 switch/framing 范围与 SUE-T 的 endpoint transport 范围。
- UALink：基于 2026-01 UALink public white paper、2025-04 UALink 200G 1.0 specification overview、UALink 2026 specification/roadmap 页面；本文按公开页面确认 200G 1.0、Common 2.0、DL/PL 2.0、Chiplet 1.01 等规格状态，硬件出货按 Astera/Marvell 等公司公告分别标注。
- Broadcom Ethernet AI portfolio：基于 Broadcom Tomahawk Ultra（2025-07-15）、Tomahawk 6（2025-06-03）、Thor Ultra（2025-10-14）和 OFC 2026 portfolio update（2026-03-12）公开资料；本文将 Tomahawk Ultra 视为低时延 scale-up 特化分支，将 Tomahawk 6 视为高带宽 scale-up/scale-out 通用主线。
- Broadcom 交换产品对比：v1.2 新增表格基于 Tomahawk Ultra、Tomahawk 6、Jericho4（2025-08-04）和 Ramon3 / BCM88920 产品页公开资料；其中 Jericho4/Ramon3 归入 scale-out / DCI / modular fabric 侧，不把它们误写成 SUE endpoint transport。
