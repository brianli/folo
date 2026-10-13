---
title: "KV Cache 存储的四种理解：重读 FMS 2026 的 CXL Panel"
source: "Readings"
category: "AI Infra"
group: "MiracleFarms"
url: "https://miraclefarms.github.io/notes/2026/10/12/fms2026-kv-cache-storage-four-vendors/"
published: 2026-10-12T15:10:00+08:00
saved: 2026-10-13T10:53:31+08:00
folo_key: "mf-readings::https://miraclefarms.github.io/notes/2026/10/12/fms2026-kv-cache-storage-four-vendors/"
tags:
  - "folo"
  - "AI_Infra"
  - "Readings"
---

# KV Cache 存储的四种理解：重读 FMS 2026 的 CXL Panel

> [!info] Readings · AI Infra · 2026-10-12 15:10 · [原文](https://miraclefarms.github.io/notes/2026/10/12/fms2026-kv-cache-storage-four-vendors/)

Liqid 看复用率，UnifabriX 看访问层级，Astera Labs 看控制器能力，Micron 看系统落地。沿着完整幻灯片，拆开容量、搬运与重算三类收益。

![[KV Cache 存储的四种理解：重读 FMS 2026 的 CXL Panel-01.png]]

*题图：四家厂商从不同位置理解 KV 存储；容量、访问路径与复用率需要分别比较。题图为本文解读示意，正文幻灯片保留原报告。*

FMS 是 Future of Memory and Storage，面向内存与存储产业的技术会议，前身为 Flash Memory Summit。会议的关注范围已从闪存扩展到内存、互连和数据中心基础设施。2026 年的 FMS 于 8 月 4–6 日在美国圣克拉拉举行，AI 推理所需的内存容量和数据搬运是其中一条重要议题。[\[4\]](https://terrapinn.com/conference/future-memory-storage/FMS-Timeline.stm)[\[2\]](https://computeexpresslink.org/blog/increasing-tokens-per-dollar-with-cxl-fms-2026-recap-4788/)

CXL 是 Compute Express Link，一种面向处理器、加速器和内存扩展的互连标准。它让 CPU 与附接设备的内存保持一致，并为扩容和资源共享提供连接基础。放到 KV Cache 场景里，问题变得具体：GPU 显存装不下的状态可以留在哪里？多台服务器能否使用同一份 KV？取回旧状态是否比重新计算更划算？[\[3\]](https://computeexpresslink.org/about-cxl/)

本文阅读的是 CXL 联盟在 FMS 2026 组织的一场 panel，也就是围绕同一主题、由多位产业讲者共同参与的专题讨论。主题是“突破内存墙：CXL 内存池化与共享如何改变 AI 推理”，由 Intel 的 Anil Godbole 主持，Liqid、UnifabriX、Astera Labs 和 Micron 分别带来自己的架构、产品与实验。这份 53 页 PDF 收录了他们的报告，适合用来比较不同厂商的设计思路。[\[2\]](https://computeexpresslink.org/blog/increasing-tokens-per-dollar-with-cxl-fms-2026-recap-4788/)

把整份材料放在一起，主线可以归纳为：**先扩大可保留的 KV 工作集，再改善共享与取回路径，最终减少重复 prefill。** 四家厂商各自回答其中不同的问题：Liqid 看资源池和复用率，UnifabriX 看 CPU/GPU 访问域，Astera Labs 看控制与管理能力，Micron 看系统组合和应用收益。

同一份报告里，CXL 与本地 DRAM 的推理吞吐几乎持平，KV 复用相对重算却带来约 6.7 倍提升。两个结果对应不同的比较：前者衡量存储位置变化的代价，后者衡量保留已有计算结果带来的收益。下面先建立共同的比较框架，再按厂商展开，最后回到这些方案能支撑哪些工程判断。[\[1\]](https://computeexpresslink.org/wp-content/uploads/2026/09/CXL_FMS-2026-Panel-Presentation_FINAL.pdf#page=49)

除题图外，正文配图采用原 PDF 的整页导出，图注页码从 PDF 第一页开始计数。测试数字保留厂商原口径，估算、已测试配置与后续路线图分别注明。

## 一、先分清四家厂商在回答什么

[

![[KV Cache 存储的四种理解：重读 FMS 2026 的 CXL Panel-02.png]]

](https://miraclefarms.github.io/assets/fms2026-kv-cache-storage-four-vendors/fig-01-slide-03-shared-memory-overview.png) 

*图 1：原 PDF 第 3 页。多个主机经 CXL switch 访问共享设备，fabric manager 管理资源。这是整场 panel 的共同起点：让已有数据可以被更多计算节点使用。正文幻灯片均保留完整页面，点击可查看原始尺寸。*

如果一个 Agent 连续围绕同一份代码或文档工作，前一轮算出的 KV 在下一轮仍可能有用。存储系统要做的事情，便是保留这份状态，让下一次调用找到它，并在取回成本足够低时省去重复 prefill。内存池扩大了可以保留的工作集，共享机制扩大了可以使用这些状态的计算节点范围。

与此同时，当前 decode 所需的活跃 KV 有更紧迫的访问要求。保留数小时后再恢复一个会话，与每生成一个 token 就访问外部内存，面对的延迟预算和访问频率差别很大。PPT 对前缀复用展示了明确结果，对 GPU 如何消费外部 KV 则只有部分架构说明。阅读时需要保留这条边界。

| 厂商 / 讲者                           | PDF 范围  | 对 KV 存储的主要理解                    | 想改变的约束                  |
| --------------------------------- | ------- | ------------------------------- | ----------------------- |
| Liqid / Sumit Puri                | 6–17 页  | 可动态组织的资源池，更大的内存让更多 KV 留存并复用     | 单机容量与资源闲置，取回时间和命中率      |
| UnifabriX / Ronen Hyatt           | 18–33 页 | 位于不同访问域的共享内存层                   | CPU 侧扩容与 GPU 侧供数之间的路径差别 |
| Astera Labs / Sandeep Dattaprasad | 34–38 页 | 需要控制器、fabric 和管理软件共同支撑的多主机资源    | 分配、隔离、复位、遥测与故障处理        |
| Micron / Luis Ancajas             | 39–51 页 | 可以独立扩容的 JBOM，结合 famfs 与推理软件形成系统 | 大工作集驻留、跨主机复用与整机经济性      |

这张分类表是我的阅读归纳。报告没有给出一套四家厂商共同运行的 benchmark，也没有统一价格、功耗或请求轨迹。因此，这四条路线适合按机制比较，性能倍率需要留在各自实验内。

**池化、共享和分层是三项不同能力。** 池化让容量能够按需分配，共享让多个使用者访问同一份数据，分层则决定哪些 KV 留在哪个位置。硬件支持其中一项，并不自动补齐另外两项的软件语义。

## 二、Liqid：内存池首先是一项复用率工程

### 2.1 从单机内存转向可分配资源

Liqid 的前半段用推理经济性解释池化。训练负载以阶段性任务为主，推理更接近持续服务；当一台服务器内存不足、另一台服务器又有闲置容量时，固定配置会让资源利用率与服务能力同时受损。第 7–12 页围绕这件事组织了推理占比、GPU 空闲、内存压力和上下文增长的背景材料。

落到产品组织方式，第 10 页展示 Liqid Matrix 管理计算服务器、GPU 池和内存池。图中内存 chassis 为 40TB，池容量最多 160TB；GPU 池展示最多 30 个 GPU、每个 chassis 十个。容量和计算资源可以沿各自的路径扩展，再由管理层组织给服务器使用。[\[1\]](https://computeexpresslink.org/wp-content/uploads/2026/09/CXL_FMS-2026-Panel-Presentation_FINAL.pdf#page=10)

[

![[KV Cache 存储的四种理解：重读 FMS 2026 的 CXL Panel-03.png]]

](https://miraclefarms.github.io/assets/fms2026-kv-cache-storage-four-vendors/fig-02-slide-10-liqid-resource-pool.png) 

*图 2：原 PDF 第 10 页。Liqid 把内存和 GPU 都组织为可管理的资源池；图中容量是产品组织示例，不是后面每个 KV 实验的统一配置。*

这套叙事也覆盖内存数据库与虚拟化。第 11 页用数据库处理时间、虚拟机数量与每虚拟机成本说明容量扩展的作用，AI 示例则给出独立的性能倍率。这些案例共享“让工作集装得下”的动机，但访问模式各异，数据库和虚拟机结果不能直接成为 KV serving 的性能保证。

### 2.2 CXL 承接主存与存储之间的 KV

第 13 页把存储层级和 KV 用途放在一起：HBM 放活跃 KV，系统 DRAM 承接 staging 与溢出，本地 SSD 承接较温的复用数据，共享文件或对象存储承接更冷的上下文。内存池扩大的是 DRAM 这一侧的容量，让更多状态在落入存储之前仍可被较快地访问。

[

![[KV Cache 存储的四种理解：重读 FMS 2026 的 CXL Panel-04.png]]

](https://miraclefarms.github.io/assets/fms2026-kv-cache-storage-four-vendors/fig-03-slide-13-liqid-storage-hierarchy.png) 

*图 3：原 PDF 第 13 页。Liqid 用成本、延迟和容量说明各层的角色。左侧 ns 数字是分层示意，不能直接当作 GPU 恢复一个 KV block 的端到端时间。*

这里的判断很明确：DRAM 的容量和成本介于 HBM 与大容量存储之间，池化可以继续扩大它可承载的工作集。但一个内存读操作的延迟，与恢复一整段 KV 的时间之间，还隔着数据量、软件路径、并行度和 GPU 接入通道。Liqid 随后的实验恰好把这些问题分成了三个比较。

### 2.3 加载时间、位置对照、命中率，分别测量

第 14 页比较加载同样标注为 4GB、128kB 粒度的 KV：NVMe 为 3,188ms，CXL 为 78ms，图中标注约 40 倍差距。这个结果支持“该路径下 CXL 取回更快”，但页面未交代 NVMe 盘数、队列深度、驱动和数据搬运方式，不能直接推导所有 SSD 后端都有相同差距。[\[1\]](https://computeexpresslink.org/wp-content/uploads/2026/09/CXL_FMS-2026-Panel-Presentation_FINAL.pdf#page=14)

[

![[KV Cache 存储的四种理解：重读 FMS 2026 的 CXL Panel-05.png]]

](https://miraclefarms.github.io/assets/fms2026-kv-cache-storage-four-vendors/fig-04-slide-14-liqid-kv-load-time.png) 

*图 4：原 PDF 第 14 页。3,188ms 与 78ms 衡量的是 KV 加载过程；原页没有给出足以解释 SSD 瓶颈来源的完整配置。*

第 15 页换成 AIPerf 的应用吞吐对照：DRAM 为 2,460 tokens/s，CXL 为 2,433 tokens/s，约保留 98.9%。图中将其概括为约 99% 持平。这回答了内存放到 CXL 后，应用性能是否明显受损；它与上一页的 CXL/NVMe 加载比较采用了不同基线。

[

![[KV Cache 存储的四种理解：重读 FMS 2026 的 CXL Panel-06.png]]

](https://miraclefarms.github.io/assets/fms2026-kv-cache-storage-four-vendors/fig-05-slide-15-liqid-dram-cxl-parity.png) 

*图 5：原 PDF 第 15 页。DRAM 与 CXL 的 token 吞吐接近，支持在该负载下以较小性能代价扩展可访问容量；“100 倍更大规模”是页面宣称，缺少对应容量扫描曲线。*

第 16 页再换一个问题：缓存保留下来后，能省多少重算？柱状图给出的值是全重算 355、70% 命中 865、90% 命中 2,018 tokens/s。命中率提高确实伴随吞吐上升，不过 2,018/355 约为 5.68，页面写的约 7 倍不能从这组三根柱子直接复算。

[

![[KV Cache 存储的四种理解：重读 FMS 2026 的 CXL Panel-07.png]]

](https://miraclefarms.github.io/assets/fms2026-kv-cache-storage-four-vendors/fig-06-slide-16-liqid-cache-reuse.png) 

*图 6：原 PDF 第 16 页。命中率改变的是重算量。图值与约 7 倍标记有出入，本文保留原值并按图值解释趋势。*

最后，第 17 页展示多个主机同时访问共享内存，以及 DAX 枚举和 famfs 软件路径。到这里，Liqid 的 KV 存储理解就连起来了：先把容量从服务器内解耦，再保留更多 KV、让更多使用者访问它，最后通过更高复用率减少 prefill。

**Liqid 的三组实验分别对应取回速度、位置替换代价和避免重算的收益。** 它们能支撑同一套池化故事，却不能合并成一个“CXL 加速倍数”。

## 三、UnifabriX：共享内存的价值取决于它靠近哪条访问路径

### 3.1 G2.5：放在系统 DRAM 与 SSD 之间

UnifabriX 从内存墙、Agent 状态累积与内存供给切入。第 21 页用 Llama 3 70B、128K 上下文、batch=1 的约 40GB KV 提醒读者，长上下文可能迫使系统为了容量增加 GPU 分片，进而增加通信和同步。第 22 页的价格与供给材料则为“提高已有内存利用率”提供产业背景。

它随后给层级编号：G1 是 GPU HBM，G2 是系统 DRAM，G3 是本地或 rack-local SSD，G4 是共享文件/对象存储。第 25 页把 NVIDIA CMX 放在 G3.5，强调其位于 Ethernet 附接 flash 这一侧。第 26 页再加入自己的 G2.5：CPU scale-up 域里的 CXL 共享内存池，位于主存和存储之间。[\[1\]](https://computeexpresslink.org/wp-content/uploads/2026/09/CXL_FMS-2026-Panel-Presentation_FINAL.pdf#page=26)

[

![[KV Cache 存储的四种理解：重读 FMS 2026 的 CXL Panel-08.png]]

](https://miraclefarms.github.io/assets/fms2026-kv-cache-storage-four-vendors/fig-07-slide-26-unifabrix-g25-memory.png) 

*图 7：原 PDF 第 26 页。G2.5 扩展 CPU 侧可访问的共享内存。G1、G2.5、G3.5 等编号是讲者的架构表达，阅读时应按访问域与媒介理解。*

G2.5 与 flash 层可以同时存在。前者把更多 KV 留在内存访问域，后者承接更大的历史状态。媒介不同，恢复时付出的成本也不同；这条分界比“都能缓存 KV”更能解释两者为何会同时出现。

第 27 页的 MPU 平台把这层内存产品化，标注 4–32TB、标准 2U 形态，可使用 DDR5、DDR4、MRDIMM，连接涉及 CXL、UALink 和 ESUN。页面还列出池化、适应性共享、RAS、链路冗余、镜像与条带化、交织、近内存处理和加密等能力。

[

![[KV Cache 存储的四种理解：重读 FMS 2026 的 CXL Panel-09.png]]

](https://miraclefarms.github.io/assets/fms2026-kv-cache-storage-four-vendors/fig-08-slide-27-unifabrix-mpu-platform.png) 

*图 8：原 PDF 第 27 页。MPU 被呈现为带交换、管理和可靠性能力的内存平台。能力清单是产品描述，该页没有逐项给出验证结果。*

### 3.2 重复运行曲线展示了缓存变热的过程

第 28 页用 Llama 3.1 70B 对照三个配置：150GB DRAM；5GB DRAM 加 150GB CXL；5GB DRAM 加 150GB Disk。横轴是重复运行次数，纵轴是平均 TTFT。三个配置第一次都处于冷启动高位，DRAM 第二次降到低位，CXL 第三次接近 DRAM，Disk 更晚下降且稳定后的延迟更高。

[

![[KV Cache 存储的四种理解：重读 FMS 2026 的 CXL Panel-10.png]]

](https://miraclefarms.github.io/assets/fms2026-kv-cache-storage-four-vendors/fig-09-slide-28-unifabrix-ttft-warmup.png) 

*图 9：原 PDF 第 28 页。曲线把冷启动与后续复用分开；第 29 页只是前三次运行的放大图。原件未提供精确数据表，本文不把目测曲线读数当成可复算实测。*

这幅图更适合解释缓存状态如何改变请求体验。它没有直接测量一次随机 KV 读取的延迟，也没有证明第一次请求可以获得与热缓存相同的表现。若服务经常更换模型、输入前缀频繁变化，或者工作集不断超出池容量，曲线里逐渐进入低位的过程就需要重新评估。

### 3.3 G1.5：继续把共享内存推向 GPU 后端

扩展 CPU 侧内存之后，GPU–CPU 之间仍要经过接入通道。第 30 页把这一段单独标出：GPU HBM 提供多 TB/s 量级带宽，G1 到 G2 的 PCIe 路径却可能限制数据进入 GPU 的速度。图中 PCIe Gen5 x16 的 128GB/s 标注明确写了双向口径，不能直接当作单向 GPU 取数带宽。

[

![[KV Cache 存储的四种理解：重读 FMS 2026 的 CXL Panel-11.png]]

](https://miraclefarms.github.io/assets/fms2026-kv-cache-storage-four-vendors/fig-10-slide-30-unifabrix-pcie-bottleneck.png) 

*图 10：原 PDF 第 30 页。G2.5 改变容量与共享范围，GPU 到 CPU 的通道约束仍然存在；图中的接口数字还需要按方向统一口径。*

第 31 页于是进一步加入 G1.5：GPU scale-up 域里的共享内存，支持 CXL 与 native 路径，通过 GPU 后端 fabric 接入。G2.5 继续承担 CPU scale-up 侧的容量扩展，两者面向不同的访问域。

[

![[KV Cache 存储的四种理解：重读 FMS 2026 的 CXL Panel-12.png]]

](https://miraclefarms.github.io/assets/fms2026-kv-cache-storage-four-vendors/fig-11-slide-31-unifabrix-g15-g25.png) 

*图 11：原 PDF 第 31 页。同一种“共享内存”被拆成 GPU 侧 G1.5 与 CPU 侧 G2.5；区分它们的关键是消费者和接入路径。*

第 32 页把 G1.5 放进机架蓝图：compute tray、memory tray、scale-up switch 与外部 MPU 平台相连，GPU 可以在机架内外访问共享内存。这张图表达了希望达到的组织方式，没有对应的 G1.5 KV serving 性能结果。前面的 TTFT 实验属于 G2.5，不能自动转给新架构。

[

![[KV Cache 存储的四种理解：重读 FMS 2026 的 CXL Panel-13.png]]

](https://miraclefarms.github.io/assets/fms2026-kv-cache-storage-four-vendors/fig-12-slide-32-unifabrix-rack-blueprint.png) 

*图 12：原 PDF 第 32 页。GPU 侧内存作为基础设施资源的概念蓝图；图示可达性与已测得的请求性能需要分别验证。*

**UnifabriX 把“KV 应该存在哪里”推进成了“哪些处理器通过哪条路径消费它”。** CPU 侧池化、GPU 侧共享和网络存储可以同时扩大状态空间，各自改变的是不同的瓶颈。

## 四、Astera Labs：KV 共享池需要一整套控制能力

### 4.1 Agent 把 CPU、GPU 与内存管理绑进同一个循环

Astera Labs 的报告很短，但补上了前两家较少展开的控制器视角。第 35 页对比普通 GenAI 与 Agentic AI：CPU 从调度和 I/O 扩展到线程协调、工具执行和多步骤规划，GPU 继续执行模型生成，内存管理则贯穿 KV 留存和取回。

[

![[KV Cache 存储的四种理解：重读 FMS 2026 的 CXL Panel-14.png]]

](https://miraclefarms.github.io/assets/fms2026-kv-cache-storage-four-vendors/fig-13-slide-35-astera-agentic-workload.png) 

*图 13：原 PDF 第 35 页。Agent 的工具和状态管理增加了 CPU 与内存压力；图中的 CPU/GPU 配比属于讲者对系统形态的描述，不是所有部署的固定比例。*

当多个主机共享内存，问题会继续延伸：谁可以使用哪一段容量？某个主机复位会不会影响别人？链路上下线如何处理？缓存数据是否需要隔离？这些行为共同决定共享池能否承载长期运行的 KV 服务。

第 36 页展示 Leo 和 Scorpio 组成的多层拓扑。Leo 位于内存控制与连接侧，Scorpio 支撑 fabric 组织；图中既有单服务器扩容，也有多个 CPU 连接多个内存资源的池化/共享方式。容量扩展、更多层级与更高利用率被放在同一张方案图内。[\[1\]](https://computeexpresslink.org/wp-content/uploads/2026/09/CXL_FMS-2026-Panel-Presentation_FINAL.pdf#page=36)

[

![[KV Cache 存储的四种理解：重读 FMS 2026 的 CXL Panel-15.png]]

](https://miraclefarms.github.io/assets/fms2026-kv-cache-storage-four-vendors/fig-14-slide-36-astera-leo-topologies.png) 

*图 14：原 PDF 第 36 页。单机扩容与多主机池化使用不同的连接组织。框架能否复用 KV，还需要在这些硬件连接之上建立数据管理。*

### 4.2 功能状态表限定了架构图能承诺什么

第 37 页给出 Leo 1 的验证进度。多主机链路稳定、主机冷/热复位、遥测、带内和带外管理标为已验证；静态内存分配/回收已验证，动态分配/回收仍在推进；多主机内存隔离仍在推进；fabric 管理的固件/COSMOS 框架已经验证，但主机软件/BMC 支持仍待到位。[\[1\]](https://computeexpresslink.org/wp-content/uploads/2026/09/CXL_FMS-2026-Panel-Presentation_FINAL.pdf#page=37)

[

![[KV Cache 存储的四种理解：重读 FMS 2026 的 CXL Panel-16.png]]

](https://miraclefarms.github.io/assets/fms2026-kv-cache-storage-four-vendors/fig-15-slide-37-astera-feature-status.png) 

*图 15：原 PDF 第 37 页。静态分配、动态分配和多主机隔离的状态不同。应按会场时的状态表评估能力，而不能把架构示意中的全部操作都视为已交付。*

RAS 与性能也有分阶段标记：CXL 2.0 错误处理已验证，额外测试还在进行；基础性能测试完成，更多应用 benchmark 还在推进。一个控制器“可以工作”，与一个多租户共享池“具备完整生产管理能力”，之间需要逐项确认这些接口和故障行为。

第 38 页的 COSMOS 则把管理工具具体化：C SDK 和 CLI 提供链路诊断、RAS、固件升级、温度与性能监控、安全管理、内存分配、热页追踪和遥测等能力。功能展示描述管理面的覆盖范围，成熟度仍应回到上一页的状态表。

[

![[KV Cache 存储的四种理解：重读 FMS 2026 的 CXL Panel-17.png]]

](https://miraclefarms.github.io/assets/fms2026-kv-cache-storage-four-vendors/fig-16-slide-38-astera-cosmos-management.png) 

*图 16：原 PDF 第 38 页。COSMOS 将链路、设备、内存与安全管理组织成工具接口，补齐容量规格之外的运维视角。*

**Astera Labs 让共享 KV 池的讨论落到了分配、隔离、恢复和可观测性。** 本节没有给出独立的 KV 推理吞吐倍率，其作用是说明可用的存储资源需要哪些底层能力支撑。

## 五、Micron：把分离式内存、文件抽象与应用收益组合成系统

### 5.1 Abaco 让内存容量独立扩展

Micron 的开场将分离式内存定义为独立 JBOM chassis 内的内存，经 CXL switching 与计算服务器解耦。它希望像扩展计算、存储和网络一样扩展内存，把目标同时放在 AI/KV、数据库、图分析和大数据索引上。第 40 页的总体性能宣称随后由参考架构和两个案例展开。

第 41 页给出 DOE/PNNL Abaco 3.0 参考架构。一个 rack 包括 fabric manager、CXL switch、五台 GPU compute server 和四个 JBOM。计算服务器标注每台四个 GPU、双路 CPU 和 CXL host adapter；JBOM 组合了 Micron RDIMM、Primemas AIC 和 Liqid chassis。[\[1\]](https://computeexpresslink.org/wp-content/uploads/2026/09/CXL_FMS-2026-Panel-Presentation_FINAL.pdf#page=41)

[

![[KV Cache 存储的四种理解：重读 FMS 2026 的 CXL Panel-18.png]]

](https://miraclefarms.github.io/assets/fms2026-kv-cache-storage-four-vendors/fig-17-slide-41-micron-abaco-architecture.png) 

*图 17：原 PDF 第 41 页。Abaco 将 JBOM 作为独立扩展单元；容量表中的可用时间是报告时的路线图，不能用来确认今天每种配置的交付状态。*

容量从单个 4U JBOM 的 10TB、20TB、40TB 到 80TB，对应四个 JBOM 的 40TB、80TB、160TB、320TB。前两档列 2026 Q3，40TB/JBOM 列 2026 Q4，80TB/JBOM 列 2027 Q4。这个跨度说明其目标是大工作集驻留与机架内存扩展，但容量表本身没有回答所有主机同时访问时能得到多少带宽。

原页还列 chassis 6,000W 功率上限和参考配置约 3,000W 的需要量。两项都是 chassis 口径，材料没有给出 KV 测试时的整机功耗、每 token 能耗或能效曲线。不能仅凭这两项数字复算性能/瓦。

### 5.2 平台级带宽与请求级结果要按配置分开

第 42 页的实验平台由 Xeon 6、每服务器两个 A100、2TB 本地 DDR5、约 30TB SSD 和 10TB CXL chassis 组成。CXL chassis 标注十个 AIC、每卡八个 128GB RDIMM，聚合读带宽 250GB/s、内存延迟 434ns。软件包括 RHEL、kernel 6.14.11、weighted software interleaving 和 famfs 更新。

[

![[KV Cache 存储的四种理解：重读 FMS 2026 的 CXL Panel-19.png]]

](https://miraclefarms.github.io/assets/fms2026-kv-cache-storage-four-vendors/fig-18-slide-42-micron-benchmark-platform.png) 

*图 18：原 PDF 第 42 页。平台表包含 A100、LLaMA3-8B 和 GPT-OSS-20B；后面 Llama 70B 的结果脚注改为 H200 平台，不能把前后配置拼成一个实验。*

这一页列 LLaMA3-8B 和 GPT-OSS-20B，还给出 AIPerf、Dynamo、10,000 请求、并发 64、输出 128 tokens 等负载信息。到第 49–50 页，具体 KV 结果的脚注却使用两台 Granite Rapids、四个 H200 和 Llama 3.1 70B。**250GB/s 和 434ns 是第 42 页的平台参数，后面的应用结果需要按其自己的脚注解读。** 原件没有建立这两组配置之间完整的映射。

### 5.3 famfs 把共享内存变成主机可使用的文件资源

第 43 页解释软硬件之间的抽象：CXL module 经 chassis 汇聚，主机 OS 枚举为 DAX device，再由 famfs 暴露为文件系统；master 创建和挂载，client 挂载共享资源。应用通过文件抽象使用这块内存，多个主机可以访问同一份资源。

[

![[KV Cache 存储的四种理解：重读 FMS 2026 的 CXL Panel-20.png]]

](https://miraclefarms.github.io/assets/fms2026-kv-cache-storage-four-vendors/fig-19-slide-43-micron-famfs-layers.png) 

*图 19：原 PDF 第 43 页。DAX 与 famfs 连接主机应用和 CXL 内存；这幅图没有展开 GPU KV 的搬运、预取、索引与一致性实现。*

文件抽象降低了主机应用接入共享内存的门槛。不过，要让两个推理实例复用同一段 KV，仍需知道哪些 block 属于哪个模型和前缀、是否完整、何时可回收，以及怎样送入 GPU。报告提到 Dynamo、KV Block Manager 和 Rapid sharing，但没有完整展开这些策略与协议。famfs 解释了“主机怎样看见共享内存”，框架层还要解释“怎样正确复用 KV”。

### 5.4 数据库案例用于解释驻留容量与共享副本

第 44 页先把三类收益放在一起：图分析与神经网络标为已验证 32 倍性能/美元，内存分析缓存标为已验证 8 倍，KV 推理标为估算 5.5 倍。这一页已经给出了证据状态的差别。

[

![[KV Cache 存储的四种理解：重读 FMS 2026 的 CXL Panel-21.png]]

](https://miraclefarms.github.io/assets/fms2026-kv-cache-storage-four-vendors/fig-20-slide-44-micron-evidence-levels.png) 

*图 20：原 PDF 第 44 页。KV 的 5.5 倍性能/美元明确标为 Estimated；其他应用的 Proven 标记也只是厂商在原件中的表述。*

随后案例一用约 3TB 的合成股票 tick 数据，面对服务器 2TB 本地 DRAM 的容量限制。负载包含极值、加权价格、快照和统计聚合。将数据放到可以容纳它的 famfs/CXL 内存后，代表查询从 46ms 降为 4ms，原页标注 11.5 倍性能、4.2 倍性能/美元和 77% TCO 降低。读/计算重负载的时间取 1,000 次运行平均值。[\[1\]](https://computeexpresslink.org/wp-content/uploads/2026/09/CXL_FMS-2026-Panel-Presentation_FINAL.pdf#page=46)

[

![[KV Cache 存储的四种理解：重读 FMS 2026 的 CXL Panel-22.png]]

](https://miraclefarms.github.io/assets/fms2026-kv-cache-storage-four-vendors/fig-21-slide-46-micron-database-single-host.png) 

*图 21：原 PDF 第 46 页。工作集超过本地 DRAM，额外内存改变了数据驻留方式；这是时序数据库结果。脚注“10×1GB 合计 10TB”存在疑似单位笔误。*

第 47 页继续让多台服务器共享一份 3TB 数据。页面标注 8 倍性能/美元、同等性能下 TCO 降低 88%，但脚注明确两台服务器并行已经测试，四台配置仍待测试。图里的四台线性扩展包含预期，不能整张图都按实测理解。

[

![[KV Cache 存储的四种理解：重读 FMS 2026 的 CXL Panel-23.png]]

](https://miraclefarms.github.io/assets/fms2026-kv-cache-storage-four-vendors/fig-22-slide-47-micron-database-shared.png) 

*图 22：原 PDF 第 47 页。一份数据供多服务器使用，展示容量共享的经济逻辑；两台已测试、四台仍待测试的限制写在原页脚注。*

这两页解释了 Micron 为何强调独立扩容和共享副本：容量充足可以减少存储访问，多使用者共享又可以减少重复保留的状态。KV pool 有相近的动机，但访问频率、重算价值和数据有效性不同，数据库的成本倍率应留在数据库案例内。

### 5.5 KV 案例有两组不同的对照

第 48 页列出 Llama 3.1 70B BF16、10K 请求、并发 64、10.2 分钟、输出 128 tokens，软件为 AIPerf、Dynamo/vLLM、KV Block Manager 和 famfs/Rapid sharing。上下文在原表写作 4KB–16KB，后页又写 4K–16K；材料没有说明这里 K 与 KB 如何对应，因此本文保留其原口径。

[

![[KV Cache 存储的四种理解：重读 FMS 2026 的 CXL Panel-24.png]]

](https://miraclefarms.github.io/assets/fms2026-kv-cache-storage-four-vendors/fig-23-slide-48-micron-kv-workload.png) 

*图 23：原 PDF 第 48 页。配置表是阅读后面结果的起点；并发 64 与结果脚注中的 4 users、上下文单位等信息尚未完整对齐。*

第 49 页先比较位置：本地 DRAM 为 23.1 tokens/s，CXL JBOM 为 23.0 tokens/s，原页概括为约 99% 吞吐持平。脚注写两台 GNR 服务器、每台两个 H200，模型为 Llama 3.1 70B，并提到 4 users。页面同时写 90% TTFT 持平，未提供逐长度数据或比值定义，不能直接转换为延迟降低 10%。[\[1\]](https://computeexpresslink.org/wp-content/uploads/2026/09/CXL_FMS-2026-Panel-Presentation_FINAL.pdf#page=49)

[

![[KV Cache 存储的四种理解：重读 FMS 2026 的 CXL Panel-25.png]]

](https://miraclefarms.github.io/assets/fms2026-kv-cache-storage-four-vendors/fig-24-slide-49-micron-dram-cxl-parity.png) 

*图 24：原 PDF 第 49 页。位置改变后 token 吞吐接近持平；token/s 按请求、用户、GPU 还是系统统计，以及 TTFT parity 的精确定义，原页没有补齐。*

第 50 页比较的则是 prefill 全重算与 KV 复用。全重算为 24 tokens/s，89% KV block 复用为 160 tokens/s，约 6.67 倍，吻合 6.7 倍标记。这页还明确多 GPU 共用分离式内存中的 KV，服务器可以使用另一服务器的 KV block。

[

![[KV Cache 存储的四种理解：重读 FMS 2026 的 CXL Panel-26.png]]

](https://miraclefarms.github.io/assets/fms2026-kv-cache-storage-four-vendors/fig-25-slide-50-micron-kv-reuse.png) 

*图 25：原 PDF 第 50 页。6.7 倍来自重算与复用的对照，89% 指 KV block 复用。TTFT 343s→59s 与旁边 2.7 倍标记不吻合，不能静默把其中一组当成已校正结果。*

TTFT 的原值为 343s 和 59s，算术比例约 5.81，但右侧写 2.7 倍。缺少原始数据时，既不能替报告挑一个倍率，也不能自行修改时间单位。能确认的是：页面用 KV 复用解释减少 prefill 的方向，绝对延迟与倍率仍需澄清。

**Micron 的明确应用结果是：在其测试条件下，CXL 位置对照接近 DRAM，跨主机 KV 复用相对全重算提高了吞吐。** 5.5 倍性能/美元仍是估算，报告缺少设备价格、功耗、统计边界和成本模型，无法据此建立完整采购预算。

### 5.6 路线图说明系统交付阶段

第 51 页回到系统落地：Micron 为 PNNL 构建的 Crete 2.0 原型于 2025 年 6 月交付，Abaco 3.0 指向 2026 Q4 商用，机架容量从 15TB 扩展到 100TB 以上。合作角色包括 PNNL、XConn、Liqid 和 Primemas。

[

![[KV Cache 存储的四种理解：重读 FMS 2026 的 CXL Panel-27.png]]

](https://miraclefarms.github.io/assets/fms2026-kv-cache-storage-four-vendors/fig-26-slide-51-micron-production-roadmap.png) 

*图 26：原 PDF 第 51 页。原型交付、商用计划和容量升级属于不同阶段。本文记录会场时的路线图，没有据此确认各配置后续实际交付状态。*

因此，Micron 对 KV 存储的理解包含硬件容量、主机软件抽象、推理框架复用和系统交付四部分。参考系统已经展示了这些部分如何组合，但生产 KV 服务所需的完整协议与长期运行数据仍有空白。

## 六、把四条路线放回同一张比较表

四家报告的共同假设是，有价值的状态会持续增长，单机里固定的内存容量难以跟上。它们选择的解法互相衔接：Liqid 组织资源，UnifabriX 选择访问位置，Astera Labs 提供连接和控制能力，Micron 构建完整系统与案例。

| 比较维度       | Liqid              | UnifabriX           | Astera Labs         | Micron                        |
| ---------- | ------------------ | ------------------- | ------------------- | ----------------------------- |
| 主要存储对象     | 可跨请求复用的 KV 与其他工作集  | CPU/GPU 不同访问域里的共享状态 | 可分配、可管理的多主机内存资源     | KV、数据库、图与分析工作集                |
| 容量怎么扩展     | 独立 chassis、池化与资源管理 | MPU 共享内存平台          | Leo/Scorpio 支撑多种拓扑  | Abaco 的 RDIMM/AIC/JBOM 组合     |
| 软件层重点      | 分配与多主机访问、famfs     | 共享域与接入路径            | COSMOS、主机软件/BMC、RAS | DAX/famfs 与 Dynamo/vLLM/KV 管理 |
| KV 收益的直接证据 | 加载、DRAM 对照、命中率三类实验 | 重复运行的 TTFT 曲线       | 本节没有独立 KV 吞吐对照      | DRAM 对照与重算/复用对照               |
| 原件中仍开放的部分  | benchmark 配置、部分倍率  | G1.5 对应实测与协议细节      | 动态分配、隔离及主机支持进度      | 配置映射、TTFT 口径、TCO 复算           |

按这个分类，CXL 的意义也更具体。它可以扩展主机内存，可以让多主机共享状态，也可以进入更靠近 GPU 的访问域；每一种用法都需要相应的控制与软件适配。只写“使用 CXL”不足以描述 KV 的物理位置和消费方式。

同样，“共享”至少涉及两步：硬件让多个主机能够访问资源，推理软件让它们能够辨识并复用正确的 block。DAX/famfs、controller 和 KV Block Manager 分别处在其中不同位置。一个环节的可达性结果，不能代替另一个环节的正确性与性能验证。

**四家厂商可以组成一条系统链，但它们的性能数字不能组成一条乘法链。** 40 倍加载差距、约 99% 位置持平、6.7 倍复用收益和 5.5 倍性能/美元估算，测量边界与证据状态都不同。

## 七、这份 Reading 留下的工程判断

这场 panel 把 KV 当作需要独立配置、共享和管理的状态资源。长上下文提高保存状态的容量要求，多轮调用提高再次使用它的机会；更大的池只有承载了会被再次使用的状态，才会减少实际重算。

原报告已经给出几类可用证据：Liqid 的加载与命中率对照，UnifabriX 的冷启动/复用曲线，Astera Labs 的控制器功能状态，Micron 的位置对照与系统案例。它们足以帮助确定下一步应该验证什么，却尚未提供一套完整、可复现的生产定容方案。

若要把报告转成工程选型，仍需固定同一组模型、请求轨迹、活跃工作集与复用策略，再分别测量容量增长、位置替换、跨主机共享和减少重算的收益。还要记录 GPU 端实际取数带宽、P99 TTFT、队列、功耗和故障恢复。尤其第 14 页的 NVMe 加载较慢，材料未分解是介质带宽、I/O 控制、并行度还是 GPU 搬运造成的，这场 panel 无法裁决 SSD 瓶颈的来源。

第 50 页的 block 复用结果也不能直接外推到逐 token 的外部 attention 供数。后者每轮都要满足计算时序，访问次数与可隐藏的延迟完全不同；报告中 GPU 侧共享架构需要新的对应实验来验证。

**KV 存储方案应沿着“留住哪些状态、谁能复用、如何取回、省下多少重算”逐项评估。** 这四个问题能把容量规格、共享架构和吞吐数字重新连成一条可检验的系统论证。

- - -

## 参考资料

\[1\] [CXL Consortium：Beyond the Memory Wall — FMS 2026 Panel Presentation](https://computeexpresslink.org/wp-content/uploads/2026/09/CXL_FMS-2026-Panel-Presentation_FINAL.pdf)

\[2\] [CXL Consortium：Increasing Tokens-per-Dollar with CXL — FMS 2026 Recap](https://computeexpresslink.org/blog/increasing-tokens-per-dollar-with-cxl-fms-2026-recap-4788/)

\[3\] [CXL Consortium：About CXL](https://computeexpresslink.org/about-cxl/)

\[4\] [FMS 官方：FMS Timeline](https://terrapinn.com/conference/future-memory-storage/FMS-Timeline.stm)
