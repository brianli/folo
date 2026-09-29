---
title: "UCIe 全景阅读：三代规范、三种封装，与一个尚未到来的 chiplet 市场"
source: "Readings"
category: "AI Infra"
group: "MiracleFarms"
url: "https://miraclefarms.github.io/notes/2026/09/03/ucie-chiplet-interconnect-survey/"
published: 2026-09-03T20:40:00+08:00
saved: 2026-09-27T16:29:31+08:00
folo_key: "mf-readings::https://miraclefarms.github.io/notes/2026/09/03/ucie-chiplet-interconnect-survey/"
tags:
  - "folo"
  - "AI_Infra"
  - "Readings"
---

# UCIe 全景阅读：三代规范、三种封装，与一个尚未到来的 chiplet 市场

> [!info] Readings · AI Infra · 2026-09-03 20:40 · [原文](https://miraclefarms.github.io/notes/2026/09/03/ucie-chiplet-interconnect-survey/)

![[UCIe 全景阅读：三代规范、三种封装，与一个尚未到来的 chiplet 市场-01.jpg]]

1965 年那篇提出摩尔定律的文章里，还藏着另一句预言：把大系统拆成分别封装、再互连起来的小功能，可能更经济。这句话在此后半个世纪几乎没有兑现——单片集成一直赢，因为它更快、更省电、也更便宜。直到 2022 年 3 月，十家公司把一份叫 UCIe 的规范推上台，Intel 的 Debendra Das Sharma 在几乎每一场关于它的演讲里都会把这句 1965 年的话放进第二页幻灯片[\[12\]](https://www.ieee-edps.com/archives/2023/c/0900sharma.pdf)。

四年过去，UCIe 已经走完四个版本，把封装内互连的物理层、链路层和管理面收敛成了一份公共契约，KPI 表把 2D、2.5D、3D 三档全部填满，最高一档的带宽密度做到 300,000 GB/s/mm²、能效 0.01 pJ/bit[\[4\]](https://arxiv.org/abs/2510.06513)。但把 chiplet 变成”像买 IP 那样买回来焊上去”的市场，标准只完成了其中最容易的一半：**到 2026 年，UCIe 仍然没有一次物理 plugfest，跨厂商互操作验证全部依赖仿真环境。** 这篇文章从五份综述和一手规范材料出发，把 UCIe 的技术版图和它没解决的问题一并摊开。

## 一、五份综述，各自回答什么问题

想快速建立 UCIe 的全貌，最省时间的路径是先读综述再读规范，因为 UCIe 规范本身是一份面向实现者的文档——它告诉你每个字段怎么填，不告诉你为什么这么设计。

第一份必读是 Das Sharma 发表在 ACM Computing Surveys 上的《An Introduction to the Universal Chiplet Interconnect Express》[\[1\]](https://dl.acm.org/doi/10.1145/3819235)，2026 年 5 月 29 日上线。它按”三代演进”的线索串起 UCIe 的全部技术组件，覆盖平面与垂直两种连接形态，并用一节讨论已有的 UCIe 实现。这是目前信息密度最高的单篇材料。

读它的时候要带一个前提：作者本人是 UCIe 规范的主要设计者和推动者。这份综述的准确性没有问题，但它对”标准能否兑现开放生态”的判断天然偏乐观——它是标准制定者写的标准介绍，不是第三方评估。**读 UCIe 的一手材料必须同时准备一份来自外部的对冲视角。**

对冲视角的第一份是 Ma 等人 2022 年发表在《CCF Transactions on High Performance Computing》上的《Survey on chiplets: interface, interconnect and integration methodology》[\[2\]](https://link.springer.com/article/10.1007/s42514-022-00093-0)。它写在 UCIe 1.0 刚发布的时点，视角是整个 chiplet 领域而非单一标准，把接口、互连和集成方法学三条线并列展开。用它来定位 UCIe 在更大坐标系里的位置，比读 UCIe 自己的材料更清楚。

第二份是 Yang 等人的《Challenges and Opportunities to Enable Large-Scale Computing via Heterogeneous Chiplets》[\[3\]](https://arxiv.org/abs/2311.16417)（ASP-DAC 2024）。它有一张我认为是全领域最有用的表——Table II 把 USR、AIB、BoW、HBM、LIPINCON、UCIe、AAC 七种 D2D 接口的能效、最高速率和容错机制并排列出。UCIe 在这张表里是 0.25–1.25 pJ/bit、32 GT/s、带 CRC 和重传；作为对照，TSMC 的 LIPINCON 是 0.424 pJ/bit 但只有 2.8 Gbps，中国芯粒产业联盟的 AAC 是 2.5 pJ/bit 换 128 Gbps。**七种接口同时存在这件事本身，就说明 UCIe 赢下的是标准话语权，还不是物理层的唯一解。**

第三份对冲来自 Feng 等人的 MICRO 2023 论文《Heterogeneous Die-to-Die Interfaces》[\[5\]](https://dl.acm.org/doi/10.1145/3613424.3614310)。它直接质疑”单一统一接口”这个前提：任何一种接口都有其适配的工作负载、规模和场景，用统一接口的 chiplet 无法在不同系统之间自由复用，面对混合负载时也表现不佳。作者提出让 chiplet 同时挂并行和串行两种接口。这篇是 CC-BY 开放获取，值得完整读一遍。

最后一份是理解 UCIe 未来方向的关键：Das Sharma 等人在 Hot Interconnects 2025 上的《On-Package Memory with UCIe》[\[4\]](https://arxiv.org/abs/2510.06513)。它提出给 UCIe 加内存语义，用来替换 HBM 和 LPDDR 的并行总线前端。这篇同时也是 UCIe 技术细节写得最清楚的公开材料之一，Table 1 那张 KPI 表在后面几节会反复用到。

建议的阅读顺序：先读 \[3\] 的 Table II 建立坐标系，再读 \[1\] 建立完整技术地图，然后用 \[5\] 打一遍反面，最后读 \[4\] 看它要去哪里。\[2\] 作为历史背景随时可查。

## 二、UCIe 标准化了什么：三层结构加一个协议插槽

UCIe 是一个三层协议：协议层、die-to-die 适配器（D2D Adapter）、物理层[\[4\]](https://arxiv.org/abs/2510.06513)。这个分层看上去平平无奇，但它做出的一个关键选择决定了 UCIe 后来的全部形态。

![[UCIe 全景阅读：三代规范、三种封装，与一个尚未到来的 chiplet 市场-02.png]]

*图 1：(a) 标准封装与几种先进封装形态（EMIB 硅桥、CoWoS 中介层、FOCoS-B 扇出中介层）；(b) UCIe 的三层结构，右侧的 Raw Mode 直接旁路 D2D 适配器接到 RDI；(c) 单模块与多模块聚合。来源：Das Sharma 等，Hot Interconnects 2025，Figure 2。*

先看物理层。UCIe 的基本单位是”模块”（module）：N 条单端、单向、全双工的数据 lane，加一条 valid、一条 tracking，每个方向再配一对差分转发时钟。UCIe-S 的 N 是 16，UCIe-A 的 N 是 64。边带（sideband）是独立的两条单端 lane——一条数据、一条 800 MHz 转发时钟，每个方向各一组，负责链路训练、寄存器访问和诊断[\[4\]](https://arxiv.org/abs/2510.06513)。1 个、2 个或 4 个模块可以聚合成一条链路，性能按倍数叠加。

边带这个设计值得单独说一句。它是”永远在线”的旁路通道[\[13\]](https://www.openfabrics.org/wp-content/uploads/2023-workshop/2023-workshop-presentations/day-2/201_DDasSharma.pdf)，训练、配置、错误上报、性能计数器读取全走这里。800 Mb/s 每方向的带宽在主数据通道面前微不足道，但它把管理面从数据面里彻底摘了出来。到 UCIe 3.0，边带的可达距离被延长到 100 mm[\[8\]](https://www.uciexpress.org/post/ucie-3-0-specification-redefining-chiplet-interconnects)，为的就是覆盖越来越大的封装尺寸。

再看协议层。UCIe 没有发明新的事务语义，它把已经存在的搬进封装：PCIe、CXL，以及一个叫 Streaming 的通道用来承载 AXI、CHI 这类片上协议[\[13\]](https://www.openfabrics.org/wp-content/uploads/2023-workshop/2023-workshop-presentations/day-2/201_DDasSharma.pdf)。图 1(b) 右上角那个紫色的 Raw Mode 更激进：它允许协议层直接旁路 D2D 适配器，接到 RDI 接口上，把 UCIe 当成一根纯粹的高速线用。

**协议层被设计成一个插槽，而不是一套固定语义，这是 UCIe 和 UALink/SUE 这类 scale-up fabric 最大的路线差异。** 前期调研显示，开放 scale-up 标准的物理层已经趋同到 100G/200G PAM4 和以太网衍生 PHY，真正的分野从数据链路层和事务层才开始[\[17\]](https://miraclefarms.github.io/notes/2026/06/30/scale-up-fabric-datalink-transaction-layer/)——UALink 重新定义了 flit、credit 和内存语义事务，SUE 保留以太网 packet 生态。UCIe 走了第三条路：它把物理层和链路层标准化到底，然后在协议层开一个口子，让差异化留在插槽上面。谁想插 CXL 就插 CXL，想插自研的片上总线就走 Streaming，什么都不想插就走 Raw Mode。

另外几个不显眼但对实现者关键的设计也落在这一层。UCIe 把配置寄存器写进了规范，用于发现和运行时的控制与状态上报，每一层都有自己的一组，而且这套机制对现有驱动是透明的[\[13\]](https://www.openfabrics.org/wp-content/uploads/2023-workshop/2023-workshop-presentations/day-2/201_DDasSharma.pdf)。规范还固定了 RDI 和 FDI 两个内部接口——FDI 是协议层到 D2D 适配器的 flit 感知接口，RDI 是适配器到物理层的原始接口——这两个接口被标准化之后，IP 供应商才可能把 PHY 和控制器拆开卖，客户也才可能混搭不同来源的 IP。UCIe 的 bump map 同样写在规范里，而且刻意为未来更小的 bump pitch 留了演进路径。

这个选择的代价在第八节会显现——插槽的灵活性买来了采用率，但也意味着”两颗都支持 UCIe 的 die”未必能通信。

## 三、KPI 表怎么读：封装形态才是自变量

理解 UCIe 的性能，最快的路径是把它那张 KPI 表读透。这张表在 UCIe 1.0 白皮书里首次出现，Hot Interconnects 2025 那篇论文把它扩展成了 2D/2.5D/3D 三列的完整版[\[4\]](https://arxiv.org/abs/2510.06513)。

![[UCIe 全景阅读：三代规范、三种封装，与一个尚未到来的 chiplet 市场-03.png]]

*图 2：UCIe 三种封装形态的完整 KPI 对照。注意底部脚注——原作者明确写出 UCIe 目前受限于 bump（”we are bump-limited”），这句话解释了为什么 3D 那一列的数据率反而最低。来源：Das Sharma 等，Hot Interconnects 2025，Table 1（数据源自 UCIe 1.0 与 2.0 规范）。*

先把三档的数字摆出来。UCIe-2D（标准封装）：数据率 4/8/12/16/24/32 GT/s，每方向 16 lane，bump pitch 100–130 µm，信道可达 25 mm，线密度带宽 28–224 GB/s/mm，面密度 22–125 GB/s/mm²，能效 0.5 pJ/bit（≤16G）或 0.6（>16G）。UCIe-2.5D（先进封装）：同样的数据率档位，每方向 64 lane，bump pitch 收到 25–55 µm，信道只剩 2 mm，线密度带宽跳到 165–1317 GB/s/mm，能效降到 0.25/0.3 pJ/bit。

UCIe-3D 这一列的数字则完全换了一个量级。bump pitch ≤1–9 µm，信道长度约等于零（混合键合），面密度带宽 4000 GB/s/mm²（9 µm）到 300,000 GB/s/mm²（1 µm），能效 0.05 到 0.01 pJ/bit，往返延迟不到 1 ns。而它的数据率是三档里最低的——不超过 4 GT/s。

**数据率越低、带宽越高，这个看似矛盾的组合正是 UCIe 性能模型的核心：这套接口是被 bump 数量而非信号速率卡住的。** 当 bump pitch 从 110 µm 收缩到 1 µm，单位面积能放下的 lane 数量增长了四个数量级，此时把每条 lane 跑到 32 GT/s 既没必要也不划算——降到 4 GT/s 换来的是每比特能耗从 0.5 pJ 掉到 0.01 pJ。混合键合把互连问题从”信号完整性”转成了”对准精度和良率”。

对做方案的人来说，这张表的读法是：**选 UCIe 不是选一个速率档位，是选一种封装成本结构。** 挑 UCIe-2D，你拿到 25 mm 的走线余量和便宜的有机基板，代价是带宽密度低一个量级；挑 UCIe-2.5D，你把自己绑在 CoWoS 或 EMIB 的产能上，2 mm 的信道长度意味着 die 必须挨着放；挑 UCIe-3D，你需要混合键合产线，而且 die 的功耗密度和散热要重新算。这三档之间不是升级关系，是三种不同的工程约束。

这张表还藏着一个容易被误读的地方：线密度带宽（GB/s/mm）和面密度带宽（GB/s/mm²）是两个不同的口径，对应两种不同的物理约束。2D 和 2.5D 的接口沿着 die 的边缘排列，能拿到多少带宽取决于你愿意分给 D2D 接口多长的 shoreline，所以线密度是主导指标；UCIe-3D 靠混合键合在整个 die 面上垂直互连，边缘长度不再是约束，表里那一格直接写着”仅面积口径”。**这两个口径不能混着比——一颗 die 的边缘长度以毫米计，面积以平方毫米计，用面密度去论证边缘接口的带宽，会把结论放大到失真。** 前期调研在 scale-up fabric 那边总结过同一类问题：公开的协议带宽比较必须标注版本和作用域，区分单 die、单 chip、交换 ASIC 和 fabric 聚合口径[\[18\]](https://miraclefarms.github.io/notes/2026/07/15/southbound-interconnect-full-stack-survey/)。UCIe 的两栏带宽是这条纪律在封装内的版本。

模块聚合是另一个需要一起算的变量。一条 UCIe 链路由 1 个、2 个或 4 个模块组成，带宽按模块数线性叠加，但 shoreline 占用同样线性增长。所以真实的设计决策落在”我愿意在这条 die 边上分出多少毫米给 D2D、剩下的留给 HBM 和其他接口”上，速率档位只是这道分配题的一个输入。多 die 拼接的芯片里，D2D 接口、HBM PHY 和对外 I/O 在争同一条边，这个分配问题往往比选速率更难。

另外两个数字容易被忽略但很关键。一是低功耗进出延迟：不到 1 ns 完成进入和退出，节能超过 85%[\[12\]](https://www.ieee-edps.com/archives/2023/c/0900sharma.pdf)。这让 UCIe 链路可以在空闲时激进地关掉，而不用担心唤醒代价。二是可靠性目标：Flit Mode 下的 FIT（十亿小时失效次数）目标是远小于 1，预期在 1E-10 量级。这个数字之所以重要，是因为封装内的 die 一旦焊上去就换不了——一条 D2D 链路的可靠性要求，比一根可插拔的 PCIe 线缆高好几个数量级。

## 四、D2D 适配器：可靠性怎么塞进 2 ns 预算

上一节那个 2 ns 的往返延迟，包含了 D2D 适配器和物理层的全部开销（从 FDI 接口到 bump 再回来）[\[12\]](https://www.ieee-edps.com/archives/2023/c/0900sharma.pdf)。在这个预算里同时塞进 CRC 校验、链路级重传和多协议复用，是 UCIe 设计里最见功夫的部分。

D2D 适配器的核心职责是可靠交付。它支持三种 flit 格式加一种旁路模式[\[14\]](https://www.design-reuse.com/blog/55294-ucie-d2d-adapter-explained-architecture-flit-mapping-reliability-and-protocol-multiplexing/)。68B flit 是给 CXL 68B 模式和 PCIe 非 flit 模式用的：在 64 字节数据前面加 2 字节 flit 头、后面缀 2 字节 CRC，这条路径的数据率上限被压在 32 GT/s。256B flit 有两个变体，其中延迟优化版是给 CXL 3.0 和 Streaming 推荐的。Raw 格式最简单——64 字节数据从协议层直通物理层，不做任何加工，JESD204E 这类流式协议靠它才能跑在 UCIe 上。

CRC 的设计有个细节值得记住：校验码覆盖 128 字节载荷，用 16 比特提供三比特翻转的检测保证；不足 128 字节的消息补零后再算。校验失败触发重放。**重传对非 Raw 格式是强制项，触发门槛是误码率超过 1e-27。** 相比 PCIe，UCIe 的重传机制被显著简化了：选择性 NAK 和接收侧重试缓冲的规则被整个删掉，所有 10 比特的重试计数器压缩成 8 比特。

这些简化不是偷懒，是延迟预算逼出来的。封装内的链路只有几毫米长，误码率极低且错误模式单一，PCIe 为长距离电缆准备的那套复杂重传逻辑在这里纯属浪费硅片面积和时钟周期。

多协议复用则提供了两个档位。常规模式让两个相同的协议实例各占一半带宽；增强模式允许不同协议各自占满 100% 带宽，代价是它们必须共用同一种 flit 格式，且不能连续发送同协议的 flit——中间要插 NOP。

把这一节和前期调研的判断对照，会看到一个有意思的镜像。在 scale-up fabric 那一侧，数据链路层的重传归宿直接决定端点硅片面积，Broadcom 的 SUE Lite 靠移除端点 reliable transport 层和 R-CRC，把 IP 面积最多砍掉 50%[\[17\]](https://miraclefarms.github.io/notes/2026/06/30/scale-up-fabric-datalink-transaction-layer/)。UCIe 面对同一道选择题时给的是另一个答案：**把重传做成强制项，让每个 UCIe 端点都背上重试缓冲的面积成本，然后开一道 Raw Mode 的门给不愿意付这笔钱的人。** 强制换来的是”任何两个 Flit Mode 实现都能互通”的底线保证；那道门则承认了这个底线并非所有场景都需要。

## 五、封装内的容错：备用 lane 与降宽，两种封装两种答案

一根 PCIe 线缆坏了可以换。一条 die-to-die 链路上的某条 lane 在装配后失效，整颗封装就是废品——除非接口本身能绕开它。UCIe 在这件事上给出的答案按封装形态分叉，而这个分叉的理由完全是经济学的。

先进封装用备用 lane，标准封装用降宽运行[\[13\]](https://www.openfabrics.org/wp-content/uploads/2023-workshop/2023-workshop-presentations/day-2/201_DDasSharma.pdf)。UCIe-A 的一个模块有 64 条数据 lane，bump pitch 只有 25–55 µm，多铺几条备用 lane 占用的面积和 shoreline 都可以忽略，坏一条就切一条顶上，链路宽度保持不变。UCIe-S 的一个模块只有 16 条 lane，bump pitch 是 100–130 µm，每一条 lane 都在吃宝贵的 die 边缘长度，铺备用的代价按比例算高得多，于是规范选择让链路以更窄的宽度继续工作，用带宽换可用性。

**同一份规范在两种封装下给出两种容错策略，分叉的理由落在 bump 的单位成本上——两者差了一个量级，而可靠性目标是同一个。** 这个设计细节把第三节那句”选 UCIe 是选一种封装成本结构”具体化了：封装形态不只决定带宽和能效，还决定了链路降级时你拿到的是”性能不变”还是”带宽打折”。

修复动作发生在链路初始化阶段。整个初始化分四步：复位、边带初始化、主带训练与修复、协议参数交换[\[14\]](https://www.design-reuse.com/blog/55294-ucie-d2d-adapter-explained-architecture-flit-mapping-reliability-and-protocol-multiplexing/)。lane 修复在第三步完成，等到第四步协议层拿到参数时，它看到的已经是一条完好的链路——协议层不需要知道底下换过几条 lane。

物理层还提供了另外两个自由度：发送侧支持 lane reversal，规范也明确支持 die 的旋转和镜像摆放[\[13\]](https://www.openfabrics.org/wp-content/uploads/2023-workshop/2023-workshop-presentations/day-2/201_DDasSharma.pdf)。这两条合起来的意思是，同一份 bump map 可以在物理布局上被翻转对接，SoC 设计者不必为了迁就接口顺序而扭曲版图。还有一个容易忽略的细节：边带那两条 lane 周围的地 bump 被刻意去掉了，为的是不额外占用 shoreline——在一个被 bump 数量卡住的接口里，每一格 shoreline 都要算账。

运行期的容错则是后两代补上的。1.1 引入了运行时健康监测与修复，以及预测性失效分析，直接动机是车规客户对十年现场可靠性的要求[\[10\]](https://www.uciexpress.org/post/ucie-consortium-releases-the-ucie-1-1-specification-at-fms-2023)；3.0 又加了发送侧的运行时重校准，让链路能在不中断服务的前提下调整时序[\[8\]](https://www.uciexpress.org/post/ucie-3-0-specification-redefining-chiplet-interconnects)。

把这套机制的边界写清楚很重要。**UCIe 的容错模型是”训练期修复加运行期监测”，它针对的是制造缺陷和器件老化这类相对静态的失效，而不是毫秒级的动态链路抖动。** 封装内的信道只有几毫米、没有连接器、没有插拔，这个假设在绝大多数情况下成立。但它意味着 UCIe 不提供 scale-up fabric 那种端口级的动态切换和快速重收敛能力——如果一个系统同时需要封装内和封装间的容错，这是两套独立的机制，各自的故障域、检测延迟和恢复路径都不一样，不能指望其中一套覆盖另一套。

## 六、四代规范的演进顺序，本身就是信息

UCIe 的四个版本发布节奏近乎精确的年更：1.0 在 2022 年 3 月 2 日随联盟成立发布，1.1 在 2023 年 8 月 8 日，2.0 在 2024 年 8 月 6 日，3.0 在 2025 年 8 月 5 日[\[7\]](https://www.uciexpress.org/press-releases)。四个版本全部向后兼容。

1.0 打的是地基：分层结构、两种封装形态、PCIe/CXL/Streaming 协议映射、完整的 KPI 目标[\[11\]](https://ieeexplore.ieee.org/document/9893865/)。

1.1 的重心落在车规[\[10\]](https://www.uciexpress.org/post/ucie-consortium-releases-the-ucie-1-1-specification-at-fms-2023)：预测性失效分析、健康监测、运行时健康监控与修复，另外补了新的 bump map 来降低封装成本。选择在第二个版本就做车规，反映的是汽车客户对 chiplet 的兴趣来得比想象中早——车规要求的是十年以上的现场可靠性和可诊断性，这类需求会倒逼标准提前把运行时监控写进规范。

2.0 是转折点。它做了三件事：把 UCIe-3D 加进来（针对混合键合优化，bump pitch 从 10–25 µm 一直覆盖到 1 µm 以下）；引入可选的可管理性特性和一套 UCIe DFx 架构（UDA），在每颗 chiplet 内部放一张管理织物，负责测试、遥测和调试；建立物理层、适配器和协议三个层次的初始合规测试框架[\[9\]](https://www.businesswire.com/news/home/20240806155624/en/UCIe-Consortium-Releases-2.0-Specification-Supporting-Manageability-System-Architecture-and-3D-Packaging)。

**用整整一代版本去做 DFx 和管理面，而不是继续推带宽，说明标准组织认识到”多厂商 chiplet 拼出来的东西怎么测试和排障”是比带宽更早撞上的墙。** 一颗封装里如果坐着来自三家公司、三个工艺节点的 die，出问题时谁负责定位？在 UDA 之前，这个问题没有标准答案。

3.0 才回过头抢速率：48 GT/s 和 64 GT/s 两个新档位，把 2.0 的带宽翻倍，UCIe-S 和 UCIe-A 都适用[\[8\]](https://www.uciexpress.org/post/ucie-3-0-specification-redefining-chiplet-interconnects)。同时引入发送侧的运行时重校准，让链路可以在不牺牲响应速度的前提下动态调整时序；边带可达距离延到 100 mm；管理面补上固件管理、低延迟信令和紧急控制。还有一个容易被忽略的新特性——通过 Raw Mode 映射 ADC/DAC 数据的连续传输模式，让对噪声敏感的设计不必生成会产生干扰的时钟。

把四代顺序连起来看：先性能，再可靠性，然后是可管理性，最后回来抢速率。**这条曲线和 UALink 在 2026 年 4 月同时发布 Chiplet 1.0 与 Manageability 1.0 的动作是同一种信号**——开放互连标准的成熟标志，从”能跑多快”转向”能不能被封装、部署、监控、升级和排障”[\[16\]](https://miraclefarms.github.io/notes/2026/06/21/ualink-open-scale-up-fabric/)。

## 七、两个外延：把 UCIe 用出封装，把内存塞进 UCIe

协议插槽的设计在两个方向上开始兑现，两个方向都超出了”封装内互连”的原始定位。

第一个方向是往外走。UCIe 定义了 retimer 用法：一颗 die 通过 UCIe 连到 retimer die，retimer 再把信号转成光或长距电信号送出封装[\[13\]](https://www.openfabrics.org/wp-content/uploads/2023-workshop/2023-workshop-presentations/day-2/201_DDasSharma.pdf)。在 Das Sharma 画的机架级图景里，CXL 通过 UCIe retimer 和共封装光学延伸到整个 pod，成为跨机架资源池化的物理载体。这条路径上 UCIe 只做一段——从计算 die 到 retimer die 的那几毫米——但它让”封装内标准”意外地成了光互连的接入口。

第二个方向是往内挖：给 UCIe 加内存语义。

![[UCIe 全景阅读：三代规范、三种封装，与一个尚未到来的 chiplet 市场-04.png]]

*图 3：(a) 三种形态——SoC 经 UCIe 连逻辑 die 再接 HBM 3D 堆栈（i）、逻辑 die 用引线键合接 LPDDR 堆栈（ii）、LPDDR die 直接原生支持 UCIe PHY（iii）；(b) 非对称 UCIe 上的内存协议，读写通道宽度不同；(c) 对称 UCIe 上的内存协议。来源：Das Sharma 等，Hot Interconnects 2025，Figure 3。*

这篇论文的出发点是一组成本和带宽的数字：片上 HBM 每比特成本是 LPDDR 的 5–10 倍，换来的是同容量下约 20 倍的带宽[\[4\]](https://arxiv.org/abs/2510.06513)。而 LPDDR 和 HBM 都还在用并行双向总线，命令和数据分开走、各跑各的频率，为的是迁就 DRAM 工艺。作者的论点是：I/O 领域过去二十年已经完成过一次同样的迁移——PCI 总线到 PCI Express、前端总线到点对点链路、平台级内存挂载转向 CXL——片上内存该轮到了。

技术上最有意思的是”非对称 UCIe”。内存访问天然是不对称的：命令只从 SoC 发向内存，读带宽通常是写带宽的两倍左右。所以他们提出让 UCIe 的两个方向使用不同宽度，比如 24 条读数据 lane 配 12 条写数据 lane，形成 2:1 的比例。在 100% 写的极端场景下这个方案只有原生 LPDDR6 一半带宽，100% 读时打平，混合读写时超过——而现实负载压倒性地偏读。他们还提出让 UCIe PHY 跑在 DRAM 频率的整数倍上，避免异步跨时钟域。

配置寄存器访问、错误上报、性能计数器和中断全部走边带的 800 Mb/s 通道，这样主数据通路可以把 CXL.io 整个去掉。论文给出的整体收益是：相比现有的 HBM4 和 LPDDR 片上方案，带宽密度最高 10 倍、延迟最低到三分之一、功耗最低到三分之一。

需要给这些数字加一条边界：**这是一份 2025 年 10 月的提案，不是已发布的规范内容，也没有第三方硅片实测。** 它的意义在于展示协议插槽的扩展方式——加一个协议映射就能吃下一个新市场，不用动物理层和适配器。但从提案到进入规范再到出货硅片，UCIe 自己的历史给出的参考周期是两到三年。

## 八、一个标准解决不了的：接口异构与缺席的 plugfest

前面六节讲的都是 UCIe 做到了什么。这一节讲它没做到什么，而这部分在标准制定者写的材料里通常只有一段。

第一个问题是接口异构没有消失。回到 Yang 等人那张对比表[\[3\]](https://arxiv.org/abs/2311.16417)：USR、AIB、BoW、HBM、LIPINCON、UCIe、AAC 七种接口同时在产，能效从 0.25 到 2.5 pJ/bit，最高速率从 2 Gbps 到 128 Gbps，容错机制各不相同。作者指出的后果很具体——当设计者要集成来自不同厂商、使用不兼容接口的 chiplet 时，就得引入”hub chiplet”做转接，而这会重复一次非经常性工程成本。标准的存在没有消灭转接层，只是把转接层需要覆盖的组合数减少了一些。

Feng 等人的批评更根本[\[5\]](https://dl.acm.org/doi/10.1145/3613424.3614310)：统一接口这个前提本身就限制了灵活性。任何一种接口都对应特定的负载、规模和场景——并行接口带宽密度高但走线短、串行接口能跨得远但能效差。用单一接口的 chiplet 无法在不同规模的系统之间自由复用，而现代计算系统面对的是混合负载。他们的方案是让 chiplet 同时挂两种接口。这个思路和 UCIe”一个接口通吃”的叙事直接冲突，而且它在 MICRO 上通过了同行评审。

第二个问题更实际：**UCIe 至今没有过一次物理 plugfest。** 2.0 版本建立了合规测试的初始框架，目标是把待测器件的主带特性对照一个已知良好的参考实现来验证。但正式的互操作与合规程序还没有开放，行业内的普遍解释是——规范发布到硅片可用之间有几年时间差，程序通常要等硅片铺开才启动。

目前填补这个空缺的是仿真：把不同公司的 UCIe IP 放进同一个仿真环境跑互操作，重点覆盖 LogPHY 和 D2D 层[\[15\]](https://chipletsummit.com/proceeding_files/a0qVV000000Jrmz/20250121_PreConG_Bunnell.PDF)。这个方法能提前发现规范理解偏差和实现层的协议缺口，也确实找出过靠读规范发现不了的问题。但仿真互操作和物理互操作之间隔着信号完整性、工艺偏差、封装装配公差和热行为——这几项恰恰是 D2D 接口最容易出事的地方。

把这两条放在一起，得到的判断是：**UCIe 已经把”两颗 die 之间的协议契约”标准化得相当彻底，但”两颗来自不同公司的 die 能不能被焊在一起并且工作”，目前还没有可验证的答案。** 前期调研在 scale-up fabric 那边得出过一条同构的结论——声称兼容某开放规范的方案，必须单列数据链路层的 replay 粒度、credit 模型、flit/PDU 尺寸和 QoS 映射，共用一套 PHY 证明不了上层语义一致[\[18\]](https://miraclefarms.github.io/notes/2026/07/15/southbound-interconnect-full-stack-survey/)。UCIe 的情况是这条结论的封装内版本，而且它连”复用同一套 PHY”这一层的物理验证都还欠着。

第三个问题在标准之外，而且可能是最难的：即使协议完全互通，商业上仍然缺答案。谁来保证 known-good die——一颗 die 在装配前如何被证明是好的，测试覆盖率由谁定义、由谁承担？多方 die 拼装后良率不达标，损失怎么在 die 供应商、封装厂和系统集成方之间分摊？封装的热预算和供电预算怎么在互不隶属的几方之间分配，谁有权在系统超温时降频谁的 die？这些不是规范能写的内容——UCIe 2.0 的 UDA 提供了跨 chiplet 的遥测和调试通道，让”看得见”成为可能，但”谁负责”是合同问题不是协议问题。

工具链这一侧的进展反而是最快的。到 2026 年，主流 EDA 厂商的 3DIC 流程已经能对 UCIe 和 HBM 做自动布线，并支持 EMIB 这类先进封装工艺[\[19\]](https://news.synopsys.com/2026-07-27-Synopsys-and-Intel-Foundry-Fast-Track-Customer-Readiness-from-Silicon-to-Systems-on-Intel-14A)。这意味着”在自己的封装里用 UCIe 连两颗自研 die”这条路径的工程摩擦已经很低。**工具链就绪、IP 就绪、规范就绪，而市场没就绪——缺的那一环恰恰是标准最初想解决的那一环。**

## 九、自研多 die 方案该怎么用这张表

把一颗大芯片拆成几颗小 die 再拼回去，是先进节点掩模成本和良率曲线压出来的做法：单芯粒面积每往上走一档，良率损失就吃掉一部分成本优势，到某个点之后拆开更划算。对沿着这条路走的团队，UCIe 的意义和”多厂商开放市场”那套叙事关系不大。

先看它落在 KPI 表的哪一格。两颗算力 die 挨着放、走 2.5D 共封，对应的是 UCIe-A 那一列——bump pitch 25–55 µm、信道长度 2 mm 以内、每方向 64 lane。这几个数字反过来约束物理布局：D2D 接口必须放在两颗 die 相邻的那条边上，接口占用的 shoreline 长度直接决定链路总带宽，die 的长宽比因此不再是自由变量。

shoreline 的账要一起算。一颗 die 的边缘要同时安排 D2D 接口、HBM PHY 和对外 I/O，被 D2D 占掉的部分就不能再用来挂 HBM。第三节那个”愿意分多少毫米给 D2D”的问题，在多 die 拼接里不是优化项而是硬约束——它同时决定了片间带宽上限和单 die 能挂的内存颗数，两个指标被同一条边耦合在一起。

第五节讲的容错分叉在这里也有直接后果。走先进封装拿到的是备用 lane 方案，单条 lane 失效由物理层在训练期切换掉，链路宽度不变、带宽不打折；走标准封装拿到的是降宽运行。这个差别对”多颗 die 对软件层表现为单一设备”这类设计几乎是前提条件：如果链路可能降宽，上层的透明性承诺就会在部分封装上变成性能不确定，调度器和通信库都得为此准备分支。**用备用 lane 换来的是可预测的性能契约，代价是先进封装的成本和产能约束——这笔交换要在选封装形态的时候就算清楚，而不是等流片回来再发现。**

**在这类场景里，UCIe 起的作用是降低自研 D2D 的设计风险，以及给封装与测试产业链一个明确的对齐目标，而不是”接上别人家的 chiplet”。** 用公开规范意味着 bump map、训练序列、flit 格式、重传逻辑都有现成的、被多方审阅过的答案，不用从零推导；也意味着 EDA 工具链、验证 IP 和测试方法学能直接复用生态里已有的东西。这两条价值在自研闭环内部就能兑现，完全不依赖多厂商互操作是否成立。

再往外一层看，前期调研在 scale-up fabric 那边观察到一个结构性现象：开放标准与生产部署之间存在系统性落差，连 UALink 董事会成员都会先用专有 fabric 保产量[\[16\]](https://miraclefarms.github.io/notes/2026/06/21/ualink-open-scale-up-fabric/)。UCIe 在封装内的情况是这个现象的翻版——采用规范的厂商很多，但绝大多数是把它用在自己封装内部的自研 die 之间。规范采用率和开放生态是两个不同的指标，前者已经很高，后者还没开始计数。

## 十、结论：读完 UCIe 应该带走什么

回到开篇那句 1965 年的预言。它在 2022 年被重新翻出来，靠的是一笔经济账而非技术突破——先进节点的掩模成本和单芯粒的良率曲线，把”拆开再拼”从一个选项压成了唯一选项。UCIe 是这个转变的产物，也是它的加速器。

四年四个版本之后，可以确认的部分是清楚的：UCIe 把封装内互连的物理层、链路层和管理面收敛成了一份被主要厂商共同签署的公共契约；它的 KPI 表覆盖了从便宜的有机基板到 1 µm 混合键合的完整跨度，带宽密度和能效在三档之间跨越四个数量级；它用”协议层作为插槽”的设计换到了极高的采用率，也为内存语义、光互连接入这类新用法留出了扩展空间。

未被证明的部分同样清楚：一个多厂商可互换的 chiplet 市场，目前只存在于规范文本和仿真环境里。没有物理 plugfest，没有开放的合规认证程序，没有公开的跨厂商互操作实测。七种 D2D 接口仍在并存，转接层没有消失。商业侧的良率归属、热预算分配和责任划分，标准根本没有触及。

对不同的读者，这两组结论指向不同的动作。如果你在设计自研的多 die 芯片，UCIe 现在就该用——它把 D2D 接口的设计风险压到了最低，生态工具链是现成的，这部分收益不依赖任何互操作承诺。如果你在评估”买第三方 chiplet 来拼”这条路，标准就绪度和生态就绪度要分开打分，后者的可信证据要等到第一次真正的物理互操作测试。

留一个开放问题：UCIe-3D 那一列的 300,000 GB/s/mm² 和 0.01 pJ/bit，是三档里唯一能显著改变系统架构假设的数字——当片间带宽逼近片内带宽、能效逼近片内互连，”一颗芯片”的边界会重新定义在哪里？Hot Interconnects 那篇把内存塞进 UCIe 的论文，可能是这个问题的第一个具体答案。

- - -

## 参考资料

\[1\] [An Introduction to the Universal Chiplet Interconnect Express® (UCIe®), ACM Computing Surveys](https://dl.acm.org/doi/10.1145/3819235)

\[2\] [Survey on chiplets: interface, interconnect and integration methodology, CCF THPC](https://link.springer.com/article/10.1007/s42514-022-00093-0)

\[3\] [Challenges and Opportunities to Enable Large-Scale Computing via Heterogeneous Chiplets](https://arxiv.org/abs/2311.16417)

\[4\] [On-Package Memory with Universal Chiplet Interconnect Express (UCIe), Hot Interconnects 2025](https://arxiv.org/abs/2510.06513)

\[5\] [Heterogeneous Die-to-Die Interfaces: Enabling More Flexible Chiplet Interconnection Systems, MICRO 2023](https://dl.acm.org/doi/10.1145/3613424.3614310)

\[6\] [UCIe Consortium Specifications](https://www.uciexpress.org/specifications)

\[7\] [UCIe Consortium Press Releases](https://www.uciexpress.org/press-releases)

\[8\] [UCIe 3.0 Specification: Redefining Chiplet Interconnects](https://www.uciexpress.org/post/ucie-3-0-specification-redefining-chiplet-interconnects)

\[9\] [UCIe Consortium Releases 2.0 Specification Supporting Manageability System Architecture and 3D Packaging](https://www.businesswire.com/news/home/20240806155624/en/UCIe-Consortium-Releases-2.0-Specification-Supporting-Manageability-System-Architecture-and-3D-Packaging)

\[10\] [UCIe Consortium Releases the UCIe 1.1 Specification at FMS 2023](https://www.uciexpress.org/post/ucie-consortium-releases-the-ucie-1-1-specification-at-fms-2023)

\[11\] [Universal Chiplet Interconnect Express (UCIe): An Open Industry Standard for Innovations With Chiplets at Package Level, IEEE](https://ieeexplore.ieee.org/document/9893865/)

\[12\] [Universal Chiplet Interconnect Express (UCIe), IEEE EDPS 2023](https://www.ieee-edps.com/archives/2023/c/0900sharma.pdf)

\[13\] [Universal Chiplet Interconnect Express (UCIe), OFA Workshop 2023](https://www.openfabrics.org/wp-content/uploads/2023-workshop/2023-workshop-presentations/day-2/201_DDasSharma.pdf)

\[14\] [UCIe D2D Adapter Explained: Architecture, Flit Mapping, Reliability, and Protocol Multiplexing](https://www.design-reuse.com/blog/55294-ucie-d2d-adapter-explained-architecture-flit-mapping-reliability-and-protocol-multiplexing/)

\[15\] [Accelerating UCIe Adoption: A Simulation-based Interoperability Program, Chiplet Summit 2025](https://chipletsummit.com/proceeding_files/a0qVV000000Jrmz/20250121_PreConG_Bunnell.PDF)

\[16\] [前期调研：UALink 把开放 scale-up fabric 推进到系统契约](https://miraclefarms.github.io/notes/2026/06/21/ualink-open-scale-up-fabric/)

\[17\] [前期调研：物理层趋同之后，scale-up fabric 的分家落在 DL 与 TL](https://miraclefarms.github.io/notes/2026/06/30/scale-up-fabric-datalink-transaction-layer/)

\[18\] [前期调研：超节点南向互联全栈综述](https://miraclefarms.github.io/notes/2026/07/15/southbound-interconnect-full-stack-survey/)

\[19\] [Synopsys and Intel Foundry Fast-Track Customer Readiness from Silicon to Systems on Intel 14A](https://news.synopsys.com/2026-07-27-Synopsys-and-Intel-Foundry-Fast-Track-Customer-Readiness-from-Silicon-to-Systems-on-Intel-14A)
