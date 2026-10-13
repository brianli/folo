---
title: "Kubernetes 环境下AI 推理多副本负载分担机制深度调研与全景架构分析报告"
source: "微信公众号"
category: "Inbox"
group: "AI基础设施"
author: "罗曙晖"
url: "https://mp.weixin.qq.com/s/lJ0KBguBr8gzkmrkzTMVUA"
published: 2026-10-14T07:36:31+08:00
saved: 2026-10-14T07:36:40+08:00
folo_key: "url::https://mp.weixin.qq.com/s/lJ0KBguBr8gzkmrkzTMVUA"
tags:
  - "folo"
  - "Inbox"
  - "微信公众号"
---

# Kubernetes 环境下AI 推理多副本负载分担机制深度调研与全景架构分析报告

> [!info] 微信公众号 · Inbox · 2026-10-14 07:36 · [原文](https://mp.weixin.qq.com/s/lJ0KBguBr8gzkmrkzTMVUA)

## 报告摘要

本报告深入剖析了在现代 Kubernetes 容器编排环境中，多副本 Pod 支撑大模型及生成式 AI 推理时的\*\*负载分担（Load Balancing / Load Sharing）\*\*机制。  
涵盖从最底层 L4 连接调度、L7 反向代理与网格、客户端直连分担，到开源引擎原生（vLLM、SGLang）、官方扩展（Kubernetes Gateway API for Inference）、解耦架构（Prefill-Decode Disaggregation、Mooncake 分布式 KV-Cache）、MoE 专家并行负载均衡（DeepSeek EPLB）以及多机并行副本组（LeaderWorkerSet）全技术栈。

## 目录导航

1.  vLLM 多副本集群原生路由与负载分担机制 

2.  SGLang 高性能多副本路由与负载分担架构 

3.  Prefill-Decode Disaggregation 异构流水线负载分流 (PD分离)

4.  Kubernetes Gateway API for Inference (K8s 官方推理扩展) 

5.  分布式 KV-Cache 亲和路由与负载防倾斜 (CHWBL 与 llm-d) 

6.  Mooncake KVCache 中心化解耦存储与全局调度 

7.  DeepSeek-V3/R1 专家并行负载均衡 (EPLB 与多副本协同) 

8.  LeaderWorkerSet (LWS) 超大模型跨节点分布式副本组负载分担 

9.  L7 网格与反向代理负载分担 (Envoy / Istio / Nginx) 

10. gRPC 客户端直连负载分担 (Headless Service + 自定义探针) 

- - -

## 1\. vLLM 多副本集群原生路由与负载分担机制

> **架构分类**：主流推理引擎集群分担  
> **核心定位**：剖析 vLLM 在 K8s 多副本部署时的 Data Parallelism 路由、Prefix Cache 亲和分发、引擎内部等待队列与显存水位打分分担算法

### 负载分担调度算法与决策核心

#### 维度：`load_balancing_algorithm`

*维度说明：具体分担算法逻辑（轮询/加权轮询、最小在飞请求 Least In-flight、基于前缀树 Radix Tree 的哈希匹配、一致性哈希与负载边界 CHWBL、双重映射 DualMap 等）*

vLLM 在多副本集群部署中支持多层次的负载分担与路由机制：

1. 官方高性能 Rust 路由器（vllm-router / vLLM Production Stack）：官方推出的独立 Rust 代理层，专为多引擎副本负载均衡设计。支持多种核心调度策略：
      - cache-aware（缓存亲和感知）：通过解析 Prompt 前缀哈希或 Session ID，将具有相同系统提示词（System Prompt）或多轮上下文特征的请求路由至同一副本；
      - power-of-two（P2C，两次随机选择）：随机抽取两个候选后端副本，评估其在飞请求与队列压力，选择负载更小的一个，有效避免惊群效应与哈希偏斜；
      - consistent-hashing（一致性哈希）：基于请求会话 Key 构建哈希环，在副本扩缩容时最小化缓存重新映射带来的未命中震荡；
      - round-robin（轮询）与 random（随机）：基础无状态分流策略。

2. 智能扩展与网关协同算法（如 KubeAI / llm-d / Envoy ext-proc）：引入带负载边界的一致性哈希（CHWBL, Consistent Hashing with Bounded Loads）。设定副本负载上限阈值 C = (1 + ε) \* L\_avg，请求优先根据前缀哈希分配给主亲和副本；若该副本活跃负载超过阈值 C，则溢流分发至哈希环上的后继副本，兼顾缓存命中率与负载均衡。

3. vLLM V1 数据并行（Data Parallelism, DP）运行时路由：在单节点多卡或跨多节点的多副本引擎内部，vLLM V1 引入原生 DP 调度器，将客户端请求透明分派至各个独立的 DP Rank Worker 实例执行。

#### 维度：`load_metrics_monitored`

*维度说明：分担决策所采纳的后端状态指标（活跃请求数 Running Requests、队列等待数 Waiting Requests、GPU 显存水位、KV Cache 块占用率、Token 生成速率）*

分担决策采纳的后端引擎状态指标主要通过 vLLM 原生 Prometheus 监控端点（/metrics）与健康探针获取：

1. 队列与在飞请求状态：
      - vllm:num\_requests\_running：当前正在执行 Continuous Batching 迭代步的模型运行中请求数；
      - vllm:num\_requests\_waiting：处于调度器等待队列（Waiting Queue）中挂起排队的请求数，是评估副本计算饱和度与调度排队延迟（Queue Latency）最直接的核心指标；
      - vllm:num\_requests\_swapped：因 GPU 显存紧张被换出至 CPU 内存的请求数，该指标大于 0 意味着显存严重承载过载。

2. KV Cache 与显存水位指标：
      - vllm:gpu\_cache\_usage\_factor：当前 GPU KV Cache 物理块分配占用率（0.0 至 1.0）；
      - vllm:cpu\_cache\_usage\_factor：CPU 内存中换出 KV Cache 的占用比率；
      - 调度配置参数 gpu\_memory\_utilization（默认 0.90）与保留水位块（watermark\_blocks），作为副本内部阻断准入的硬边界。

3. 吞吐与处理速率指标：
      - vllm:avg\_prompt\_throughput\_tok\_per\_s：Prefill 阶段的 Token 吞吐率；
      - vllm:avg\_generation\_throughput\_tok\_per\_s：Decode 阶段的 Token 生成速率；

4. 探针健康状态：通过 HTTP /health 端点定期探测各 Pod 探针状态，用于快速剔除异常或未完成冷启动的副本。

#### 维度：`dispatch_granularity`

*维度说明：请求分发粒度（TCP 连接级、HTTP/2 请求级、Token 生成 Chunk 级、Prefill/Decode 阶段级）*

vLLM 多副本集群的请求分发粒度呈现分层结构：

1. HTTP/2 与 REST/gRPC 请求级粒度：外部客户端至负载均衡网关/vllm-router 层面，以标准的 OpenAI 兼容 API 请求（/v1/chat/completions、/v1/completions）为基本调度单元。Router 解析请求头、模型名称及 Prompt 特征后进行整体分派。

2. Prefill / Decode 两阶段解耦粒度（PD 分离模式）：在启动 PD 分离流水线架构时，分发粒度细化为阶段级。前端路由器首先将请求的 Prefill 计算调度给 Prefill 副本池；当 Prefill 结束且 KV Cache 传输完成之后，由路由器协调转入 Decode 副本池进行单 Token 自回归迭代生成。

3. Token Chunk 级分发协同：配合 Chunked Prefill 机制，长文本 Prompt 在副本内部被拆解为固定大小（如 512 或 2048）的 Token Chunk 执行，与正在生成的 Decode 请求在每次迭代中协同批处理，但此粒度由引擎内部迭代调度器执行，外部代理不对单个 Token Chunk 做网络二次路由。

4. 流式长连接维持：针对流式输出（SSE/Streaming），路由器在分发初始请求后，与选定的目标副本保持单向长连接通道，逐 Token 持续转发响应直到结束标记（EOS）产生。

### 状态亲和与负载均衡的动态权衡

#### 维度：`cache_affinity_vs_load_balance`

*维度说明：如何平衡‘提高 KV Cache / Prefix Caching 命中率（倾向发给已有缓存的同一 Pod）’与‘防止单 Pod 显存负载倾斜过载（倾向发给空闲 Pod）’*

在多副本集群中平衡 KV Cache 亲和性与防负载倾斜的核心机制包括：

1. 冲突本质剖析：高缓存亲和（倾向将共享相同 System Prompt、多轮历史或 RAG 知识库的请求持续发往同一 Pod）能最大化复用 BlockManager 中的物理 KV 块，大幅降低首字延迟（TTFT）；但若某类特定提示词处于热点流量，会导致该 Pod 显存块耗尽、Waiting 队列剧烈增长，而其余副本空闲，造成严重的显存倾斜与长尾延迟恶化。

2. 负载边界限流与溢流机制（CHWBL / 动态阈值门控）：路由层维护各 Pod 的在飞请求数或实时缓存利用率。当主亲和 Pod 的在飞请求超过设定阈值（例如超出集群均值的 20%）或等待队列非空时，路由器主动触发溢流（Spillover），将后续请求调度给空闲的次选 Pod。

3. 前缀热点主动复制（Prefix Replication）：当高频前缀的流量触发溢流至新 Pod 后，新 Pod 会自发在本地显存构建该前缀的 KV Cache 副本，使集群内部动态形成多个具备该前缀亲和能力的并行服务节点，实现缓存热度与负载容量的自动动态均衡。

4. LRU 缓存协同感知：vLLM 内部通过 Prefix Caching 跟踪逻辑块哈希与物理块映射。路由器配合一致性哈希，降低跨 Pod 随意分配导致的频繁全量重新 Prefill，显著削减总体计算浪费。

#### 维度：`multi_lora_affinity_dispatch`

*维度说明：针对挂载多个微调 LoRA 的多副本，如何感知各 Pod 当前已驻留的 Adapter 权重并做亲和性负载分担，减少显存置换开销*

针对多副本挂载多微调 LoRA Adapter 的场景，vLLM 采用以下协同分担机制：

1. 引擎内部适配器管理机制：vLLM 通过 LRUCacheLoRAModelRunner 动态加载 LoRA 权重，在 GPU 显存中预留槽位（受 --max-loras 和 --max-lora-rank 限制），并在 CPU 内存中维护更大的 LRU 缓存池（--max-cpu-loras）。

2. 盲目轮询导致的显存抖动（Thrashing）：若前端代理无法感知各副本驻留的 LoRA 权重，请求随机分发会导致各 Pod 频繁在 GPU 显存与 CPU 内存之间换入换出 LoRA 权重，产生严重的权重加载阻塞与延迟抖动。

3. LoRA 亲和路由分担：网关路由器解析请求载荷中的 model 字段（指定基础模型与具体 LoRA 变体），通过 Adapter 亲和策略（Adapter Affinity Routing）将相同 LoRA 的请求定向分发到已加载该权重的特定 Pod 子集。仅当该子集高负载或队列积压时，才溢流至备用副本进行冷加载，从而最大化 LoRA 显存命中率并降低冷换开销。

### 解耦架构与流水线负载分流

#### 维度：`pd_disaggregated_load_sharing`

*维度说明：Prefill 阶段与 Decode 阶段的跨 Pod 流水线解耦分担机制，请求流转路径与跨节点 KV-Cache 传输网络协同（RDMA/RoCE/NCCL）*

vLLM 原生支持 Prefill 与 Decode 阶段解耦（Prefill-Decode Disaggregation）的流水线负载分担：

1. 架构拓扑分工：将集群 Pod 划分为 Prefill 副本池与 Decode 副本池。Prefill Pod 配置高张量并行与高算力 GPU，专注于高并发 Prompt 矩阵乘法计算；Decode Pod 配置适合大显存带宽的卡型与低张量并行，专注于自回归单 Token 生成，彻底解决长 Prompt 预填充打断 Decode 迭代引起的延迟尖刺（Inter-Token Latency, ITL）。

2. 请求调度与流转路径：请求首先到达 vllm-router，路由器将其转发给 Prefill Pod 执行前向预填充；Prefill 节点将生成的 KV 状态直接通过高速互连网络流转写入 Decode Pod 的显存；随后路由器将请求的自回归生成阶段交给 Decode Pod 继续执行，最终通过流式通道返回客户端。

3. 跨节点 KV Cache 传输网络协同：
      - NixlConnector：基于 NVIDIA Inference Transfer Library（NIXL）实现的端到端传输连接器，底层依托 UCX（Unified Communication X）协议栈，在 InfiniBand 或 RoCEv2 网络上利用 GPUDirect RDMA 实现跨主机显存零拷贝传输；
      - MooncakeConnector：整合 Mooncake 高性能传输引擎，实现高带宽低时延的跨卡跨节点 KV Cache 块流动；
      - LMCacheConnector：配合分布式分层存储机制，在跨节点网络传输与远程分布式缓存间实现 KV 块的高效搬运与复用。

#### 维度：`chunked_prefill_concurrency`

*维度说明：变长长文本 Prompt 的分块预填充（Chunked Prefill）对多副本并发负载平滑分担的支撑效果*

分块预填充（Chunked Prefill）在 vLLM 多副本集群中对负载平滑分担起到关键支撑作用：

1. 核心运行逻辑：通过参数 --enable-chunked-prefill（vLLM V1 默认开启）配合 --max-num-batched-tokens（如 512、2048），当收到超长上下文 Prompt（如数万 Token）时，调度器将其切分为多个 Chunk，在每次模型迭代步中与 Decode 请求共同组 Batch 执行。

2. 消除气泡与平滑单 Pod 负载：传统非分块模式下，巨型 Prompt 的 Prefill 会独占 GPU 数百毫秒甚至数秒，导致同 Pod 内其它 Decode 请求严重饥饿挂起。Chunked Prefill 消除调度断层，将巨型计算均摊到多个迭代步中。

3. 提升集群全局负载评估精度：由于每个 Pod 的单次迭代时间维持在稳定的毫秒级微区间内，网关路由器观测到的 Pod 队列等待耗时与吞吐指标更加连续、稳定，避免因偶发长 Prompt 阻塞而导致监控指标出现假死或剧烈震荡，从而使负载均衡器的分发决策更加敏捷和准确。

### 过载保护、反压与容灾伸缩

#### 维度：`overload_protection_and_admission`

*维度说明：单 Pod 即将显存击穿（OOM）或队列满时的准入控制（Admission Control）、排队缓冲与溢流分流（Spillover）机制*

vLLM 多副本集群在显存击穿与过载场景下构筑了多层准入控制与保护屏障：

1. 引擎内部准入控制与水位硬约束：调度器通过 max\_num\_seqs（最大并发序列数）、max\_num\_batched\_tokens 以及显存水位系数 gpu\_memory\_utilization 进行准入裁决。在每次调度迭代前评估剩余空闲物理块数量，若不足以支撑后续分配，新请求被截留在 Waiting Queue 中排队。

2. 显存击穿与驱逐抢占机制（Preemption）：若运行中的 Decode 请求持续增长导致 GPU KV 块耗尽，调度器执行请求抢占（Preemption）：
      - recompute 模式：丢弃被抢占请求的已分配显存块，将请求回退至 Waiting 队列首部，待显存充裕时重新执行 Prefill 计算；
      - swap 模式：将部分运行中请求的 KV Cache 块通过 PCIe 换出到 CPU 系统内存（Swapped Queue），待 GPU 显存回落后再换回继续生成。

3. 队列饱和与主动反压：当引擎内部排队请求数超过阈值或达到超时限制时，vLLM 直接返回 HTTP 429（Too Many Requests）或 HTTP 503（Service Unavailable）状态码，向调用方提供明确反压信号。

4. 网关层断路器与溢流（Circuit Breaking & Spillover）：vllm-router 或外部智能代理实时统计各 Pod 错误率与排队深度。当某 Pod 出现显存过载或频繁返回 429/503 时，代理立即触发断路器降级，将后续请求重定向至备用或轻载 Pod，实现集群流量自适应泄洪。

#### 维度：`k8s_autoscaling_and_rebalancing`

*维度说明：与 Kubernetes HPA/KEDA 水平伸缩的联动，扩容新增 Pod 时的冷启动预热，以及缩容/故障时的流量优雅下线（Drain）与平滑再均衡*

在 Kubernetes 生产环境中，vLLM 与弹性伸缩和生命周期管理的协同包括：

1. 基于 KEDA / Prometheus 的自定义指标 HPA 联动：传统基于 CPU/Memory 利用率的 HPA 对大模型推理失效。推荐通过 KEDA ScaledObject 监听 vLLM 核心业务指标：
      - vllm:num\_requests\_waiting 队列等待数（例如平均每 Pod 等待数 > 2 触发扩容）；
      - vllm:gpu\_cache\_usage\_factor 显存利用率（例如均值 > 0.85 触发扩容）。

2. 冷启动预热与流量挂载控制：大模型权重加载耗时长达数分钟，且初始化时需进行 CUDA Graph 预编译。必须配置严格的 K8s StartupProbe 和 ReadinessProbe 指向 /health 端点，确保模型权重加载和空跑预热完成后才将 Pod 挂入 Endpoints，防止冷启动期间流量注入引发请求失败。

3. 权重加速与存储方案：利用高带宽共享存储（如 Ceph、JuiceFS、NFS）或 K8s Image Volume 特性预挂载权重，缩短扩容冷启动时间。

4. 缩容优雅下线（Drain）与平滑再均衡：
      - Pod 接收到 K8s 发出的 SIGTERM 信号后，立即向探针标记为 Unready，Service/Router 将其从转发列表中摘除，不再分配新请求；
      - Pod 在 terminationGracePeriodSeconds（推荐设置为 60-180 秒）窗口期内继续处理正在执行的存量流式生成任务，直至存量在飞请求全部正常关闭后安全退出，保障缩容期间零业务中断。

#### 维度：`streaming_failover_resilience`

*维度说明：长连接流式输出（SSE/gRPC Streaming）长达数十秒过程中，Pod 崩溃或被抢占时的容灾与重试机制*

针对 SSE（Server-Sent Events）或 gRPC 流式长连接数十秒交互中的容灾能力：

1. 状态绑定与透明故障转移限制：大模型推理的自回归解码是强状态的，中间 KV Cache 驻留在特定 Pod 的 GPU 显存内。因此，若正在生成 Token 的 Pod 发生崩溃、OOM 终止或被 Spot 实例抢占，现有四层/七层代理无法做到物理显存级别的即时无感知透明热迁移。

2. 网关与客户端协同重试：网关检测到底层 Pod 连接断开后，向客户端抛出可重试错误；客户端或外层智能 Agent 发起重试，并带上原始 Prompt（可选拼接已接收的部分内容）。

3. 配合分布式缓存与 Prefix Cache 快速恢复：在配置了分布式 KV 缓存共享（如 LMCache）或统一路由的环境下，重试请求被分发给新副本时，新副本可直接命中已持久化或共享的 Prompt 前缀 KV Cache，极大缩短二次 Prefill 耗时，实现近乎即时的流式恢复。

4. 优雅终止期规避风险：在 K8s 计划内维护和缩容场景中，通过足够长的优雅终止窗口（Graceful Drain），允许绝大部分长连接流式请求完整走完 EOS 流程，将崩溃断连概率降至最低。

### 性能开销与落地可行性

#### 维度：`routing_overhead_and_latency`

*维度说明：分担决策自身引入的额外网络跳数、代理处理耗时（TTFT 影响）与 CPU/内存额外开销*

vLLM 原生路由体系引入的性能开销与延迟分析：

1. 代理网络跳数与首字延迟（TTFT）影响：引入独立部署的 vllm-router 增加了一跳集群内网络转发（典型 Intra-cluster RTT 约为 0.2ms - 0.5ms）。相较大模型首字时间（TTFT，通常在数十毫秒至数百毫秒），该网络耗时占比极小（小于 1%）。

2. 代理自身的处理耗时：基于 Rust 编写的高性能 vllm-router 采用异步事件驱动架构（Tokio/Hyper），内部逻辑以纳秒/微秒级执行，单请求路由决策耗时通常小于 0.8ms（P99），对端到端吞吐与首字时延无显著阻碍。

3. 计算开销与内存占用：vllm-router 内存开销极低（常驻仅需数十 MB 至数百 MB），单核 CPU 即可处理上万 QPS 的路由转发与指标聚合。

4. 基于 Envoy ext-proc 或重型网关对比：若采用外部 Envoy ext-proc 方案，因需要跨进程 gRPC 调用并解析 Token 字段，可能引入 1ms - 3ms 额外开销；而 vllm-router 属于专用轻量流式代理，性能表现更为卓越。

#### 维度：`production_readiness_and_cases`

*维度说明：企业级生产落地成熟度、典型标杆案例与主流开源生态配套*

vLLM 多副本集群架构在企业级生产环境中的落地成熟度：

1. 生态成熟度与官方标准化：
      - vLLM Production Stack：官方基于 Helm 与 Kubernetes 维护的生产级全栈方案，打包集成了 vllm-router、Grafana 仪表盘与 Prometheus 监控，支持一键部署多副本集群与 PD 分离集群；
      - LeaderWorkerSet（LWS）：与 Kubernetes SIG 官方联合支持，针对跨节点 TP/PP 分布式大模型副本提供原生多 Pod 协同生命周期与入口调度；
      - KubeAI 与 KubeRay：云原生主流 AI 调度平台已深度集成 vLLM 作为核心推理运行时。

2. 行业标杆与生产落地案例：
      - Anyscale、Mistral AI、Baseten、DeepInfra 等全球顶尖大模型托管平台大规模采用 vLLM 作为主力推理引擎；
      - 字节跳动、腾讯、阿里巴巴等国内头部云厂商与互联网大厂在公有云及内部大模型底座中广泛基于 Kubernetes + vLLM 架构构建大规模弹性推理服务，支撑亿级日常推理调用。

- - -

## 2\. SGLang 高性能多副本路由与负载分担架构

> **架构分类**：主流推理引擎集群分担  
> **核心定位**：深入 SGLang Router 基于 RadixAttention 的前缀树缓存命中调度、多 Worker 实例负载均衡打分与 Multi-LoRA 亲和分流

### 负载分担调度算法与决策核心

#### 维度：`load_balancing_algorithm`

*维度说明：具体分担算法逻辑（轮询/加权轮询、最小在飞请求 Least In-flight、基于前缀树 Radix Tree 的哈希匹配、一致性哈希与负载边界 CHWBL、双重映射 DualMap 等）*

SGLang 具备业界领先的软硬件协同分流架构，其核心是独立的 Rust 高性能路由器（sgl-router，亦称 SGLang Model Gateway / SMG）：

1. cache\_aware（前缀树缓存感知调度算法，默认推荐策略）：
      - 路由器内存中为每个后端 Worker 维护一颗近似前缀树（Approximate Radix Tree，即 approx\_tree），实时镜像后端 Worker 上 RadixAttention 的 KV Cache 树结构与已缓存 Token 序列；
      - 请求到达时，路由器将 Prompt Token 与各 Worker 的近似前缀树进行最长公共前缀匹配（Prefix Matching），计算各 Worker 的预期前缀命中率（Cache Overlap Ratio）；
      - 调度打分与决策分支：
        a. 缓存亲和判定：若匹配 Token 比例高于缓存阈值 cache\_threshold（默认 0.5），优先将请求调度至前缀树匹配长度最大的 Worker；
        b. 负载偏斜门控防线（Anti-Skew Load Gate）：同时监控各 Worker 的活跃在飞请求数。若最高负载与最低负载之差超过绝对阈值 balance\_abs\_threshold（默认 32），且负载比值超过相对阈值 balance\_rel\_threshold（默认 1.0001 至 1.1），则负载均衡门控生效，强制覆盖缓存亲和，将请求分派至在飞队列最短的 Worker（Shortest Queue）；
        c. 无命中时的冷分布：若前缀匹配比例低于阈值，则将请求分配至当前前缀树尺寸最小或负载最低的 Worker，以平衡全局显存空间占用。

2. 其它通用路由策略：
      - power\_of\_two（P2C）：随机采样两个 Worker，选择其中在飞请求数较小者，具备极低仲裁开销；
      - round\_robin 与 random：标准轮询与随机策略，用于基准对照或纯无状态短文本推理。

#### 维度：`load_metrics_monitored`

*维度说明：分担决策所采纳的后端状态指标（活跃请求数 Running Requests、队列等待数 Waiting Requests、GPU 显存水位、KV Cache 块占用率、Token 生成速率）*

SGLang 路由器与后端 Worker 之间通过高频心跳、原子计数以及 Prometheus 端点（/metrics）监控全方位状态指标：

1. 调度队列与在飞负载：
      - 路由器内部原子计数器：在代理层实时原子递增/递减各 Worker 的活跃在飞请求数（Active In-flight Requests），实现零延迟微秒级负载追踪；
      - sglang:num\_running\_reqs（新版为 sglang\_num\_running\_reqs）：Worker 当前处于 Batch 计算中的并发请求数；
      - sglang:num\_queue\_reqs：Worker 内部等待调度队列中的排队请求数。

2. KV Cache 与显存空间指标：
      - sglang:token\_usage：当前 Worker 的 KV Cache 显存块使用比例（0.0 至 1.0），当超过 0.9 时表明显存饱和；
      - sglang:cache\_hit\_rate：基于 RadixAttention 的实测前缀缓存命中率；
      - approx\_tree\_size：路由器近似前缀树中的节点总量与估算 Token 容量，用于衡量 Worker 缓存池膨胀程度。

3. 服务延时与吞吐指标：
      - sglang:gen\_throughput：各 Worker 的实时每秒 Token 生成速率；
      - sglang:e2e\_request\_latency\_seconds：端到端请求时延与首字延迟（TTFT）分布直方图。

4. 健康探针状态：定期探活 Worker 的 /health 端点，实时捕获节点离线或内核挂起事件。

#### 维度：`dispatch_granularity`

*维度说明：请求分发粒度（TCP 连接级、HTTP/2 请求级、Token 生成 Chunk 级、Prefill/Decode 阶段级）*

SGLang 架构在请求全链路中实现了细粒度分流与控制：

1. 请求与 Token 序列匹配级粒度：客户端发送标准的 HTTP/REST 或 gRPC 推理请求到达 sgl-router。路由器解析 Prompt 并进行分词匹配，以整个请求的 Prompt Token 序列作为 Radix Tree 路由查找粒度，决策分发至特定 Worker。

2. Prefill / Decode 解耦阶段级粒度（PD 分离模式）：在启用 Prefill-Decode 分离架构时，请求生命周期被拆解为两个执行阶段：
      - Prefill 阶段由路由器派发给高算力 Prefill Worker 实例集群执行；
      - 预填充完成并经由 RDMA 网络传输 KV Cache 后，后续的自回归解码阶段转由 Decode Worker 实例集群执行。

3. Chunked Prefill 迭代级粒度：超长 Prompt 在 Worker 内部被切分为 Token Chunk，与正在运行的 Decode 序列在每轮迭代步中动态拼批，由 Worker 调度器以微批次（Micro-batch）粒度流转调度。

4. 流式通道粒度：对于 Streaming 请求，sgl-router 建立下行 SSE 长连接，并与选定的 Worker 维持流式传输隧道，透明透传逐 Token 生成事件。

### 状态亲和与负载均衡的动态权衡

#### 维度：`cache_affinity_vs_load_balance`

*维度说明：如何平衡‘提高 KV Cache / Prefix Caching 命中率（倾向发给已有缓存的同一 Pod）’与‘防止单 Pod 显存负载倾斜过载（倾向发给空闲 Pod）’*

SGLang 通过深度的 RadixAttention 亲和性与动态负载门控算法，精巧化解‘缓存亲和’与‘显存负载倾斜’的根本矛盾：

1. 矛盾点：在多轮对话、长文档 Agent 交互或代码补全场景中，大量请求带有相同的前缀。若机械地将请求全部发给命中该前缀的同一个 Worker，该 Worker 的显存利用率（token\_usage）与排队数（num\_queue\_reqs）将迅速被打满并触发 OOM 或排队剧增，而其它副本却在空跑。

2. 双重门控防御（Dual-Threshold Gating）：
      - 缓存准入门槛（cache\_threshold）：只有当前缀重合度足够高（默认超过 50%）才激发缓存亲和定向，防止因微小的公共前缀（如仅几个无意义系统 Token）导致流量盲目倾斜；
      - 负载抗倾斜硬门槛（balance\_abs\_threshold & balance\_rel\_threshold）：当目标 Worker 的活跃在飞请求数比最空闲 Worker 多出 32 个以上，且负载比值超过 1.1 时，路由器判定该节点处于过载临界态，主动打破缓存亲和，强制将请求引流至轻载 Worker。

3. 缓存自适应多活复制：被引流到轻载 Worker 的请求在执行 Prefill 后，会在该 Worker 的 RadixAttention 树中新建一份相同前缀的缓存分支。随后，路由器中的两棵 approx\_tree 均记录了该前缀，该热点前缀从单副本独占平滑转变为双副本/多副本并发分担，实现自适应动态负载均衡。

4. 近似树 LRU 协同驱逐：当 Worker 显存承压触发 Radix 树的 LRU 叶子节点淘汰时，路由器同步裁剪对应 Worker 的 approx\_tree 节点，确保路由决策始终基于有效的缓存分布状态。

#### 维度：`multi_lora_affinity_dispatch`

*维度说明：针对挂载多个微调 LoRA 的多副本，如何感知各 Pod 当前已驻留的 Adapter 权重并做亲和性负载分担，减少显存置换开销*

在挂载多个 LoRA 微调权重的生产多副本场景下，SGLang 的亲和分流与管理机制表现为：

1. 引擎底层 SGMV 核心支持：SGLang Worker 内部通过高效的 SGMV（Segmented Gather Matrix-Vector Multiplication）Triton 算子，原生支持在一个 Batch 内并发混合执行不同的 LoRA Adapter，显著降低了多 LoRA 共享基础模型的计算开销。

2. 权重动态换入换出与驱逐控制：Worker 显存中保存活跃 LoRA 槽位，其余权重缓存在 CPU Host 内存中。引擎提供 --lora\_eviction\_policy（支持 lru 与 fifo）管理适配器显存淘汰，并通过 lora\_drain\_wait\_threshold 防止特定 LoRA 任务长时间挂起或饿死。

3. 路由器层 LoRA 亲和分发：在 SGLang Model Gateway（SMG）层，路由器解析请求中的 LoRA 标识（如 model 参数或请求头中的 Adapter ID）。优先将带有特定 LoRA 的请求分派给已经驻留该 LoRA 显存权重的 Worker 节点，大幅降低从 CPU/磁盘反复搬运 LoRA 权重的开销（Adapter Thrashing），实现吞吐量与首字时延的显著提升。

### 解耦架构与流水线负载分流

#### 维度：`pd_disaggregated_load_sharing`

*维度说明：Prefill 阶段与 Decode 阶段的跨 Pod 流水线解耦分担机制，请求流转路径与跨节点 KV-Cache 传输网络协同（RDMA/RoCE/NCCL）*

SGLang 具备卓越的 Prefill-Decode 解耦（PD 分离）多副本流水线架构：

1. 角色解耦与资源隔离：将多副本 Pod 划分为 Prefill 节点组与 Decode 节点组。Prefill 节点配置高张量并行与算力密集型卡型，Decode 节点配置高显存带宽与低并行度卡型，彻底避免长 Prompt 抢占 GPU 计算资源导致正在生成的会话产生严重的 Token 抖动（ITL）。

2. 协同分流路径：客户端请求发送至 sgl-router，路由器首先将请求路由给最优的 Prefill Worker 执行前向 Prompt 计算；预填充完成后，生成的完整 KV Cache 块被高效推送至目标 Decode Worker；Decode Worker 接管请求开始自回归迭代，并将 Token 流式透传给路由器返回客户端。

3. 跨节点 KV Cache 传输网络协同（Mooncake 深度集成）：
      - Mooncake Transfer Engine：SGLang 原生集成月之暗面（Moonshot AI）开源的 Mooncake 传输引擎，利用 RDMA（InfiniBand / RoCEv2）实现跨节点显存级零拷贝直传，传输吞吐可达网络物理线速（100Gbps - 400Gbps）；
      - 绕过计算单元（SM-free Transfer）：在数据搬运期间完全不消耗 GPU 流式多处理器（SM）算力，杜绝网络传输对正在进行的推理计算产生微架构层面的干扰；
      - 分布式 KV 内存池化：Mooncake 统一汇聚各节点的 DRAM 与 SSD 资源，形成全局去中心化分层 KV 缓存池，使 Decode 节点及后续请求能够直接复用全局前缀，无需二次计算。

#### 维度：`chunked_prefill_concurrency`

*维度说明：变长长文本 Prompt 的分块预填充（Chunked Prefill）对多副本并发负载平滑分担的支撑效果*

SGLang 通过分块预填充（Chunked Prefill）技术为多副本并发负载平滑提供强力保障：

1. 参数与算法实现：通过配置 --chunked-prefill-size（如 512、1024、2048 等尺寸），当输入超长 Prompt（如 32k、128k Token）时，SGLang 拒绝单次全量占用计算内核，而是将 Prompt 离散切分为固定长度的 Chunk 分步执行。

2. 消除调度断层与计算长尾：在传统的非分块并发中，超长文本的 Prefill 具有极高的独占性，会导致 Worker 在长达数秒的时间内无法处理任何新请求或生成 Decode Token。Chunked Prefill 使 Prefill 计算细粒度化，让 Decode 请求得以穿插交替运行，大幅压低 P99 ITL 延迟。

3. 多副本负载均衡的平滑效应：Chunked Prefill 将算力消耗拉平为均匀的时间切片，使得 Worker 对外暴露的队列长度与处理延迟保持线性和可预测。sgl-router 能够精准捕获 Worker 的即时处理能力，避免将请求误发至看似在飞数低但正被超长 Prefill 锁死算力的“假空闲”节点。

### 过载保护、反压与容灾伸缩

#### 维度：`overload_protection_and_admission`

*维度说明：单 Pod 即将显存击穿（OOM）或队列满时的准入控制（Admission Control）、排队缓冲与溢流分流（Spillover）机制*

SGLang 多副本系统设计了从网关代理到底层引擎的完备准入控制与过载保护：

1. Worker 层准入与显存保护约束：各 Worker 通过 --max-running-requests 控制最大并发执行数，通过 --mem-fraction-static（默认 0.88-0.90）锁定模型权重与静态显存，剩余空间全部划入 RadixAttention KV 动态池。若请求所需上下文超出当前剩余可用物理块，调度器自动进行排队缓冲。

2. 优雅反压与错误抛出：当内部排队队列超出容量上限或超时阈值时，Worker 触发主动限流机制，向客户端返回标准 HTTP 429 或 503 错误，向网关暴露清晰的反压信号。

3. 网关级断路器与自适应流量溢流：sgl-router 实时跟踪各 Worker 的健康状况与错误率指标。当检测到某个 Worker 显存濒临击穿、连续响应超时或返回 429 报错时，网关熔断机制（Circuit Breaker）立即介入，将该节点权重清零并暂停分发新流量，将积压流量安全溢流（Spillover）至其余健康 Worker，待故障节点显存恢复并成功响应探活探测后自动平滑复位挂载。

#### 维度：`k8s_autoscaling_and_rebalancing`

*维度说明：与 Kubernetes HPA/KEDA 水平伸缩的联动，扩容新增 Pod 时的冷启动预热，以及缩容/故障时的流量优雅下线（Drain）与平滑再均衡*

在云原生 Kubernetes 编排体系下，SGLang 的弹性伸缩与生命周期协同机制包括：

1. 基于 KEDA / Prometheus 的智能化 HPA 水平伸缩：由于大模型服务对 CPU 使用率不敏感，生产环境通常采用 KEDA ScaledObject 监听 SGLang 原生 Prometheus 监控指标：
      - sglang:num\_queue\_reqs（等待队列请求数）：例如设置平均队列积压超过 3-5 时立即触发 Pod 扩容；
      - sglang:token\_usage（KV Cache 水位）：例如均值超过 0.85 时提前扩容，防范突发流量导致显存溢出。

2. 容器冷启动预热治理：大模型加载涉及数十 GB 权重的磁盘读取与 Triton 算子 JIT 编译。SGLang 配合 K8s StartupProbe 和 ReadinessProbe 定期轮询 /health。只有当权重完全载入 GPU、CUDA 初始化及空跑测试全部成功后，探针才返回 HTTP 200，K8s 随之将 Pod 注册入 Endpoints 并让 sgl-router 纳入可用 Worker 列表，彻底杜绝冷启动未就绪即接流导致的请求溃败。

3. 缩容优雅下线（Graceful Drain）：当 K8s 缩容或节点滚动升级发出 SIGTERM 信号时，Pod 立即切断 Readiness 探针。sgl-router 毫秒级从后端池中摘除该 Worker 实例，阻断新流量流入；同时给 Pod 留足 terminationGracePeriodSeconds（如 90-180 秒），确保正在进行的流式生成完整输出完毕后安全退出，做到零断流平滑再均衡。

#### 维度：`streaming_failover_resilience`

*维度说明：长连接流式输出（SSE/gRPC Streaming）长达数十秒过程中，Pod 崩溃或被抢占时的容灾与重试机制*

针对数十秒乃至上百秒的长连接流式输出（SSE/gRPC Streaming）在 Pod 故障时的容灾与容错方案：

1. 状态持久性局限：大模型自回归解码高度依赖中间状态（当前生成位置的完整 KV Cache）。一旦承载该会话的 Pod 遭遇硬件故障、OOM-Killer 强制杀进程或底层虚拟机被云厂商抢占中断，当前物理显存内的 KV 数据瞬间丢失，四层与传统七层代理无法实现毫秒级透明跨节点上下文恢复。

2. 网关探测与可恢复重试协同：sgl-router 具备流式连接中断检测能力。当探测到后端 Worker 的 TCP 连接异常中断时，网关及时向调用端反馈规范的断连错误信息。

3. 基于 RadixAttention 与分布式存储的秒级回血：若客户端或智能体网关发起请求重试，由于 SGLang 原生基于 RadixAttention，若集群配置了 Mooncake 分布式 KV Cache 存储池或其它副本已持有该会话的前缀块，新接盘的 Worker 可以直接命中此前生成的全部 Prompt 与上下文 KV 缓存，几乎以零 Prefill 开销直接继续自回归生成，将重试引发的延迟惩罚降到最低极限。

### 性能开销与落地可行性

#### 维度：`routing_overhead_and_latency`

*维度说明：分担决策自身引入的额外网络跳数、代理处理耗时（TTFT 影响）与 CPU/内存额外开销*

SGLang 高性能路由架构在性能损耗与延迟方面的工程表现：

1. 极低路由决策耗时（Sub-millisecond Routing）：sgl-router 采用纯 Rust 语言编写，底层基于 Tokio 异步网络框架与轻量级 HTTP 引擎开发。其内部近似前缀树查找基于紧凑优化的 Radix Trie 数据结构，单次最长公共前缀匹配与决策打分耗时通常在 10 至 50 微秒（Microseconds）级别。

2. 网络跳数与首字延迟（TTFT）影响：引入独立部署的 sgl-router 仅增加一跳局域网传输延迟（在典型 K8s 容器网络 CNI 或 VPC 网络下为 0.1ms 至 0.3ms）。相较大模型端到端数百毫秒的 TTFT，此网络开销完全在可忽略的测量误差范围之内（占比 < 0.2%）。

3. 内存与 CPU 开销控制：sgl-router 内存占用极度紧凑（即使镜像数千节点的树节点，常驻内存通常不足 100MB），单实例在轻量级 CPU 资源配置下即可承载数万并发连接与高吞吐请求分流。

#### 维度：`production_readiness_and_cases`

*维度说明：企业级生产落地成熟度、典型标杆案例与主流开源生态配套*

SGLang 高性能多副本与路由架构在企业级生产环境中的落地成熟度：

1. 头部基础设施标杆采用：
      - LMSYS 官方生产环境：全球知名的大模型竞技场（Chatbot Arena）大规模部署 SGLang，承载全球海量匿名用户的超高并发、多样化多轮对战评测流量，长期经受极端真实复杂流量洗礼；
      - 月之暗面（Moonshot AI / Kimi）：深度结合 Mooncake 与 SGLang 构建长文本万亿 Token 级别的大规模跨节点分离式推理集群；
      - 深度求索（DeepSeek）、零一万物（01.AI）、阿里巴巴等国内顶级大模型团队及开源社区在私有化集群中深度落地 SGLang 支撑生产推理；
      - Together AI、RunPod 等海外算力云与推理 API 平台将 SGLang 作为官方顶级推荐推理引擎。

2. 社区与云原生生态配套：
      - 深度兼容 OpenAI API 规范，支持无缝替换旧版推理栈；
      - 联合开源社区推出 Open Model Engine（OME）与 Kubernetes 原生调度编排方案，具备生产级 Helm Chart、Grafana 监控看板及自动化故障转移体系。

- - -

## 3\. Prefill-Decode Disaggregation 异构流水线负载分流 (PD分离)

> **架构分类**：两阶段解耦分担架构  
> **核心定位**：以 vLLM/SGLang/Mooncake 为代表，将多副本拆分为 Prefill Pod 池与 Decode Pod 池，前端 Router 进行两阶段协同与跨卡/跨机 KV-Cache 传输分担

### 负载分担调度算法与决策核心

#### 维度：`load_balancing_algorithm`

*维度说明：具体分担算法逻辑（轮询/加权轮询、最小在飞请求 Least In-flight、基于前缀树 Radix Tree 的哈希匹配、一致性哈希与负载边界 CHWBL、双重映射 DualMap 等）*

采用前置智能协调器（如 sgl-router、Mooncake Global Scheduler 或 vLLM NIXL/Proxy）进行两阶段流水线解耦调度。

1. Prefill 阶段调度：采用前缀缓存感知与排队深度混合算法（Cache-Aware + Shortest Queue Hybrid Scheduling）。基于 Radix Tree 或前缀哈希匹配（RadixAttention）计算各 Prefill Pod 的前缀缓存重用块数，并结合活跃在飞请求数（Least In-flight）进行加权打分（如 Score = α \* Cache\_Hit\_Ratio - β \* Queue\_Depth），在优先复用已有 KV Cache 的同时避免单节点过载。

2. Decode 阶段调度：采用基于显存块可用余量与在飞并发序列数的加权最小负载分流算法（Weighted Least Load）。监控各 Decode Pod 预估显存余量（KV Cache Block Availability）与生成步进，将完成 Prefill 的请求派发给吞吐最高且显存充裕的 Decode 节点，确保自回归 Token 生成的高吞吐与低抖动。

3. 分桶自适应路由（Bucket-based Routing）：按 Prompt Token 长度对请求分类分流，短文本请求可直连混部 Pod，长文本请求则定向分流至高性能 PD 专用异构集群。

#### 维度：`load_metrics_monitored`

*维度说明：分担决策所采纳的后端状态指标（活跃请求数 Running Requests、队列等待数 Waiting Requests、GPU 显存水位、KV Cache 块占用率、Token 生成速率）*

分担决策采纳分阶段的多维度运行时指标：

1. Prefill Pod 状态指标：当前排队及在飞预填充 Token 总量（Waiting & Running Prefill Tokens）、GPU 计算核心（Tensor Core）算力饱和度、已构建前缀树的深度与局部命中率、实时吞吐率。

2. Decode Pod 状态指标：GPU 显存 KV Cache 块占用率（KV Block Allocation Rate）、当前活跃解码请求数（Running Decode Requests / In-flight Sequences）、未来 Token 显存预分配余量、单 Token 生成步进延迟（Inter-Token Latency, ITL）。

3. 跨节点传输网络指标：RDMA/RoCE 网卡队列对（Queue Pairs）在飞字节数、跨机传输延迟与带宽占用率、GPU 接收侧暂存缓冲区（Staging Buffer）水位。

#### 维度：`dispatch_granularity`

*维度说明：请求分发粒度（TCP 连接级、HTTP/2 请求级、Token 生成 Chunk 级、Prefill/Decode 阶段级）*

分发粒度为两阶段流水线级（Prefill/Decode 阶段级）与 Chunk 块级混合：

1. 入口接入级：客户端向前端代理发起 HTTP/2 或 gRPC 会话请求。

2. 流水线阶段级：调度器在请求生命周期内将其拆解为独立执行的‘预填充任务’与‘解码任务’，分别定向到不同的计算拓扑池。

3. Chunk 块级分发：在长 Prompt 场景下，支持分块预填充（Chunked Prefill），每计算完一段 Chunk 即可向跨卡/跨机通道异步传输，与后续 Decode 计算流水线形成重叠。

4. Token 输出级：Decode Pod 生成的每个自回归 Token 经长连接（SSE/gRPC Streaming）原路透传给前端 Router 并实时推流至客户端。

### 状态亲和与负载均衡的动态权衡

#### 维度：`cache_affinity_vs_load_balance`

*维度说明：如何平衡‘提高 KV Cache / Prefix Caching 命中率（倾向发给已有缓存的同一 Pod）’与‘防止单 Pod 显存负载倾斜过载（倾向发给空闲 Pod）’*

通过动态软亲和阈值与分布式解耦存储平衡缓存命中与负载倾斜：

1. 动态软亲和与边界约束（Consistent Hashing with Bounded Loads / Soft Affinity）：调度器设定负载倾斜上限阈值（如单 Pod 负载不得超过集群平均负载的 1.25~1.5 倍）。当某 Prefill Pod 命中率极高但队列深度逼近阈值时，调度器主动打破亲和性，将请求分发至空闲 Pod 进行冷算，以微小的重算代价消除长尾排队。

2. 中心化/分层解耦存储（以 Mooncake 为代表）：将 KV Cache 从单一 Pod 的显存中解耦，持久化在集群跨节点的 CPU DRAM 与分布式固态存储池中。任意 Prefill Pod 计算出的 KV Cache 均可被任意 Decode Pod 远程挂载拉取，从底层存储架构上消除了‘必须绑定特定物理 Pod 显存’的亲和性约束，从根本上防止显存倾斜。

#### 维度：`multi_lora_affinity_dispatch`

*维度说明：针对挂载多个微调 LoRA 的多副本，如何感知各 Pod 当前已驻留的 Adapter 权重并做亲和性负载分担，减少显存置换开销*

针对挂载多个微调 LoRA 权重的多副本集群，构建适配器感知分担机制：

1. 调度器全局同步各 Decode/Prefill Pod 的 LoRA 驻留注册表（Loaded Adapter Registry）及显存驻留热度。

2. 请求携带特定 LoRA 标识时，调度器执行 LoRA 亲和分发，优先路由至已加载该 Adapter 的 Pod，避免发生数百毫秒的动态权重显存置换。

3. 当已有实例均处于高负载时，调度器结合空闲 Pod 的动态显存余量打分，挑选最合适实例进行流式热加载，并通过平滑窗口限制频繁换入换出（Thrashing）。

### 解耦架构与流水线负载分流

#### 维度：`pd_disaggregated_load_sharing`

*维度说明：Prefill 阶段与 Decode 阶段的跨 Pod 流水线解耦分担机制，请求流转路径与跨节点 KV-Cache 传输网络协同（RDMA/RoCE/NCCL）*

Prefill 与 Decode 阶段跨 Pod 流水线协同流转路径如下：

1. 请求路由与分配：客户端请求到达集群 Router，Router 选定 Prefill Pod 计算 Prompt，并预先在 Decode Pod 池中选定接收节点并预留显存 Block ID。

2. 预填充计算：Prefill Pod 并行计算全部 Prompt Token，将生成的 Key/Value 矩阵写入本地显存。

3. 跨节点高速直连传输：计算完成后，Prefill Pod 与 Decode Pod 通过无损高带宽网络（基于 RDMA/RoCE v2 协议、NVSHMEM 或优化 NCCL P2P 通信，结合 GPU Direct RDMA 旁路技术）直接将 KV Cache 张量传输至 Decode Pod 的显存或 GPU 环形暂存区（Staging Buffer），绕过 CPU 内存复制与 TCP 协议栈损耗。

4. 解码流式输出：Decode Pod 确认收到 KV Cache 后接管上下文，自回归生成第一个 Token 并进入连续生成循环，将响应流直接由 SSE/gRPC 连接推送给 Router 并返回用户。

#### 维度：`chunked_prefill_concurrency`

*维度说明：变长长文本 Prompt 的分块预填充（Chunked Prefill）对多副本并发负载平滑分担的支撑效果*

分块预填充（Chunked Prefill）在 PD 分离架构中发挥关键平滑作用：

1. 避免队头阻塞（Head-of-Line Blocking）：超长 Prompt（如 32K~128K Token）若单次全量执行，会独占 Prefill GPU 达数百毫秒以上。Chunked Prefill 将 Prompt 拆分为固定块（如每块 512 或 1024 Token），使得 Prefill Pod 可以在多个并发长短请求间交错计算。

2. 计算与通信深度重叠（Overlap）：前一段 Chunk 计算完毕即可触发跨节点 RDMA 传输，与下一段 Chunk 的 GPU GEMM 矩阵计算重叠掩盖，显著降低端到端首 Token 延迟（TTFT）。

3. 平抑负载毛刺：使多副本 Pod 间的计算水位保持平稳，大幅提高集群并发承载上限与吞吐稳定性。

### 过载保护、反压与容灾伸缩

#### 维度：`overload_protection_and_admission`

*维度说明：单 Pod 即将显存击穿（OOM）或队列满时的准入控制（Admission Control）、排队缓冲与溢流分流（Spillover）机制*

构建基于显存与队列双水位的多级准入反压与溢流机制：

1. 两级准入控制（Two-Tier Admission Control）：Router 层维护全局请求并发窗口与租户优先级队列；Pod 内部依据显存空闲块设置阈值（如 KV 占用率 > 90% 触发预警，> 95% 拒绝新任务）。

2. 跨阶段动态反压（Dynamic Backpressure）：当 Decode Pod 显存普遍紧张时，向 Router 发送反压信号，Router 立即减缓向 Prefill Pod 派发新请求的速率，防止 Prefill 端产生的大量 KV Cache 无法在 Decode 端着陆而造成显存或缓冲区爆仓。

3. 溢流分流（Spillover）：针对超额突发流量，Router 触发溢流策略，将低优先级请求分流至降级模型池、备用弹性 Pod 组或返回携带 Retry-After 头的 429 限流保护。

#### 维度：`k8s_autoscaling_and_rebalancing`

*维度说明：与 Kubernetes HPA/KEDA 水平伸缩的联动，扩容新增 Pod 时的冷启动预热，以及缩容/故障时的流量优雅下线（Drain）与平滑再均衡*

深度适配 Kubernetes 原生编排与异构伸缩体系：

1. 异构独立 HPA/KEDA 联动：Prefill Pod 与 Decode Pod 分属独立的 Deployment/StatefulSet。Prefill 池基于瞬时 `waiting_prefill_tokens` 和排队延迟驱动伸缩；Decode 池基于 `gpu_kv_cache_usage_factor` 与活跃生成序列数伸缩，实现算力密集型与带宽密集型 Pod 的弹性解耦。

2. 冷启动隔离与预热（Cold Start & Warm-up）：大模型 Pod 启动需数分钟拉取模型并编译 CUDA Graph。结合 Kubernetes StartupProbe 与内部预热测试请求，未完成初始化前对 Router 保持不可见，就绪后按加权缓慢升温引入流量。

3. 优雅下线（Graceful Drain）：滚动更新或缩容时，利用 PreStop 钩子通知 Router 摘除节点。Prefill Pod 等待当前 Chunk 传输完成；Decode Pod 等待已有流式长连接输出全部结束（Drain In-flight Streams），确保零服务中断与零 Token 丢弃。

#### 维度：`streaming_failover_resilience`

*维度说明：长连接流式输出（SSE/gRPC Streaming）长达数十秒过程中，Pod 崩溃或被抢占时的容灾与重试机制*

应对长连接流式输出过程中的 Pod 异常与崩溃：

1. Router 端流式状态追踪与断点恢复：Router 维护已推送 Token 偏移量。若 Decode Pod 在生成中途崩溃，Router 捕获连接异常，将‘原始 Prompt + 已生成 Token 序列’作为新的输入上下文，重派给 Prefill Pod 重新生成 KV Cache 并接入新的 Decode Pod 继续流式生成。

2. 基于持久化 KV 存储（如 Mooncake）的免重算接管：若系统配备分布式全局 KV Cache 缓存池，新分配的 Decode Pod 可直接根据 Cache Checkpoint 从共享内存/存储层挂载历史 KV 块，无需全量重算即可瞬间恢复后续 Token 生成。

3. 客户端优雅降级：若重试超限，流式接口返回结构化错误并附带已生成段落，避免客户端长轮询悬挂。

### 性能开销与落地可行性

#### 维度：`routing_overhead_and_latency`

*维度说明：分担决策自身引入的额外网络跳数、代理处理耗时（TTFT 影响）与 CPU/内存额外开销*

分担决策与跨节点流转的性能开销特征：

1. 调度决策延迟：基于 C++/Rust 高性能实现的智能 Router，单次前缀树检索与加权打分决策耗时仅 0.2ms ~ 1.5ms，占端到端耗时比率小于 0.5%，开销完全可忽略。

2. 跨机传输延迟（核心开销）：在 400Gbps/800Gbps RDMA/RoCE 网络下，传输典型 4K 上下文 KV Cache 耗时仅需 5ms ~ 20ms，远小于 Prefill 计算节省的耗时；但在 100Gbps 普通 TCP 跨机网络下，传输延迟可达 80ms ~ 200ms 以上，会严重侵蚀首 Token 延迟收益。

3. 资源占用：Router 仅需少量 CPU 与内存开销；主要开销在于集群需要配备专用的 RoCE/RDMA 无损网卡硬件设施。

#### 维度：`production_readiness_and_cases`

*维度说明：企业级生产落地成熟度、典型标杆案例与主流开源生态配套*

在工业界已取得高度成熟的生产验证：

1. 标杆生产案例：Moonshot AI（月之暗面）基于 Mooncake 在大规模集群中全面落地 PD 分离，支撑 Kimi 200 万超长文本服务，集群吞吐提升 3~5 倍；阿里云百炼、腾讯云、DeepSeek、字节跳动等在大模型推理中台广泛落地基于 PD 分离的异构推理池。

2. 开源生态配套：SGLang 原生支持 SGLang Router + Remote KV Transfer；vLLM 官方推进 Disaggregated Prefill (RFC) 及 vLLM V1 分布式 KV Transfer 架构；开源分布式缓存引擎 Mooncake、NVIDIA Dynamo 及 TensorRT-LLM 提供了完备的 PD 分离支撑组件。

- - -

## 4\. Kubernetes Gateway API for Inference (K8s 官方推理扩展)

> **架构分类**：云原生官方前沿网关  
> **核心定位**：Kubernetes SIG 推理网关规范，结合 Envoy ext-proc 协议获取多副本运行时显存/在飞指标，实现动态模型后端池负载分担

### 负载分担调度算法与决策核心

#### 维度：`load_balancing_algorithm`

*维度说明：具体分担算法逻辑（轮询/加权轮询、最小在飞请求 Least In-flight、基于前缀树 Radix Tree 的哈希匹配、一致性哈希与负载边界 CHWBL、双重映射 DualMap 等）*

基于 Kubernetes Gateway API 官方标准扩展架构，采用 Envoy 统一网关结合外部处理（ext-proc）机制与端点选择器（Endpoint Picker, EPP）协同算法：

1. 动态负荷加权调度（Queue-Depth & In-flight Adaptive Scoring）：EPP 作为控制面与数据面扩展，拦截网关请求，通过持续采集或推送接收到的各 Pod 活跃在飞请求数（Active/In-flight Requests）与排队等待数（Waiting Queue Depth）进行动态综合打分，将请求精确分派到实时负载最低的 Pod，避免传统 L4/L7 简单轮询引发的尾部延迟剧烈抖动。

2. KV Cache 亲和性分担（KV-Cache Utilization / Prefix Locality）：EPP 分析请求 Prompt 前缀特征或上游客户端标识，结合各 Pod 的 Cache 水位状态，将具有公共上下文或系统提示词的请求定向至具有最高缓存命中潜力的 Pod，显著削减首 Token 延迟（TTFT）。

3. 多级 SLO 优先级与模型降级路由（Priority-Aware & Multi-Objective Routing）：基于 `InferenceModel` CRD 定义的不同请求优先级（Criticality），高优先级请求独占或优先接入主推理池，普通请求在高峰期弹性调度至备用算力池。

4. LoRA 适配器亲和路由（LoRA Adapter Affinity）：识别请求中的目标微调适配器，优先定向至已在显存中驻留该 LoRA 权重的 Pod 实例，规避频繁动态加载开销。

#### 维度：`load_metrics_monitored`

*维度说明：分担决策所采纳的后端状态指标（活跃请求数 Running Requests、队列等待数 Waiting Requests、GPU 显存水位、KV Cache 块占用率、Token 生成速率）*

EPP 通过直接拉取/后端主动上报方式监控推理引擎运行时指标：

1. 引擎在飞与队列指标：当前执行中的在飞请求数（`num_requests_running`）、排队等待执行请求数（`num_requests_waiting`）。

2. 显存与 KV 缓存水位指标：GPU KV Cache 块占用百分比/空闲块数量（`gpu_cache_usage_factor` / `free_gpu_blocks`）。

3. LoRA 适配器就绪状态：各 Pod 当前已驻留激活的 LoRA 列表与适配器显存池剩余容量。

4. 网关与端点观测度量：端到端与排队延迟直方图（`inference_objective_request_duration_seconds`）、各 Pod 队列深度分布（`inference_pool_per_pod_queue_size`）、请求过载脱落/限流频次（Shedding Events Count）。

#### 维度：`dispatch_granularity`

*维度说明：请求分发粒度（TCP 连接级、HTTP/2 请求级、Token 生成 Chunk 级、Prefill/Decode 阶段级）*

请求分发粒度为 HTTP/2 / gRPC 请求级（Request-level）与流式数据透明转发：

1. 请求拦截级：客户端向 Gateway 提交推理请求（如 OpenAI 兼容的 `/v1/chat/completions` REST/gRPC 请求），Envoy 网关层在解析出 HTTP Header 或初步 Request Payload 时触发 `ext-proc` 过滤器。

2. 端点选路级：Envoy 将请求元数据同步推送给 EPP，EPP 在毫秒级内完成后端实例打分与选路决策，返回指示 Envoy 重定向到具体目标 Pod 的指令（如重写上游路由或注入路由目的标签），决策在请求级生效。

3. 流式响应 Chunk 级：选路完成后，Envoy 直连目标后端 Pod，后端的自回归流式响应（SSE / gRPC Streaming）由 Envoy 原路高效透传，EPP 不介入单 Token Chunk 的转发路径，保证数据面极低代理损耗。

### 状态亲和与负载均衡的动态权衡

#### 维度：`cache_affinity_vs_load_balance`

*维度说明：如何平衡‘提高 KV Cache / Prefix Caching 命中率（倾向发给已有缓存的同一 Pod）’与‘防止单 Pod 显存负载倾斜过载（倾向发给空闲 Pod）’*

在网关层面通过多目标打分函数权衡缓存命中与负载倾斜：

1. 多目标动态打分公式（Multi-objective Scoring Function）：调度器计算综合选路分值 `Score = W_cache * Cache_Hit_Score - W_load * Load_Penalty`。当某 Pod 前缀命中率极高但队列深度逼近过载警戒线时，负载惩罚项迅速增大，强制调度器将请求转移至低负荷 Pod，避免单 Pod 因追求‘缓存热点’而发生排队积压击穿。

2. 结合一致性哈希与负载边界（CHWBL）：在多副本横向扩展时，利用带负载边界的一致性哈希算法，将相同前缀哈希的请求映射至主副 Pod 集合中，兼顾缓存局部性与全局均匀分摊。

3. 缓存感知隔离池：在 `InferencePool` 拓扑中支持将长前缀请求与高频短请求划分至不同模型池子集，防止短请求对长前缀缓存造成无序置换。

#### 维度：`multi_lora_affinity_dispatch`

*维度说明：针对挂载多个微调 LoRA 的多副本，如何感知各 Pod 当前已驻留的 Adapter 权重并做亲和性负载分担，减少显存置换开销*

针对多 LoRA 共享底座架构的高效感知分担：

1. CRD 声明式映射：通过 `InferenceModel` CRD 将不同的 LoRA 变体逻辑声明为模型端点，并指向底座模型所在的同一 `InferencePool`。

2. 动态感知与亲和调度：EPP 维护集群内各 Pod 的 LoRA 驻留快照。当收到指向特定 LoRA 的推理请求时，优先将请求派发给已完成该 Adapter 显存加载的 Pod。

3. 动态加载决策与防抖：若目标 LoRA 未在任何 Pod 驻留或驻留 Pod 均严重饱和，EPP 选择显存余量最充裕且队列最短的 Pod 承接，触发后端的按需热加载，并通过平滑窗口打分限制单 Pod 频繁卸载换入造成的 GPU 显存颠簸（Thrashing）。

### 解耦架构与流水线负载分流

#### 维度：`pd_disaggregated_load_sharing`

*维度说明：Prefill 阶段与 Decode 阶段的跨 Pod 流水线解耦分担机制，请求流转路径与跨节点 KV-Cache 传输网络协同（RDMA/RoCE/NCCL）*

Gateway API for Inference 在跨阶段解耦架构中的协同支持：

1. 多池抽象解耦：在 Kubernetes 中将 Prefill 集群和 Decode 集群分别抽象为两个独立的 `InferencePool`（如 `prefill-pool` 与 `decode-pool`）。

2. 协同流水线路由：网关可作为全集群统一访问入口，利用 `InferenceModel` 将入站推理请求首跳路由给 Prefill 池中的最优端点；Prefill 节点完成预填充后，结合底层 RDMA 跨节点传输 KV Cache，同时网关或前端协调器将流式输出生命周期交由 Decode 池负责，实现云原生网关层对 PD 分离集群的统一纳管与流量分发。

#### 维度：`chunked_prefill_concurrency`

*维度说明：变长长文本 Prompt 的分块预填充（Chunked Prefill）对多副本并发负载平滑分担的支撑效果*

利用 Chunked Prefill 增强网关层并发调度的均匀性与平滑度：

1. 消除超长大请求造成的调度不均：传统全量预填充会导致 Pod 长时间无响应，造成网关指标感知严重滞后。后端启用分块预填充后，单次计算粒度变小，指标反馈更加平滑和线性。

2. 精确预期排队耗时估算：EPP 结合后端上报的 Chunk 计算耗时与当前等待 Token 数，能精准估算出各 Pod 的预期可服务时间（Expected Wait Time），从而更平滑地将并发长短请求打散到各副本，避免网关误判后端假死。

### 过载保护、反压与容灾伸缩

#### 维度：`overload_protection_and_admission`

*维度说明：单 Pod 即将显存击穿（OOM）或队列满时的准入控制（Admission Control）、排队缓冲与溢流分流（Spillover）机制*

构建基于 Envoy + EPP 的统一云原生准入控制、排队缓冲与溢流保护：

1. 网关层主动准入与负载脱落（Load Shedding）：当所有后端的 KV 显存占用率或排队规模超过设定的临界安全阈值时，EPP 直接驱动 Envoy 在网络入口层执行负载脱落，向客户端返回 429 Too Many Requests（附带标准 Retry-After 头）或 503 错误，阻断流量穿透并保护 GPU 实例不发生 OOM 崩溃。

2. 优先级排队缓冲（Priority Queuing）：在 EPP 与网关层设置缓冲等待队列，依据 `InferenceModel` 的 Criticality 配置，优先保障高优先级生产流量准入，低优先级请求排队缓冲。

3. 动态溢流分流（Spillover）：支持在 `InferenceModel` 中配置备用池（Fallback/Spillover Pool）。当主 `InferencePool` 饱和时，EPP 动态修改转发上游目标，将流量溢流至次级低配集群或公有云弹性算力池。

#### 维度：`k8s_autoscaling_and_rebalancing`

*维度说明：与 Kubernetes HPA/KEDA 水平伸缩的联动，扩容新增 Pod 时的冷启动预热，以及缩容/故障时的流量优雅下线（Drain）与平滑再均衡*

无缝集成云原生自动伸缩机制与优雅生命周期管理：

1. 驱动 KEDA / HPA 智能伸缩：EPP 汇聚输出高质量的池级模型指标（如 `inference_pool_per_pod_queue_size`、等待队列深度、平均服务延迟），直接作为 KEDA 外部指标源（External Metrics），驱动 Pod 水平扩容，克服传统 CPU/GPU 利用率难以准确反映大模型排队负荷的弊端。

2. 容器冷启动隔离与预热引流：结合 Kubernetes EndpointSlice 机制，新扩容的 Pod 在拉取数百 GB 模型权重及 CUDA Graph 初始化完成前处于隔离状态；EPP 通过健康检查探针确认引擎真正就绪后，才将其纳入调度候选池，并可支持加权小流量预热引流。

3. 流量优雅下线（Graceful Draining）：缩容或滚动更新时，Pod 被标记为 Terminating，K8s 控制面将该端点从 `InferencePool` 摘除。EPP 立即停止向其分发新请求，Envoy 保持已有流式连接持续通信，直至进行中的在飞 Token 全部生成完毕再正常退出，保障业务零报错。

#### 维度：`streaming_failover_resilience`

*维度说明：长连接流式输出（SSE/gRPC Streaming）长达数十秒过程中，Pod 崩溃或被抢占时的容灾与重试机制*

针对 SSE / gRPC Streaming 长达数十秒生成周期的容灾与恢复机制：

1. 首 Token 阶段透明故障转移（Pre-TTFT Failover）：如果后端 Pod 在握手、排队或生成第一个 Token 之前崩溃或超时，Envoy 捕获上游错误并在 EPP 决策下，将请求安全、无感知地重试路由至健康的副本 Pod，对客户端完全透明。

2. 流式传输中途故障防护：若流式生成已开始、部分 Token 已下发给客户端，Envoy 避免粗暴重试导致内容重复；Envoy 快速切断连接并返回特定可解析的流式终止错误码，通知客户端 SDK 触发断点恢复逻辑。

3. 主动健康检查与断路器（Circuit Breaking）：EPP 实时监控 Pod 异常率，连续失败达到阈值时立即将对应 Pod 熔断下线，防止坏节点持续吞噬请求。

### 性能开销与落地可行性

#### 维度：`routing_overhead_and_latency`

*维度说明：分担决策自身引入的额外网络跳数、代理处理耗时（TTFT 影响）与 CPU/内存额外开销*

性能开销与代理损耗的精准控制：

1. 代理链路跳数：在客户端与 Pod 之间引入了一层 Envoy Gateway 与 EPP 扩展。

2. `ext-proc`

    选路延迟：Envoy 与同机或本地 DaemonSet 部署的 EPP 通过 gRPC Unix Domain Socket (UDS) 或高效 IPC 通信，单次选路耗时仅约为 1ms ~ 5ms，相较于动辄数百毫秒乃至数十秒的 LLM 推理延迟，网络与代理开销占比通常小于 1%~2%，且完全不阻塞 Token 生成阶段的数据面传输。

3. 资源开销：Envoy 采用 C++ 实现，性能极高且内存消耗极低（通常百兆以内）；EPP 采用 Go 语言轻量实现，单实例可支撑每秒数千 QPS 的智能路由调度决策。

#### 维度：`production_readiness_and_cases`

*维度说明：企业级生产落地成熟度、典型标杆案例与主流开源生态配套*

具备极高的工业级背书与云原生标准化前景：

1. 官方规范与组织：由 Kubernetes 官方 SIG Network、WG Serving 以及 Envoy 社区核心专家联合制定并在 `kubernetes-sigs/gateway-api-inference-extension` 持续迭代，是目前 Kubernetes 体系内最具权威性和前瞻性的 AI 推理网关规范。

2. 生产落地与标杆支持：Google Cloud 正式基于该规范推出并落地 **GKE Inference Gateway**；Envoy Gateway 官方原生支持推理扩展；Istio、NGINX Gateway Fabric、阿里云 Higress 等主流开源服务网关与云厂商全面跟进兼容适配；开源社区 `llm-d` (llm-d-router) 等项目基于该规范构建高性能生产路由层。

3. 后端生态整合：深度兼容 vLLM、TGI、SGLang、Triton 等主流开源推理框架，与 KEDA、Ray on Kubernetes 形成完整的全栈云原生 AI 服务解决方案。

- - -

## 5\. 分布式 KV-Cache 亲和路由与负载防倾斜 (CHWBL 与 llm-d)

> **架构分类**：分布式缓存与防倾斜调度  
> **核心定位**：结合一致性哈希与负载边界（Consistent Hashing with Bounded Loads），在多副本横向扩展时平衡缓存命中率与单 Pod 显存负载倾斜

### 负载分担调度算法与决策核心

#### 维度：`load_balancing_algorithm`

*维度说明：具体分担算法逻辑（轮询/加权轮询、最小在飞请求 Least In-flight、基于前缀树 Radix Tree 的哈希匹配、一致性哈希与负载边界 CHWBL、双重映射 DualMap 等）*

核心算法基于有界负载一致性哈希（Consistent Hashing with Bounded Loads, CHWBL，由 Mirrokni 等人提出）与前缀树启发式打分（Prefix Radix-Tree Scoring）相结合的动态调度机制。在 CNCF 沙箱项目 llm-d 及 Kubernetes Gateway API Inference Extension (GAIE) 中，核心组件 Endpoint Picker (EPP) 作为 Envoy ext-proc 外部处理插件或独立 L7 调度器运行。其算法逻辑包含：1. 请求到达时，提取 System Prompt、多轮对话前缀或 Token 序列哈希（如 SHA-256/MurmurHash3），映射到包含大量虚拟节点（Virtual Nodes）的一致性哈希环；2. 设定容量边界乘数 c = 1 + ε（典型取值 1.15~1.25），依据集群实时活跃负载动态计算各 Pod 允许承载的最大请求上限 L\_max = ⌈c \* (总活跃负载 / N)⌉；3. 首选主节点判定：优先匹配哈希环上顺时针命中的首个 Worker Pod；4. 防倾斜溢流分流（Bounded Overflow）：若首选 Pod 当前负载已达到或超过 L\_max，算法沿哈希环顺时针向后遍历探测下一个可用节点（Next Replica），直至找到未过载的 Pod 分发，实现局部散列与平滑级联溢流；5. 双重映射与双选策略（DualMap / Power of Two Choices）：通过两个独立哈希函数映射两个候选 Pod，结合两者的实时排队预期时延与前缀重叠度进行动态优选，在 SLO 临界状态自动平滑退化至最小在飞请求（Least In-flight）调度。

#### 维度：`load_metrics_monitored`

*维度说明：分担决策所采纳的后端状态指标（活跃请求数 Running Requests、队列等待数 Waiting Requests、GPU 显存水位、KV Cache 块占用率、Token 生成速率）*

分担决策深度采集并采纳以下多维后端与模型运行时指标：1. 活跃在飞请求数（In-flight / Running Requests）：实时跟踪每个 Pod 正在执行 Prefill 或 Decode 的请求并发量；2. 引擎内部等待队列深度（Waiting Queue Depth）：统计 vLLM / SGLang 内部调度器的排队积压请求数量；3. GPU 显存与 KV-Cache 块占用率（KV-Cache Block Usage Ratio）：通过 vLLM/SGLang 暴露的 Prometheus / OpenTelemetry 导出器，实时监控 gpu\_cache\_usage\_factor（已分配 KV 块占总可用块的比例）与空闲显存（Free Memory）；4. 前缀缓存状态指纹（Prefix Cache Fingerprint / Hit Ratio）：EPP 维护每个 Pod 的粗粒度 Radix Tree 状态缓存或 Bloom Filter，跟踪各 Pod 当前驻留的前缀哈希集合；5. Token 生成速率与排队时延预估：根据各 Pod 当前输出 Token 速率（Tokens/sec）与上下文处理积压，预估新请求的排队等待时长（TTFT Expectation）。

#### 维度：`dispatch_granularity`

*维度说明：请求分发粒度（TCP 连接级、HTTP/2 请求级、Token 生成 Chunk 级、Prefill/Decode 阶段级）*

主要工作在 HTTP/1.1、HTTP/2 请求级（Request-level）与 gRPC 单次 RPC 调用粒度。通过 Envoy 网关的 ext-proc（External Processing）协议在请求头部（Header）及 Body 初始流式到达时进行拦截，由 EPP 解析 Prompt 前缀并执行打分路由，随后由 Envoy 建立上游连接转发至目标 Pod。对于请求内部的流水线分发：在标准单体多副本模式下，单个请求的全生命周期（Prefill + Decode）绑定至同一 Pod；在 PD 解耦（Prefill-Decode Disaggregation）架构中，llm-d/EPP 在网关入口负责将请求首先分派给选定的 Prefill Pod，并协同后端推理引擎触发跨 Pod KV-Cache 传输与 Decode 阶段交接。

### 状态亲和与负载均衡的动态权衡

#### 维度：`cache_affinity_vs_load_balance`

*维度说明：如何平衡‘提高 KV Cache / Prefix Caching 命中率（倾向发给已有缓存的同一 Pod）’与‘防止单 Pod 显存负载倾斜过载（倾向发给空闲 Pod）’*

在高并发长文本（如多轮对话、长文档 RAG）场景中，同一公共前缀极易引发严重的热点集中，导致单一 Pod 显存打满、队列积压劣化 TTFT 乃至触发 OOM；若单纯采用无状态轮询，则会导致全集群 KV-Cache 命中率为零，各 Pod 重复进行昂贵的 Prefill 计算。系统通过以下三重机制实现动态平衡：1. 严格的有界负载硬约束（Bounded Loads Constraint）：以集群平均负载动态乘以 (1 + ε) 为容量红线，只要单 Pod 负载未越界，100% 优先保证前缀亲和命中；一旦越界，坚决牺牲命中率将流量分流至次优空闲节点，确保排队时延有界；2. 综合打分平衡函数（Composite Scorer）：EPP 采用多目标联合打分 S = W\_cache \* Cache\_Hit\_Score - W\_load \* Load\_Penalty\_Score，随 Pod 负载攀升逐步增加负载惩罚权重，实现从偏好缓存命中向偏好负载均衡的平滑过渡；3. 热点前缀多副本自适应扩散（Hotspot Self-Replication）：当热点前缀请求溢流至次优 Pod 后，次优 Pod 会自动计算并在本地生成该前缀的 KV-Cache，使热点前缀从单 Pod 副本自然扩展为多 Pod 副本，后续请求即可在多个 Pod 间并行分摊亲和流量。

#### 维度：`multi_lora_affinity_dispatch`

*维度说明：针对挂载多个微调 LoRA 的多副本，如何感知各 Pod 当前已驻留的 Adapter 权重并做亲和性负载分担，减少显存置换开销*

EPP 具备 Multi-LoRA 适配器感知能力。网关解析请求体或 Header 中的 lora\_name / model 标识，维护集群内各个 Pod 当前已激活驻留的 LoRA 权重状态表（LoRA Warm Cache Table）。调度决策时，优先将请求派发至已加载目标 LoRA 且显存处于安全水位的 Pod；仅当所有已常驻该 LoRA 的 Pod 均触碰负载边界时，才分流至未加载该 LoRA 的空闲 Pod，并通知后端触发动态权重换入（利用 S-LoRA / Paged LoRA 等内存分页加载），从而最大限度降低 GPU 显存频繁换入换出引入的毫秒级阻塞与显存碎片开销。

### 解耦架构与流水线负载分流

#### 维度：`pd_disaggregated_load_sharing`

*维度说明：Prefill 阶段与 Decode 阶段的跨 Pod 流水线解耦分担机制，请求流转路径与跨节点 KV-Cache 传输网络协同（RDMA/RoCE/NCCL）*

深度协同 Prefill 与 Decode 阶段解耦流水线分担：1. 请求分流路径：Gateway/EPP 首先将长 Prompt 路由至具备高 Prefix 缓存命中率且算力充足的 Prefill 实例池进行高效预填充计算；2. 跨节点传输协同：Prefill Pod 计算完毕后，通过跨节点高速通信通道（基于 RDMA/RoCE 零拷贝网络或 NCCL P2P 链路）直接将生成的 KV-Cache 块推送至指定的 Decode Pod；3. 协同调度编排：llm-d/EPP 作为全局编排中枢，在向 Prefill Pod 派发请求前，预先向 Decode Pod 申请或探测目标 KV-Cache 显存块资源，确保 Prefill 完成后 Decode 端具备充足显存承接，避免跨节点传输时因 Decode 节点显存不足导致的阻塞与二次换入换出。

#### 维度：`chunked_prefill_concurrency`

*维度说明：变长长文本 Prompt 的分块预填充（Chunked Prefill）对多副本并发负载平滑分担的支撑效果*

全面协同后端引擎的 Chunked Prefill（分块预填充）特性。针对数万 Token 的超长 Prompt，后端将其切分为多个 Chunk（如每块 512~1024 Token）分步执行计算。在多副本负载分担层面，Chunked Prefill 显著消除了大 Prompt 一次性独占 GPU 算力数秒导致的计算倾斜与 Decode 气泡（降低推理抖动），使得各 Pod 的在飞 Token 负载处于平稳可预测状态，为 CHWBL 算法提供了更加高精度的负载度量基准，防止因单次大 Prefill 误判 Pod 负载状态而导致过度溢流。

### 过载保护、反压与容灾伸缩

#### 维度：`overload_protection_and_admission`

*维度说明：单 Pod 即将显存击穿（OOM）或队列满时的准入控制（Admission Control）、排队缓冲与溢流分流（Spillover）机制*

提供端到端的准入控制（Admission Control）与溢流分流（Spillover）过载保护：1. 网关前置快速失败：当目标 InferencePool 内所有 Pod 的 KV-Cache 占用率普遍超过高水位阈值（如 90%~95%）或总排队深度突破安全红线时，EPP 直接在网关层返回 HTTP 429（Too Many Requests），保护后端免于显存溢出（OOM）与雪崩式崩溃；2. 租户与优先级排队缓冲：支持在 EPP 内部维护优先队列，在轻微过载时对非关键请求实施排队缓冲反压，优先保障高 SLA 业务请求的派发；3. 弹性溢流分流（Spillover）：当主推理池达到负载上限时，依据路由规则将请求自动溢流至公共备用推理池或云端弹性突发实例承担。

#### 维度：`k8s_autoscaling_and_rebalancing`

*维度说明：与 Kubernetes HPA/KEDA 水平伸缩的联动，扩容新增 Pod 时的冷启动预热，以及缩容/故障时的流量优雅下线（Drain）与平滑再均衡*

与 Kubernetes 云原生弹性伸缩和运维生命周期深度融合：1. 弹性伸缩联动（HPA / KEDA）：EPP 汇聚并对外暴露模型维度的排队深度（Waiting Requests）、KV-Cache 显存占用率和预估 TTFT 指标，驱动 KEDA External Scaler 实施秒级响应的 Horizontal Pod Autoscaler；2. 扩容冷启动预热：新 Pod 就绪（Readiness 探针通过）后动态注册至 CHWBL 哈希环，EPP 采用软启动策略（Warm-up Routing），先向新 Pod 分派无状态或低重合度的短文本请求，使其逐步在本地积累 Prefix 缓存，避免冷启动瞬时承接大流量导致吞吐骤降；3. 优雅下线（Drain）与再均衡：当节点缩容或收到 SIGTERM 信号时，EPP 立即将该 Pod 标记为 Terminating 并从活跃哈希环中剔除，停止派发新请求，同时保留对既有流式连接的数据中继直至全部完成，实现无损平滑下线。

#### 维度：`streaming_failover_resilience`

*维度说明：长连接流式输出（SSE/gRPC Streaming）长达数十秒过程中，Pod 崩溃或被抢占时的容灾与重试机制*

针对 SSE（Server-Sent Events）与 gRPC Streaming 持续数十秒的流式输出，构建健壮的容灾重试机制：1. 断点感知与重试：若上游 Pod 在解码过程中意外崩溃（如底层驱动异常或不可抗力中断），Envoy 捕获断开事件；若客户端支持带 Token 偏移量的断点重试请求，EPP 将请求重新路由至哈希环上的次选 Pod；2. 快速冷启复原：由于前缀树特征或跨节点分布式缓存的存在，次选 Pod 可以迅速匹配或复用前半段上下文的 KV-Cache，极大缩短二次 Prefill 恢复时间；3. 优雅异常信令：对于不支持断点恢复的普通客户端，网关在流式管道中注入明确的标准 JSON 错误事件通知前端，避免连接静默挂起。

### 性能开销与落地可行性

#### 维度：`routing_overhead_and_latency`

*维度说明：分担决策自身引入的额外网络跳数、代理处理耗时（TTFT 影响）与 CPU/内存额外开销*

1. 网络跳数与代理耗时：采用 Envoy ext-proc 架构，增加 1 次同节点 Unix Domain Socket 或 Pod 间轻量 gRPC 调用，单次代理处理延迟控制在 13ms 以内；2. 调度算法开销：EPP 内部基于哈希表与树状结构维护状态，CHWBL 查找与打分计算耗时低于 1ms，CPU 与内存开销极低（Go/Rust 实现单实例仅需数十 MB 内存即可支撑数千 QPS）；3. 综合性能收益：虽然引入了 2ms 左右的网关代理延迟，但通过将分布式 Prefix Cache 命中率提升 30%70%，避免了重复计算成千上万的 Prompt Tokens，将端到端首字延迟（TTFT）大幅缩短 40%~80%，整体吞吐量获得数倍提升，综合性价比极高。

#### 维度：`production_readiness_and_cases`

*维度说明：企业级生产落地成熟度、典型标杆案例与主流开源生态配套*

具备极高的生产就绪度（Production-Ready）：1. 开源生态：llm-d 作为 CNCF 沙箱项目，由 Red Hat、Google Cloud、IBM Research、CoreWeave、NVIDIA 等业界巨头联合共建；Kubernetes 官方 SIG-Network / SIG-Apps 大力推进 Gateway API Inference Extension (GAIE) 标准制定；2. 生态兼容：无缝集成 vLLM、SGLang、Envoy AI Gateway 与 KubeAI，成为下一代标准云原生大模型推理调度控制面的基准实现；3. 标杆落地：在 Red Hat OpenShift AI、IBM watsonx 以及主流超大规模公有云 AI 基础设施中完成大规模生产验证。

- - -

## 6\. Mooncake KVCache 中心化解耦存储与全局调度

> **架构分类**：分布式存储与全局流控  
> **核心定位**：跨 DRAM/SSD/GPU 构建分布式 KV-Cache 存储层，配合全局调度器实现高并发长文本场景下的多副本自适应负载分担

### 负载分担调度算法与决策核心

#### 维度：`load_balancing_algorithm`

*维度说明：具体分担算法逻辑（轮询/加权轮询、最小在飞请求 Least In-flight、基于前缀树 Radix Tree 的哈希匹配、一致性哈希与负载边界 CHWBL、双重映射 DualMap 等）*

核心采用基于以 KVCache 为中心的两级协同负载分担架构：Mooncake Conductor（全局中枢调度器）+ Mooncake Store（分布式分层解耦存储池）。其负载分担与调度决策算法流程包含：1. 缓存局部性与拓扑重叠感知匹配：Conductor 维护集群全局的 KVCache 分布与 Prefix 指纹视图。请求到达时，首先检索其 Prompt 前缀在集群多级存储介质（本地 GPU HBM / 远端 Host CPU DRAM / 本地与远端 NVMe SSD）中的物理分布；2. 传输与重算代价量化模型（Transfer vs. Recompute Cost Model）：基于自研 Transfer Engine 的实时网络拓扑带宽矩阵，动态评估从远端节点拉取 KV-Cache 的传输耗时与本地 GPU 重新计算 Prefill 的算力耗时，当且仅当传输耗时显著低于重算耗时且目标节点负载安全时，触发跨节点缓存复用分担；3. 联合两阶段解耦配对算法（Joint Prefill-Decode Placement）：将请求拆解，依据 Prefill 实例池的算力吞吐负载与 Decode 实例池的显存及 TBT 负载，协同挑选出综合代价最低的最优 (Prefill Pod, Decode Pod) 节点对；4. 基于预测的早期拒绝机制（Prediction-based Early Rejection）：在突发高并发场景下，Conductor 预测当前请求在 Prefill 完成后，Decode 节点池是否具备满足 TTFT 与 TBT SLO 的并发承载能力，若预测无法达标，在请求入口处即刻执行主动早拒绝，杜绝系统无效吞吐与雪崩。

#### 维度：`load_metrics_monitored`

*维度说明：分担决策所采纳的后端状态指标（活跃请求数 Running Requests、队列等待数 Waiting Requests、GPU 显存水位、KV Cache 块占用率、Token 生成速率）*

分担决策深度采集并联合监控多维分布式硬件与模型服务指标：1. 分布式存储分层水位：各节点的 GPU HBM 显存占用率、CPU Host DRAM 共享缓存占用率、本地 NVMe SSD 写入吞吐与磁盘余量；2. 计算实例动态负载：Prefill Pod 当前正在处理的 Prompt Tokens 吞吐量及在飞批处理请求数；Decode Pod 当前激活的并发序列数（Active Batch Size）、单步解码迭代时延（Step Time）以及 Token 生成速率（Tokens/sec/GPU）；3. 全局网络与传输通道状态：跨节点 InfiniBand / RoCE RDMA 高速网络的实时链路带宽利用率、PFC 拥塞状态以及 Mooncake Transfer Engine 正在执行的跨机 KV-Cache 传输队列积压量；4. 服务等级目标（SLO）达成率：端到端请求响应延迟、首字时延（TTFT）与 Token 间时延（TBT / Inter-token Latency）的 P95/P99 达标百分比。

#### 维度：`dispatch_granularity`

*维度说明：请求分发粒度（TCP 连接级、HTTP/2 请求级、Token 生成 Chunk 级、Prefill/Decode 阶段级）*

实行两阶段流水线解耦（Phase-level Disaggregation）与会话生命周期级的细粒度分发：1. 接入层请求级分发：Conductor 统一接入客户端 HTTP/gRPC 请求；2. 计算阶段解耦分发：首先将长文本 Prompt 分发给 Prefill 节点池中特定的 Prefill Pod 执行密集矩阵运算；Prefill 计算结束后，通过 Transfer Engine 跨节点零拷贝搬运 KV-Cache，随后将生成状态移交给 Decode 节点池中特定的 Decode Pod 执行自回归解码；3. 分块预填充（Chunked Prefill）并发切片：对于百万（1M+）级别的超长上下文，支持将 Prompt 切片分块，在多个 Prefill 节点间实现更细粒度的并发分担计算与流水线汇聚。

### 状态亲和与负载均衡的动态权衡

#### 维度：`cache_affinity_vs_load_balance`

*维度说明：如何平衡‘提高 KV Cache / Prefix Caching 命中率（倾向发给已有缓存的同一 Pod）’与‘防止单 Pod 显存负载倾斜过载（倾向发给空闲 Pod）’*

彻底颠覆了传统架构中 KV-Cache 与单一 GPU 物理绑定的矛盾桎梏：传统架构下，为了追求缓存命中率必须将长文本请求强绑定至特定 GPU，极易导致拥有热点缓存的 GPU 算力暴满而其余节点闲置；若分流至空闲 GPU 又会丧失缓存造成灾难性的重复 Prefill。Mooncake 通过中心化解耦架构实现动态平衡：1. 存储与计算分离的虚拟资源池化：将全集群各宿主机未被充分利用的 CPU DRAM 和 NVMe SSD 聚合为统一的分布式缓存池（Mooncake Store），打破单卡显存容量壁垒；2. 高速网络驱动的自由流动分担：当持有缓存原件的 GPU 算力过载时，Conductor 果断将计算任务分发给算力空闲的异地 GPU Pod，异地 GPU 利用 200Gbps~400Gbps RDMA 零拷贝以数十 GB/s 的线速在数十毫秒内直接从远端 DRAM/SSD 换入所需前缀的 KV-Cache，达成‘计算负载全集群绝对均衡，缓存数据按需超高速跨机流动’；3. 分级缓存自适应冷热分层：高频前缀保留在 GPU HBM，次高频前缀沉降至 Host DRAM，低频长尾前缀归档至 NVMe SSD，实现极高缓存命中率与无倾斜计算调度的完美统一。

#### 维度：`multi_lora_affinity_dispatch`

*维度说明：针对挂载多个微调 LoRA 的多副本，如何感知各 Pod 当前已驻留的 Adapter 权重并做亲和性负载分担，减少显存置换开销*

Conductor 实时同步全集群 Prefill 与 Decode 节点当前驻留的 LoRA Adapter 权重状态拓扑图。在调度分发时，将 LoRA 权重本地命中率作为关键权重项纳入多目标评分器；同时依托 Mooncake Store 跨节点的全局共享内存传输能力，高频 LoRA 权重可预热缓存在分布式 DRAM 中，当非驻留 Pod 需要承接微调任务时，通过 RDMA 毫秒级极速载入权重，有效平摊 Multi-LoRA 动态加载带来的冷启动时延开销。

### 解耦架构与流水线负载分流

#### 维度：`pd_disaggregated_load_sharing`

*维度说明：Prefill 阶段与 Decode 阶段的跨 Pod 流水线解耦分担机制，请求流转路径与跨节点 KV-Cache 传输网络协同（RDMA/RoCE/NCCL）*

Mooncake 是工业界将 Prefill-Decode（PD）解耦架构推向万卡级规模的代表性里程碑：1. 异构资源解耦分配：Prefill 池针对计算密集型（Compute-bound）特点进行算力密集型卡型配置，Decode 池针对显存带宽受限型（Memory-bandwidth-bound）特点针对性调优，消除了传统混合部署下 Prefill 突发对 Decode 步进时延的剧烈抢占干扰；2. 跨机传输通信协同：自研高性能 Transfer Engine 深度整合 RDMA（InfiniBand/RoCE）和 GPU Direct RDMA（GDR）技术，实现无须 CPU 参与的跨节点 GPU-to-GPU 零拷贝极速传输；3. 流水线级闭环流转：Conductor 统一掌管 Prefill Pod 算力调度、KV-Cache 跨网络传输路由以及 Decode Pod 显存块预锁定，实现长 Prompt 预填充与自回归生成之间的毫秒级平滑流水线交接。

#### 维度：`chunked_prefill_concurrency`

*维度说明：变长长文本 Prompt 的分块预填充（Chunked Prefill）对多副本并发负载平滑分担的支撑效果*

针对百万级（1M+）超长上下文大模型推理场景，深度结合 Chunked Prefill 机制：1. 计算与传输流水线重叠（Overlap Pipeline）：Prefill 节点在分块计算后续 Chunk 时，前序已生成的 KV-Cache Chunk 已通过 RDMA 异步流式向 Mooncake Store 或目标 Decode Pod 传输，将计算延迟与跨节点传输延迟完全重叠；2. 峰值负载平滑：将单次长达数秒的巨型 Prefill 脉冲切割为均质的小计算块，极大平抑了多副本节点的瞬时算力冲击，提升了集群整体的并发调度弹性与吞吐稳定性。

### 过载保护、反压与容灾伸缩

#### 维度：`overload_protection_and_admission`

*维度说明：单 Pod 即将显存击穿（OOM）或队列满时的准入控制（Admission Control）、排队缓冲与溢流分流（Spillover）机制*

构建多层次、预测性的全链路过载保护与反压体系：1. 预测性早拒绝（Prediction-based Early Rejection）：Conductor 依据排队理论与历史解码负载模型，在请求进入系统前预判其完成 Prefill 后目标 Decode 实例能否按 SLO 要求（TTFT/TBT）接管解码；若预估系统未来将发生过载违约，立刻在入口实施早拒绝，坚决杜绝‘无效执行完高昂 Prefill 后因 Decode 拥塞而超时丢弃’的算力严重浪费；2. 存储多级分流与动态驱逐：当 GPU HBM 显存达到警戒阈值时，自动将非活跃 KV 块置换（Evict）至 Host DRAM，DRAM 饱和时溢流至 NVMe SSD，避免发生 GPU OOM 崩溃；3. 全局反压联动：当全集群存储与网络链路接近饱和时，Conductor 动态调低请求准入配额，触发客户端排队等待与速率限制（Rate Limiting）。

#### 维度：`k8s_autoscaling_and_rebalancing`

*维度说明：与 Kubernetes HPA/KEDA 水平伸缩的联动，扩容新增 Pod 时的冷启动预热，以及缩容/故障时的流量优雅下线（Drain）与平滑再均衡*

深度适配 Kubernetes 原生容器编排与水平伸缩体系：1. 异构池化独立弹性伸缩：在 K8s 中通过独立的 Deployment 或 Custom Resource 分离部署 Prefill Pod 池与 Decode Pod 池，结合 KEDA 采集 Conductor 暴露的 Prefill 队列 Token 吞吐与 Decode 活跃序列指标，实现两阶段独立细粒度 HPA 伸缩；2. 存储状态解耦带来的极速再均衡：由于 KV-Cache 与单卡物理脱耦、持久暂存在分布式存储层中，在 Pod 缩容、故障驱逐或节点被抢占时，不会丢失已有的上下文缓存资产；新扩容的 Pod 加入集群后可立即作为纯算力节点接入，直接从 Mooncake Store 读取缓存开始服务，实现真正的秒级冷启动与零损耗集群动态再均衡。

#### 维度：`streaming_failover_resilience`

*维度说明：长连接流式输出（SSE/gRPC Streaming）长达数十秒过程中，Pod 崩溃或被抢占时的容灾与重试机制*

针对超长文本流式生成场景提供工业级高韧性容错：1. 会话状态解耦存储：长文本会话中每步生成的 KV 状态均由 Mooncake Store 进行分布式缓冲与异步归档；2. 节点崩溃秒级自愈：若负责解码的 Decode Pod 突发硬件挂起或被 K8s 抢占，Conductor 快速探知并在百毫秒内将该流式上下文无缝迁移调度至同规格的备用 Decode Pod；备用 Pod 通过 RDMA 极速重载该会话在存储层中的历史 KV 缓存，从断点直接恢复自回归解码，保障前端 SSE/gRPC 连接不会因单点故障中断。

### 性能开销与落地可行性

#### 维度：`routing_overhead_and_latency`

*维度说明：分担决策自身引入的额外网络跳数、代理处理耗时（TTFT 影响）与 CPU/内存额外开销*

1. 调度控制面耗时：Conductor 作为高性能集中式调度器，单次路径与节点决策耗时通常在 13ms 之间；2. 跨网络缓存传输开销：Transfer Engine 采用 RDMA 零拷贝旁路传输，传输速率达 30GB/s50GB/s，百 KB 状态微秒级同步，数万 Token 的大容量 KV-Cache 传输仅耗费数十毫秒，远快于本地 GPU 重新进行 Prefill 计算所需的数百毫秒至数秒耗时；3. 整体端到端性能表现：在 Kimi 实际生产环境下，该架构将请求并发承载能力提升了 75%，在长文本模拟测试中吞吐量提升高达 525%，平均首字延迟（TTFT）与长尾延迟得到显著压降，显存利用率成倍提高。

#### 维度：`production_readiness_and_cases`

*维度说明：企业级生产落地成熟度、典型标杆案例与主流开源生态配套*

具备顶尖的工业生产级成熟度与学术标杆地位：1. 标杆生产案例：作为月之暗面（Moonshot AI）核心大模型应用‘Kimi’的底层基石生产架构，稳定支撑数亿次高并发长文本（200K~200万 Token）生产请求；2. 顶级学术荣誉：系统技术论文荣获存储系统顶级学术会议 USENIX FAST 2025 的最佳论文奖（Erik Riedel Best Paper Award）；3. 开源生态建设：在 GitHub（kvcache-ai/Mooncake）以 Apache 2.0 协议完全开源，2026年正式进入 PyTorch 官方生态体系，并与 vLLM、SGLang、TensorRT-LLM 深度集成，成为全行业分布式 KVCache 解耦存储与 PD 分离的标准典范。

- - -

## 7\. DeepSeek-V3/R1 专家并行负载均衡 (EPLB 与多副本协同)

> **架构分类**：MoE 专家并行负载均衡  
> **核心定位**：针对大模型 MoE 专家并行场景下的计算倾斜，通过冗余专家复制、动态通信重叠与多副本请求分发实现负载均衡

### 负载分担调度算法与决策核心

#### 维度：`load_balancing_algorithm`

*维度说明：具体分担算法逻辑（轮询/加权轮询、最小在飞请求 Least In-flight、基于前缀树 Radix Tree 的哈希匹配、一致性哈希与负载边界 CHWBL、双重映射 DualMap 等）*

DeepSeek-V3/R1 采用“宏观多副本集群路由 + 微观卡间专家并行负载均衡（EPLB）”的两层协同分担机制：

1. 宏观集群多副本路由分发：在多副本集群外层（如 SGLang Router、vllm-router 或自研高性能网关），基于 Prompt 前缀哈希感知（Prefix Cache-Aware）结合最小在飞请求（Least In-flight）或带边界的一致性哈希（CHWBL），将用户会话与长提示词分派给各个独立的 671B 分布式推理副本（单副本内部由 32/64 卡通过 EP=32/64 或 TP+EP 构成完整实例）；

2. 微观专家并行负载均衡算法（EPLB, Expert Parallelism Load Balancer）：
      - 冗余专家动态复制（Redundant Expert Replication）：MoE 架构（256 个路由专家 + 1 个共享专家，激活 top-8）存在严重的专家访问长尾倾斜，导致承载热点专家的 GPU 成为计算瓶颈（Straggler）。EPLB 实时/周期性统计专家激活频次，对访问频次极高的热点专家在不同 GPU 节点上创建冗余副本（例如由单卡独占扩展为 2~4 卡多副本），冷门专家则保持单副本；
      - 专家放置最优化装箱（Optimal Expert Placement）：根据统计数据求解装箱优化问题，提供分层负载均衡（Hierarchical Load Balancing，优先平衡节点间负载并优化节点内通信）与全局负载均衡（Global Load Balancing，跨集群全局统筹专家最优放置）两种策略；
      - 组受限选路与动态 Token 导流：对于存在多副本的专家，Gate 路由层在 Token 级选路时，根据各候选副本所在 GPU 的实时负载或哈希策略将 Token 导流至相对空闲的卡，彻底消除卡间计算倾斜；
      - 通信与计算双向重叠（DualPipe & DeepEP）：借助自研通信库 DeepEP，将 MoE 的跨卡跨机 Dispatch/Combine All-to-All 集合通信与前向密集注意力计算完全异步重叠，消除通信阻塞延迟。

#### 维度：`load_metrics_monitored`

*维度说明：分担决策所采纳的后端状态指标（活跃请求数 Running Requests、队列等待数 Waiting Requests、GPU 显存水位、KV Cache 块占用率、Token 生成速率）*

EPLB 与外层调度器协同采纳的多层次监控指标：

1. 专家级激活与计算负载指标：
      - expert\_hit\_count（各专家被 Gate 路由选择并激活的 Token 累计计数与滑动窗口命中率）；
      - expert\_computation\_time（单专家前向 GEMM 计算耗时）；
      - redundant\_expert\_load\_skew（冗余专家各副本间的 Token 分派偏差率）；

2. 跨卡通信与排队指标：
      - DeepEP 通信耗时与 All-to-All 缓冲区积压延迟（All-to-All Dispatch/Combine Latency）；
      - 各 EP 卡的等待队列长度与在飞 Token 批处理规模（Batched Tokens Skew）；

3. 显存与全实例水位指标：
      - 各 GPU 的 KV Cache 物理块分配率与激活显存占用；
      - 外层路由感知的整个 671B 副本组活跃请求数（Running Requests）与调度队列等待数（Waiting Requests）。

#### 维度：`dispatch_granularity`

*维度说明：请求分发粒度（TCP 连接级、HTTP/2 请求级、Token 生成 Chunk 级、Prefill/Decode 阶段级）*

呈现三级分发粒度协同：

1. L7 网关层：HTTP/2 与 REST/gRPC 请求级粒度。外部客户端与网关之间以单次 Prompt 请求或对话为调度单元，分发至特定 671B 分布式副本组；

2. 阶段级（PD 分离架构下）：Prefill 阶段与 Decode 阶段分发。大批量 Token 预填充交由专用的 Prefill EP 组计算，自回归单步生成交由 Decode EP 组执行；

3. 引擎内核专家并行层：细粒度 Token 级（Token-level）分发。在每个 Decoder 层的 MoE 前向过程中，每个 Token 的 Gate 路由器独立计算 Top-8 专家的概率权重，并通过跨卡/跨机 DeepEP All-to-All 通信，将单个 Token 的隐状态分派给对应的目标专家所在 GPU 执行，并在完成计算后通过 Combine 算子聚合并行结果。

### 状态亲和与负载均衡的动态权衡

#### 维度：`cache_affinity_vs_load_balance`

*维度说明：如何平衡‘提高 KV Cache / Prefix Caching 命中率（倾向发给已有缓存的同一 Pod）’与‘防止单 Pod 显存负载倾斜过载（倾向发给空闲 Pod）’*

在多副本集群中平衡 KV Cache 亲和性与专家负载防倾斜的核心机制包括：

1. 冲突本质剖析：若外层路由器为了最大化 Prefix Caching 命中率，将具有相同上下文（如长代码库或长系统提示词）的大量请求持续打发给同一个 671B 副本，尽管该副本内部 EPLB 能通过冗余专家抹平卡间倾斜，但整个副本的总吞吐与显存仍会迅速见顶，Waiting 队列积压剧增；

2. 双层防倾斜控制：
      - 外层副本调度引入带负载边界的一致性哈希（CHWBL）：设定副本队列上限阈值，当目标副本在飞请求或排队深度超过阈值（如均值 120%）时，强制打破缓存亲和，溢流转发至其他空闲的 671B 副本，以端到端时延为第一优先；
      - 副本内部 EPLB 自适应分流：即便是亲和分发的长 Prompt 集中在某副本，EPLB 的专家冗余复制也能保证不会因为代码/推理专家的局部高频调用而拖垮单张 GPU，使得单副本承载倾斜流量的弹性上限大幅提升。

#### 维度：`multi_lora_affinity_dispatch`

*维度说明：针对挂载多个微调 LoRA 的多副本，如何感知各 Pod 当前已驻留的 Adapter 权重并做亲和性负载分担，减少显存置换开销*

针对 671B 巨型 MoE 模型的多微调 LoRA 协同分担机制：

1. 权重切片同步挑战：在 MoE 结构中，微调 LoRA 往往针对所有专家或特定路由专家注入。由于单卡显存受限且基础权重跨数十张 GPU 切分，运行时动态换入换出 LoRA 开销极大；

2. 集中亲和调度：外层路由器根据请求中的 lora\_id 实施 Adapter 亲和调度（Adapter Affinity Routing），将相同微调任务的请求绑定到预先加载该 LoRA 切片的特定 671B 副本组；

3. 组内协同加载与缓存：各 EP Rank 在初始化或动态加载时同步拉取属于本卡专家的 LoRA 分片权重，配合 LRU 缓存保持活跃，避免不同 LoRA 频繁换入换出导致 All-to-All 通信等待中断。

### 解耦架构与流水线负载分流

#### 维度：`pd_disaggregated_load_sharing`

*维度说明：Prefill 阶段与 Decode 阶段的跨 Pod 流水线解耦分担机制，请求流转路径与跨节点 KV-Cache 传输网络协同（RDMA/RoCE/NCCL）*

Prefill 与 Decode 阶段解耦下的专家并行负载协同：

1. Prefill 与 Decode 节点异构分工：Prefill 阶段 Token 密度极高（计算密集型），热点专家引起的计算倾斜最为剧烈，Prefill EP 组中 EPLB 冗余专家复制与装箱优化带来的加速收益最高；Decode 阶段为访存与通信密集型（Token-by-token），更关注 All-to-All 通信的时延，DeepEP 的底层优化尤为关键；

2. 请求流转路径：客户端请求首先路由至 Prefill 副本组（如 4 节点 32 卡 EP=32），EPLB 高速完成所有 Prompt Token 的 MoE 计算与 KV Cache 生成；随后通过专用高速 RDMA 网络（RoCEv2/IB）将分布式 KV Cache 块传输写入指定的 Decode 副本组显存；

3. 跨机通信协同：采用 Mooncake / NIXL 连接器与 DeepEP，在跨节点通信中实现零拷贝 GPU Direct RDMA，极大缩短 KV 传输时间，使流水线负载平滑转交。

#### 维度：`chunked_prefill_concurrency`

*维度说明：变长长文本 Prompt 的分块预填充（Chunked Prefill）对多副本并发负载平滑分担的支撑效果*

分块预填充（Chunked Prefill）在 DeepSeek-V3/R1 MoE 场景下的平滑支撑效果：

1. 削平 MoE 专家突发冲击：在未开启 Chunked Prefill 时，数万 Token 的超长 Prompt 会瞬间向特定热点专家发送上万个激活 Token，导致对应 GPU 的 All-to-All 接收缓冲区与矩阵乘算子发生突发尖刺阻塞；

2. 平滑分担效果：通过将长 Prompt 切分为固定规格（如 512 或 1024 Tokens）的 Chunk 分步执行，将对热点专家的并发计算与跨卡传输分散到多个迭代步中，使得 GPU 算力利用率和通信网络利用率维持在平缓的高位，有效降低了跨请求之间的尾部抖动。

### 过载保护、反压与容灾伸缩

#### 维度：`overload_protection_and_admission`

*维度说明：单 Pod 即将显存击穿（OOM）或队列满时的准入控制（Admission Control）、排队缓冲与溢流分流（Spillover）机制*

全链路过载保护与准入控制屏障：

1. 专家级 Capacity Factor 与溢流保底：Gate 路由设定专家的最大容量因子（Expert Capacity Factor）。当分派给某一专家副本的 Token 数量超过容量阈值时，触发溢流保底策略（丢弃多余 Token 计算、Fallback 至备选副本或回退至共享专家），防止单卡因 Token 过多而显存溢出（OOM）或严重拖慢整体步频；

2. 副本级队列反压与准入控制：Leader 协调器或引擎调度器监控分布式显存可用块与队列深度。当并发请求超过 max\_num\_seqs 或显存利用率触及安全红线（如 0.92）时，执行准入截留，并在前端快速返回 HTTP 429 反压；

3. 网关层熔断与溢流重试：前端路由器监测各 671B 副本的排队等待时长与错误率，触发断路器机制将新流量平滑切换至集群内其他轻载实例。

#### 维度：`k8s_autoscaling_and_rebalancing`

*维度说明：与 Kubernetes HPA/KEDA 水平伸缩的联动，扩容新增 Pod 时的冷启动预热，以及缩容/故障时的流量优雅下线（Drain）与平滑再均衡*

在 Kubernetes 环境下的弹性伸缩与拓扑协同：

1. 以整组超节点（Super-Pod）为单位扩缩容：由于 DeepSeek-V3/R1 单副本通常跨越 4 台 8 卡节点（共 32 卡）或 8 台 8 卡节点（共 64 卡），K8s HPA/KEDA 的伸缩粒度不能是单 Pod，而必须借助 LeaderWorkerSet（LWS）以整组 4~8 个 Pod 为不可分割单元进行协同伸缩；

2. 指标联动：通过 Prometheus/KEDA 监控集群整体 Token 吞吐与副本组排队等待数联动触发扩缩容；

3. 启动预热与拓扑重平衡：扩容启动新 671B 副本组时，由于权重高达数百 GB，通过分布式共享文件系统（如 JuiceFS/GPFS）结合 RoCE 预加载，并通过 StartupProbe 完成 CUDA Graph 与 EPLB 初始专家装箱热身，最后上线挂入 Service；

4. 缩容优雅 Drain：缩容时向 Leader Pod 发送 SIGTERM，LWS 确保在长达 180~300 秒的优雅窗口内处理完存量流式生成，再统一回收全组 Pod。

#### 维度：`streaming_failover_resilience`

*维度说明：长连接流式输出（SSE/gRPC Streaming）长达数十秒过程中，Pod 崩溃或被抢占时的容灾与重试机制*

针对长思维链推理（R1 长推理数十秒至数分钟）的长连接容灾：

1. 状态绑定与崩溃处理：DeepSeek-R1 推理具有极长思维链（CoT 往往输出数千至上万 Token），中间状态跨 32~64 卡深度分布式绑定。若组内任意单卡崩溃或发生 ECC 错误，整个分布式副本组无法在显存级别无缝单卡热迁；

2. 外层网关与客户端透明重试：网关（如 SGLang Router / Envoy）检测到与当前副本组连接异常断开后，立即介入，将原始 Prompt 以及已安全生成的断点内容重新封装，以新请求形式分派至其他健康的 671B 副本组；

3. 分布式前缀缓存秒级续推：得益于各副本间共享的分布式缓存层（如 Mooncake / LMCache），备用副本可直接秒级复用已生成的历史 Token KV Cache，仅需重算极少量的末端 Token 即可无缝继续流式吐字。

### 性能开销与落地可行性

#### 维度：`routing_overhead_and_latency`

*维度说明：分担决策自身引入的额外网络跳数、代理处理耗时（TTFT 影响）与 CPU/内存额外开销*

负载分担与路由决策的性能开销：

1. EPLB 装箱优化耗时：EPLB 的专家复制与装箱算法属于离线或准动态运行（通常按批次或分钟级周期根据滑动窗口统计重新计算专家重排策略），算法求解耗时仅为数毫秒至数十毫秒，完全在后台异步执行，对线上推理无运行时阻塞；

2. 运行时 Token 路由延迟：前向计算中的 Gate 路由（门控网络 GEMM 与 Top-8 排序）耗时极低（通常仅需微秒级），主要开销在于跨卡跨节点的 All-to-All 通信。通过 DeepEP 优化的 RDMA 原语与 DualPipe 通信隐藏，跨机通信开销被绝大部分重叠进注意力与密集计算中；

3. 外层代理网络开销：外层 SGLang Router 或专用代理引入的网络中转耗时约为 0.3ms - 0.8ms，相较于 671B 模型的整体首字延迟（TTFT 通常在 200ms - 800ms），代理开销占比小于 0.2%，具有极高执行效率。

#### 维度：`production_readiness_and_cases`

*维度说明：企业级生产落地成熟度、典型标杆案例与主流开源生态配套*

生产落地成熟度与典型生态应用：

1. 官方与开源生态支撑：DeepSeek 官方开源 EPLB 算法与 DeepEP 高性能通信库，并已被主流大模型推理框架 SGLang、vLLM 以及 TensorRT-LLM 官方深度集成，成为 671B MoE 模型标准部署优化方案；

2. 标杆生产落地案例：
      - DeepSeek 官方线上 API 与网页端生产集群全面依赖 EPLB 与 DualPipe 支撑亿级日活调用；
      - 阿里云、腾讯云、火山引擎、百度智能云等头部公有云均基于 K8s + LWS + EPLB 架构推出了 DeepSeek-V3/R1 专属推理加速服务与算力解决方案；
      - 金融、科研等私有化大模型算力中心广泛采用该方案解决 671B 跨机推理的卡间负载严重失衡难题。

- - -

## 8\. LeaderWorkerSet (LWS) 超大模型跨节点分布式副本组负载分担

> **架构分类**：多机分布式副本拓扑  
> **核心定位**：针对 70B/405B 等跨多机张量/流水线并行的大模型副本，实现多 Pod 协同生命周期管理与入口流量仅向 Leader Pod 分派的负载均衡

### 负载分担调度算法与决策核心

#### 维度：`load_balancing_algorithm`

*维度说明：具体分担算法逻辑（轮询/加权轮询、最小在飞请求 Least In-flight、基于前缀树 Radix Tree 的哈希匹配、一致性哈希与负载边界 CHWBL、双重映射 DualMap 等）*

LeaderWorkerSet (LWS) 是 Kubernetes SIG 官方推出的分布式大模型推理与训练协同工作负载控制器，其负载分担核心遵循“组内紧耦合并行计算 + 组间多副本入口收敛负载分派”机制：

1. 组间多副本入口路由分发：LWS 将超大模型（如 Llama-3-405B、DeepSeek-671B、Qwen-72B 多机 TP/PP）部署为多个相互独立的分布式副本组（Groups）。每个组包含 1 个 Leader Pod 与 N 个 Worker Pod。前端负载均衡器（K8s Service / Ingress / Gateway API / Envoy / vllm-router）通过 Service Selector 严格过滤标签 leaderworkerset.sigs.k8s.io/worker-index: "0"，只将外部网络流量分发至各个组的 Leader Pod；分担算法可采用加权轮询（Weighted Round-Robin）、最小在飞请求（Least In-flight）、基于前缀树的哈希路由（Radix-Tree Hash / Prefix Cache Aware）或根据 Leader 汇报的组内综合负载打分；

2. 组内跨机协同计算负载分摊：外部流量仅打入 Leader Pod，Leader Pod 内部作为计算主节点（运行 HTTP API Server、分布式调度器如 Ray Head 或 Torchrun Rank 0），接收到请求后，通过内部高性能分布式互连（NCCL/MPI/gRPC/Ray GCS），按照张量并行（Tensor Parallelism）切片、流水线并行（Pipeline Parallelism）阶段或专家并行（Expert Parallelism）将计算任务分发给本组内的各个 Worker Pod 协同执行。Worker Pod 仅负责计算分摊，完全对外部流量不可见。

#### 维度：`load_metrics_monitored`

*维度说明：分担决策所采纳的后端状态指标（活跃请求数 Running Requests、队列等待数 Waiting Requests、GPU 显存水位、KV Cache 块占用率、Token 生成速率）*

分担决策所采纳的状态指标呈现全组聚合特征，均由 Leader Pod 对外统一暴露：

1. 组内聚合在飞与队列指标：
      - num\_requests\_running（当前整个分布式副本组正在执行模型迭代的并发请求数）；
      - num\_requests\_waiting（调度器中排队等待分配计算资源的请求数）；

2. 组内多机 GPU 显存与 KV 状态：
      - 全组 GPU 显存峰值与均值占用率（由 Leader 通过心跳汇集各 Worker 显存）；
      - 分布式 KV Cache 块占用比例（gpu\_cache\_usage\_factor），反映全组是否接近显存饱和；

3. 节点与网络拓扑健康指标：
      - 组内各 Worker Pod 的网络连通性与 RDMA 通信健康探针（通过 Leader 的 Headless Service 周期性自检）；
      - Token 生成速率与单次前向 step 耗时，反映整组是否存在跨机拖后腿节点（Straggler）。

#### 维度：`dispatch_granularity`

*维度说明：请求分发粒度（TCP 连接级、HTTP/2 请求级、Token 生成 Chunk 级、Prefill/Decode 阶段级）*

内外两层严格解耦的调度粒度：

1. 外部流量分发粒度：标准的 HTTP/2 或 gRPC 请求级（Request-level）粒度。外部负载均衡器不对请求做任何切片，将完整客户端请求直接转发至目标副本组的 Leader Pod；

2. 阶段级（在配合 DisaggregatedSet 时）：当多组 LWS 分别充当 Prefill 集群组与 Decode 集群组时，外部网关根据 Prefill 与 Decode 阶段进行两阶段跨组分流；

3. 组内执行粒度：微观算子与张量分片级（Layer / Tensor Chunk 级）。Leader Pod 将请求解析为 Token 序列后，在每一次模型前向迭代中，通过底层 NCCL 集合通信将张量矩阵分块分摊至各个 Worker Pod，走跨卡 RoCE/IB 网络进行 AllReduce / All-to-All 运算，外部网关完全不干涉该内部粒度。

### 状态亲和与负载均衡的动态权衡

#### 维度：`cache_affinity_vs_load_balance`

*维度说明：如何平衡‘提高 KV Cache / Prefix Caching 命中率（倾向发给已有缓存的同一 Pod）’与‘防止单 Pod 显存负载倾斜过载（倾向发给空闲 Pod）’*

平衡多分布式组间的缓存亲和与显存负载倾斜：

1. 组级 KV Cache 隔离现状：由于每个 LWS 副本组是跨多节点的分布式独立引擎（其 KV Cache 分布式切分在组内所有 Worker 的 GPU 显存中），不同 LWS 副本组之间的物理显存默认不互通；

2. 亲和分担策略：网关层针对具有相同 Prompt 前缀或会话 ID 的请求，通过一致性哈希倾向分发至同一个 LWS 副本组的 Leader Pod，以最大化复用该组内已构建的分布式 Prefix Caching；

3. 负载边界门控（CHWBL）：为防止特定前缀请求暴增导致某一 LWS 副本组显存击穿，网关设定在飞请求与显存边界阈值。当主亲和 LWS 组负载越界时，网关触发溢流（Spillover），将后续请求调度给其它轻载 LWS 副本组的 Leader Pod，在保证集群吞吐的前提下进行自适应再平衡。

#### 维度：`multi_lora_affinity_dispatch`

*维度说明：针对挂载多个微调 LoRA 的多副本，如何感知各 Pod 当前已驻留的 Adapter 权重并做亲和性负载分担，减少显存置换开销*

超大模型跨机多 LoRA 协同分发机制：

1. 权重切片同步挑战：在 70B/405B 等超大模型中，基础模型权重跨多机切分（如 TP=16），对应的 LoRA Adapter 权重切片也必须在组内 Leader 和所有 Worker Pod 上协同一致加载；

2. 集中亲和调度：网关识别请求中的 LoRA 标识，通过 LoRA 亲和路由（LoRA Affinity Routing）将请求直接分派到当前已经加载了该 LoRA 的 LWS 副本组 Leader；

3. 组内协同热加载：若必须加载新 LoRA，Leader Pod 接收请求后向所有 Worker Pod 广播加载指令，所有 Worker 同步从共享存储加载对应分片，避免了多 Pod 加载状态不一致导致的分布式计算死锁。

### 解耦架构与流水线负载分流

#### 维度：`pd_disaggregated_load_sharing`

*维度说明：Prefill 阶段与 Decode 阶段的跨 Pod 流水线解耦分担机制，请求流转路径与跨节点 KV-Cache 传输网络协同（RDMA/RoCE/NCCL）*

LWS 在 PD 分离解耦架构中的协同分担机制：

1. 拓扑编排：借助 Kubernetes SIG 的更高层 API DisaggregatedSet (DS)，分别定义两套 LWS 资源：一组 Prefill LWS 组（通常配置更高并行度与高算力 GPU 节点）和一组 Decode LWS 组（配置高显存带宽与更精简并行度的节点）；

2. 协同分担路径：外部 Router 将请求首发给 Prefill LWS 组的 Leader Pod，Leader 协同其 Worker 完成超大模型的并行 Prefill 计算，并将分布式生成的 KV Cache 通过 GPUDirect RDMA 直接写入目标 Decode LWS 组对应 Worker 的 GPU 显存；

3. 流水线交接：Prefill 完成后，Router 将请求转交至 Decode LWS 组的 Leader Pod，由 Decode 组接管后续的自回归单 Token 生成，实现超大模型跨机集群的高效流水线解耦分担。

#### 维度：`chunked_prefill_concurrency`

*维度说明：变长长文本 Prompt 的分块预填充（Chunked Prefill）对多副本并发负载平滑分担的支撑效果*

分块预填充对分布式多机多卡负载的平滑作用：

1. 缓解跨机集合通信拥塞：在 70B/405B 模型中，跨机张量并行需要频繁执行跨节点的 NCCL AllReduce。长文本 Prompt 如果全量一次性 Prefill，会造成持续数百毫秒的跨机网卡满载风暴与计算阻塞；

2. 并发平滑效果：Chunked Prefill 将超长 Prompt 拆分为小批量 Chunk，使跨机通信与 GPU 计算平摊到各个微步迭代中，显著抑制了跨机通信长尾延迟（Straggler Effect），使得整组副本对外表现出更平稳的吞吐率和更低的首字延迟抖动。

### 过载保护、反压与容灾伸缩

#### 维度：`overload_protection_and_admission`

*维度说明：单 Pod 即将显存击穿（OOM）或队列满时的准入控制（Admission Control）、排队缓冲与溢流分流（Spillover）机制*

以 Leader 为核心的组级准入控制与过载保护：

1. Leader 统一度量门控：Leader Pod 汇集全组 Worker 的实时状态，掌握全组真实的剩余 KV Cache 块数量。当全组显存达到预设水位（如 90%）或等待队列超过最大限制时，Leader 在入口处执行硬准入控制（Admission Control），拦截后续请求进入推理计算管线；

2. 显存防击穿与抢占：若在飞 Decode 请求导致组内任一 Worker 显存即将击穿，Leader 触发全局抢占调度（Recompute 或 Swap 策略），组内各 Worker 协同丢弃或换出指定请求的局部显存块；

3. 快速反压与外部溢流：Leader Pod 阻断请求时，直接向客户端或前置网关返回 HTTP 429 / 503 状态码。前置网关捕获后立即触发限流与溢流逻辑，将后续请求调度给其余具备计算冗余的 LWS 副本组。

#### 维度：`k8s_autoscaling_and_rebalancing`

*维度说明：与 Kubernetes HPA/KEDA 水平伸缩的联动，扩容新增 Pod 时的冷启动预热，以及缩容/故障时的流量优雅下线（Drain）与平滑再均衡*

LWS 与 Kubernetes 弹性体系的深度整合：

1. 组级协同扩缩容（Gang Scheduling / All-or-Nothing）：通过扩展子资源（Scale Subresource），LWS 原生支持 HPA 与 KEDA 联动调节 spec.replicas（副本组数量）。扩容时，必须保证 1 个 Leader Pod 与 N 个 Worker Pod 所需的 GPU 资源全部就绪才统一调度启动（配合 Kueue / Volcano 实现 Gang Scheduling），避免产生资源碎片；

2. 依赖注入与自动化无头服务发现：LWS 自动为每个组建立独立的 Headless Service，并在 Pod 内部注入环境变量 LWS\_LEADER\_ADDRESS，Worker 启动时自动与 Leader 握手组网，大幅简化分布式启动流程；

3. 优雅缩容与下线（Drain）：缩容触发时，LWS 控制器以整组为单位进行销毁。Leader Pod 收到 SIGTERM 信号后自动向 Service 探针标记 Unready，从前置网关的负载分担 Endpoints 中移除，在配置的优雅窗口期（terminationGracePeriodSeconds）内处理完组内在飞请求后整组退出。

#### 维度：`streaming_failover_resilience`

*维度说明：长连接流式输出（SSE/gRPC Streaming）长达数十秒过程中，Pod 崩溃或被抢占时的容灾与重试机制*

针对分布式组长连接流式输出的容灾与自愈：

1. 组内同生共死生命周期策略（All-or-Nothing Restart Policy）：由于跨节点 TP/PP 并行高度依赖 NCCL 通信，组内任意 1 个 Worker 出现硬件故障、OOM 或网络掉线都会导致全组死锁。LWS 提供了 restartPolicy: RecreateGroupOnPodRestart，一旦组内单 Pod 异常退出，控制器自动重启或重建整组 Pod，避免悬挂与僵尸进程；

2. 网关层透明重试：在流式生成期间若 Leader 因整组故障断开连接，前置负载均衡代理立即感知，向调用方提供容灾兜底，并将请求重定向至备用健康的 LWS 副本组；

3. 结合分布式持久缓存快速恢复：重试请求打入新 LWS 副本组 Leader 后，若启用了跨机分布式 KV 缓存机制，可快速从外部共享存储中拉取已生成的上下文，实现极短延迟内的流式接续生成。

### 性能开销与落地可行性

#### 维度：`routing_overhead_and_latency`

*维度说明：分担决策自身引入的额外网络跳数、代理处理耗时（TTFT 影响）与 CPU/内存额外开销*

LWS 架构对系统性能与网络延迟的影响分析：

1. 零额外数据面代理开销：LWS 本质是 Kubernetes 控制面 CRD 与 Operator，完全不介入推理数据面。外部请求经过 K8s Service / Ingress 仅需一跳直接打入 Leader Pod（单跳网络延迟通常 < 0.5ms），与常规单 Pod 部署毫无差别，数据面路由开销为零；

2. 组内通信延迟优化：组内通信完全由专用的底层物理网络（RoCEv2 / InfiniBand）承载，利用 Leader 自动生成的内部 Headless DNS 与专用虚拟子网通信，绕过 K8s iptables/ipvs 虚拟网络损耗；

3. 控制器轻量级开销：LWS Controller 仅负责监听 CRD 状态并管理 Pod 生命周期，CPU 与内存开销极小（单副本占用通常在几十 MB 级别），不占用 GPU 宿主机的宝贵算力。

#### 维度：`production_readiness_and_cases`

*维度说明：企业级生产落地成熟度、典型标杆案例与主流开源生态配套*

生产落地成熟度与主流行业实践：

1. Kubernetes SIG 官方标准项目：由 Google Cloud、Red Hat 等顶级云原生团队联合发起并捐赠给 Kubernetes SIG 官方维护，专门解决 AI/ML 分布式多机容器编排标准缺失痛点；

2. 顶尖推理引擎官方深度集成：vLLM、SGLang、Ray Train/Serve、Triton Inference Server 官方文档均将 LWS 推荐为多机多卡分布式部署的首选标准方案；

3. 行业标杆案例：Google Cloud GKE 将 LWS 作为大规模 TPU/GPU 集群运行 70B/405B 大模型的官方白皮书参考架构；国内外主流 AI 算力平台与智算中心（如字节跳动、阿里巴巴、各大云厂商 AI 容器平台）在大规模落地多机推理集群时普遍采用 LWS 进行生产级运维编排。

- - -

## 9\. L7 网格与反向代理负载分担 (Envoy / Istio / Nginx)

> **架构分类**：反向代理与服务网格  
> **核心定位**：分析 Envoy/Istio 基于 Least Request 与 gRPC/SSE 多路复用的七层分流表现，及其无法感知显存与 Token 长度的局限

### 负载分担调度算法与决策核心

#### 维度：`load_balancing_algorithm`

*维度说明：具体分担算法逻辑（轮询/加权轮询、最小在飞请求 Least In-flight、基于前缀树 Radix Tree 的哈希匹配、一致性哈希与负载边界 CHWBL、双重映射 DualMap 等）*

Envoy/Istio 与 Nginx 在七层负载均衡中采用经典的微服务分流算法，但面对 AI 大模型推理时暴露出显著的适用性断层：

1. 核心分流算法逻辑：
      - 最少请求数（Least Request / P2C，Power of Two Choices）：Envoy/Istio 默认的主力分担算法，通过随机采样两个后端实例并选择当前在飞请求数（Active Requests）较少的一个进行转发；
      - 轮询与加权轮询（Round Robin / Weighted Round Robin）：Nginx 与 Envoy 基础分担算法，按固定或静态加权顺序轮替派发请求；
      - 一致性哈希（Ring Hash / Maglev）：依据 HTTP Header（如 session-id、API Key、User-Id）对后端 Pod 环进行散列，用于实现无状态的长连接会话保持；
      - 最少连接数（least\_conn）：Nginx 监控当前活跃 TCP/HTTP 连接数进行分流。

2. AI 推理场景下的算法失效与困境：
      - 算力与负载不均假象：传统微服务请求耗时在毫秒级且方差小，Least Request 效果极佳；但大模型推理是长周期流式生成（数十秒），且计算开销受 Prompt Token 长度（Prefill 矩阵计算量 O(N^2)）与生成 Token 数量（Decode 显存与访存带宽）极端决定。一个正在处理 32K 上下文长文本的 Pod 与一个处理 50 Token 短问答的 Pod，显存占用与计算压力可能有数十倍差距。Least Request 仅将二者均视作‘1 个在飞请求’，甚至会将新请求盲目派发给处于显存崩溃边缘的长文本处理节点，引发灾难性排队或 OOM；
      - 缺少前缀树哈希支持：原生反向代理不支持对请求体（JSON Prompt）做分词（Tokenize）和 Radix Tree 树形哈希匹配，无法将具有公共系统前缀的请求聚合调度；

3. 扩展增强尝试：业界常借助 Envoy 的 ext-proc（External Processor filter）或 WebAssembly（Wasm）扩展插件，在 Envoy 接收到请求时异步回调外部智能调度器（如 K8s SIG Inference Gateway）获取路由决策，但在高 QPS 下会带来额外开销。

#### 维度：`load_metrics_monitored`

*维度说明：分担决策所采纳的后端状态指标（活跃请求数 Running Requests、队列等待数 Waiting Requests、GPU 显存水位、KV Cache 块占用率、Token 生成速率）*

Envoy/Istio/Nginx 原生分担决策采纳的指标完全局限于 L4 传输层与 L7 应用层网络指标：

1. 采纳的网络层度量指标：
      - 活跃在飞请求数（upstream\_rq\_active）：当前代理与各 upstream Pod 正在交互中的并发请求计数；
      - 活跃连接数（upstream\_cx\_active）：维护中的 TCP/HTTP2 连接数量；
      - 历史响应时延（RTT / Peak EWMA）：部分自适应算法依据近期请求的历史响应时间滑动均值做权重调整；
      - 连接异常与 5xx 错误率：用于主动剔除异常 Pod 的健康度量。

2. 无法感知的推理运行时核心盲区：
      - 显存水位与物理块占用：完全无法感知各 Pod GPU 显存利用率、PagedAttention 物理 KV-Cache 块占用率（如 vllm:gpu\_cache\_usage\_factor）；
      - 引擎等待队列（Waiting Queue）：无法洞察推理引擎内部处于排队挂起状态的请求堆积数（如 vllm:num\_requests\_waiting），造成代理认为 Pod 正在快速处理但实际排队延迟呈指数爆炸；
      - Token 生成动态速率：无法获知 Decode 阶段的每秒生成 Token 速率与显存带宽饱和度；
      - 请求计算复杂度（Prompt Length / Max Tokens）：默认不解析 Request Body 中的 Token 长度，将极长 Prompt 与极短 Prompt 等同视之，产生严重评估盲区。

#### 维度：`dispatch_granularity`

*维度说明：请求分发粒度（TCP 连接级、HTTP/2 请求级、Token 生成 Chunk 级、Prefill/Decode 阶段级）*

L7 网格与反向代理的请求分发粒度体现为应用协议级调度：

1. HTTP/2 与 gRPC 请求/流级（Stream-level）粒度：
      - Envoy 原生具备深度 HTTP/2 与 gRPC 协议栈支持，支持单个 TCP 连接内的多路复用（Multiplexing）。外部客户端可以在单个长连接中发起多个并行 gRPC 调用，Envoy 能够将每个独立的 Stream 精确解复用并分派给不同的后端 Pod，彻底解决了 L4（iptables/IPVS）负载均衡中单个长连接被焊死在单 Pod 的致命缺陷；
      - Nginx 在支持 grpc\_pass 时同样能实现请求级代理分发，维持上游动态连接池。

2. SSE（Server-Sent Events）长连接管道级分发：
      - 针对大模型主流的 HTTP POST /v1/chat/completions（stream=true），代理与选定 Pod 建立单向流式通道，逐个分块（Chunk）透明转发 Token 文本流，直至接收到 \[DONE\] 标记或连接正常关闭。

3. 推理底层解耦粒度的绝对缺失：
      - 无法实现 Prefill 阶段与 Decode 阶段的细粒度流水线解耦调度（PD 分离），外部代理将一次完整的流式生成视为原子黑盒事务；
      - 无法实现 Token Chunk 级的动态跨卡或跨 Pod 协同迁移。

### 状态亲和与负载均衡的动态权衡

#### 维度：`cache_affinity_vs_load_balance`

*维度说明：如何平衡‘提高 KV Cache / Prefix Caching 命中率（倾向发给已有缓存的同一 Pod）’与‘防止单 Pod 显存负载倾斜过载（倾向发给空闲 Pod）’*

在多副本推理集群中，Envoy/Istio/Nginx 在 KV-Cache 亲和性与防负载倾斜之间存在天然难以调和的矛盾：

1. 缓存亲和性感知能力极弱：
      - 原生代理不解构 Request Payload，无法提取共享的 System Prompt、Few-shot 示例或 RAG 文档前缀；
      - 仅能依赖 Ring Hash / Maglev 策略，根据客户端 IP、User-ID 或 Session-ID 请求头做哈希路由。但大模型缓存亲和的核心是‘Prompt 文本内容前缀重合’而非‘同一用户’。不同用户访问相同文档无法命中缓存，同一用户的不同提问却被硬绑定在同一 Pod，导致集群 Prefix Cache 命中率极其低下，重复 Prefill 计算严重浪费 GPU 算力。

2. 负载倾斜与边界防范不足：
      - 即使采用基于请求头的一致性哈希，若某特定业务 Key 突发热点流量，原生哈希机制会将该 Key 的全部流量压向单一 Pod，导致该 Pod 显存耗尽并剧烈排队；
      - 尽管 Envoy 支持带负载边界的一致性哈希（Consistent Hashing with Bounded Loads），但其负载边界计算仅依据‘活跃连接数’，无法获知真实的显存块剩余情况，极易出现代理判定未过载而 Pod 内部 KV-Cache 已经击穿的现象。

#### 维度：`multi_lora_affinity_dispatch`

*维度说明：针对挂载多个微调 LoRA 的多副本，如何感知各 Pod 当前已驻留的 Adapter 权重并做亲和性负载分担，减少显存置换开销*

针对挂载多个微调 LoRA Adapter 的多副本 Pod 集群：

1. 静态请求头匹配分流：
      - Envoy RouteConfiguration 与 Istio VirtualService 支持精细的 HTTP Header 匹配规则。若客户端或网关在请求头中显式传入特定字段（如 `x-model-adapter: lora-medical-v1`），代理可将其定向分发到挂载有对应 LoRA 的特定 Subset Pod 集群；

2. 动态驻留感知的完全缺失：
      - 推理引擎（如 vLLM/SGLang）支持在运行时通过 LRU 策略在 GPU/CPU 之间动态换入换出数十个 LoRA Adapter。原生 Envoy/Istio 无法感知各 Pod 当前实时驻留了哪些 Adapter 权重；
      - 若不作静态子集拆分，随机或轮询分发会导致请求被随机打到不同 Pod，各 Pod 被迫高频触发 LoRA 权重从 CPU/磁盘到 GPU 显存的冷加载与置换（Thrashing 抖动），导致推理延迟发生秒级长尾尖刺。

### 解耦架构与流水线负载分流

#### 维度：`pd_disaggregated_load_sharing`

*维度说明：Prefill 阶段与 Decode 阶段的跨 Pod 流水线解耦分担机制，请求流转路径与跨节点 KV-Cache 传输网络协同（RDMA/RoCE/NCCL）*

在 Prefill-Decode 解耦（PD 分离）架构中的协同能力：

1. 原生两阶段流转机制不支持：
      - Envoy / Istio / Nginx 遵循标准的客户端-服务端单次请求-响应代理范式。PD 分离架构要求请求先在 Prefill Pod 完成提示词前向计算与 KV Cache 构造，通过高速 RDMA/RoCE 互连传输至 Decode Pod，再由 Decode Pod 持续流式吐字返回给客户端。通用 L7 代理完全不具备这种跨异构 Pod 的两阶段状态机编排与传输握手跟踪能力；

2. 工业界折中架构：
      - L7 代理仅作为外层统一接入点，接收请求后转发给专门的外部协调调度器（如 Mooncake、vLLM Disaggregated Router 或独立 API Server），由专用调度器完成跨 Prefill 池与 Decode 池的流水线分担；代理层本身不对两阶段数据流作直接拆分。

#### 维度：`chunked_prefill_concurrency`

*维度说明：变长长文本 Prompt 的分块预填充（Chunked Prefill）对多副本并发负载平滑分担的支撑效果*

分块预填充（Chunked Prefill）对 L7 代理层负载分担的间接影响：

1. 消除超长计算导致的代理超时断流：
      - 在未开启 Chunked Prefill 时，数万 Token 的超长 Prompt 会独占 GPU 计算数秒甚至十数秒，导致同 Pod 内其他在飞请求的流式 Token 输出完全停滞，极易触发 Envoy 的 `stream_idle_timeout` 或 Nginx 的 `proxy_read_timeout` 导致连接被代理层异常掐断；
      - 开启 Chunked Prefill 后，后端 Pod 每次执行均被切片为固定尺寸（如 512/2048 Tokens）的平滑批次，请求持续获得吐字推进，彻底杜绝了代理层因假死而误判 Pod 故障；

2. 平滑 Least Request 决策曲线：
      - 避免了单个长 Prompt 霸占算力导致的响应延迟剧烈波动，使 Envoy 基于历史 RTT 和活跃请求数的决策更加稳定，减少调度震荡。

### 过载保护、反压与容灾伸缩

#### 维度：`overload_protection_and_admission`

*维度说明：单 Pod 即将显存击穿（OOM）或队列满时的准入控制（Admission Control）、排队缓冲与溢流分流（Spillover）机制*

Envoy 与 Istio 具备成熟全面的通用网络级准入控制与过载保护机制，但与 AI 显存保护存在脱节：

1. 成熟的 L7 熔断与流量节流（Circuit Breaking）：
      - Envoy 支持严格限制每个集群的 `max_connections`、`max_pending_requests`、`max_requests` 与 `max_retries`。当等待队列或活跃请求超过阈值时，直接在代理层快速拒绝并返回 HTTP 503 Service Unavailable 或 429 Too Many Requests，阻断流量穿透到后端 Pod；

2. 异常点检测与故障剔除（Outlier Detection）：
      - 当某个推理 Pod 因 CUDA OOM、显存崩溃或内部死锁频繁返回 HTTP 5xx 错误或连接超时时，Envoy 的异常检测机制可自动将该 Pod 从上游负载均衡集群中驱逐（Ejection）一段冷却时间，避免流量持续落入故障节点；

3. 自适应并发控制（Adaptive Concurrency Limit）：
      - Envoy 内置自适应并发过滤插件，基于梯度延迟采样算法监控 RTT 变化，在后端排队延迟飙升时主动限流；

4. 核心局限：
      - 所有的准入控制阈值均为静态网络连接/请求并发数，无法动态映射到后端模型动态可变的高显存消耗。若突发数个超长文本并发，即便请求数远低于 Envoy 的 max\_requests 限制，依然会导致后端 GPU 显存瞬间击穿。

#### 维度：`k8s_autoscaling_and_rebalancing`

*维度说明：与 Kubernetes HPA/KEDA 水平伸缩的联动，扩容新增 Pod 时的冷启动预热，以及缩容/故障时的流量优雅下线（Drain）与平滑再均衡*

与 Kubernetes 生态弹性伸缩及生命周期管理紧密契合：

1. 指标暴露与 HPA/KEDA 联动：
      - Envoy/Istio 暴露详尽细致的 Prometheus 指标（如 `envoy_cluster_upstream_rq_active`、`istio_requests_total`、`response_duration_seconds`）。运维团队可基于网关聚合的实时在飞请求数或 P95/P99 延迟配置 KEDA ScaledObject，实现比原生 K8s 基于 CPU/Memory 利用率敏锐得多的水平弹性伸缩；

2. 服务发现与热更新（EDS）：
      - Envoy 深度集成 K8s 端点发现服务（Endpoint Discovery Service, EDS），实时同步 Endpoints/EndpointSlice 变更。当推理 Pod 扩容并完成权重加载、就绪探针（Readiness Probe）变为 Ready 后，Envoy 毫秒级将其纳入集群分发池，零流量丢失且无需 reload 进程；

3. 缩容优雅下线（Drain）：
      - 当 Pod 收到终止信号（SIGTERM）时，Istio/Envoy 启动连接排空（Drain）流程：停止接收新请求，同时对现存活跃的长连接流式输出（SSE/gRPC）提供长达数分钟的宽限时间（配合 terminationGracePeriodSeconds），等待流式传输自然完成，实现缩容平滑再均衡。

#### 维度：`streaming_failover_resilience`

*维度说明：长连接流式输出（SSE/gRPC Streaming）长达数十秒过程中，Pod 崩溃或被抢占时的容灾与重试机制*

针对数十秒流式长连接（SSE/gRPC Streaming）运行中 Pod 崩溃时的容灾表现：

1. 流中故障不可重试原则：
      - 大模型流式输出具有强时序状态。在 Envoy/Nginx 的 HTTP 规范中，一旦后端 Pod 已经返回了初始响应头（Response Headers）或者已经向客户端流式输出了部分 Token 数据包，代理即判定该事务已进入‘已提交（Committed）’流状态；
      - 若此时后端 Pod 发生 OOM 崩溃、Pod 被抢占或网络中断，Envoy 严格禁止将请求透明重试转移至其他 Pod（防止客户端收到重复或格式错乱的数据），直接向客户端发送连接重置（RESET\_STREAM）或 HTTP 502/503 异常中断；

2. 恢复成本与应用层兜底：
      - 客户端只能捕获异常后重新发起全量请求，造成前期已生成的 Token 算力全部白费；缺少应用层断点续传（Checkpoint Resumption）机制。

### 性能开销与落地可行性

#### 维度：`routing_overhead_and_latency`

*维度说明：分担决策自身引入的额外网络跳数、代理处理耗时（TTFT 影响）与 CPU/内存额外开销*

L7 代理自身引入的性能损耗与开销评估：

1. 网络跳数与首字延迟（TTFT）增量：
      - 集中式网关架构（如 Envoy Ingress Gateway）：引入一跳额外内部网络，RTT 增加约 0.2ms - 0.8ms；基于 C++ 开发的高性能异步事件模型（Epoll/Libevent），Envoy 自身的内部路由与解析处理延迟在微秒级（通常 < 0.5ms），相较大模型数百毫秒的 TTFT 而言几乎微不足道（开销占比 < 1%）；
      - Service Mesh Sidecar 拦截模式（如 Istio Sidecar）：每个请求需经过 iptables 两次拦截与 4 次内核态/用户态上下文切换（客户端 Envoy -> 服务端 Envoy），在高并发连续流式通信下会产生 2ms - 6ms 额外延迟并消耗可观的 CPU 资源，因此大规模大模型推理极少采用全量 Sidecar 网格注入，普遍偏向使用集中式 Ingress 独立网关；

2. 内存与吞吐损耗：
      - Envoy 内存管理精良，单进程维持数万并发长连接仅消耗数百 MB 内存，支持数十 Gbps 吞吐流式透明转发。

#### 维度：`production_readiness_and_cases`

*维度说明：企业级生产落地成熟度、典型标杆案例与主流开源生态配套*

L7 网格与反向代理方案的生产成熟度与落地生态：

1. 企业级生产成熟度极高（传统 L7 领域王者）：
      - Envoy / Istio / Nginx 经过全球顶级互联网高并发严苛检验，在协议合规、安全防护（mTLS 零信任、WAF、JWT 鉴权）、全局限流、可观测性与云原生标准生态配套方面处于绝对统治地位；

2. 大模型推理场景下的工业界标杆方案与架构演进：
      - 边缘统一网关层标杆：OpenAI、Anthropic、Cloudflare 以及各大云厂商统一大模型 API 入口，均无一例外采用 Envoy / Nginx 构建最外层 API Ingress，统一负责 TLS 终结、计费鉴权、租户级配额限流与模型路径分发；
      - 内层演进方案（Envoy + ext-proc / SIG Inference Gateway）：针对通用代理无法感知显存与 Cache 的致命短板，业界最新前沿（如 Kubernetes SIG 推理网关规范 Gateway API for Inference）基于 Envoy 的 ext-proc 接口构建专用的推理扩展控制器，将通用七层代理与大模型运行时指标深度串联，形成‘Envoy 高性能七层转发 + 专用推理智能控制面’的标杆演进范式。

- - -

## 10\. gRPC 客户端直连负载分担 (Headless Service + 自定义探针)

> **架构分类**：客户端直连架构  
> **核心定位**：客户端解析 Headless Service DNS，直接维护与各 Pod 的长连接池，跳过中间代理单点，基于业务端探针实现零损耗负载分担

### 负载分担调度算法与决策核心

#### 维度：`load_balancing_algorithm`

*维度说明：具体分担算法逻辑（轮询/加权轮询、最小在飞请求 Least In-flight、基于前缀树 Radix Tree 的哈希匹配、一致性哈希与负载边界 CHWBL、双重映射 DualMap 等）*

gRPC 客户端直连架构通过将负载均衡逻辑下沉至客户端进程内部，摆脱了中间代理的单点瓶颈与性能损耗：

1. 官方内置基础均衡策略：
      - Round Robin（轮询）：客户端解析 Headless Service 返回的全部 Pod IP 列表，为每个 Pod 建立一个 Subchannel，在每次发起 RPC 调用时轮流挑选就绪（READY）的子信道；
      - Pick First（首次命中）：仅与首个健康 Pod 建立连接并发送全部请求，仅作冷备容灾，不适用于并发负载分流；
      - Weighted Round Robin（WRR）：配合探针反馈的静态或动态权重，依比例分配 RPC 流。

2. 自定义业务探针增强的高级分担算法：
      - 最小在飞流调度（Least In-flight Streams / P2C with Probes）：客户端拦截器在本地实时跟踪每个 Subchannel 正在传输中的并发流计数（In-flight RPC Count）。新请求到达时，采用 P2C（Power of Two Choices）在后端池中随机挑选两个 Pod，并优先分派给在飞流较小且探针评分更高的 Pod，既避免了惊群效应，又实现了轻量级动态自适应；
      - 指数加权移动平均响应时间（Peak EWMA / P2C-EWMA）：结合带内探针返回的真实首字延迟（TTFT）与平均处理耗时，计算每个 Pod 的动态响应代价，优先流向计算速率最快的 Pod；
      - 客户端前缀感知散列（Client-Side Prefix-Aware Hashing）：客户端在内存中对请求 Prompt 计算 Hash，优先尝试映射至固定的候选 Pod 子集以提升 KV Cache 命中率；当带内探针指示该 Pod 显存或队列饱和时，触发客户端本地溢流切换。

#### 维度：`load_metrics_monitored`

*维度说明：分担决策所采纳的后端状态指标（活跃请求数 Running Requests、队列等待数 Waiting Requests、GPU 显存水位、KV Cache 块占用率、Token 生成速率）*

客户端直连架构通过多通道探针机制，能够获取到极为精准、实时的模型引擎深层指标：

1. 带内流式探针（In-band Trailers / Response Headers Probing）：
      - 推理引擎（如定制版 vLLM/SGLang gRPC 接口）在流式 RPC 的响应头（Initial Metadata）或结束帧（Trailing Metadata）中，直接回传当前 Pod 运行时的内部深层状态：
        \* `x-inference-running-requests`：当前正在执行 Continuous Batching 的活跃请求数；
        \* `x-inference-waiting-requests`：当前调度器等待队列中的积压请求数；
        \* `x-inference-gpu-cache-factor`：GPU 显存物理块的实时占用率；
      - 客户端 gRPC ClientInterceptor 拦截器在接收到首个 Token 或完成时，以零额外网络往返（Zero Extra RTT）瞬时更新本地连接池对应的 Pod 负载权重。

2. 带外异步健康探测（Out-of-band Metrics Poller）：
      - 客户端后台启动轻量异步协程，以毫秒级周期（如 200ms - 500ms）向各 Pod IP 发起轻量健康探测或拉取 `/metrics`，维护 Pod 健康度矩阵；

3. 客户端链路级感知指标：
      - 客户端本地记录的各 Pod TCP 连接握手延迟、TCP RTT、首字生成延迟（TTFT）、每秒接收 Token 吞吐率及 gRPC 状态码（如 `UNAVAILABLE`、`RESOURCE_EXHAUSTED`）。

#### 维度：`dispatch_granularity`

*维度说明：请求分发粒度（TCP 连接级、HTTP/2 请求级、Token 生成 Chunk 级、Prefill/Decode 阶段级）*

客户端直连架构具备极致的请求分发粒度与连接灵活性：

1. 原生 gRPC Stream / 请求级绝对解耦：
      - 客户端通过 Headless Service 获取后端 N 个 Pod 的真实 IP，并分别与这 N 个 Pod 建立并保持独立的 HTTP/2 底层 TCP 连接（Subchannels）。当业务代码发起一次流式推理调用时，客户端负载均衡器在本地挑选出最优的 Subchannel，并在该连接上分配一个独立的 HTTP/2 Stream ID；
      - 彻底破除 L4 负载均衡陷阱：在标准 K8s ClusterIP Service（四层 iptables/IPVS）下，gRPC 客户端与单个 Pod 建立长连接后，后续千万次 Stream 都会永久锁死在同一 Pod 上造成严重偏斜；而 Headless 直连使每个 Stream 都能自由、动态地分派到不同的 Pod IP，实现了真正的单请求级精确分担；

2. 阶段解耦粒度的协同局限：
      - 对于将请求拆分为 Prefill 阶段与 Decode 阶段的 PD 分离架构，客户端通用直连层无法自动进行跨 Pod 的显存 RDMA 握手调度，必须依赖高度定制的客户端 SDK 编排二阶段 RPC，增加了业务端实现的复杂度。

### 状态亲和与负载均衡的动态权衡

#### 维度：`cache_affinity_vs_load_balance`

*维度说明：如何平衡‘提高 KV Cache / Prefix Caching 命中率（倾向发给已有缓存的同一 Pod）’与‘防止单 Pod 显存负载倾斜过载（倾向发给空闲 Pod）’*

客户端掌握全量请求上下文，在实现 Cache 亲和与防负载倾斜方面具备天然先验优势：

1. 零解析开销的客户端 Prompt 前缀亲和：
      - 中间代理层需要解构巨大的 HTTP 请求体（JSON Parsing）才能获取 Prompt 文本；而客户端本身就是 Prompt 的生成者，天然拥有原始文本、对话历史和 System Prompt 指针；
      - 客户端可在本地直接对 System Prompt 或会话 Session-ID 进行哈希计算，将同一会话或共享长文档前缀的请求锁定派发给特定 Pod，极大提升后端 Pod 的 Prefix Caching 命中率，避免反复重新 Prefill；

2. 基于探针反馈的本地自适应防倾斜（Client-side Spillover）：
      - 客户端监控各 Pod 回传的带内显存利用率（`gpu_cache_factor`）与等待队列数（`waiting-requests`）；
      - 当亲和 Pod 的显存块利用率超过安全警戒水位（例如 > 85%）或 Waiting 队列大于 0 时，客户端调度器判定亲和 Pod 存在过载风险，立刻绕过亲和锁定，将新请求溢流（Spillover）派发至空闲的备选 Pod，并在客户端动态平滑更新亲和缓存映射表，在保证全局高吞吐的同时杜绝单 Pod 倾斜崩溃。

#### 维度：`multi_lora_affinity_dispatch`

*维度说明：针对挂载多个微调 LoRA 的多副本，如何感知各 Pod 当前已驻留的 Adapter 权重并做亲和性负载分担，减少显存置换开销*

在多副本部署挂载多个微调 LoRA Adapter 场景下的客户端协同：

1. 客户端本地 LoRA 路由注册表（LoRA-to-Pod Mapping）：
      - 客户端通过结合 K8s Pod 标签（Labels/Annotations）、带外查询接口或初始化握手，在内存中维护每个 Pod 当前驻留或分配的 LoRA 模型列表；
      - 当业务端发起特定微调模型调用（如 `model: \"llama-3-8b-lora-code\"`）时，客户端负载均衡选择器先在候选 Subchannel 集合中过滤出已驻留该 LoRA 的 Pod 节点，仅在目标子集中执行 Least In-flight 调度，彻底消除因随机分流导致的跨 Pod 频繁换入换出显存抖动（Thrashing）；

2. 降级动态加载策略：
      - 若所有驻留该 LoRA 的目标 Pod 均处于高负载排队状态，客户端可基于策略降级溢流至通用空闲 Pod，并在请求参数中触发该 Pod 的动态拉取加载。

### 解耦架构与流水线负载分流

#### 维度：`pd_disaggregated_load_sharing`

*维度说明：Prefill 阶段与 Decode 阶段的跨 Pod 流水线解耦分担机制，请求流转路径与跨节点 KV-Cache 传输网络协同（RDMA/RoCE/NCCL）*

客户端直连在 Prefill-Decode 解耦架构下的集成与流水线分流：

1. 客户端主导的双 Headless Service 编排模式：
      - 在集群中分别部署 Prefill Headless Service 和 Decode Headless Service。客户端 SDK 维护两个独立的连接池；
      - 调用流程：客户端首先直连 Prefill Pod 发送 Prompt，Prefill Pod 执行高并行 Prefill 计算，并将 KV Cache 通过高速网络（RDMA/RoCE）推入分布式缓存或直接同步至 Decode Pod；Prefill Pod 向客户端返回上下文 Token 或 KV 句柄（Context Handle）；客户端随即携带该 Handle 直连 Decode Pod 发起自回归流式生成；

2. 方案代价与局限：
      - 该模式要求将复杂的两阶段容灾、节点寻址与超时逻辑内嵌在客户端应用中，使得客户端变得极重（Fat Client），多语言支持与升级迭代包袱沉重。因此在工业界，PD 分离更普遍选择在服务端网关层做内部中转，客户端直连通常只作为统一入口的直连通信。

#### 维度：`chunked_prefill_concurrency`

*维度说明：变长长文本 Prompt 的分块预填充（Chunked Prefill）对多副本并发负载平滑分担的支撑效果*

分块预填充（Chunked Prefill）对客户端直连负载分担的性能强化：

1. 消除带内指标震荡与探针失真：
      - 若后端 Pod 缺乏 Chunked Prefill，超长文本计算会瞬间霸占整个 GPU 算力数秒，期间该 Pod 无法处理或吐出任何其他请求的 Token，导致客户端记录的单次 RPC 响应时间产生极端毛刺；
      - 开启 Chunked Prefill 后，后端 Pod 在每次微步迭代中均匀交织 Prefill 片段与 Decode Token，单步迭代维持在数十毫秒的恒定节奏，使客户端收集到的带内延迟指标高度平滑稳定；

2. 提升客户端加权算法精度：
      - 稳定的迭代延迟使客户端基于 EWMA 或最小响应时间的调度器能够做出真实可信的权重分配，避免因长 Prompt 偶发阻塞导致客户端误将健康 Pod 判定为不可用并发生剧烈切流。

### 过载保护、反压与容灾伸缩

#### 维度：`overload_protection_and_admission`

*维度说明：单 Pod 即将显存击穿（OOM）或队列满时的准入控制（Admission Control）、排队缓冲与溢流分流（Spillover）机制*

客户端直连架构将过载保护推至最前线，实现极度灵敏的零等待熔断与反压：

1. 客户端并发控制器（Client-Side Concurrency Limiter）：
      - 客户端本地为每个后端 Pod 设定并发上限阈值（如单个 Pod 最大允许 16 个并发 Stream）。一旦达到上限，新请求在客户端本地内存排队或直接报错，绝不将过载压力打向后端 Pod；

2. 毫秒级带内反馈断路器（In-band Circuit Breaking）：
      - 当后端 Pod 显存耗尽或 Waiting 队列超过硬限制时，在 gRPC 响应 Trailer 中标记过载状态，或直接返回 gRPC `RESOURCE_EXHAUSTED` 错误；客户端拦截器捕获该信号后，在毫秒级将该 Subchannel 标记为熔断冷却状态，在冷却窗口内完全阻断新请求流入，比传统基于外部健康检查的熔断反应快 1-2 个数量级；

3. 客户端削峰排队缓冲（Local Queueing）：
      - 面对偶发高并发突发洪峰，客户端在进程内存池中做短时间排队平滑注入，避免瞬间流量冲垮后端推理引擎。

#### 维度：`k8s_autoscaling_and_rebalancing`

*维度说明：与 Kubernetes HPA/KEDA 水平伸缩的联动，扩容新增 Pod 时的冷启动预热，以及缩容/故障时的流量优雅下线（Drain）与平滑再均衡*

与 Kubernetes 弹性伸缩和 Headless 节点发现的联动机制：

1. gRPC Name Resolver 与 DNS 刷新挑战：
      - 客户端通过 gRPC 原生 Name Resolver（`dns:///headless-svc.namespace.svc.cluster.local:port`）监听 DNS 变更；
      - K8s CoreDNS 更新与客户端 DNS 缓存默认存在 1s - 5s 的刷新延迟。在 Pod 频繁水平自动伸缩（HPA/KEDA）的动态场景下，可能出现扩容 Pod 无法被客户端立即发现、或缩容 Pod 已死但客户端仍尝试建连的现象；
      - 高级工业级解法：在客户端引入基于 client-go / K8s API 直接监听 EndpointSlice 事件的自定义 Resolver，实现 Pod IP 变更的毫秒级推送与零延迟连接池更新；

2. 缩容优雅下线与 GOAWAY 帧处理：
      - 当推理 Pod 被 K8s 调度终止（SIGTERM）时，后端 gRPC Server 触发优雅停机逻辑，主动向上游客户端发送 HTTP/2 `GOAWAY` 帧；客户端接收到该帧后，立即将该 Subchannel 从活跃分流池中移出，不再向其分派任何新 Stream，同时保持现有流式响应不受干扰地继续吐字传输直至结束，实现缩容平滑再均衡与零报错下线。

#### 维度：`streaming_failover_resilience`

*维度说明：长连接流式输出（SSE/gRPC Streaming）长达数十秒过程中，Pod 崩溃或被抢占时的容灾与重试机制*

流式长连接传输中 Pod 崩溃时的客户端自主容灾与断点重试：

1. 掌握上下文控制权的断点重试优势：
      - 与中间反向代理在流中断时无法重试的硬约束截然不同，客户端是流式数据的直接消费方，精确知晓已经接收到了多少个 Token 以及最新生成的文本上下文；

2. 智能上下文拼接重试（Client-side Resumption）：
      - 当直连的 Pod 在流式吐字数十秒后突然发生物理宕机、OOM 终止或网络中断，客户端 gRPC 拦截器捕获异常断连事件，立即剔除该故障 Pod；
      - 客户端调度器将原始 Prompt 与已成功接收的 Token 文本进行本地拼接，生成新的续写 Prompt，透明直连另一个健康的副本 Pod 发起重试；结合后端的 Prefix Caching，新 Pod 可瞬间命中前半段缓存并继续无缝吐字，实现用户体验近乎无感的高韧性容灾。

### 性能开销与落地可行性

#### 维度：`routing_overhead_and_latency`

*维度说明：分担决策自身引入的额外网络跳数、代理处理耗时（TTFT 影响）与 CPU/内存额外开销*

网络链路与系统资源开销分析：

1. 极致的零中间跳数与绝对延迟优势：
      - 拓扑结构为客户端与推理 Pod 点对点（P2P）直连，中间网络跳数为绝对最优（0 额外网络跳数，直连 Pod 物理 IP）；
      - 首字延迟（TTFT）大幅缩短：消除了反向代理层带来的两次 TCP 协议栈编解码、上下文切换与网络转发开销，端到端 TTFT 延迟相比集中式网关代理降低 0.5ms - 2ms。在实时语音流（Voice AI）、金融风控、实时代码补全等极度严苛的毫秒级交互场景中具备决定性性能优势；

2. 彻底破除网关单点带宽瓶颈：
      - 流量完全去中心化分散在各个客户端与 Pod 之间，消除了集中式反向代理网关集群的横向扩容成本、带宽上限瓶颈及单点故障风险；

3. 客户端开销：
      - 客户端进程需要维护通往所有 Pod 的 HTTP/2 长连接与健康探针轮询，在超大规模集群（如上千个推理 Pod）下，客户端连接池的内存开销与文件描述符占用会线性增加，需通过 Subchannel 池化或子集划分（Subsetting）优化。

#### 维度：`production_readiness_and_cases`

*维度说明：企业级生产落地成熟度、典型标杆案例与主流开源生态配套*

客户端直连架构的生产落地评估与企业实践：

1. 架构权衡与重客户端（Fat Client）挑战：
      - 该架构的核心痛点在于将复杂的连接治理、DNS 解析、探针评估与重试逻辑深度耦合在客户端 SDK 内部。若企业业务涉及多门编程语言（Python、Go、Java、C++、Node.js），跨语言维护一套高可靠、算法一致的 Client SDK 研发与治理成本极高；

2. 工业界标杆案例与主流方案：
      - Google gRPC Proxyless Service Mesh：Google 官方推行的无代理服务网格架构，客户端直接使用 gRPC 库，通过 xDS 协议由集中式控制面（如 Traffic Director / Istio）统一下发后端端点与路由策略，客户端执行直连通信，完美兼顾了‘集中控制治理’与‘客户端直连零损耗’；
      - 互联网大厂内部微服务与高并发推理底座：字节跳动、美团、Uber、Netflix 在其核心大模型低时延在线推理链路、Embedding 向量召回服务中，普遍采用 Headless Service + 自研 RPC 客户端智能探针直连架构，作为支撑万卡级算力集群的高吞吐核心基础设施。

- - -

## 技术选型决策与架构演进建议

### 1\. 架构选型对照表

| 方案类别         | 代表组件                      | 调度决策层级       | 缓存亲和度            | 负载感知维度               | 适用吞吐区间       | 推荐场景                  |
| ------------ | ------------------------- | ------------ | ---------------- | -------------------- | ------------ | --------------------- |
| **基础代理层**    | Envoy / Istio             | L7 HTTP/gRPC | 无                | 在飞连接/请求数             | < 100 QPS    | 通用微服务网关初级接入           |
| **客户端直连**    | gRPC Headless             | 客户端进程内       | 弱/可定制            | 业务端带内探针              | 100~500 QPS  | 内部微服务调用，极致低延迟         |
| **引擎原生路由**   | vLLM / SGLang Router      | 引擎应用层        | 强 (Prefix/Radix) | 等待队列 + 显存水位          | 200~2000 QPS | 同构 LLM 多副本推理主力方案      |
| **云原生标准网关**  | Gateway API for Inference | K8s 官方扩展     | 强 (动态更新)         | 运行时指标 (ext-proc)     | 500~5000 QPS | **企业级云原生标准化首选**       |
| **P/D 解耦架构** | Mooncake / vLLM PD        | 异构流水线两阶段     | 极强 (全分布式)        | Prefill算力 + Decode显存 | 1000+ QPS    | **长文本、高并发、超低尾延迟核心业务** |
| **MoE 专家并行** | DeepSeek EPLB             | 卡间 + 副本间     | 强                | 专家激活频次 + 冗余权重        | 500+ QPS     | 深度求索 V3/R1 大规模 MoE 部署 |
| **多机逻辑副本**   | LeaderWorkerSet (LWS)     | 多 Pod 协同组拓扑  | 继承引擎层            | 组健康度 + Leader负载      | 超大模型专用       | 70B / 405B 跨机分布式并行托管  |

### 2\. 生产落地演进路线

1. **起步验证阶段**

   ：采用 **SGLang Router** 或 **vLLM Router** 配合 K8s Deployment，开启 Prefix Caching 与 Chunked Prefill，前端挂载基础 L7 Ingress 进行 TLS 终结与配额限制；

2. **规范化平台阶段**

   ：采用 **Kubernetes Gateway API for Inference (Envoy Gateway + EPP)**，将多副本 vLLM/SGLang 纳管为 `InferencePool`，利用 `ext-proc` 协议闭环显存与队列反压，并配合 **LeaderWorkerSet** 管理大模型分布式实例；

3. **极致性能与长文本阶段**

   ：全面演进至 **Prefill-Decode Disaggregation (PD分离) + Mooncake 分布式 KV 存储**，通过高速 RDMA 消除长 Prompt 阻塞，实现 GPU 计算与带宽的极致利用率。
