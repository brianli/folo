---
title: "Google TPU 架构演进：从单卡 ASIC 到可调度互联域"
source: "Essays"
category: "AI Infra"
group: "MiracleFarms"
url: "https://miraclefarms.github.io/notes/2026/06/22/google-tpu-architecture-evolution-interconnect/"
published: 2026-06-22T22:50:00+08:00
saved: 2026-09-27T16:27:24+08:00
folo_key: "mf-essays::https://miraclefarms.github.io/notes/2026/06/22/google-tpu-architecture-evolution-interconnect/"
tags:
  - "folo"
  - "AI_Infra"
  - "Essays"
---

# Google TPU 架构演进：从单卡 ASIC 到可调度互联域

> [!info] Essays · AI Infra · 2026-06-22 22:50 · [原文](https://miraclefarms.github.io/notes/2026/06/22/google-tpu-architecture-evolution-interconnect/)

![[Google TPU 架构演进：从单卡 ASIC 到可调度互联域-01.jpg]]

读 TPU 的代际史，很容易被两类数字牵着走：一类是 MXU / TensorCore 的峰值算力，另一类是 HBM 容量和带宽。它们当然重要，但如果只看这些，v4 的 OCS、v5e Multislice、TPU 8t 的 Virgo、TPU 8i 的 Boardfly 都会变成边角料。Google TPU 的演进更像一条互联域扩张史：从单机推理加速卡，到 2D torus 训练 Pod，再到 OCS 可重构 3D torus、ICI/DCN 两级集群，以及按训练/推理拆分的第八代系统。

本文关心三个问题。第一，每一代 TPU 的架构变化到底服务什么 workload。第二，ICI、OCS、DCN、Pathways 这些系统层机制如何逐渐盖过单芯片参数。第三，为什么 TPU 8 会拆成 8t training 与 8i inference 两条不同拓扑路线。

## 一、v1：先把推理塞进 Google 现有数据中心

> **本节论点**：TPU v1 的核心约束来自部署环境，而不是训练集群。它要像一张数据中心加速卡那样接入现有服务器，用确定性数据流、软件管理片上内存和大矩阵乘阵列换低延迟推理吞吐。 **本节核心参考**：
> 
> - ★★★ [In-Datacenter Performance Analysis of a Tensor Processing Unit](https://arxiv.org/abs/1704.04760)
> - ★★ [An in-depth look at Google’s first Tensor Processing Unit](https://cloud.google.com/blog/products/ai-machine-learning/an-in-depth-look-at-googles-first-tensor-processing-unit-tpu)

![[Google TPU 架构演进：从单卡 ASIC 到可调度互联域-02.png]]

*图 1：第一代 TPU 板卡。来源：Google Cloud Blog。*

TPU v1 是一张 28nm、700MHz 的推理 ASIC，2015 年已经在 Google 数据中心内运行。论文给出的核心配置是 65,536 个 8-bit MAC 组成的 Matrix Multiply Unit、28 MiB 片上软件管理内存、约 92 TOPS 峰值吞吐[\[1\]](https://arxiv.org/abs/1704.04760)。Google Cloud 后来的解读补充了部署细节：v1 板卡插在类似硬盘槽位的位置，通过 PCIe Gen3 x16 接主机，运行功耗约 40W[\[2\]](https://cloud.google.com/blog/products/ai-machine-learning/an-in-depth-look-at-googles-first-tensor-processing-unit-tpu)。

这代 TPU 没有今天的 Pod 叙事。它解决的是“现有在线服务里，CPU/GPU 在 99p latency 约束下不够划算”的问题。硬件选择也很直接：把复杂控制流、cache hierarchy、out-of-order 执行等通用处理器能力压掉，换成大 systolic array、CISC 风格指令和可预测的数据搬运。v1 论文报告，相比当时的数据中心 CPU/GPU，TPU 在神经网络推理上达到 15-30 倍性能和 30-80 倍性能/瓦收益[\[1\]](https://arxiv.org/abs/1704.04760)。

从后视镜看，v1 留下了两条路线。一条是 systolic array + software-managed memory 的芯片路线，后来延伸成 MXU / TensorCore / VMEM / HBM 分层。另一条是更隐蔽的系统路线：TPU 从一开始就是为 Google 自己的数据中心约束服务的硬件，而不是先做一颗通用芯片再找应用。

## 二、v2/v3：训练把 TPU 推进 2D torus Supercomputer

> **本节论点**：v2/v3 的转折点在芯片外部。训练需要频繁同步梯度、权重和激活，TPU 因此从单卡推理 ASIC 变成由 BF16、HBM、ICI、XLA 和 TensorFlow 共同定义的训练超算。 **本节核心参考**：
> 
> - ★★★ [A Domain-Specific Supercomputer for Training Deep Neural Networks](https://gwern.net/doc/ai/scaling/hardware/2020-jouppi.pdf)
> - ★★ [Cloud TPU v3 documentation](https://docs.cloud.google.com/tpu/docs/v3)

v2/v3 之后，TPU 的名字里虽然还保留着“processor”，但产品形态已经进入 supercomputer。Jouppi 等人在 TPU v2/v3 论文里把这套系统拆成六个共同设计对象：TensorFlow、XLA compiler、TPU chips、bfloat16、Inter-Core Interconnect，以及 pod-scale 芯片组织。论文强调，v2/v3 的训练收益来自软硬件共同设计，不能简化成单颗芯片替换[\[3\]](https://gwern.net/doc/ai/scaling/hardware/2020-jouppi.pdf)。

v1 面向已训练模型的 INT8 推理，v2/v3 面向训练，差别不只是精度从 INT8 扩到 BF16。训练的通信模式更重：数据并行要 all-reduce 梯度，模型并行要交换 activation 或 partial result，embedding 和推荐模型还会引入稀疏访问。TPU v2/v3 因此引入 HBM 和专用 ICI，并用 2D torus 支撑 Pod 内通信。论文图示用 16x16 2D torus 解释 TPU v2 supercomputer；Cloud TPU v2 文档又把 v2 slice 描述为 512 chips composed with reconfigurable high-speed links[\[4\]](https://docs.cloud.google.com/tpu/docs/v2)。这两个口径不宜硬合并，成文时可以理解为论文解释的拓扑示意与 Cloud 产品文档的 slice 规格处在不同上下文。

v3 延续 2D torus 路线，把 Pod 扩到 1024 chips。Cloud TPU v3 文档给出的单芯片规格是 123 BF16 TFLOPS、32 GiB HBM2、900 GB/s HBM bandwidth；Pod 侧则是 126 PFLOPS、all-reduce bandwidth 340 TB/s、bisection bandwidth 6.4 TB/s[\[5\]](https://docs.cloud.google.com/tpu/docs/v3)。这些指标提醒我们：v3 的价值不只在“每颗芯片更快”，还在 Pod 内 all-reduce 成为可被训练框架依赖的硬件路径。

这也是 TPU 和普通数据中心网络的早期分界。DCN 负责把机器接入数据中心，ICI 负责把 TPU 芯片组织成训练域。只要模型切分落在 ICI 域内，编译器和 runtime 就能以更稳定的延迟/带宽假设安排 parallelism；一旦跨出这个域，通信模式就要重新设计。

## 三、v4：OCS 让拓扑成为可调度资源

> **本节论点**：TPU v4 的架构创新不只在 4096 chips 和 3D torus。OCS 把 Pod 从静态布线升级为可重构资源池，让拓扑选择、碎片治理、故障绕行和增量部署进入同一个系统问题。 **本节核心参考**：
> 
> - ★★★ [TPU v4: An Optically Reconfigurable Supercomputer for ML](https://arxiv.org/abs/2304.01433)
> - ★★★ [Resiliency at Scale: Managing Google’s TPUv4 ML Supercomputer](https://www.usenix.org/system/files/nsdi24-zu.pdf)
> - ★★ [Cloud TPU v4 documentation](https://docs.cloud.google.com/tpu/docs/v4)

v4 是 TPU 集群架构的第一个大拐点。Cloud 文档给出的 v4 单芯片规格是 275 BF16 TFLOPS、32 GiB HBM2、1200 GB/s HBM bandwidth；Pod 规模是 4096 chips、1.1 EFLOPS、all-reduce bandwidth 1.1 PB/s、bisection bandwidth 24 TB/s[\[6\]](https://docs.cloud.google.com/tpu/docs/v4)。这些数字已经很大，但 v4 论文把更多篇幅给了 OCS 和 SparseCore。SparseCore 用约 5% die/power 加速 embedding-heavy 模型，论文报告 5-7 倍 embedding 模型加速；OCS 则用不到 5% 成本、不到 3% power 引入可重构光互联[\[7\]](https://arxiv.org/abs/2304.01433)。

NSDI 2024 的 TPUv4 resiliency 论文把这套系统讲得更清楚：v4 以 4x4x4 cube 为资源单元，每个 cube 由 16 台机器、每台 4 颗 TPUv4 chip 构成，cube 的六个方向暴露光 ICI 链路，再通过 Palomar OCS 拼成更大的 3D torus 或 twisted torus。一个 v4 supercomputer 有 64 个 cube、6144 条 optical ICI links 和 48 个 OCS；整个 Pod 是单一 ICI domain，任意 TPU 之间可 RDMA[\[8\]](https://www.usenix.org/system/files/nsdi24-zu.pdf)。

OCS 的价值不该按“它是不是普通交换机”来理解。它是 circuit switching：在作业开始前或恢复时改变光链路连接关系，让同一批物理 cube 组成不同的逻辑拓扑。这样可以缓解四类问题：第一，给不同作业分配不同大小和形状的 slice；第二，用 twisted torus 提高特定形状下的 bisection bandwidth；第三，绕开坏链路或坏 cube；第四，把新硬件逐步接入 Pod，而不需要一次性停机重布线。

Google Cloud v4 文档也保留了这种“双重口径”：基础 interconnect topology 写作 3D mesh，但大于一定形状的 slice 可配置为 3D torus，一些 3D slice 还支持 twisted torus。比如 4x4x8 twisted topology 相比非 twisted 形态理论 bisection bandwidth 提高约 70%[\[6\]](https://docs.cloud.google.com/tpu/docs/v4)。这说明 v4 的 Pod 已经不再只是“很多芯片接在一起”，而是一组可被调度器、拓扑选择器和 resiliency 系统共同管理的互联资源。

## 四、v5e/v5p：产品线分化，ICI/DCN 两级集群成型

> **本节论点**：v5 代开始出现清晰分工：v5e 用较小 2D torus 域追求性价比和可组合 scale-out，v5p 用更大的 3D torus 域服务大模型训练。Multislice 把 TPU 集群从单 Pod 问题推进到 ICI/DCN 分层问题。 **本节核心参考**：
> 
> - ★★ [Cloud TPU v5e documentation](https://docs.cloud.google.com/tpu/docs/v5e)
> - ★★ [Cloud TPU v5p documentation](https://docs.cloud.google.com/tpu/docs/v5p)
> - ★★ [World’s largest distributed LLM training job on TPU v5e](https://cloud.google.com/blog/products/compute/the-worlds-largest-distributed-llm-training-job-on-tpu-v5e)

v5e 和 v5p 放在一起看，能看出 TPU 产品线从“每代一个大 Pod”转向“同代不同优化目标”。v5e 是 256-chip 2D torus，单芯片 197 BF16 TFLOPS、16 GB HBM、400 GB/s ICI bandwidth；文档写得很直接：serving 通常支持单 host 最多 8 chips，training 最多 256 chips[\[9\]](https://docs.cloud.google.com/tpu/docs/v5e)。它的强项落在成本、供给和 GKE/Cloud TPU 使用便利性，而非单 Pod 最大规模。

v5p 则站在另一端。Cloud TPU v5p 文档给出 8960 chips per Pod，单芯片 459 BF16/FP8 TFLOPS、95 GiB HBM、1200 GB/s ICI bandwidth，interconnect topology 为 3D torus。单 slice training 最大 6144 chips，Multislice 可扩到 18,432 chips；部分 3D slice 还支持 twisted torus[\[10\]](https://docs.cloud.google.com/tpu/docs/v5p)。Google Cloud 发布 v5p 时把它放进 AI Hypercomputer 叙事，强调相比 v4，v5p 对大模型训练可达到 2.8 倍训练速度提升，second-generation SparseCore 对 embedding-dense 模型训练有 1.9 倍提升[\[11\]](https://cloud.google.com/blog/products/ai-machine-learning/introducing-cloud-tpu-v5p-and-ai-hypercomputer)。

v5e 的 Multislice 案例更能说明系统边界如何变化。Google Cloud 公开的 v5e 训练演示把 50,944 个 v5e chips 组织在 199 个 v5e pods 上，BF16 算力约 10 exa-FLOPs；slice 内通过 ICI 通信，pods 之间通过 Jupiter DCN 连接[\[12\]](https://cloud.google.com/blog/products/compute/the-worlds-largest-distributed-llm-training-job-on-tpu-v5e)。Multislice 文档也明确区分了两层网络：ICI 是 Pod 内高带宽低延迟链路，DCN 延迟更高、吞吐更低，但可以承载跨 slice 的数据并行通信[\[13\]](https://cloud.google.com/blog/products/compute/using-cloud-tpu-multislice-to-scale-ai-workloads)。

这和分布式训练里的并行策略分层正好对上。TP、CP、EP 这类高频、低延迟通信更适合留在 ICI 域；DP/FSDP 的跨副本同步可以被放到 DCN 上，通过 XLA/GSPMD 生成通信并尽量和计算重叠。v5e 证明“小 Pod”不等于“小集群”：单个 ICI 域可以较小，整体训练规模由 ICI 域、DCN、compiler 和 runtime 一起外推。

## 五、Trillium/Ironwood：推理压力进入 TPU 规格表

> **本节论点**：Trillium 继续强化通用训练/推理的单芯片能力，Ironwood 则把大 HBM、SparseCore、9216-chip 3D torus 和推理优先叙事绑在一起。TPU 的代际压力开始从训练 step time 扩展到 decode、MoE、KV working set 和尾延迟。 **本节核心参考**：
> 
> - ★★ [Cloud TPU v6e documentation](https://docs.cloud.google.com/tpu/docs/v6e)
> - ★★ [TPU7x / Ironwood documentation](https://docs.cloud.google.com/tpu/docs/tpu7x)
> - ★★ [Ironwood: The first Google TPU for the age of inference](https://blog.google/innovation-and-ai/infrastructure-and-cloud/google-cloud/ironwood-tpu-age-of-inference/)

Trillium，也就是 TPU v6e，延续 v5e 的 256-chip 2D torus 产品形态，但单芯片能力大幅抬升：1 个 TensorCore、2 个 MXU、918 BF16 TFLOPS、32 GB HBM、1638 GB/s HBM bandwidth、800 GB/s ICI bandwidth[\[14\]](https://docs.cloud.google.com/tpu/docs/v6e)。它仍然适合训练、fine-tuning 和 serving 的混合场景，形态上像 v5e 的代际加强版。

Ironwood 的语气明显变了。Google 官方博客把它称为“first Google TPU for the age of inference”，强调 decode-heavy inference、reasoning 和 MoE workloads[\[15\]](https://blog.google/innovation-and-ai/infrastructure-and-cloud/google-cloud/ironwood-tpu-age-of-inference/)。Cloud TPU7x 文档的表格则给出更产品化的口径：9216 chips per Pod，2 个 TensorCores、4 个 SparseCores，单芯片 2307 BF16 TFLOPS / 4614 FP8 TFLOPS，192 GiB HBM，7380 GB/s HBM bandwidth，1200 GB/s ICI bandwidth，100 Gbps DCN bandwidth[\[16\]](https://docs.cloud.google.com/tpu/docs/tpu7x)。

这里要保留一个表述边界：Ironwood 仍然面向 large-scale AI training and inference，不能写成纯推理芯片。但它把推理压力正式推到架构叙事中心：大模型 reasoning 会让输出 token 路径更长，MoE 会带来专家路由和 all-to-all，长上下文与 agentic workload 会放大 KV working set 对 HBM/VMEM 的压力。单芯片 FLOPS 仍然有用，系统瓶颈却越来越像“数据能否在正确时间抵达正确芯片”。

## 六、TPU 8：训练与推理拓扑正式分道

> **本节论点**：TPU 8t/8i 把 TPU 架构分成两种通信模式：训练侧继续扩大 3D torus 与 DCN fabric，推理侧改用 Boardfly 和 CAE 压低 all-to-all / collective latency。workload pattern 已经强到足以反向决定网络拓扑。 **本节核心参考**：
> 
> - ★★★ [TPU 8t and TPU 8i technical deep dive](https://cloud.google.com/blog/products/compute/tpu-8t-and-tpu-8i-technical-deep-dive)
> - ★★ [Our eighth generation TPUs: two chips for the agentic era](https://blog.google/innovation-and-ai/infrastructure-and-cloud/google-cloud/eighth-generation-tpu-agentic-era/)
> - ★★ [Pathways: Asynchronous Distributed Dataflow for ML](https://arxiv.org/abs/2203.12533)

![[Google TPU 架构演进：从单卡 ASIC 到可调度互联域-03.png]]

*图 2：TPU 8t rack-level connectivity to Virgo fabric。来源：Google Cloud Blog。*

TPU 8 目前主要来自 2026 年 4 月 Google 官方发布和技术 deep dive，仍需按“coming soon / official technical preview narrative”理解，不要当成已稳定多年的 Cloud TPU 版本文档。即便如此，它给出的架构方向已经很清晰：第八代拆成 TPU 8t 和 TPU 8i。8t 面向大规模 pre-training，8i 面向 post-training、serving 和 high-concurrency reasoning[\[17\]](https://cloud.google.com/blog/products/compute/tpu-8t-and-tpu-8i-technical-deep-dive)。

8t 延续 3D torus，单 superpod 9600 chips，加入 native FP4、SparseCore、VPU/MXU overlap 等芯片侧增强；更关键的是 Virgo Network。Google 称 Virgo 使用 high-radix switches、flat two-layer non-blocking topology、多 control domain，相比前代提供 2x scale-up ICI 和最高 4x scale-out DCN bandwidth。单 fabric 可连接超过 134,000 个 TPU 8t chips，non-blocking bisection bandwidth 最高 47 petabits/s，配合 JAX 和 Pathways 可把训练扩展到超过 100 万 TPU chips 的 logical cluster[\[17\]](https://cloud.google.com/blog/products/compute/tpu-8t-and-tpu-8i-technical-deep-dive)。

Pathways 在这里承担的是 runtime 抽象。它把异步分布式 dataflow、gang scheduling、dedicated interconnect transfer 和 single-controller programming model 放在一起，论文报告在 2048 个 TPU 上可以保持接近 100% accelerator utilization，并支持通过 DCN 连接多个 island 的 pipeline[\[18\]](https://arxiv.org/abs/2203.12533)。从 v5e Multislice 到 TPU 8t Virgo，Google 的训练路线越来越像“ICI 域 + DCN fabric + runtime 控制面”的组合，而不是一个孤立 Pod。

![[Google TPU 架构演进：从单卡 ASIC 到可调度互联域-04.png]]

*图 3：TPU 8i Boardfly：4-chip building block、8-board group、36 groups 通过 OCS 组织成 Pod。来源：Google Cloud Blog。*

8i 的选择更有意思。它没有继续把 3D torus 当作默认答案，而是为 MoE / reasoning 的 all-to-all 通信设计 Boardfly。Google deep dive 解释，3D torus 很适合 dense training 里的邻居通信和规整 reduction，但 8x8x16 的 1024-chip torus 最远通信需要 16 hops；Boardfly 用 4-chip building block、8-board fully connected group、36 groups 通过 OCS 连接，把 1024-chip Pod 的最大网络直径降到 7 hops，降幅约 56%[\[17\]](https://cloud.google.com/blog/products/compute/tpu-8t-and-tpu-8i-technical-deep-dive)。

8i 的芯片规格也服务同一目标：288 GB HBM、384 MB on-chip SRAM、Arm Axion host、19.2 Tb/s ICI，以及替代上一代 SparseCore 的 Collectives Acceleration Engine。官方说法是，CAE 可把 on-chip collective latency 降低最多 5 倍；更大的 SRAM 用来容纳 reasoning 模型的 KV cache working set，减少长上下文 decode 时核心空转[\[19\]](https://blog.google/innovation-and-ai/infrastructure-and-cloud/google-cloud/eighth-generation-tpu-agentic-era/)。

至此，TPU 的架构主线已经从“矩阵乘阵列更大”推进到“不同 workload 需要不同互联域”。训练要吞吐、同步和容错，8t 继续扩大 3D torus 与 Virgo scale-out fabric；推理要低尾延迟、MoE 路由和 KV locality，8i 改用 Boardfly、CAE 和大 SRAM。

## 七、代际速查表：每一代的架构重心

> **本节论点**：TPU 的代际差异可以压成三个维度：芯片内算力/内存、Pod 内 ICI 拓扑、Pod 外 DCN/runtime。越到后期，第三个维度越决定系统边界。 **本节核心参考**：
> 
> - ★★ [Cloud TPU v5p documentation](https://docs.cloud.google.com/tpu/docs/v5p)
> - ★★ [TPU7x / Ironwood documentation](https://docs.cloud.google.com/tpu/docs/tpu7x)
> - ★★★ [TPU 8t and TPU 8i technical deep dive](https://cloud.google.com/blog/products/compute/tpu-8t-and-tpu-8i-technical-deep-dive)

| 代际                 | 核心定位                      | 互联 / 集群形态                                          | 架构意义                                                             |
| ------------------ | ------------------------- | -------------------------------------------------- | ---------------------------------------------------------------- |
| TPU v1             | 数据中心在线推理 ASIC             | PCIe 接主机，单卡加速                                      | systolic array + software-managed memory 成为基本路线                  |
| TPU v2             | Cloud TPU / 训练 TPU 起点     | 论文用 16x16 2D torus 解释；Cloud 文档另有 512-chip slice 口径 | BF16、HBM、ICI、XLA 把 TPU 推向训练超算                                    |
| TPU v3             | 更大 2D torus Pod           | 1024 chips，2D torus                                | all-reduce / bisection 成为训练规格的一部分                                |
| TPU v4             | 可重构光互联超算                  | 4096-chip Pod，3D mesh/torus，OCS 支持 twisted torus   | 拓扑选择、碎片治理、故障绕行进入硬件系统设计                                           |
| TPU v5e            | 成本型训练/推理                  | 256-chip 2D torus，Multislice 跨 Pod                 | 用 ICI/DCN 分层把小 ICI 域外推成大训练集群                                     |
| TPU v5p            | 大模型训练 Pod                 | 8960 chips per Pod，3D torus，单 slice 最大 6144 chips  | 同代高性能训练路线，强调 HBM、ICI 与 SparseCore                                |
| TPU v6e / Trillium | 通用训练、fine-tuning、serving  | 256-chip 2D torus                                  | 单芯片 compute、HBM、ICI 比 v5e 大幅增强                                   |
| TPU7x / Ironwood   | 推理优先的大规模 TPU              | 9216 chips per Pod，3D torus                        | 把 decode-heavy inference、MoE、HBM working set 推到叙事中心              |
| TPU 8t             | 大规模 pre-training          | 9600-chip 3D torus + Virgo scale-out fabric        | 训练集群边界从 Pod 推进到 datacenter fabric / logical million-chip cluster |
| TPU 8i             | reasoning / serving / MoE | Boardfly + OCS，1024 active chips 级低直径 Pod          | 推理通信模式开始决定芯片和网络拓扑                                                |

这张表背后的判断是：TPU 的产品边界不断上移。v1 的边界是“主机里的一张加速卡”；v2/v3 的边界是“2D torus Pod”；v4 的边界是“OCS 可重构 ICI domain”；v5e/v5p 的边界变成“ICI + DCN + compiler/runtime”；TPU 8 则把训练和推理拆成两套不同的拓扑假设。

对 AI Infra 读者来说，这比单纯比较 TFLOPS 更有用。训练系统选型要问：TP/CP/EP 是否留在低延迟域，DP/FSDP 是否能稳定跨 DCN，故障和碎片会不会改变可用拓扑。推理系统选型要问：MoE all-to-all、KV working set、collective latency、host 数据准备和尾延迟是否被硬件路径覆盖。TPU 的每一代都在回答这些问题，只是答案越来越不像“芯片参数”，越来越像“系统契约”。

- - -

## 参考资料

\[1\] [In-Datacenter Performance Analysis of a Tensor Processing Unit](https://arxiv.org/abs/1704.04760)

\[2\] [An in-depth look at Google’s first Tensor Processing Unit](https://cloud.google.com/blog/products/ai-machine-learning/an-in-depth-look-at-googles-first-tensor-processing-unit-tpu)

\[3\] [A Domain-Specific Supercomputer for Training Deep Neural Networks](https://gwern.net/doc/ai/scaling/hardware/2020-jouppi.pdf)

\[4\] [Cloud TPU v2 documentation](https://docs.cloud.google.com/tpu/docs/v2)

\[5\] [Cloud TPU v3 documentation](https://docs.cloud.google.com/tpu/docs/v3)

\[6\] [Cloud TPU v4 documentation](https://docs.cloud.google.com/tpu/docs/v4)

\[7\] [TPU v4: An Optically Reconfigurable Supercomputer for ML](https://arxiv.org/abs/2304.01433)

\[8\] [Resiliency at Scale: Managing Google’s TPUv4 ML Supercomputer](https://www.usenix.org/system/files/nsdi24-zu.pdf)

\[9\] [Cloud TPU v5e documentation](https://docs.cloud.google.com/tpu/docs/v5e)

\[10\] [Cloud TPU v5p documentation](https://docs.cloud.google.com/tpu/docs/v5p)

\[11\] [Introducing Cloud TPU v5p and AI Hypercomputer](https://cloud.google.com/blog/products/ai-machine-learning/introducing-cloud-tpu-v5p-and-ai-hypercomputer)

\[12\] [World’s largest distributed LLM training job on TPU v5e](https://cloud.google.com/blog/products/compute/the-worlds-largest-distributed-llm-training-job-on-tpu-v5e)

\[13\] [Using Cloud TPU Multislice to scale AI workloads](https://cloud.google.com/blog/products/compute/using-cloud-tpu-multislice-to-scale-ai-workloads)

\[14\] [Cloud TPU v6e documentation](https://docs.cloud.google.com/tpu/docs/v6e)

\[15\] [Ironwood: The first Google TPU for the age of inference](https://blog.google/innovation-and-ai/infrastructure-and-cloud/google-cloud/ironwood-tpu-age-of-inference/)

\[16\] [TPU7x / Ironwood documentation](https://docs.cloud.google.com/tpu/docs/tpu7x)

\[17\] [Inside the eighth-generation TPU: An architecture deep dive](https://cloud.google.com/blog/products/compute/tpu-8t-and-tpu-8i-technical-deep-dive)

\[18\] [Pathways: Asynchronous Distributed Dataflow for ML](https://arxiv.org/abs/2203.12533)

\[19\] [Our eighth generation TPUs: two chips for the agentic era](https://blog.google/innovation-and-ai/infrastructure-and-cloud/google-cloud/eighth-generation-tpu-agentic-era/)
