---
title: "KVCache：从推理工作负载到系统设计范式"
source: "Essays"
category: "AI Infra"
group: "MiracleFarms"
url: "https://miraclefarms.github.io/notes/2026/06/02/kvcache-workload-to-system-design/"
published: 2026-06-02T21:40:00+08:00
saved: 2026-09-27T16:30:06+08:00
folo_key: "mf-essays::https://miraclefarms.github.io/notes/2026/06/02/kvcache-workload-to-system-design/"
tags:
  - "folo"
  - "AI_Infra"
  - "Essays"
---

# KVCache：从推理工作负载到系统设计范式

> [!info] Essays · AI Infra · 2026-06-02 21:40 · [原文](https://miraclefarms.github.io/notes/2026/06/02/kvcache-workload-to-system-design/)

> **版本历史**
> 
> | 版本    | 日期         | 说明                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
> | ----- | ---------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
> | v1.0  | 2026-06-02 | 初稿，自顶向下梳理 KVCache 从工作负载到系统设计范式，覆盖 agentic workload、attention 变体、框架管理、PD 分离、集群 KV pool、CXL 硬件                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
> | v1.1  | 2026-06-03 | 结构重组：每章围绕一张核心图展开主线叙事（5→13 图），分支材料（线性注意力、量化压缩、AF 分离、弹性成本、系统抽象、网络通信）移入附录 8.5-8.10 并补充核心观点；附录 8.1 增加核心判断列，8.3 增加引用标注                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
> | v1.2  | 2026-06-03 | 第二轮 review：补充芯片视角（decode arithmetic intensity、DMA 需求分类、CXL 延迟量级对比、bytes/token/layer 归一化指标）、系统视角（identity 正确性风险具体案例、开放问题优先级）、网络视角（RDMA 流量定量参数、传输路径选择、QoS 讨论）                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
> | v1.3  | 2026-06-04 | 按 essay 新范式重构：每章开头新增”本节论点 + 章节级星级参考”导读块（E1）；统一把”站内文章”改称”前期调研文章”，便于站外单独发表（E4）；完成全文引用三校验（E5）——修正端到端延迟 8.7→8.6 倍、对齐 Beluga 吞吐口径、标注 Penguin 容量参数未能二次抓取核对、注明 MiMo/Dynamo 部分数字为图中读出                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
> | v1.4  | 2026-06-04 | 前五章深度重构（专家讨论轮）：第一章补”封装性判据”论证为何是系统约束 + prompt caching 行业计费对比；2.1 四特征补机制论据（读放大可推导 R/W≈O(N)、prefix 分层突变率栈、无状态路由致会话迁移、tool call 三类非模型瓶颈），2.2 厘清 effective hit length 与 hit rate 的三处分叉（绝对 vs 比值、幽灵命中=正确性问题、连续性空洞）；4.1 改纯叙事+前向引用、4.2 厘清 Mooncake Master=元数据目录服务（区分服务网关/调度器）、4.3 对齐 L1-L4⇔G1-G4 四级分层并标注 L2.5/L3.5 半层+加入 offload 经济性算例、4.4 补平台化定义与 taxonomy 轴说明 + KVBM↔HiCache↔LMCache 同位且 runtime 中立、4.5 拆为”推理框架（转置含机制维度）+ 框架外组件（按角色层）”两表；5.1 补 PD 路线定位（Conditional Disaggregation 运行时组合、角色启动固定、DeepSeek EP32/EP144 非对称 xPyD）、5.3 补”逻辑池”路线（全局索引 + KV-aware routing）并缝合 5.2↔5.3。新增一手引用 \[32\]–\[38\]。第六章硬件形态另行讨论，本版未动                                                                            |
> | v1.5  | 2026-06-04 | 第六章硬件数值校正 + 引用清理：修正 §6.1 HBM 访问延迟（1-2ns→100-300ns，并指明 HBM 相对 DDR/CXL 的优势在带宽而非延迟）；修正 §6.1/§6.2 decode 单 token 读取量与全文 32GB 口径对齐（400MB 实为单层，全模型 ~80 层约 32GB，HBM ~9.5ms / CXL ~640ms）；修正 §6.2 H100 roofline 拐点（6.5→约 295 FLOP/byte，并注明 arithmetic intensity 与层数无关）；§2.1 读写比推导对照实测改为 \[20,40\] 区间口径；§4.3/§8.10 GQA 32GB 标注 80 层口径以与图 4 具体型号区分；§4.2/§5.3 厘清 92.2% 与 >95% 来自不同实验设定；补全孤儿引用 \[21\]\[23\]\[24\]\[26\]\[27\]\[28\]\[30\] 的正文内联标注                                                                                                                                                                                                                                                    |
> | v1.6  | 2026-06-04 | 第二轮第五章深度讨论 + 4.3 概念对齐表：§4.3 把 G1-G4/L1-L4 分层改成开头对齐表（标题 L1/L2/L3→L1–L4，半层 L2.5/L3.5 入表，图 8 caption 标注”延迟为量级示意，精确值见 §6.1”）；§5.1 修正”EP32/EP144 ⟹ decode 更重”的串轴错误（厘清”实例数比例 xPyD”与”单实例 EP 规模”两轴，DeepSeek EP32/EP144 实为 1P1D 形态的实例内非对称，补 vLLM 1P3D/3P1D 社区案例与 Dynamo runtime-reconfigurable xPyD），为 Sarathi-Serve 加 gloss + chunked context 别名，加粗三处主线判断；§5.2 补 KV-aware 调度的两层落点（集群路由层 vs 引擎调度层、三框架归属、缝合 §4.2 前门/目录）+ MiMo 两信号”重算成本 vs 排队成本”的朴素机制 + tier/link-aware 信号前沿；§5.3 补”当前判断 + 未来展望”（B 路由打底 + A 物理池持久后备的混合共识，A↔B 接口标准化是胜负手，呼应第七章）。新增一手引用 \[39\]–\[41\]                                                                                                                                  |
> | v1.7  | 2026-06-05 | 第二轮第六/八章讨论落地：§2.1 第一点删读放大形式推导改朴素说明（回退 v1.4 的 C\_i/望远镜/\[20,40\] 口径）；§6.1 点破 CXL pooling（2.0 分区独占）vs sharing（3.x 多 host coherent 同享）、指明”rack 内多 host 共享同一片”严格依赖 3.x 且其量产生态到 2026 仍早；§6.2 修正 decode arithmetic intensity 单位口径（原 0.5 实为 MAC/byte，与 295 FLOP/byte 混用 → 统一 1.0 FLOP/byte vs 295）、补 GQA query-head 口径注与”batching 救不了 attention”结构性结论、MLA 强度改定性、layout 例子改 layer-first↔page-first；§8.6 补线性注意力代表模型（Mamba/RWKV/RetNet + Qwen3.5/Kimi Linear/Jamba 混合）与”定长状态快照按边界 resume、不可切片、混合模型与 KV 并管取命中下界”机制；§8.8 重构为 Attention-FFN(MoE) 分离/AFD（原 agent 工具执行内容移除，已分布于 2.1/8.5/8.9）。新增一手引用 \[42\] MegaScale-Infer、\[43\] NVIDIA–Groq                                                           |
> | v1.8  | 2026-06-08 | 按 SP / LW review 修订：采纳 SP 建议，在第三章单独新增 DeepSeek V4 CSA/HCA 范式讨论，补 sliding + CSA/HCA 的多池融合与 ShadowRadix/HiSparse 系统含义；采纳 LW 指正，核对 Kimi K2.6 config 与 modeling code 后将 MLA cache 口径从 latent rank 512 修正为 `kv_lora_rank 512 + qk_rope_head_dim 64 = 576`，并重写 §6.2 MLA arithmetic intensity 判断：DeepSeek 128-head MLA 已接近 H100 BF16 roofline 拐点，不再表述为”远低于”                                                                                                                                                                                                                                                                                                                                     |
> | v1.9  | 2026-06-09 | 整体结构按“自顶向下、围绕 KVCache 存储层级”重排：业务模式（含新增 §2.3 GQA/MLA 让 KV 随上下文线性增长的背景图文、§2.4 Claude Code 的 KVCache budget 与布局）→ 调度逻辑（原 5.2 前移为独立第三章）→ 框架实现 → 分层实现（单实例 L1–L4 / 多实例 PD / 集群池）→ CXL 硬件 → Attention 算法演进（原第三章 SWA/Hybrid/CSA-HCA 后移收尾）。全文图号、章号、附录号随之重排，引用三校验通过                                                                                                                                                                                                                                                                                                                                                                                                                                |
> | v1.10 | 2026-06-09 | 充实第三章「调度逻辑」：拆成 §3.1 信号演化（队列长度→前缀命中+负载→KV 物理位置）、§3.2 三种路由实现的精度坐标（vLLM 近似前缀树 / llm-d KVEvents 精确事件 365GB↔339KB / Dynamo Flash Indexer 1.7 亿 ops/s）+ llm-d 精确 vs 近似 57× P90 TTFT 基准与「可观测性是天花板」判断、§3.3 反热点张力与 Lodestar 学习式路由终点（异构 ~4.4×，呼应国内多芯片混推）。新增一手引用 \[51\]–\[55\]，引用三校验通过                                                                                                                                                                                                                                                                                                                                                                                                              |
> | v1.11 | 2026-06-10 | 调度章节补 demand fetch 边界条件与 SM-free 约束（KVDrive/ShadowServe）；CXL 章节重构为「按流量语义拆分」框架，补 CloudMatrix384 三平面生产判例（UB 池访问 + RDMA P→D 交接）、EP/TP 争用分析（DualPath/DeepSeek 硬件反思）、HBM-as-cache 终局方向（HyperOffload）；新增一手引用 \[56\]–\[62\]                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
> | v1.12 | 2026-06-10 | 全文引用三校验复审：修复 KVBM 设计文档失效链接（\[13\] 迁至新版 docs 路径）与 §3 导读块 9 处弯引号导致的失效 anchor；修正 §3.2 路由器归属（近似 radix tree 是 sgl-router 的设计，vLLM Router 实为一致性哈希 sticky 路由）；”GPU/CPU 层”证据改挂 llm-d 官方博客 \[51\]；CloudMatrix 芯片名按论文口径改 Ascend 910；ShadowServe 阈值 25–30%→30%、KVDrive 超 50%→近 50%、Lodestar/ShadowServe 收益补”最高”口径；Groq 交易改为”授权 + acqui-hire、金额为媒体报道”；Penguin 容量经通稿转载核对为 3TB DDR5 + 8TB CXL 合计 11TB；MiMo 图 7 秒数标注图中数据                                                                                                                                                                                                                                                                                |
> | v1.13 | 2026-07-13 | 月度社区动向更新（2026-06-13 至 07-13 窗口：vLLM v0.23–0.25 / SGLang v0.5.13–0.5.15 / TRT-LLM 1.3.0rc19–rc20 / LMCache v0.5.1 / Mooncake main）：§3.2 Mooncake Conductor 从设计文档落成开源 C++ 目录服务；§4.4 修正”vLLM 最外包”判断（第一方 OffloadingConnector 多级分层 + 对象存储二级层）并补 TRT-LLM 暴露 block-hash chain / KV 压缩管理框架、LMCache MP 模式与 Device-DAX；§5.2 PD 数据面走向缓存感知增量同步（SGLang decode 侧 HiCache、vLLM NIXL push）；§5.3 Mooncake connector 集成深化；§6.1/9.10 KV 传输 QoS 从开放问题进入实现（TENT 顺序/时限/通道三轴 + SGLang RFC #28631）；§7.2/9.6 线性注意力 prefix caching 产品化（vLLM Marconi 式准入、SGLang int8 checkpoint pool 与 UnifiedTree 默认化）。新增引用 \[63\]–\[80\]                                                                                          |
> | v1.14 | 2026-07-22 | 解除文章加密（去掉 `locked`），并做一轮补漏与表述修订。**补上两条被前期调研证明、正文却漏引的主线证据**：SAC 用 CXL cache-line 取数把稀疏 demand fetch 从”无法前置”变成”不必前置”（§3.3、§6.2），TraCT 把 P→D 同步交接直接放上 CXL 共享内存（§6.1）——据此把 v1.11 的”按流量语义拆分”框架修正为”语义决定形态、fabric 半径决定可达”。§3.3 用 Price of Anarchy 的饱和拐点和 DualMap 的双哈希映射，把反热点张力从”打分函数里减一项”改写为有相变点、有专门机制的问题。社区窗口 2026-07-13 至 07-22：LMCache v0.5.2（coordinator 在 L1 做全局 P2P token 匹配、pin/unpin token API、Azure Blob/Valkey/Bigtable 等 L2 适配器）、Mooncake TENT 暴露 RDMA NIC 负载统计与 Store 的 L2→L1 提升重试和主机段配额、SGLang v0.5.15 的 MLA decode context parallelism 与 HiCache 联动。数据口径修订：Beluga 76.8s 补齐”对照组是 256-token block”、89.6%/7.35× 标注为 cache-hit 轮次口径，§5.1 的 MLA/DSA 会话体积补 80 层归一化说明。新增引用 \[81\]–\[92\] |
> 
> 本文初稿调研截至 2026-06-02，v1.8 于 2026-06-08 补充核对 Kimi K2.6 配置、DeepSeek V4 CSA/HCA 与 MLA roofline 口径，v1.9 于 2026-06-09 按”围绕存储层级自顶向下”的主线重排章节顺序并在业务模式章补齐 GQA/MLA 增长背景与 Claude Code budget，v1.11 于 2026-06-10 补充调度与 CXL 章节（流量语义拆分、CloudMatrix384、demand fetch、SM-free、HyperOffload），v1.12 同日完成全文 62 条引用的链接与数字复核，v1.13 于 2026-07-13 按最近一个月五个社区的 release 与 merged PR 完成月度动向更新，v1.14 于 2026-07-22 解锁全文并补齐 SAC / TraCT / Price of Anarchy / DualMap 四条此前漏引的一手证据、按 07-13 至 07-22 窗口刷新 LMCache 与 Mooncake 动向；主要复用 MiracleFarms 已发布的 KVCache / Agentic Inference / CXL / Mooncake / NIXL / KVBM / HiCache 系列文章，并补充少量公开一手资料；除非特别说明，框架和产品状态均按公开页面或文章描述。

KVCache 重新变成架构问题，起点来自工作负载变化，单纯更大的上下文窗口只是表层现象。一次普通 chat 请求也会生成 KV cache，但它通常是一轮、几千 token、请求结束后状态随进程释放；一次 Claude Code、OpenClaw 或企业内部 coding agent 任务，会在几十轮调用里反复携带 system prompt、工具 schema、仓库上下文、历史观察和计划状态。vLLM x Mooncake 对 Codex/SWE-bench Pro traces 的公开分析给了一个很适合开场的数字：610 条 trace 的中位轮数是 33，第 30 轮上下文约 80K token，最长超过 180K token，平均输入输出 token 比约 131:1[\[1\]](https://vllm.ai/blog/2026-05-06-mooncake-store)。**这里的关键不只是请求变长；同一段 prefix 会在会话生命周期里被一次又一次读取。**

从这组负载往下看，推理系统的核心矛盾被压缩成三件事：算力利用率、显存容量、服务质量。算力利用率要求 GPU 不要在 prefill 重算和 I/O 等待里空转；显存容量要求 KV block 不要被碎片、重复和冷数据占满；服务质量要求首 token 延迟、尾延迟、吞吐和任务完成时间都可控。**KVCache 已经从 attention 的实现细节，演变成连接工作负载、模型结构、runtime、部署拓扑和硬件形态的系统约束。**

这篇综述按一条自顶向下、围绕 KVCache 存储层级展开的主线推进：先看 agentic 业务模式如何改变 KVCache 的生命周期，并补齐 GQA/MLA 让 KV 随上下文线性增长的背景；再看调度逻辑如何从“看算力”升级到“看 KV state”；接着比较三大框架把 KVCache 管理放在哪一层；然后沿存储层级走完单实例 L1–L4、多实例 PD 分离与集群级 KV pool，并落到 CXL 硬件形态；最后回到 attention 算法演进如何反过来改写 KV 的形态。正文每节围绕一张核心图展开主干叙事，分支材料和详细展开放在附录。

在展开之前，先用一张总览图把全文要讨论的系统架构收在一起（图 0）——它既是后续各章的导航，也是”KVCache 已成为跨层系统约束”这一判断的结构化呈现。

![[KVCache：从推理工作负载到系统设计范式-01.svg]]

*图 0：KVCache 中心的分布式推理系统架构总览（半径视角）。顶部是 agentic 业务模式（长上下文、多轮，输入输出比 131:1、读写比 11.7:1）——**KVCache 的加入把服务从无状态推成有状态**，每个请求都拖着一坨可复用、有物理位置的 KV；右上角点明 attention 结构（GQA/MLA、多态 KV）决定每个半径里 KV 的体积与斜率（第二/七章）。中间是控制平面：第三章的调度信号演化（队列长度 → 前缀命中+负载 → KV 物理位置 → 学习式）与第四章的缓存管理层（KVBM / HiCache / LMCache），二者共同决定每个半径上”**请求向状态迁移，还是状态向请求迁移**“，并经 KV Directory / Indexer（可观测脊柱，不在数据路径）查询与喂事件。下半部是沿存储层级展开的四个嵌套半径——单机内（S0：L1 HBM / L2 DDR，引擎内调度）⊂ 单实例内（S1：P/D 相位与跨卡 KV 传输，L2.5 CXL）⊂ 单集群内（S2：KV-aware 路由＝负载均衡，物理池 A / 逻辑池 B，L3 SSD / L4 共享池）⊂ 多集群间（S3：跨区域联合代价与学习式路由）。底部温度条标出热（近 GPU，L1）到冷（远 GPU，L4）：**调度半径 S0–S3 与存储半径 L1–L4 一一对应，同一个 KV 局部性权衡在四个半径上重演**。本图为 MiracleFarms 绘制，后续各章是这张总览图的局部展开。*

## 一、KVCache 为什么从实现细节变成系统约束

> **本节论点**：KVCache 的定义没变，但它的系统属性已经从”单机显存里的 attention 中间结果”，扩张成连接工作负载、模型结构、runtime、部署拓扑和硬件形态的系统约束；这一章要回答的核心问题是——为什么 2026 年要把 KVCache 当成跨层设计对象重新审视。 **本节核心参考**：
> 
> - ★★★ **vLLM x Mooncake 官方博客**[\[1\]](https://vllm.ai/blog/2026-05-06-mooncake-store) — 610 条 Codex/SWE-bench 生产 trace，把 cache 命中率从 1.7% 拉到 92.2% 的一手数据，是本节核心图（图 1）的来源
> - ★★ PagedAttention 论文[\[2\]](https://arxiv.org/abs/2309.06180) — 奠定单机显存 KV 管理范式，界定”上一代问题”的边界
> - ★★ Anthropic Prompt Caching 文档[\[3\]](https://platform.claude.com/docs/en/build-with-claude/prompt-caching) — 应用层把 prefix 稳定性显式做成计费与 TTL 能力（5 分钟默认 / 1 小时可选）

KVCache 的经典定义很简单：Transformer 自回归生成时，缓存历史 key/value，decode 阶段不用为每个新 token 重新计算完整历史投影。这个定义仍然正确，但已经不够用了。**对架构团队来说，更重要的是它的系统属性**：KVCache 的体积随上下文、层数、KV head 数、head dimension、精度和 attention 结构增长；它在 decode 阶段被高频读取，在 prefill 后被写入，在多轮场景里可能被跨请求复用，在 PD 分离里会跨节点传输，在 offload 里会跨 HBM、DDR、SSD、远端存储迁移。

**判断一个东西是实现细节还是系统约束，有一个直接的判据：它能否被某一层独立封装、对其余层隐藏。**PagedAttention 时代，KVCache 能被 runtime 在单机显存内封装干净，调度器、网络和硬件都不必知道它的存在——那时它是实现细节。2026 年它封装不住了：体积由模型结构决定（attention 变体可让 100K token 下 KV 相差超 10 倍，见第二章），命中率由调度与拓扑决定，迁移成本由硬件层级决定，复用价值由工作负载决定。没有任何一层能独立把它藏起来，每一层的设计都要为它让路——这正是开篇所说”系统约束”的确切含义。换个角度看，KVCache 同时压在算力利用率、显存容量、服务质量这三个目标的交点上：你在显存上省下的，会在重算的算力或 TTFT 上吐回去，没有哪一层能局部最优地单独处置它。

图 1 展示了一条典型的 agentic trace 中 prefix 与新增 token 的关系。每轮真正新增的 token 只是尾部小段，历史 system prompt、skill/memory 和 prior turns 才是主要输入。**KVCache 的复用价值来自这些稳定 prefix。**

![[KVCache：从推理工作负载到系统设计范式-02.svg]]

*图 1：Agentic trace 中每轮的输入构成。橙色区域为已有 prefix，蓝色为本轮新增 token。中位 33 轮、输入输出比 131:1 意味着 KV 复用是系统效率的主要杠杆。注意这是 append-only 的理想形态：真实 trace 会被 thinking token 裁剪和 context compaction 打断，蓝色新增里还混着体量往往更大的 tool result（详见 2.1）。来源：vLLM x Mooncake 官方博客[\[1\]](https://vllm.ai/blog/2026-05-06-mooncake-store)。*

PagedAttention 当年把问题第一次说清楚：LLM serving 需要同时服务足够多请求，但每个请求的 KV cache 巨大、动态增长和收缩，碎片和重复会限制 batch size；vLLM 用类似虚拟内存分页的方式把 KV 拆成固定 block，实现近零浪费和跨请求共享[\[2\]](https://arxiv.org/abs/2309.06180)。**这一代问题主要发生在单机 HBM 内部，核心是”如何不浪费显存”。**

2026 年的问题已经跨出单机。Anthropic prompt caching 文档把应用层 cache 显式暴露成计费和 TTL 机制：多轮对话中 cache point 会自动向前移动，旧内容读 cache，新内容写 cache；默认 TTL 是 5 分钟，1 小时 TTL 需要更高写入成本[\[3\]](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)。这说明 API 层已经把 prefix 稳定性视为产品能力。而且这是全行业模式，只是计费哲学不同——Anthropic 按写入收 premium（1 小时 TTL 写入更贵）、Google Gemini 对显式缓存按存储时长收租、OpenAI 与 DeepSeek 则自动缓存、只在读端打折；不同定价路线背后是同一个判断：prefix 稳定性本身已成为可计价的资源。与此同时，vLLM x Mooncake 证明同一类 prefix 还必须跨实例共享：本地 KV cache 在负载均衡和会话迁移后会变成孤岛，**分布式 KV pool 才能把 cache hit 率从 1.7% 拉到 92.2%，带来 3.8 倍吞吐、46 倍 P50 TTFT 和 8.6 倍端到端延迟改善**[\[1\]](https://vllm.ai/blog/2026-05-06-mooncake-store)。

因此，KVCache 的讨论边界要向上移一层：**让已经算过、仍然有价值的状态，以低于重算的成本被再次使用。** 这个判断贯穿全文。

## 二、业务模式：Agentic 负载与 KVCache 的增长

> **本节论点**：agentic 业务模式把 KVCache 从单请求的临时资源，推成会话级、工作流级的状态资产——真正压在系统上的是反复读取不断增长的历史（读放大），而 KV 到底有多大，由 GQA/MLA 这类 attention 结构决定的斜率说了算；应用层 harness（如 Claude Code）则把“怎么摆放上下文”直接做成 KV 的预算与命中策略。 **本节核心参考**：
> 
> - ★★★ **NVIDIA Dynamo 技术博客**[\[22\]](https://developer.nvidia.com/blog/full-stack-optimizations-for-agentic-inference-with-nvidia-dynamo/) — 读放大主线证据（图 2，11.7:1 累计读写比）
> - ★★★ **vLLM x Mooncake 官方博客**[\[1\]](https://vllm.ai/blog/2026-05-06-mooncake-store) — 131:1 输入输出比、92.2% 跨实例命中率（图 3）
> - ★★ MiMo V2.5 官方博客[\[5\]](https://mimo.xiaomi.com/zh/blog/mimo-v2-5-inference) — 用“窗口安全长度”把 hit rate 细化为 effective hit length
> - ★★ 前期调研文章《主流 Attention 算法全景》[\[49\]](https://miraclefarms.github.io/notes/2026/04/20/mainstream-attention-algorithms-overview/) — 图 4 的 GQA/MLA 缓存量对比来源
> - ★ 前期调研文章《Claude Code 论文再读》[\[50\]](https://miraclefarms.github.io/notes/2026/06/01/claude-code-design-space-reading/) — 图 6 上下文构造与记忆层级来源；工作负载五类设计要求展开见附录 9.5

### 2.1 读放大：agentic 工作负载的核心特征

图 2 来自 NVIDIA Dynamo 技术博客，展示了一条 agentic 工作负载中 KV cache 的累计读写量[\[22\]](https://developer.nvidia.com/blog/full-stack-optimizations-for-agentic-inference-with-nvidia-dynamo/)。关键数字：**40 轮 API 调用后，累计 cache reads 达到 891K token，cache writes 仅 76K token，读写比 11.7:1。**Uncached input（黄色虚线）几乎贴底，说明绝大部分输入都命中了 cache。

![[KVCache：从推理工作负载到系统设计范式-03.webp]]

*图 2：一条 agentic trace 的累计 cache read/write/uncached input。读写比 11.7:1 表明 agentic 负载的本质是”反复读取不断增长的历史状态”。来源：NVIDIA Dynamo 技术博客[\[22\]](https://developer.nvidia.com/blog/full-stack-optimizations-for-agentic-inference-with-nvidia-dynamo/)。*

这张图把 agentic 负载的四个稳定特征压缩成一条曲线：

**第一，输入输出比极端不对称。** vLLM x Mooncake 的 Codex trace 给出 131:1[\[1\]](https://vllm.ai/blog/2026-05-06-mooncake-store)；NVIDIA Dynamo 的 11.7:1 读写比从另一个维度印证了同一件事——真正压在 GPU 和缓存系统上的，是那段不断复用、不断变长的输入历史，生成输出只占很小一头。道理很朴素：每轮都要对到目前为止的全部历史做一遍 attention，却只把尾部新增的一小段写进缓存；于是读量随轮数不断累加、写量几乎不增，读写比自然随对话变长而放大。系统含义是：同一段 KV 只写一次，却要在后续每一轮被反复读取，于是”这段 KV 放在哪、回读多快”成了主导成本项——**placement 与 locality 从次要问题升级为一等问题。**

**第二，prefix 高度稳定但不是全局静态。** 更准确地说，上下文是一个按突变率分层的栈：真静态层（system prompt、工具 schema、repo instruction、few-shot，整会话不变，写一次永久命中）、append-only 层（对话历史、工具结果、错误日志，只在尾部追加，追加点之前逐 byte 不变）、易失层（context compaction 重写历史、工具结果里的时间戳/request-id、按轮变化的动态工具列表、RAG 检索块——任何一个改了中段，其后缓存全部失效）。如果 prefix 全局静态，缓存一次永不失效是 trivial 问题；如果全易失，缓存毫无意义；**正因为它介于两者之间，cache 管理（断点摆放、淘汰、有效命中长度）才成为真问题。**Anthropic 文档中 cache breakpoint 的精确匹配机制解释了为什么 Claude Code、Manus 这类 harness 会把这三层拆得非常细、以最大化可缓存跨度并隔离易失插入物[\[3\]](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)。

**第三，会话会迁移——而且是设计使然，不是意外。** 无状态路由是可扩展性的默认：生产环境是一队同构副本挂在负载均衡器后，按 least-load 或 round-robin 分发，故意不做 session 黏附，否则少数长会话会钉死特定副本造成热点。叠加轮间长 gap（工具执行或人类思考的几秒到几分钟，保留实例就是浪费 GPU）和 autoscaling/故障（扩缩容、spot 抢占、滚动发布都会增删副本），同一 session 的后续 turn 大概率落到另一台机器。本地 prefix cache 跟不上会话迁移，于是缓存丢失就等于整条 prefix 重算。你当然可以用 sticky session 保住本地 KV，但那要牺牲负载均衡和弹性——**集群级 KV pool 的价值正是解耦”请求在哪跑”和”状态在哪存”。**这一点有直接证据：本地缓存在 round-robin 路由下命中率只有 1.7%，共享池把它拉到 92.2%[\[1\]](https://vllm.ai/blog/2026-05-06-mooncake-store)。

**第四，工具调用会制造非模型瓶颈，有三个面。** 其一，工具执行在 GPU 之外、延迟可变（跑测试、爬网页、查库、执行代码，100ms 到数十秒），这期间模型空转、但会话 KV 要么占着 HBM 要么被迫 offload，关键路径不再是模型前向。其二，失败会形成上下文放大的反馈环：一次失败的 tool call 把错误观察写回上下文，触发重试新一轮，新一轮又重读已变大的上下文，失败密集的 trace 里 KV 压力呈超线性增长。其三，**工具结果（文件 dump、检索返回）往往比模型输出大得多，是输入增长的真正大头**——131:1 里”输入”那一头很大程度是工具产生的。前期调研文章《Agentic 推理的性能重构》对十篇论文的综合分析系统论证了这一特征[\[4\]](https://miraclefarms.github.io/notes/2026/05/28/agentic-inference-kv-cache-benchmark-survey/)。

### 2.2 从 hit rate 到 effective hit length

很多系统报告 cache hit rate，但 agentic 场景里更应关注有效命中长度。图 3 展示了 vLLM x Mooncake 在 agentic trace 下的性能对比。Mooncake offloading 相比无 offloading：**吞吐提升 3.8 倍（7,669 vs 2,012 tok/s/GPU），P50 TTFT 从 82.93s 降到 1.80s（46 倍），端到端延迟从 93.3s 降到 10.8s（8.6 倍）**[\[1\]](https://vllm.ai/blog/2026-05-06-mooncake-store)。

![[KVCache：从推理工作负载到系统设计范式-04.png]]

*图 3：Kimi-K2.5-NVFP4 在 codex\_swebenchpro（12 GPUs, 50 convs）上的性能对比。分布式 KV pool 把 92.2% 命中率转化成可量化的吞吐和延迟收益。来源：vLLM x Mooncake 官方博客[\[1\]](https://vllm.ai/blog/2026-05-06-mooncake-store)。*

**92.2% 命中率之所以转化成如此显著的收益，是因为几乎整个历史 prefix 都能跨实例恢复**——不只是 system prompt 头部的几 K token，而是到当前轮为止的完整会话上下文。MiMo V2.5 的经验也印证了这一点：Hybrid SWA 把 KV 体积压到约 1/7，但 SWA 的窗口有效期只有最近 128 token，团队为前缀树引入”窗口安全长度”匹配规则，确保 Full Attention 和 SWA 状态同时可用后才算有效命中[\[5\]](https://mimo.xiaomi.com/zh/blog/mimo-v2-5-inference)。

这里要厘清一个常被混淆的口径：**effective hit length 并不只是 hit rate 换一种写法。系统实际上报的命中率会系统性高估可用缓存**，二者在三处分叉。其一是绝对数与比值之别——省下的算力正比于不重算的绝对 token 数，而非比值：50% 命中的 200K prompt 比 90% 命中的 1K prompt 省得多，两者对请求的排序甚至相反。其二是”匹配”不等于”可用”——前缀树报的是 token 序列匹配，但 Hybrid SWA 下匹配上的 token 其 KV 可能已经失效（SWA 窗口过期、或 Full/SWA 状态不齐）；这种”幽灵命中”若被当真复用，产出的是 silent corruption 而不只是变慢，所以 effective hit length 同时是一道正确性闸门（详见附录 9.5）。其三是连续性——KV 复用要求从位置 0 起连续有效，中间有一个空洞，其后的缓存块再多也作废，block 级命中率再高也无济于事。MiMo 的”窗口安全长度”正是把上报从”匹配长度”校正成”匹配且状态齐备的连续长度”。文章前面说 92.2% 命中率之所以转化成 3.8 倍吞吐，关键就在于那个数是 effective 的——**一个名义高、effective 低的系统，看板好看却既不省钱也可能出错。**

**Agentic workload 对 KVCache 系统的五类设计要求**（身份、生命周期、位置、调度、可观测性）详见附录 9.5。

### 2.3 从 GQA 到 MLA：KV 随上下文线性增长，斜率由 attention 结构决定

在进入调度和系统设计之前，要先把“KVCache 的体积从哪里来”讲清楚，否则后面“显存放不下”“offload 到下一层”这些判断都会悬空。**KVCache 的体积随上下文长度线性增长**：每读入或生成一个 token，模型都要为每一层缓存它的 key 和 value，序列有多长、缓存就累积多少条。这条线性关系的斜率——每 token 每层要存多少字节——由 attention 结构决定。

图 4 把四种主流结构并排画出来，斜纹标出的就是推理时真正要缓存的部分。MHA 给每个 query head 配一套独立 K/V，缓存量最大；GQA 让一组 query head 共享一组 K/V，把要缓存的 KV head 数压下来；MQA 走到极端，所有 head 共享一组；MLA 换了一条路，把完整 K/V 压成一段低秩 latent 来缓存，用时再投影回来。**同样长的上下文，这四种结构缓存的字节数能差出近一个数量级，而这正是后面所有显存与分层决策的起点。**

![[KVCache：从推理工作负载到系统设计范式-05.png]]

*图 4：四种 attention 结构在推理时缓存的 K/V（斜纹部分）对比。MHA 每个 query head 一套 K/V；GQA 让一组 query head 共享 K/V，缓存的 KV head 数下降；MQA 全部共享一组；MLA 缓存的是一段压缩 latent，用时再投影。缓存多少，决定 KV 随上下文增长的斜率。来源：前期调研文章《主流 Attention 算法全景》[\[49\]](https://miraclefarms.github.io/notes/2026/04/20/mainstream-attention-algorithms-overview/)。*

Full Attention 的 KVCache 容量可粗略写成：层数 × token 数 × KV heads × head dim × 2（K/V）× bytes。MHA、MQA、GQA 主要改变 KV heads 这一项；**MLA 则用 latent cache 替代完整 K/V，在结构层面压低 KV 容量。**

图 5 展示了 2026 年代表性模型在 100K token BF16 口径下的 KV Cache 体积横向对比[\[6\]](https://miraclefarms.github.io/notes/2026/06/01/kvcache-growth-attention-variants-essay/)。数据来自各模型官方 config 和技术报告。

![[KVCache：从推理工作负载到系统设计范式-06.png]]

*图 5：100K token BF16 下五款代表性模型的 KV Cache 体积。Qwen3.5（Hybrid GDN+GQA，15 full attn layers）约 2.4GB，Kimi K2.6（MLA，61 layers）约 7.0GB，DeepSeek V3.2（DSA/MLA+Indexer）约 7.8GB，GLM 5.1（DSA，78 layers）约 10.0GB，MiniMax M2.7（GQA，8 KV heads）约 25.4GB。最大最小相差超 10 倍，差异主要来自 attention 结构而非总参数量。数据来源：各模型官方 config[\[6\]](https://miraclefarms.github.io/notes/2026/06/01/kvcache-growth-attention-variants-essay/)。*

绝对 GB 数受层数和 token 数双重影响，对芯片和调度团队来说，归一化为 **bytes/token/layer** 更适合横向比较 attention 结构的存储效率。以 BF16（2 bytes/element）为基准：MiniMax M2.7（GQA, 8 KV heads, head\_dim 128）每 token 每层需存储 8 × 128 × 2 × 2 = 4,096 bytes；Kimi K2.6 的 Hugging Face config 写明 `kv_lora_rank=512`、`qk_rope_head_dim=64`，modeling code 里 `kv_a_proj_with_mqa` 的输出也正好被 split 成这两段，因此按实际 MLA cache state 应计为 512 + 64 = 576 个 BF16 元素，即 1,152 bytes/token/layer，而不是只看 latent rank 的 1,024 bytes/token/layer[\[46\]](https://huggingface.co/moonshotai/Kimi-K2.6/resolve/main/config.json)[\[47\]](https://huggingface.co/moonshotai/Kimi-K2.6/raw/main/modeling_deepseek.py)；Qwen3.5 的 GDN 层仅需维护固定大小 recurrent state，对应 bytes/token/layer 接近零。**这个归一化指标直接决定调度器的资源估算公式——同样 100K prompt，按 token 数乘以模型特定的 KV slope 才能得到准确的显存需求。**

这张图对系统设计的含义非常直接：**调度器看到同样 100K prompt，不能只按 token 数估算资源。**MLA/DSA/CSA 这类结构性压缩带来的机会是 KV payload 更小、batch 上限更高、PD 分离传输压力更低。挑战也直接：系统必须理解不同物理状态。DeepSeek V4 里同一个 token 可能对应 SWA、C4、C128、indexer、compress state 等多种物理形态[\[23\]](https://huggingface.co/deepseek-ai/DeepSeek-V4-Pro/blob/main/DeepSeek_V4.pdf)；Tair + SGLang 的 Shadow Radix 正是为这类多态 KV 建立逻辑坐标和 shadow pool 映射[\[7\]](https://www.alibabacloud.com/blog/alibaba-cloud-tair-partners-with-sglang-to-build-hicache-constructing-a-new-cache-paradigm-for-agentic-inference_602767)。**这类多态 KV 如何反过来改写缓存语义，留到第七章展开。**

### 2.4 Claude Code 的 KVCache budget 与布局

**（v1.9 新增）** 把前面几节的判断落到一个具体 harness 上，就能看清“业务模式”到底给 KVCache 提了什么要求。Claude Code 是公开材料里把上下文当成预算来管理的典型：它的上下文并不只是聊天历史，而是系统提示、工具定义、CLAUDE.md 层级、skills、session storage 和文件记忆等多个来源拼装出来的（图 6）。这些来源的突变率天差地别——系统提示和工具 schema 整会话不变，文件记忆按需载入，聊天历史只在尾部追加——harness 要做的，就是把稳定的放前面、易变的放后面，让 cache breakpoint 尽量靠后，从而最大化可缓存跨度。

![[KVCache：从推理工作负载到系统设计范式-07.png]]

*图 6：Claude Code 的上下文来源——系统提示、工具、CLAUDE.md 层级、skills、session storage、auto memory——按可重建性和失效方式被分层治理；上下文窗口在这里是受治理的运行时资源，不是越大越好的文本框。布局顺序直接决定哪些段落落在稳定的可缓存区。来源：前期调研文章《Claude Code 论文再读》[\[50\]](https://miraclefarms.github.io/notes/2026/06/01/claude-code-design-space-reading/)。*

**让稳定 prefix 持续命中 cache read，才是真正决定 agent 成本的那一项——上下文窗口有多大反而次要。**Anthropic 的 prompt caching 把这件事直接做成计费与 TTL 接口：命中 cache read 只要未命中 input 的约十分之一价钱，1 小时 TTL 专为工具执行那几十秒的 gap 续命[\[3\]](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)。于是 harness 的上下文工程，本质上是在为下游的 KVCache 预算和布局服务：每多设一个 cache breakpoint、每一次 context compaction 重写历史，都会在 KV 层面变成一次缓存失效和一段重算。这把第一章“prefix 稳定性已成为可计价资源”的判断，从 API 计费一路压到了 KV 存储层——**应用层怎么摆放上下文，直接决定了 KV 这摊状态的命中率、留存时长，以及它该落在存储层级的哪一层。**

## 三、调度逻辑：从 queue length 到 state locality

> **本节论点**：业务模式确定后，调度的对象就从”算力够不够”变成”KV state 在哪、回读多贵”。一条暗线贯穿本章——调度信号沿 `队列长度 → 前缀命中 + 负载 → KV 物理位置 → 在线学出来` 逐级演化；KVCache 把缓存命中与未命中之间约一个数量级的成本差摆上台面，而只有调度层能吃到这个杠杆。但调度信号的可见度有物理天花板——query-aware demand fetch 原理上无法前置，取数路径必须 SM-free，这两条结构性约束是调度优化越不过的边界，也是第六章讨论内存语义通道收益的出发点。 **本节核心参考**：
> 
> - ★★★ **MiMo V2.5 官方博客**[\[5\]](https://mimo.xiaomi.com/zh/blog/mimo-v2-5-inference) — 本节核心图（图 7），prefix\_match\_percentage + load 路由的生产实测
> - ★★★ **llm-d：KV-Cache Wins You Can See**[\[51\]](https://llm-d.ai/blog/kvcache-wins-you-can-see) — 8 pod / 16 H100 上精确 vs 近似调度的一手吞吐/TTFT 对比，量化”可观测性是天花板”
> - ★★ NVIDIA Dynamo / SGLang KV-aware routing[\[22\]](https://developer.nvidia.com/blog/full-stack-optimizations-for-agentic-inference-with-nvidia-dynamo/)[\[35\]](https://www.lmsys.org/blog/2024-12-04-sglang-v0-4/) — Flash Indexer 与 cache-aware 负载均衡的集群路由落点
> - ★★ Lodestar 论文[\[53\]](https://arxiv.org/abs/2606.00946) — 把路由从”写规则”改成”学规则”的信号终点，异构集群收益最高约 4.4×
> - ★★ KVDrive 论文[\[60\]](https://arxiv.org/abs/2605.18071) / ShadowServe 论文[\[59\]](https://arxiv.org/abs/2509.16857) — KV 选择+取数占 decode 时间近 50%，约 20% 关键 KV 访问为 demand fetch 落在关键路径（KVDrive）；SM-free 约束与 SmartNIC 卸载后 loaded TPOT 最高 ↓2.2×（ShadowServe）
> - ★★ Price of Anarchy 论文[\[84\]](https://arxiv.org/abs/2606.17081) / DualMap 论文[\[86\]](https://arxiv.org/abs/2602.06502) — 反热点张力的饱和相变点（拐点后第一格为并发 128）与双哈希映射解法（同 TTFT SLO 下有效容量最高 ↑2.25×）
> - ★ 前期调研文章《大规模分布式推理调度层（上 / 下）》[\[54\]](https://miraclefarms.github.io/notes/2026/06/08/inference-scheduling-kv-locality-survey-part1/)[\[55\]](https://miraclefarms.github.io/notes/2026/06/08/inference-scheduling-kv-locality-survey-part2/) — 四个调度尺度与信号演化主线的完整梳理

调度逻辑放在框架实现之前讲，是因为它定义了后面所有实现要满足的控制目标——具体落到 vLLM/SGLang/TRT-LLM 哪个组件，留到第四章。

### 3.1 信号的演化：从队列长度到 KV 物理位置

过去两年推理优化的重心在 kernel——FlashAttention、GEMM/MoE 融合、量化路径，让单位 token 更便宜。但再快的 kernel 也救不回一次本不该发生的 prefill：当一段三万 token 的前缀已经在某个 worker 的显存里算好，把请求送去别处重算，赔掉的是整段 prefill，远不止百分之几的算力。llm-d 给出的估算是缓存命中与未命中之间约 10 倍的成本差[\[51\]](https://llm-d.ai/blog/kvcache-wins-you-can-see)，**这个数量级的杠杆只有调度层能吃到**——这正是调度从负载均衡里分化成独立主战场的物理原因。顺着这条线，调度信号自己在演化：最早看队列长度，谁空给谁；接着前缀命中率进入决策，路由偏向最可能命中已有 KV 的节点；再往后又进一步：路由要读的是 KV 的物理位置本身——它在哪个实例、哪一层存储。

传统 serving router 常看 queue length、GPU utilization、request length。**KVCache-aware 调度要加三类信号**：prefix locality（哪个 worker 已有这段 prefix）、effective compute delta（命中 95K 时只需计算新增 5K）、存储和网络压力（RDMA queue 满或 G2 host pool 满时不适合接长前缀 load-back）。

这类调度其实落在两层，不该混为一谈。一层是**引擎内调度器**（vLLM V1 Scheduler、SGLang 的 RadixAttention、TRT-LLM 的 KVCacheManager），在单实例内做 continuous batching 与本地 prefix 命中；另一层是**集群路由器/网关**，跨实例决定请求发往哪个 worker，需要全局 prefix locality。三大框架的落点不同：vLLM 的跨实例 KV-aware routing 不在内核，外包给 production-stack router 或 llm-d 网关（Endpoint Picker + KV-Cache Indexer）[\[34\]](https://developers.redhat.com/articles/2025/10/07/master-kv-cache-aware-routing-llm-d-efficient-ai-inference)；SGLang 用第一方的 sgl-router（维护每 worker 的近似 radix tree 做 cache-aware 负载均衡）[\[35\]](https://www.lmsys.org/blog/2024-12-04-sglang-v0-4/)；TRT-LLM 自身无集群路由，靠 Dynamo 的 KV-aware router（KVIndexer + overlap score）[\[36\]](https://docs.nvidia.com/dynamo/latest/user-guides/kv-cache-aware-routing)。而 **KVBM、HiCache、LMCache 不是路由器，它们是管理层，向上发 KV 事件、把缓存状态喂给这些 indexer**。这里先点出一组区分（第四章 4.2 会细说）：集群路由器是前门，它查询的 indexer（KVIndexer、KV-Cache Indexer、近似 radix tree、Mooncake Master）就是目录；引擎内的 prefix caching 则负责产生喂给目录的 KV 事件。

图 7 展示了 MiMo V2.5 的生产调度效果[\[5\]](https://mimo.xiaomi.com/zh/blog/mimo-v2-5-inference)。

![[KVCache：从推理工作负载到系统设计范式-08.png]]

*图 7：MiMo 把 prefix\_match\_percentage 和 normalized\_load 结合做路由决策。上图为 Long Request TTFT：P90 从 6.88s 降到 4.78s（30.5%↓），P99 从 18.50s 降到 16.57s（10.4%↓）；下图为 Short Request，改善不显著（具体秒数为图中数据，博客正文只给出”P90 降低 30%”的相对口径）。KV-aware 调度对长 prefix 请求效果最明显。来源：MiMo 官方博客[\[5\]](https://mimo.xiaomi.com/zh/blog/mimo-v2-5-inference)。*

MiMo 的路由器（属上面说的集群路由层，信号则由引擎内的 SWA-aware prefix tree 产出）把 prefix\_match\_percentage 和 normalized\_load 结合权衡，背后的道理很朴素：前者衡量**重算成本**（命中得多就能跳过大段 prefill），后者衡量**排队成本**（worker 越忙，就算有缓存也要排队）。只看命中率，热门 prefix 会把请求全导向同一台 worker、排成热点，命中率好看但实际延迟更差；只看负载，请求被均匀打到没有缓存的 worker，又要全量重算。**两者结合，才能挑出”既缓存了我大半前缀、又不太忙”的那台。**图 7 长短请求的对比正是这个道理的实测印证：长请求前缀长、重算成本项大，locality 路由收益显著（P90 TTFT 降 30.5%）；短请求没多少可省，光均衡负载就够，结合信号收益不明显。

MiMo 实测 L2 cache 命中率提升约 25%、单机输入吞吐提升约 30%[\[5\]](https://mimo.xiaomi.com/zh/blog/mimo-v2-5-inference)，是这条路上最朴素一步的生产收益。

### 3.2 三种路由实现：精度与代价的坐标

**（v1.10 新增）** 要按 KV 位置路由，先得回答一个工程问题：路由器怎么知道每段 KV 现在在哪？围绕这份位置信息的获取方式，开源路由器分出三个精度坐标。近似一端是 SGLang 的 sgl-router——路由侧为每个 worker 维护一棵近似 radix tree，不与 worker 同步、开销极低，但缓存频繁迁移、淘汰时，近似视图会和真实状态产生偏差[\[35\]](https://www.lmsys.org/blog/2024-12-04-sglang-v0-4/)；vLLM Router 走得更省，索性不维护任何缓存视图，用 Rust 实现的一致性哈希把同一会话黏到同一 worker，靠 sticky 路由间接保住 KV 复用[\[52\]](https://vllm.ai/blog/2025-12-13-vllm-router-release)。精确一端是 llm-d：每个 vLLM pod 主动发出 KVEvents，路由侧用两层索引把 token 前缀映射到具体 pod，掌握的是精确位置——管理 365GB 缓存池，索引只占约 339KB 内存，数据与元数据之比超过百万比一[\[51\]](https://llm-d.ai/blog/kvcache-wins-you-can-see)。再把精确做到极致吞吐，是 Dynamo 的 Flash Indexer——全局 KV block 索引支持每秒 1.7 亿次查询，路由目标升级为“前缀重叠分数 + decode 负载”的联合最小化[\[22\]](https://developer.nvidia.com/blog/full-stack-optimizations-for-agentic-inference-with-nvidia-dynamo/)。**同一条轴上：从无视图的哈希黏附、近似模拟，到精确事件推送，再到高吞吐精确索引，精度逐级上升，维护这份信息的基础设施也逐级变重。**

这份精度值多少钱，llm-d 把它量化了出来：8 个 vLLM pod、16 张 H100 的仿真负载上，精确调度的 P90 TTFT 是 0.542 秒，近似调度是 31.083 秒——快 57 倍；吞吐上精确比近似高 25%，比完全不感知缓存的配置高出一倍多[\[51\]](https://llm-d.ai/blog/kvcache-wins-you-can-see)。这组数字指向一个判断：**KV 感知调度的能力上限，由 KV 状态可观测性的上限决定**——路由器能做到多聪明，最终卡在它能多精确、多实时地知道每段 KV 在哪，打分函数再精巧也越不过这条可观测性天花板。沿这个思路，可纳入的信号还有很多：命中前缀所在的存储层级（在 HBM 还是 SSD，取回成本差一个数量级，llm-d 的 indexer 已记录 block 在 GPU 还是 CPU 层[\[51\]](https://llm-d.ai/blog/kvcache-wins-you-can-see)）、effective hit length、剩余 KV 预算与逐出风险、补齐缺失前缀的跨节点搬运代价（NVLink vs 跨 rack RDMA vs CXL）。把这些成本项一起喂进去、输出一个统一的执行计划，正是附录 9.9 设想的“像数据库优化器一样的 KV-aware 控制面”——tier-aware、link-aware 路由是看得见的下一步。

**（v1.13 更新）** 这条精度坐标上 7 月刚落进一个新玩家。Mooncake 论文里的 Conductor 调度器此前只存在于设计文档——开源仓库里只有架构图和 RFC（#977），运行时代码一直缺位。2026-07-12 合入的 PR #2833 把它落成 mooncake-store 下的独立 C++ 服务：通过 ZMQ 订阅 vLLM 与 Mooncake 的 BlockStored/BlockRemoved KV 事件流（支持 replay），维护前缀命中索引，经 HTTP `/query` 向路由器返回各实例的命中情况[\[75\]](https://github.com/kvcache-ai/mooncake/pull/2833)。放进本节的坐标系里，它落在“精确事件推送”一档——与 llm-d KVEvents 思路同款，但以框架中立的独立目录组件出现：**路由器不必绑定 llm-d 或 Dynamo 的网关栈，也能拿到精确的 KV 位置视图。**对第八章排在最前那条开放问题来说，这是目录接口标准化的一次实质投票。

**（v1.14 新增）** 上一段说 link-aware 路由是看得见的下一步，7 月下旬这条路铺上了第一块砖。Mooncake TENT 在 07-20 合入的 PR #2996，把 RDMA NIC 的负载统计沿 Transport 接口向上暴露[\[88\]](https://github.com/kvcache-ai/Mooncake/pull/2996)，两天后又给传输指标补上 per-transport 标签。单看是两个可观测性补丁，放进本节的坐标系里含义要大一些：**此前路由器能读到的只有 KV 在哪，现在传输层开始告诉它，取过来的那条链路眼下堵不堵。**打分函数要从「命中长度减负载」扩展成「命中长度减负载减取回代价」，缺的一直是最后这一项的实时来源。这层信号目前还只到传输库为止——没有哪个开源路由器在消费它，从 NIC 计数器到路由决策之间的那段管道尚未接上。

### 3.3 反热点张力与学习式终点

**（v1.10 新增）** 可观测性带来的精确有一个甩不掉的影子。当路由器精确知道某段热门前缀在哪台实例，它会把相关请求都导过去，结果那台 worker 过热、别处闲置——缓存亲和与负载均衡在这里正面冲突。MiMo 打分函数里那个减负载项、配合等待时间惩罚防止长请求饥饿，就是在缝合这道冲突[\[5\]](https://mimo.xiaomi.com/zh/blog/mimo-v2-5-inference)；Mooncake 在 Conductor 里把它写成显式策略：系统均衡时走 cache-first，失衡时切到 load-first，并从低负载的一组里随机挑选避免制造新热点[\[26\]](https://miraclefarms.github.io/notes/2026/06/02/mooncake-unified-data-plane-architecture/)。**反热点是 KV 感知调度的内生张力，集群、相位、跨区域每一层都要重新平衡一次**——只要调度开始追逐缓存局部性，就得同时防着自己把负载堆成尖峰。

**（v1.14 新增）** 上面这些做法都把反热点当成打分函数里的一个修正项，过去两个月有两篇工作把它推进了一层。第一层是知道这道冲突什么时候真的开始咬人。《The Price of Anarchy in Disaggregated Inference》把 Dynamo 的 Planner、Smart Router 和 KV Block Manager 建模成三个耦合博弈——P/D 资源博弈、KV 缓存上的自私缓存博弈、请求路由的拥塞博弈——并在 3 节点 B200 集群上用 Nemotron-4-340B 和 Llama-3.1-70B 实测。结论是这套系统存在相变：饱和之前，各组件各自最优带来的无政府代价有界；越过饱和拐点（两个模型的拐点后第一个网格点都是并发 128），超线性延迟和缓存外部性把它推上去。作者据此做了一个实时检测饱和、动态从缓存亲和切向负载均衡的自适应控制器，在 70B 1P/5D 拓扑上把饱和期的无政府代价估计量压掉 3.1 倍（66.4 → 21.5），代价是约 13% 吞吐[\[84\]](https://arxiv.org/abs/2606.17081)；参数 sweep 的逐格数据与三层博弈的完整拆解见前期调研文章《P/D 分离推理的失控点》[\[85\]](https://miraclefarms.github.io/notes/2026/07/03/dynamo-disaggregated-inference-price-of-anarchy/)。**缓存亲和在饱和之前基本是白拿的收益，越过拐点它才开始和负载均衡抢东西**——这解释了为什么不少团队在压测里调不出参数差异、一上生产峰值就翻车：调参实验根本没跑到相变点那一侧。要留意这里的口径，论文报的是自己定义的经验估计量而非严格意义上的 PoA 上界，跨系统横比时不能直接搬数。

第二层是换掉机制本身。DualMap 指出既有调度器都困在同一个映射空间里，只能把一部分请求交给亲和路由、另一部分交给负载均衡，两个目标始终此消彼长；它的做法是用两个独立哈希函数把每个请求映到两个候选实例，再按当前系统状态挑更好的那个——”二选一的力量”让共享前缀更容易撞在一起，不同前缀又被自然打散，同一 TTFT SLO 下有效请求容量最高提升 2.25 倍[\[86\]](https://arxiv.org/abs/2602.06502)。**把亲和与均衡从一个打分函数里的加减项，改成两个候选之间的选择题，这道张力才第一次有了不靠调权重的解法。**这条路的边界也清楚：双哈希把命中概率提高到”更容易共置”，代价是放弃了单映射下”同前缀必然同实例”的确定性，对超长会话的稳定复用是否够用，公开数据还不足以下结论。

到这里，前面所有调度都还是人写的规则——观察负载、提炼公式、调好权重。这套办法在同构集群、规整负载下工作得很好，裂缝在三个条件同时出现时显露：执行高度依赖输入、batching 与 KV 复用造成强跨请求耦合、延迟对上下文长度和加速器型号高度非线性[\[53\]](https://arxiv.org/abs/2606.00946)——而这恰好就是真实生产负载的样子。Lodestar 把路由从“写规则”改成“学规则”：持续采集集群快照训练一个在线奖励预测器，直接朝最小化 TTFT 优化，约 5 分钟就能收敛出有效策略[\[53\]](https://arxiv.org/abs/2606.00946)。最该读的是异构那一栏——同构集群 TTFT 最高改善约 2.1×，异构直接跳到最高约 4.4×（平均口径分别为 1.41× 与 1.47×），**因为异构正是静态代价模型最表达不了的地方**：不同型号加速器的延迟曲线各异、相互还非线性耦合，人写的公式很难覆盖全[\[53\]](https://arxiv.org/abs/2606.00946)。这一点对国内多类型芯片混推尤其有意义——一个集群里同时插着好几家加速器、算子支持与精度对齐各不相同，固定打分函数的天花板会很快撞到，而这恰是学习式路由的主场。规则不会就此退场，更可能是规则托底、学习增益，长期共存，这仍是活跃未定的判断。

**（v1.11 更新）** 即使信号齐备、路由学出来，仍有一类 miss 原理上无法前置搬运。主流 KV I/O 路径（Mooncake、vLLM 异步 prefetch）在 admission 时发起搬运、让传输被后续计算掩盖——这有一个隐含前提：调度器此时已知道要搬哪些 block。**query-aware 稀疏访问打破了这个前提**：Quest 这类按 decode step 做 Top-k page 选择的方案[\[62\]](https://arxiv.org/abs/2406.10774)，选哪些页由本步 query 向量决定，admission 时原理上无法预知。KVDrive 的时间分解显示 KV 选择与取数合计占 decode 时间近 50%，滑动窗口缓存能让约 80% 的关键 KV 直接命中 GPU，剩下约 20% 是 demand fetch，落在 decode 关键路径上、没有任何计算可掩盖[\[60\]](https://arxiv.org/abs/2605.18071)。还有一个独立的干扰通道：ShadowServe 实测 KV 解压/反量化 kernel 与 decode 计算在 GPU 上并发——arithmetic decoding 场景下，**无论使用 CUDA streams、MPS 还是 Green Context，都无法把双方 slowdown 同时压到 30% 以下**；把整条取数流水卸载到 SmartNIC 后 loaded TPOT 最高降低 2.2 倍[\[59\]](https://arxiv.org/abs/2509.16857)。这两个限制收束成同一个系统要求：取数路径必须是 SM-free 的纯 DMA——这是调度层无法靠信号优化绕开的物理约束，也是第六章讨论内存语义通道的出发点。

**（v1.14 更新）** 上面这段停在”无法前置”这个诊断上，SAC 顺着它给出了一条出路：既然预取注定盖不住 query-aware 的 top-k 选择，那就把单次取数做到便宜得可以直接留在关键路径上。SAC 实测从 128K 上下文里捞稀疏 KV 时，RDMA 的检索延迟是本地 DRAM 的 4.0–19.7 倍、并随取数条数恶化到毫秒量级，而 CXL 的 cache-line load/store 配 layer-first 布局只有本地 DRAM 的 1.04–1.64 倍[\[81\]](https://arxiv.org/abs/2606.19746)（该实验的完整边界与取数曲线见前期调研文章《SAC：用 CXL 给稀疏注意力做一个只取 top-k 的分离式 KV Cache》[\[82\]](https://miraclefarms.github.io/notes/2026/06/21/sac-cxl-sparse-attention-kv-cache/)）。**“无法前置”是调度层的死结，换到取数便宜的底座上它就不再需要被解开**——这条线在 §6.2 收口。

**请求调度必须理解 KV state，不能只看算力；而它理解得多深，最终由 KV 可观测性与这套学习能力的上限决定。**

## 四、框架实现：三大框架把 KVCache 管理放到了哪一层

> **本节论点**：vLLM、SGLang、TRT-LLM 三条路线起点不同，但都把 KVCache 管理推进了 scheduler 的主控制回路；框架外再分出缓存管理、数据移动、平台化三类角色。这是一个多组件问题，主线落在“管理平面的三类角色如何分工”。 **本节核心参考**：
> 
> - ★★★ **NVIDIA Dynamo KVBM 设计文档**[\[13\]](https://docs.nvidia.com/dynamo/design-docs/component-design/kvbm-design) — 本节核心图（图 10 组件架构）来源，把缓存管理 / 数据移动 / 平台化三类角色讲清楚
> - ★★ PagedAttention 论文[\[2\]](https://arxiv.org/abs/2309.06180) — vLLM 的 block-first 起点
> - ★★ SGLang HiCache 设计文档[\[10\]](https://docs.sglang.io/advanced_features/hicache_design.html) — SGLang 的 prefix-first 起点（图 8 RadixAttention）
> - ★★ vLLM x Mooncake 官方博客[\[1\]](https://vllm.ai/blog/2026-05-06-mooncake-store) — 跨实例 prefix caching 架构（图 9）
> - ★ 前期调研文章《KV Cache 前缀匹配的设计分野》[\[12\]](https://miraclefarms.github.io/notes/2026/05/13/kvcache-prefix-matching-design/) — 三框架源码级对比

### 4.1 三大框架的出发点

图 8 展示了 SGLang 的 RadixAttention 概览——以 radix tree 为核心的 prefix-first KVCache 管理方式。不同框架从不同起点出发，**但都在向 KVCache 一等公民的方向收敛。**

![[KVCache：从推理工作负载到系统设计范式-09.jpg]]

*图 8：SGLang 用 radix tree 组织共享前缀，天然服务多轮、多分支、多请求复用。节点合并与分裂操作可视化了 prefix cache 如何跟随请求动态演变。来源：SGLang RadixAttention 论文[\[31\]](https://arxiv.org/abs/2312.07104)。*

**vLLM 的出发点是 block-first。** PagedAttention 把 KV 拆成固定 block，通过 block table 映射逻辑 token 到物理显存页，解决连续分配和碎片问题[\[2\]](https://arxiv.org/abs/2309.06180)。后续 vLLM V1 进一步把 scheduler、KVCacheManager、BlockPool 和 connector 串成一条主路径。

**SGLang 的出发点是 prefix-first。** RadixAttention 把共享前缀做成 radix tree（如图 8），天然服务多轮和多分支复用；HiCache 再把 GPU HBM、host memory、local disk、remote distributed storage 统一成层级缓存[\[10\]](https://docs.sglang.io/advanced_features/hicache_design.html)。

**TensorRT-LLM 的出发点更偏 engine-integrated。** 官方 KV Cache System 文档支持跨请求复用、offloading、prioritized eviction、variable attention window size，并配合 MQA/GQA 优化[\[11\]](https://nvidia.github.io/TensorRT-LLM/features/kvcache.html)。

三者的关系不是替代。vLLM 更像操作系统页表，SGLang 更像专为 prompt 复用设计的索引树，TRT-LLM 更像推理 engine 内建的高性能内存子系统。需要强调，block-first / prefix-first / engine-integrated 描述的是三者的架构重心和主索引结构，而非互斥特性——今天三家底层都用 paged KV、也都支持 prefix caching（SGLang 的 radix tree 其实是架在 paged block 之上的索引，vLLM 后来也补齐了 prefix caching）。**真正的分野在于主索引结构带来的匹配粒度、分支表达、并发行为和扩展方向**，逐维对比见 4.4 表 1。前期调研文章已有三大框架的 KVCache 源码级对比[\[12\]](https://miraclefarms.github.io/notes/2026/05/13/kvcache-prefix-matching-design/)。

### 4.2 Prefix caching：从本地命中到跨实例命中

**Prefix caching 是 KVCache 从”省单次计算”走向”省系统成本”的第一步。**图 9 展示了 vLLM + Mooncake 的分布式 KV pool 架构[\[1\]](https://vllm.ai/blog/2026-05-06-mooncake-store)。

![[KVCache：从推理工作负载到系统设计范式-10.svg]]

*图 9：vLLM 多实例嵌入 Mooncake client，共享集群级 Mooncake Store。Master 管 metadata，workers 通过 RDMA 在 GPU HBM 与分布式 DRAM/SSD pool 间搬运 KV。Scheduler 侧先把 prompt token block hash 后查询 Mooncake master，worker 侧把 GPU KV cache memory 注册为 RDMA buffer，用后台线程通过 GPUDirect RDMA 搬运。来源：vLLM 官方博客[\[1\]](https://vllm.ai/blog/2026-05-06-mooncake-store)。*

本地 prefix cache 用链式哈希（vLLM）、RadixTree（SGLang）、多维 BlockKey（TRT-LLM）等数据结构实现自动前缀匹配。但当请求可能被路由到任意实例时，**本地 hash table 命中只是第一层，远端 KV pool 命中才是 agentic serving 的关键。**图 9 的架构实现了 round-robin 路由下仍保持 95% 以上 hit rate[\[1\]](https://vllm.ai/blog/2026-05-06-mooncake-store)（这是 12–60 GPU 扩展实验的口径，与开篇 92.2% 那组 12 GPU / 50 conv 的具体设定不同），这是 prefix cache 边界扩张的直接证据。

图 9 里的 Mooncake Master 容易被误读，值得点明：**它是分布式 KV 存储的元数据/目录服务**，对标分布式存储里的 HDFS NameNode 或 Alluxio Master——记录”哪个 block 在哪个节点、哪段内存”、负责分配与淘汰，但本身不在数据路径上（worker 查到位置后直接走 RDMA 点对点搬运，数据不经过 Master）。**它不是服务网关：网关/路由器是请求的”前门”，决定请求发往哪个实例；Master 是 KV 的”目录”。**两者的关系是——做 KV-aware 路由的网关会去查 Master 这类索引来判断 prefix 落在哪台机器（图 9 中 scheduler 把 block hash 后查询 master 正是这一步）。这层”前门”与”目录”的区分，第三章的 KV-aware 调度已经用到。

### 4.3 管理平面：KVBM、Mooncake、NIXL 的分工

图 10 展示了 NVIDIA Dynamo KVBM 的组件架构，把缓存管理的三类角色讲得很清楚[\[13\]](https://docs.nvidia.com/dynamo/design-docs/component-design/kvbm-design)。

![[KVCache：从推理工作负载到系统设计范式-11.svg]]

*图 10：KVBM 上接 TRT-LLM / vLLM / SGLang 三大 runtime（通过 Connector Interface），中间是 Core（KVBlockManager、Scheduler、State、Events、Metrics），下层是 Storage Pools（Device、Host、Disk、NFS Disk），底部通过 NIXL 对接 Block/Local File/Remote File/Remote Object/Cloud 等后端。来源：NVIDIA Dynamo KVBM 设计文档[\[13\]](https://docs.nvidia.com/dynamo/design-docs/component-design/kvbm-design)。*

这张图把管理平面的三类角色分得很清楚。**第一类是缓存管理层**（HiCache、KVBM、LMCache），关心某段 KV 是否存在、是否值得保留、放在哪一层、什么时候迁移、如何暴露给 scheduler。这三者其实是同一角色在不同生态里的对应物：KVBM 属 NVIDIA Dynamo，但它是 runtime 中立的——通过 Connector Interface 同时上接 TRT-LLM / vLLM / SGLang（图 10），并不绑定 TRT-LLM[\[13\]](https://docs.nvidia.com/dynamo/design-docs/component-design/kvbm-design)[\[28\]](https://miraclefarms.github.io/notes/2026/06/02/nvidia-dynamo-kvbm-architecture-field-note/)；HiCache 是 SGLang 内建的同位组件；LMCache 则是 vLLM 经 KVConnector 外接、并可跨引擎共享的同位组件。**第二类是数据移动/存储 substrate**（Mooncake、NIXL），Mooncake 同时有 Transfer Engine 和 Store，已进入 PyTorch 生态[\[14\]](https://pytorch.org/blog/mooncake-joins-pytorch-ecosystem/)[\[26\]](https://miraclefarms.github.io/notes/2026/06/02/mooncake-unified-data-plane-architecture/)；NIXL 提供 agent、descriptor、metadata、backend plugin 抽象[\[15\]](https://developer.nvidia.com/blog/enhancing-distributed-inference-performance-with-the-nvidia-inference-transfer-library/)[\[27\]](https://miraclefarms.github.io/notes/2026/05/26/nvidia-nixl-inference-transfer-layer/)。这里要纠正一个常见误解：KVBM 与 Mooncake 不是同类——KVBM 是管理层（决定放哪、留不留），Mooncake 是它会去消费的底座（Mooncake 的 Transfer Engine 对标 NIXL，Store 对标 KVBM 的 G4 远端后端）。**第三类是平台化/商业化服务**（Tair KVCache、华为 UCM、Penguin MemoryAI），UCM 文档称与 vLLM 集成后在多轮对话和长上下文 reasoning 场景降低 3-10 倍推理延迟[\[16\]](https://ucm.readthedocs.io/en/latest/index.html)。所谓平台化，是把前两类能力打包成可租用或购买、由厂商运维的产品，并补上开源框架结构上不提供的运营层——多租户隔离、RAS、telemetry、跨框架运营，乃至硬件 appliance（Penguin 是 CXL KV server 形态）。

需要说明，这三类角色并不在同一根坐标轴上：前两类是功能切分（管理策略 vs 传输/存储机制），第三类是”谁拥有、运维、售卖”的产业维度，一个平台产品内部往往同时包含前两类、再裹上运营层。把它们并列只是为了呈现完整生态；它们的真正分工，下一节会用两张表按”层”重新摆清。无论如何，一个趋势是清楚的：**KVCache 管理正在从开源框架能力变成 AI 云和硬件厂商的系统产品。**

### 4.4 框架对比：两张表

推理框架和框架外的 KV 组件不是一类东西，分两张表看更清楚。

表 1 比较三个推理框架——它们的真正分野不在”强项”，而在把 KVCache 管理的多少层放进框架、多少外包出去。因为只有三个框架，把维度当行、框架当列更易容纳完整对比：

| 维度            | vLLM                                                | SGLang                | TRT-LLM                          |
| ------------- | --------------------------------------------------- | --------------------- | -------------------------------- |
| 架构重心          | block-first                                         | prefix-first          | engine-integrated                |
| 主索引结构         | 链式哈希表 + block table                                 | radix tree            | 引擎内 KVCacheManager + 多维 BlockKey |
| 匹配粒度          | block（默认 16，向下取整）                                   | token（最细，不取整）         | block（多维 key）                    |
| 前缀查找成本        | O(1) 哈希                                             | O(匹配深度)，指针追逐          | 编译期协同优化                          |
| 分支 / fork 表达  | refcount 隐式                                         | 树原生（多轮/采样/agent fork） | 引擎管理                             |
| 并发行为          | 哈希易分片，锁竞争小                                          | 树 split/merge 有锁竞争    | 引擎统一管理                           |
| L1（HBM）       | 自带 BlockPool                                        | 自带                    | 内建                               |
| L2/L3 分层      | 接口外包、实现内建中（第一方 OffloadingConnector 多级分层，v1.13 更新见下） | 自带 HiCache 四级         | 内建 offload/eviction              |
| 跨实例 / 集群 pool | 外包（connector）                                       | HiCache + 远端后端        | 外接 connector + 数据面               |
| 开放性           | 中（生态广）                                              | 高（树 + tier 开放）        | 低（编译/封闭）                         |
| 最适场景          | 通用 serving、生态集成                                     | 多轮 agent、共享前缀         | 极致吞吐、单栈部署                        |

读这张表的关键是”L2/L3 分层”那一行：**三家的真正差异是把分层管理放进框架的幅度**——SGLang 最自带（HiCache 把四级全包），vLLM 最外包（只定义 KVConnector 接口，把 L2/L3 和集群 pool 全部交给外部组件），TRT-LLM 居中（本地分层内建、跨节点外接）。

**（v1.13 更新）** 上面”vLLM 最外包”的判断在过去一个月需要修正。v0.23 起，vLLM 的第一方 OffloadingConnector 长成了一个多级 offload 框架：CPU tier 之下可挂文件系统和 S3 类对象存储二级层（经 NIXL 访问）[\[63\]](https://github.com/vllm-project/vllm/releases/tag/v0.23.0)[\[66\]](https://github.com/vllm-project/vllm/pull/41968)，offload 决策支持 per-request 策略钩子；v0.24 又补上多层异步批量查询、self-describing KV 事件和分层占用 metrics[\[64\]](https://github.com/vllm-project/vllm/releases/tag/v0.24.0)。架构上它仍走 KVConnector 接口（这条”前缀缓存与卸载刻意解耦”设计线的源码级拆解见前期调研文章《vLLM 前缀缓存与分层卸载》[\[80\]](https://miraclefarms.github.io/notes/2026/06/16/vllm-prefix-cache-offload/)），但管理层本体已经进了主仓库——**这一行应改读为”接口外包、实现内建中”：vLLM 正在长出自己的 LMCache/HiCache 同位组件。**对表 2 的生态含义是，管理层三个同位物旁边挤进来一个框架第一方实现，外置管理层的生存空间被压向跨引擎共享和运营级功能。同月 TRT-LLM 沿相反方向移动：把引擎内状态向外暴露——存量 block-hash chain 主动交给 connector[\[72\]](https://github.com/NVIDIA/TensorRT-LLM/pull/14806)、cache\_salt 写进 KV 事件流，并新增 BaseResourceManager 级的 KV 压缩管理框架与 DSv4 稀疏缓存适配器[\[73\]](https://github.com/NVIDIA/TensorRT-LLM/pull/15106)，engine-integrated 路线开始给外部管理层留出标准挂点。LMCache 则在跨引擎与介质两个方向推进：v0.5.1 的多进程（MP）模式走向成熟，本地缓存层新增 Device-DAX 介质支持——CXL/持久内存设备以内存映射方式直接充当缓存层（对应本文口径的 L2.5 半层，第一次有了开源管理层消费者）[\[74\]](https://github.com/LMCache/LMCache/releases/tag/v0.5.1)。三家移动方向各异，但都在往同一个接缝挤：**管理层与引擎之间的接口正在变厚、变标准。**

**（v1.14 更新）** 被框架第一方实现挤压的外置管理层，7 月给出了它的答卷方向。LMCache v0.5.2（2026-07-22）在三处同时加码：多进程模式下 coordinator 开始在 L1 层用 P2P 做全局 token 匹配；缓存治理长出显式 API——pin/unpin token、按 token 删除、chunk 驱逐带默认配额；L2 后端一口气接上 Azure Blob、Valkey（单机与集群）、按 layer-group 分片的 Cloud Bigtable 和 SageMaker HyperPod 适配器[\[87\]](https://github.com/LMCache/LMCache/releases/tag/v0.5.2)。三件事指向同一个定位选择：**当引擎自己长出分层实现，外置管理层的差异化就落在”跨引擎、跨介质、可被运营”这一侧**——第一方实现很难去接一堆云厂商的对象存储和托管 KV 服务，也不会为多租户做 pin 和配额。§4.3 说平台化是”把前两类能力打包再裹上运营层”，LMCache 这一版是开源管理层自己把运营层长了出来。

表 2 比较框架外的 KV 管理/数据面组件。它们最容易被混为一类，**但其实分属不同”角色层”——这一列才是理解它们的钥匙**：

| 组件                   | 角色层                   | runtime 绑定       | 同位 / 对标                                                 | 关键边界                  |
| -------------------- | --------------------- | ---------------- | ------------------------------------------------------- | --------------------- |
| KVBM（Dynamo）         | 管理层                   | 中立，接三框架          | ↔ HiCache、LMCache                                       | 缺大规模公开 benchmark      |
| LMCache              | 管理层                   | 跨引擎（vLLM+SGLang） | ↔ HiCache、KVBM                                          | 经 connector 接入，受框架接口限 |
| NIXL                 | 传输底座                  | 中立               | ↔ Mooncake Transfer Engine                              | 不替代调度和 cache value 判断 |
| Mooncake             | 传输+存储底座（含 Master 元数据） | 中立（PyTorch 生态）   | Transfer↔NIXL；Store↔远端后端；Master↔NameNode/Alluxio Master | 收益受集群拓扑和集成状态影响        |
| Tair / UCM / Penguin | 平台/运营层                | 厂商栈              | 把管理层 + 底座打包成产品                                          | 商业/生态绑定               |

读这张表的关键是”角色层”：KVBM 是管理层（和框架内的 HiCache、外置的 LMCache 同位），NIXL/Mooncake 是底座，平台产品则把前两者打包再加运营。

两张表之间是供需关系：**表 1 里一个框架外包得越多，就要从表 2 取越多件**——vLLM 外包最多，通常配 LMCache（管理）+ Mooncake/NIXL（底座）；SGLang 自带 HiCache，只需从表 2 取底座；TRT-LLM 本地自足，仅跨节点时接 NIXL/Mooncake。

## 五、分层实现：从单实例 L1–L4 到多实例与集群

> **本节论点**：沿存储层级自顶向下走——单实例先在 L1–L4 之间 offload/prefetch，多实例 PD 分离把 KV 变成跨节点 payload，集群级再用物理池或逻辑池让任意实例可复用。这一章会反复看到：分层是经济上的硬约束，不做就得在显存或重算上加倍付钱。AF 分离、弹性与成本优化等支线只点名，展开见附录 9.8、9.9。 **本节核心参考**：
> 
> - ★★★ **NVIDIA Dynamo KVBM 设计文档**[\[13\]](https://docs.nvidia.com/dynamo/design-docs/component-design/kvbm-design) — 单实例四级 offload/onboard 流水线（图 11）
> - ★★★ **Mooncake / PyTorch 博客**[\[14\]](https://pytorch.org/blog/mooncake-joins-pytorch-ecosystem/) — 多实例 KVCache-centric PD 架构（图 12）
> - ★★ vLLM x Mooncake 官方博客[\[1\]](https://vllm.ai/blog/2026-05-06-mooncake-store) — round-robin 路由下 >95% 命中率的集群池扩展数据
> - ★ Beluga 论文[\[18\]](https://arxiv.org/abs/2511.20172) / 前期调研文章《Tutti：SSD 做 KV Cache》[\[17\]](https://miraclefarms.github.io/notes/2026/05/23/tutti-ssd-kv-cache/) — 集群 KV pool 三种形态的旁证

### 5.1 单实例分层存储：L1–L4 四级 KVCache

**KV 不只待在 GPU 显存里，而是在一个”越往下越慢但越大、共享范围越广”的存储层级上迁移。**业界正在形成一套相对稳定的四级语言——通用记 L1–L4，NVIDIA KVBM 记作 G1–G4，一一对应[\[13\]](https://docs.nvidia.com/dynamo/design-docs/component-design/kvbm-design)[\[21\]](https://docs.nvidia.com/dynamo/components/kvbm)。先对齐这套词汇：

| 层级       | KVBM | 介质                            | 速度 ↔ 容量           | 共享范围       | 典型存放                              |
| -------- | ---- | ----------------------------- | ----------------- | ---------- | --------------------------------- |
| **L1**   | G1   | GPU HBM                       | 最快、最小、带宽最高        | 单 GPU      | hot KV、当前 decode working set      |
| **L2**   | G2   | CPU pinned DRAM               | 次之，10–100× HBM 容量 | 单节点        | warm prefix、可快速 load-back 的 block |
| *(L2.5)* | —    | CXL memory pool               | *内存语义，介于 L2–L3*   | *rack 内共享* | *warm tier（详见第六章）*                |
| **L3**   | G3   | local SSD / NVMe              | \~ms，TBs 规模       | 单节点        | cold 本地 KV                        |
| *(L3.5)* | —    | GDS / DPU 加速存储                | *存储语义，加速到近内存吞吐*   | —          | *加速 cold / 远端（详见第六章）*             |
| **L4**   | G4   | remote / distributed / shared | \~ms（RDMA），容量最大   | 集群共享       | Mooncake Store、3FS、对象存储           |

旧的”三层”说法把本地盘和远端共享都塞进 L3，会混淆 G3 与 G4，本文统一用四级口径。两个介于完整层之间的”半层”——L2.5（CXL，内存语义）与 L3.5（GDS/DPU 加速存储，存储语义）——填的是同一段容量空隙但访问语义不同，硬件形态在第六章展开。

图 11 来自 NVIDIA Dynamo KVBM 设计文档，展示了 KV block 在 device、host、disk、remote storage 四级之间的 offload/onboard 流水线[\[13\]](https://docs.nvidia.com/dynamo/design-docs/component-design/kvbm-design)。

![[KVCache：从推理工作负载到系统设计范式-12.png]]

*图 11：KVBM 的 TransferManager 管理 D2H、H2D、H2Disk、Disk2D 等路径。每级存储有不同延迟/容量特征：GPU HBM ~ns 级最快最小；CPU pinned DRAM ~μs 级，10-100x GPU 容量；Local NVMe ~ms 级，TBs 规模；Remote Storage ~ms 级（RDMA），集群共享（延迟为量级示意，精确值与带宽对比见 §6.1）。来源：NVIDIA Dynamo KVBM 设计文档[\[13\]](https://docs.nvidia.com/dynamo/design-docs/component-design/kvbm-design)。*

**这套分层是经济账逼出来的强制项，谈不上可选优化。**用归一化口径感受一下：一张 H200（141GB HBM）跑 GQA 稠密模型，扣掉权重后约 60GB 可用于 KV；一条 100K token 的长 agent 会话按 GQA 约需 32GB KV（80 层口径；与图 5 中具体型号因层数不同而有差异），也就是单卡全部 KV 预算只够 1–2 条长会话、或几十条短 chat。当一个 agent 在等待工具执行的几十秒里把这 32GB 继续 pin 在 HBM，就等于占着二十多个活跃槽位养一个空闲会话——offload 到 L2/L3 是唯一经济的选择。结构性压缩在这里直接放大并发：同样按 80 层归一化，MLA（576 元素/token/层）把每条会话压到约 9GB，实测型号则随层数浮动——图 5 里 61 层的 Kimi K2.6 是 7.0GB、78 层 DSA 的 GLM 5.1 是 10.0GB。**同样 60GB 预算能并发约 6 条长会话，是 GQA 的三倍**——这也解释了第二章那条”调度器不能只按 token 数估算资源”。

SGLang HiCache 把这套层级讲得最清楚：GPU memory、host memory、local disk/remote distributed storage 统一成 hierarchy，**热数据留 GPU，冷数据 offload 到低层，并用异步 prefetch 在请求到来前拉回**[\[7\]](https://www.alibabacloud.com/blog/alibaba-cloud-tair-partners-with-sglang-to-build-hicache-constructing-a-new-cache-paradigm-for-agentic-inference_602767)；如何预测性地跨层调度这类 prefetch/offload 本身也已成为独立研究课题[\[24\]](https://arxiv.org/abs/2604.26968)。HiCache 还特别强调 Page-first host layout 作为桥：remote storage 按 page 连续读写，GPU 按 layer 访问，通过 host 端布局转换避免反复 `.contiguous()` 拷贝。

**（v1.14 更新）** 这套层级此前的讨论几乎全在下行——什么时候把 KV 踢下去、踢到哪一层。7 月 Mooncake Store 的两个补丁提醒我们上行同样要管：一个给”只剩本地盘副本的热 key”加了 L2→L1 提升的后台重试，让一次提升失败（host 内存暂时不够、并发冲突）不至于把热数据永久钉在 L3[\[89\]](https://github.com/kvcache-ai/Mooncake/pull/2690)；另一个给 host 段的 pin 操作加上配额，防止 pin 无限吃掉 L2 容量[\[90\]](https://github.com/kvcache-ai/Mooncake/pull/2945)。两件事拼起来是同一件：**当分层缓存开始服务多个租户和多个引擎，”哪些 KV 有权长期占住上层”就得由配额和准入来判，不能再交给 LRU 的自然淘汰。**这和上一节 LMCache 长出 pin/unpin API 是同一个方向——分层存储正在从缓存变成一个带配额的状态服务。同期 SGLang 也在收紧 L2 的容量账：HiCache 的 L3 预取此前按”潜在命中长度”先在 host 侧预留，改成先查 L3、再按真实连续命中页延迟分配，host 内存不足时只保留 page 对齐且仍高于预取阈值的那段前缀，避免远端 miss 和短命中把 L2 预留撑爆（这条链路的源码级拆解见前期调研文章《Mooncake 与 SGLang 的结合点全景》[\[91\]](https://miraclefarms.github.io/notes/2026/07/17/sglang-mooncake-integration-panorama/)）。**下层命中的不确定性，最终会变成上层的容量管理问题——预留时机本身就是一个调度变量。**

### 5.2 多实例：PD 分离让 KVCache 变成网络 payload

Prefill-Decode 分离把计算密集的 prefill 和延迟敏感的 decode 拆到不同 GPU 或节点上。好处是资源池可以按相位优化；代价是 prefill 生成的 KV 必须交给 decode，**KVCache 从本地显存状态变成跨节点 payload**。

图 12 来自 Mooncake 论文，展示了以 KVCache 为中心的 PD 分离架构[\[14\]](https://pytorch.org/blog/mooncake-joins-pytorch-ecosystem/)。

![[KVCache：从推理工作负载到系统设计范式-13.png]]

*图 12：KVCache-centric Conductor 把 Prefill Pool 和 Decoding Pool 连接起来。Prefill 侧目标是最大化 cache 复用（s.t. TTFT SLO），Decoding 侧目标是最大化吞吐（s.t. TBT SLO）。中间的 KVCache Pool 通过 Inter-node KVCache Transfer 实现跨实例 KV 共享。Cache-aware Prefill Scheduler 和 KVCache Balance Scheduler 联合决定请求路由。来源：Mooncake/PyTorch 博客[\[14\]](https://pytorch.org/blog/mooncake-joins-pytorch-ecosystem/)。*

NVIDIA NIXL 博客把 PD 场景列为核心用例：**disaggregated serving 需要在 prefill 和 decode worker 间高吞吐、低延迟移动 KV blocks**；同一套库还要支持 long-context KV storage、weight transfer、RL weight streaming 和 elastic EP[\[15\]](https://developer.nvidia.com/blog/enhancing-distributed-inference-performance-with-the-nvidia-inference-transfer-library/)。

多轮 agent 场景还会把 PD 分离推向更细粒度。第 2 轮之后，历史 KV 可能已经在 decode 侧；新一轮只追加少量 token；全量 prefill 会重算大量历史；只对 append 做 prefill 并保留历史位置更划算。这意味着 PD 分离的调度对象已经细到”**请求内不同阶段的 KV 生命周期**“这一级，整个请求不再是合用的粒度。

**（v1.13 更新）** 这个预判在过去一个月落成了框架代码。SGLang 把 HiCache 接进了 decode 侧：多轮会话的历史 KV 若已经在 decode 节点的分层缓存里，新一轮 P→D 交接只增量传输缺失的部分，prefill 侧不再整段重发[\[71\]](https://github.com/sgl-project/sglang/pull/26227)——传输对象从”本轮全部 KV”缩小到”两侧缓存状态的差集”。vLLM 侧则给 NixlConnector 补上 prefill 主动 push 的模式（此前只有 decode 拉取一种方向），P→D 交接的发起方可以按拓扑与负载选择[\[64\]](https://github.com/vllm-project/vllm/releases/tag/v0.24.0)。合起来看，**PD 分离的数据面正在从”每轮全量交接”细化成”缓存感知的增量同步”**——这也再次印证 5.3 的判断：调度与存储是同一个问题的两面。

**（v1.14 更新）** 7 月 SGLang v0.5.15 又在 decode 侧加了一根轴：MLA 模型（DeepSeek V3、Kimi K2 一系）支持 decode context parallelism，同一条会话的 KV 在 decode 实例内部再按上下文切片分到多个 rank 上，HiCache 的分层链路随之要跟着 CP 布局走[\[92\]](https://github.com/sgl-project/sglang/releases/tag/v0.5.15)。**于是”KV 在哪”这个问题在 decode 侧多了一层：先问在哪个实例，再问在实例内的哪个 context-parallel 分片。**对上一节的目录服务来说，这是索引粒度必须跟着并行策略走的又一个例子——第三章说路由的天花板是可观测性，而可观测的对象本身正在被并行策略持续切碎。

把 Mooncake 放进技术谱系里看，能避免几个常见误解。PD 分离今天已是大规模 serving 的标准做法，但它不是唯一、也不是简单的选择。最根本的分叉是”要不要分离”：与之竞争的是 chunked prefill（代表系统 Sarathi-Serve[\[39\]](https://arxiv.org/abs/2403.02310)，TRT-LLM 中称 chunked context、vLLM 原生默认）——把 prefill 切小块与 decode 交错进同一 batch，KV 不出本地、无传输开销，在中小规模常更优。二者并非互斥：NVIDIA Dynamo 的 Conditional Disaggregation 会在运行时按 prefill 长度和队列状态逐请求决定走聚合（就地 chunked）还是分离，短请求甚至直接在 decode 引擎里 piggyback 完成 chunked prefill[\[32\]](https://docs.nvidia.com/dynamo/v-0-7-1/design-docs/disaggregated-serving)——所以现状更接近”运行时组合”而非”二选一”。分离阵营内部还有第二根分叉：以什么为中心。Mooncake 是 KVCache-centric（全局 KV pool 一等公民、cache-aware 调度），而 DistServe[\[37\]](https://arxiv.org/abs/2401.09670)、Splitwise[\[38\]](https://arxiv.org/abs/2311.18677) 这类更偏算力/放置（优化 prefill/decode 资源配比与 goodput，Splitwise 还研究异构硬件）。**Mooncake 的领先正在于它 2024 年就押注”cache 是中心”，这一判断到 agentic 时代才被充分验证**——这张架构图两年未过时，原因即在此。

几个常被误解的生产事实值得点明。其一，P:D 不是固定 1:1，但要分清两个常被混淆的维度：**实例数比例**（xPyD 里的 x:y，按负载可配——vLLM 社区已有 1P3D / 3P1D 的 xPyD 实现[\[40\]](https://github.com/vllm-project/vllm/pull/18242)，NVIDIA Dynamo 支持运行时增删 prefill/decode worker[\[32\]](https://docs.nvidia.com/dynamo/v-0-7-1/design-docs/disaggregated-serving)）和**单实例并行规模**（每个 prefill/decode 实例各横跨多少卡）。DeepSeek 官方披露的 prefill EP32（4 节点）、decode EP144（18 节点）说的是后者——单个 decode 实例的 EP 规模约为 prefill 实例的 4.5×，反映 decode 侧大规模 MoE serving 需要更大的专家并行；这是 1P1D 形态下的”实例内非对称”，不能直接读成”decode 实例数更多”[\[33\]](https://github.com/deepseek-ai/open-infra-index/blob/main/202502OpenSourceWeek/day_6_one_more_thing_deepseekV3R1_inference_system_overview.md)。其二，prefill/decode 角色在多数框架中启动时即固定、不可互换（SGLang、vLLM 部署时定死，Dynamo 的 prefill worker 只执行 prefill）；唯一近似”互换”的是上面那种 decode 引擎 piggyback 短 prefill，且是单向[\[32\]](https://docs.nvidia.com/dynamo/v-0-7-1/design-docs/disaggregated-serving)。把这几根分叉收成一句：所有取舍的支点都是”**KV 跨节点传输贵不贵**“——传输便宜（快互联、或 MLA 这类小 KV）则分离更划算，传输贵（GQA 这类大 KV）则同置/chunked 更稳。模型的 attention 结构（决定 KV 体积，见第二章）会直接影响一个团队该站哪一派。

### 5.3 集群级 KVCache Pool

集群级 KV pool 有两条路线。**路线 A 是物理共享池**——把 KV 搬到一个共享的 L4 存储层，任意实例都能读写；它有三种主流形态，各有数据支撑：

**分布式 DRAM/RDMA pool（Mooncake Store）**：目前 agentic serving 最有公开生产数据的路线。vLLM 数据显示，在 12 到 60 GB200 GPU 扩展下，round-robin routing 压力下仍保持 95% 以上命中率[\[1\]](https://vllm.ai/blog/2026-05-06-mooncake-store)。

**SSD/文件系统层级（HiCache + 3FS / KVBM G3-G4）**：容量大、成本低，但前期调研文章《Tutti：SSD 做 KV Cache》的结论明确：SSD 方案的关键瓶颈往往落在大量小 I/O 的控制面开销，而非标称带宽[\[17\]](https://miraclefarms.github.io/notes/2026/05/23/tutti-ssd-kv-cache/)。

**CXL memory pool（Beluga）**：Beluga 论文提出用 CXL switch-based memory pool 让 GPU/CPU 共享大容量 KVCache，cache-hit 场景相对 RDMA baseline 把 TTFT 降低 89.6%，QPS 提升 7.35 倍[\[18\]](https://arxiv.org/abs/2511.20172)。这条路线的核心优势在下一章展开。

**路线 B 是逻辑池**——不另设共享存储，而是把每个节点本地的 L1/L2/L3 用一个全局（近似）索引统一起来，靠 KV-aware 路由把请求送到已持有该 prefix 的节点，只在必要时才显式点对点搬运。**它的哲学是”把请求搬到数据”，而非路线 A 的”把数据搬到共享池”。**三要素都有一手实现：全局索引——NVIDIA Dynamo 的 KVIndexer 把各 worker 的 KV 事件汇成全局 RadixTree[\[36\]](https://docs.nvidia.com/dynamo/latest/user-guides/kv-cache-aware-routing)，SGLang router 维护每个 worker 的近似 radix tree，llm-d 的 KV-Cache Indexer 给出跨 vLLM pod 的近实时 block locality（且记录在 GPU 还是 CPU 层）[\[34\]](https://developers.redhat.com/articles/2025/10/07/master-kv-cache-aware-routing-llm-d-efficient-ai-inference)[\[51\]](https://llm-d.ai/blog/kvcache-wins-you-can-see)；KV-aware 调度——Dynamo 按 cache-overlap score 选 worker，SGLang 的 cache-aware 负载均衡器实测带来约 1.9 倍吞吐与 3.8 倍命中率提升[\[35\]](https://www.lmsys.org/blog/2024-12-04-sglang-v0-4/)，llm-d 的 Endpoint Picker 按 KV locality 打分；显式搬运——命中本地即路由不搬，跨节点才走 NIXL/Mooncake。

两条路线各有短板，所以生产里常组合使用：路线 B 便宜（只动元数据、命中即路由），但受”locality 与负载均衡冲突”约束、索引是近似的会误路由、且没有持久性（持有节点一旦下线 KV 即丢）；路线 A 多一层存储与搬运成本，却能让 KV 在 worker 增删和会话迁移后仍可被任意实例 load-back。**典型做法是先按 B 路由到本地命中，miss 或会话迁移时回退 A 做 load-back。**这也把第三章的 KV-aware 调度和本节的集群池缝在一起：第三章的路由信号加上一个全局索引，本身就是逻辑形态的集群池——**调度与存储在这里是同一个问题的两面。**

**当前判断与未来展望。** 落到判断：**社区已经收敛成”逻辑池路由打底、物理池做持久后备”的混合，而非二选一。**逻辑池（KV-aware routing）因为只动元数据、框架原生，已是 vLLM、SGLang、Dynamo、llm-d 的普适默认；物理共享池则在 2026 快速产品化与标准化——vLLM 已官方集成 Mooncake Store[\[1\]](https://vllm.ai/blog/2026-05-06-mooncake-store)，LMCache 与 Mooncake 进一步互通[\[41\]](https://blog.lmcache.ai/en/2026/05/26/when-open-source-meets-open-source-a-joint-effort-between-lmcache-and-mooncake/)，NVMe offload 也开始走向标准化。**（v1.13 更新）**这条产品化线过去一个月继续加速：vLLM 的 Mooncake connector 在 v0.24/v0.25 两个版本里补齐流水线并行 PD、异步查询、compact chunk-hash 零拷贝查询、SWA block 跳过，以及 GDN/MLA 模型支持与 DCP 场景的并行 KV 加载[\[64\]](https://github.com/vllm-project/vllm/releases/tag/v0.24.0)[\[65\]](https://github.com/vllm-project/vllm/releases/tag/v0.25.0)——集成深度已经从”能用”推进到”跟着模型结构走”。有一处对比不该略过：公开最亮眼的命中率（92.2% / >95%）都跑在物理池上，因为逻辑池没有持久性——持有节点一旦下线 KV 即丢，而 agentic 会话的寿命远超任何单节点。往前看（这是判断而非定论）：物理池会沉淀为标准化的分层共享底座、坐在路由层之下，**真正的胜负手是路由层与存储池之间的接口标准化**——这恰是第八章列为最紧迫的开放问题。对下一代系统设计的含义有二：其一，不必在”逻辑池还是物理池”间二选一，而应面向”路由热路径 + 标准化分层后备”的接缝来构建；其二，一旦物理池成为跨 fleet 共享存储，KV identity 的正确性（见附录 9.5）就从次要约束升为承重项——跨实例复用错一个 adapter、精度或 attention state 版本，就是 silent corruption。

**（v1.14 更新）** 上面把路线 B 默认放在集群路由层，这个归属 7 月被打破了一次。LMCache v0.5.2 让多进程模式的 coordinator 直接在 L1 层用 P2P 做全局 token 匹配[\[87\]](https://github.com/LMCache/LMCache/releases/tag/v0.5.2)——请求已经落到某台机器之后，管理层仍然可以去别的实例的 GPU/host 缓存里把命中段捞回来。**逻辑池原来是路由器的专利，现在管理层自己也做了一份：路由前用它选实例，路由后用它补缺口，同一份全局视图被两层各取所需。**这对第八章那条接口标准化的开放问题是个补充论据——需要统一的不只是”路由器怎么查目录”，还有”管理层怎么查同一份目录”，两个消费者共用一套 KV 位置视图才是终局形态。

## 六、CXL 会把 KVCache pool 推向什么硬件形态

> **（v1.14 更新）本节论点**：KVCache 走哪条通道由两件事共同决定——**流量语义决定它想要什么形态**（异步、碎片化的池访问天然吃内存语义，同步大块的 P→D 交接对消息语义容忍度高），**fabric 半径决定它能不能拿到**（rack 内有共享内存就都能走南向，跨超节点只剩北向 RDMA）。v1.11 把这两条压成了一句”P→D 留北向”，TraCT 表明那是半径约束的推论，不是语义本身的性质。CXL 在池访问侧的结构性优势来自 RDMA 对碎片化 KV 的控制面税；内存语义最不可替代的场景是稀疏 demand fetch，前置搬运原理上覆盖不了它。推理芯片竞争正从”单 token 算力”扩展到”片上和片外数据移动能力”。推理系统三层抽象、网络通信设计启发等支线展开见附录 9.10。 **本节核心参考**：
> 
> - ★★★ **Beluga 论文**[\[18\]](https://arxiv.org/abs/2511.20172) — RDMA 控制面税量化（75%，16KB 传输 10.55μs 中仅 2.68μs 为数据），cache-hit 轮次 TTFT ↓89.6%、QPS ↑7.35×，本节核心图（图 13）来源
> - ★★★ **TraCT 论文**[\[83\]](https://arxiv.org/abs/2512.18194) — 把 P→D 同步交接直接放上 CXL 共享内存、绕开 NIC 跳，平均 TTFT ↓最高 9.8×、P99 ↓最高 6.2×
> - ★★★ **SAC 论文**[\[81\]](https://arxiv.org/abs/2606.19746) — 稀疏 attention 只取 top-k，CXL 取数延迟为本地 DRAM 的 1.04–1.64×（RDMA 为 4.0–19.7×）
> - ★★★ **CloudMatrix-Infer 论文**[\[56\]](https://arxiv.org/abs/2506.12708) — UB 平面池访问 + RDMA 平面 P→D 交接的三平面生产判例，跨节点延迟增加 <1μs
> - ★★★ **ShadowServe 论文**[\[59\]](https://arxiv.org/abs/2509.16857) — 取数路径 SM-free 是系统边界条件，SmartNIC 卸载后 loaded TPOT ↓2.2×
> - ★★ DeepSeek-V3 硬件反思[\[57\]](https://arxiv.org/abs/2505.09343) / DualPath 论文[\[58\]](https://arxiv.org/abs/2602.21548) — NVLink 无法按流量类型分带宽的一手证词；1% 带宽 + VL QoS 下 KV 与 EP 集合通信共存实测
> - ★★ NVIDIA Dynamo 技术博客[\[22\]](https://developer.nvidia.com/blog/full-stack-optimizations-for-agentic-inference-with-nvidia-dynamo/) — 图 14 四级存储层级，串起片上 / 搬运 / 互联三点启发
> - ★ CXL Consortium 官方说明[\[19\]](https://computeexpresslink.org/about-cxl/) / 前期调研文章《CXL + KVCache 现状调研》[\[29\]](https://miraclefarms.github.io/notes/2026/05/13/cxl-kvcache-survey/) — CXL.mem/pooling/fabric 能力边界与商业化进展

### 6.1 CXL Memory Pool 与 KVCache

**（v1.11 更新）** 把集群里多个节点的 CPU DRAM 拼成共享 KV pool——这件事已无争议，SGLang HiCache、Mooncake Store、Dynamo KVBM 都在做。有争议的是搬运通道，而**按流量语义拆分**是比”CXL 还是 RDMA”更有解释力的框架：异步的池访问（L1↔L2 offload/prefetch）碎片小、次数多，天然吃内存语义；同步的大块 KV 交接（P→D handoff）连续、一次性，对消息语义的容忍度高得多。CXL 对异步池访问这一侧的结构性优势来自 RDMA 与碎片化 KV 形态的错配：以 Qwen-32B 的 GQA 配置为例，一个 16-token 的 KV block 展开为 128 个互不连续的 20KB 片段（64 层，每层 K/V 各一份），RDMA 处理这种形态的 sg-list 税账 Beluga 算清楚了——**一次 16KB 传输总耗时 10.55μs，真实数据搬运仅 2.68μs，约 75% 是同步与控制开销**[\[18\]](https://arxiv.org/abs/2511.20172)。粒度的杀伤力在同一篇论文里有直接对照：Mooncake 用 256-token block 时 cache-hit TTFT 是 13.0s，换成 vLLM 原生的 16-token block 粒度就涨到 76.8s，比干脆重算还慢[\[18\]](https://arxiv.org/abs/2511.20172)。CXL.mem 的 load/store 绕开了这层税：细粒度访问边际成本接近本地访存，控制面 round-trip 从 RDMA-RC 的 8.39μs 降到 CXL-RPC 的 2.11μs，差 4 倍。

CXL Type 3 设备通过 CXL.mem 把设备侧内存暴露给 host；CXL 2.0 引入 switching 和 pooling；CXL 3.x 继续增强 fabric 和 sharing 能力[\[19\]](https://computeexpresslink.org/about-cxl/)。这里 pooling 与 sharing 要分清：2.0 的 pooling 是把一块内存按分区独占地分给不同 host（同一段同时只归一个 host），真正让多个 host 同时 coherent 读同一片 KV 的 memory sharing 要到 CXL 3.x 才有——所以下文说的”rack 内多 host 共享同一片”严格依赖 3.x，而 3.x 的量产生态（支持的 CPU、switch、RAS）到 2026 年仍处早期。

图 13 来自 Beluga 论文，展示了 CXL memory pool 承接大规模 KVCache 的架构[\[18\]](https://arxiv.org/abs/2511.20172)（工程重心的前期调研详见[\[30\]](https://miraclefarms.github.io/notes/2026/05/13/beluga-cxl-kvcache-memory-pool/)）。

![[KVCache：从推理工作负载到系统设计范式-14.png]]

*图 13：Beluga 用 CXL memory pool 替代 RDMA memory pool 承接 KVCache offload。核心差异：RDMA 走消息语义（碎片传输 + 控制路径开销），CXL 走 load/store memory 语义（byte-addressable，rack 内多 host 可共享同一片内存）。Cache-hit 场景 TTFT 降低 89.6%，QPS 提升 7.35 倍。来源：Beluga 论文[\[18\]](https://arxiv.org/abs/2511.20172)。*

为什么这件事重要？当前分层存储里，CPU DRAM 往往是每台机器自己的 L2 孤岛。**CXL memory pool 有机会让 rack 内多个 host 访问同一片 byte-addressable memory，把”发消息取 KV”的路径改成”读共享地址空间里的 KV”。**

**（v1.11 新增）** 「按流量语义拆分」不只是一个框架，它已经有了生产判例。华为 CloudMatrix384（384 颗 Ascend 910 NPU，UB fabric 互连）部署 DeepSeek 的论文是目前公开的国产超节点 KV pool 实测[\[56\]](https://arxiv.org/abs/2506.12708)：集群内 CPU 侧 DRAM 聚合成分布式内存池（EMS），所有 prefill 和 decode NPU 经 UB 平面以 P2P 内存语义直接访问池内 KV 数据——**跨节点访问延迟增加不到 1μs、带宽衰减低于 3%**；prefill 到 decode 的同步 KV 交接则被刻意放在 RDMA 平面，避开 UB 上 decode 关键的 token dispatch 流量。Ascend 910 的 UB 硬件还提供两条路径并存：AIV-direct（计算核直写远端 NPU 内存）与 SDMA（大块搬运引擎）——「延迟敏感小消息 vs 吞吐导向大块」的分流是硬件层就内置的设计原则，KV 流量的平面分配只是这个原则在系统层的延续。

**（v1.14 更新）** 上一版把”P→D 同步交接留在北向”写成了框架的一部分，这个表述需要收紧。TraCT 做的正是相反的事：它在 rack 尺度上把 CXL 共享内存同时当作 KV 传输介质和 rack 级前缀缓存，GPU 通过 CXL 的 load/store 与 DMA 直接读写 KV block，把 P→D 路径上的 NIC 跳整条去掉；实现在 Dynamo 上，相对 RDMA 与 DRAM 缓存基线，平均 TTFT 最高降低 9.8 倍、P99 最高降低 6.2 倍、峰值吞吐最高提升 1.6 倍[\[83\]](https://arxiv.org/abs/2512.18194)。**同步交接之所以在 CloudMatrix384 上被放到 RDMA 平面，是因为那套拓扑里 UB 平面同时背着 decode 的 token dispatch，让路是排队决策；换到一个 P→D 双方都挂在同一片共享内存上的 rack，它就没有理由绕回 NIC。**修正后的框架该这么读：流量语义决定它想要什么形态，fabric 半径决定它能不能拿到——rack 内两类流量都够得着南向，跨超节点则只剩北向。TraCT 自己也标出了这条路的代价：CXL 内存不保证 coherent，跨节点互斥、可见性和数据管理都得用软件补（它给出的是一套两级跨节点同步机制），这些正是本节末尾那份生产成熟度清单里尚未勾掉的项。

**（v1.11 新增）** 南向共享会和 EP/TP 集合通信打架吗？核心风险不在平均带宽——**KV 大块突发恰好砸进某次 decode dispatch/combine，才会推高尾延迟（head-of-line blocking）；这是排队问题，和长期平均占比几乎无关**。DualPath 的 InfiniBand 实测用 Virtual Lane 仲裁把约 99% 带宽预留给高优先级集合通信、KV 流量只占约 1% 配额，KV 在集合通信的 sub-ms 间歇里 burst 完成，TPOT 与无 KV 流量的基线基本持平（论文图中数据）[\[58\]](https://arxiv.org/abs/2602.21548)。真正的结构性风险在机制缺失：DeepSeek 团队硬件反思论文直接点名——**当前硬件无法在 NVLink 和 PCIe 上按流量类型动态分配带宽**，KV cache 从 CPU 内存进 GPU 时可打满 PCIe、与 EP 通信争用后造成性能退化和延迟尖刺；论文呼吁未来硬件把 PCIe traffic class 暴露给用户层[\[57\]](https://arxiv.org/abs/2505.09343)。这里有一个反直觉的结论：隔离机制的成熟度上，北向领先南向——InfiniBand 有 VL 仲裁，RoCE 有 DSCP 到硬件 traffic class 的映射，NVLink 类 fabric 为单应用集合通信设计，从未考虑多类流量共存。如果 UB 类互连真正暴露了可用的流量分级（MatrixLink 有此宣称但缺公开实测），在「KV 池共存」场景上就握有 NVLink 生态没有的结构性筹码。

**（v1.13 更新）** 软件栈没有等硬件把机制补齐。Mooncake TENT 在 6 月底到 7 月初连续合入两条 RFC 的实现，把 KV 传输的 QoS 拆成三条正交轴：顺序（优先级队列，此前已合入）、时限（`deadline_ns` 进准入队列，用 MLU = remaining\_bytes / (remaining\_time × available\_bw) 做可行性判据，注定超时的传输直接丢弃并向上层发 `LOCAL_DECODE_SUGGESTED` 信号——与其浪费带宽搬一个必然超时的 KV，不如让 decode 节点本地重算）[\[76\]](https://github.com/kvcache-ai/mooncake/issues/2519)、通道（TrafficClass 枚举把 KV\_PUT / KV\_GET / EP\_DISPATCH / CTRL 等业务语义映射到 IB SL/TC 与独立 QP 池）[\[77\]](https://github.com/kvcache-ai/mooncake/issues/2568)。SGLang 侧的社区 RFC 也在把请求级 priority 沿 P→D 数据路径打通到传输队列，点名 P→D 交接与 HiCache 后台写共享同一张 NIC 却没有隔离机制[\[78\]](https://github.com/sgl-project/sglang/issues/28631)。DeepSeek 硬件反思呼吁的”把流量分级暴露给用户层”，传输层先用软件把语义接了过来——**硬件缺失的按类分带宽机制，正在被”传输层准入 + 链路层通道映射”自上而下补齐**。三轴机制的逐条拆解与边界（全部 opt-in、端到端闭环未完成）见前期调研文章《Mooncake TENT 的服务质量三轴》[\[79\]](https://miraclefarms.github.io/notes/2026/07/08/mooncake-tent-qos-deadline-traffic-class-reading/)。

但 CXL 的边界同样清楚。GPU 不能自动把 CXL memory 当 HBM 读；**更关键的是 decode 是带宽密集型访问，CXL 的可用带宽远低于 HBM，不能放 decode 高频访问的 hot KV。**先厘清延迟量级：HBM 访问延迟约 100-300ns，本地 DDR5 约 80-100ns，CXL Type 3 memory（经过 PCIe/CXL link）约 150-300ns，跨 CXL switch 的 pooled memory 可能达到 200-400ns——**可见 HBM 相对 DDR/CXL 的核心优势并不在访问延迟（三者量级相近），而在带宽。**decode 阶段每生成 1 token 都要遍历所有层、读一遍完整 KV history：100K context 的 GQA 模型（约 80 层）每 token 需读 ~32GB KV，在 HBM 上耗时约 9.5ms（H100 3.35TB/s），换成 CXL memory（假设 ~50GB/s per link）则需 ~640ms，差近两个数量级。**因此 CXL 的合理定位是 warm tier：承接 prefix cache、idle session KV、PD transfer 中间态和 rack 内共享 KV server，把 active decode working set 留给 HBM。**Penguin Solutions 2026 年 3 月发布的 MemoryAI KV cache server 与 NVIDIA Dynamo KV cache offloading 兼容[\[20\]](https://ir.penguinsolutions.com/news/news-details/2026/Penguin-Solutions-Introduces-Industrys-First-Production-Ready-CXL-Based-KV-Cache-Server/default.aspx)（官方新闻稿标称 3TB DDR5 加最高 8TB CXL 内存、合计 11TB；原始 IR 页面有反爬，容量数字经 Business Wire 通稿转载核对），说明 CXL KV server 不再只是论文图，但能否成为通用 AI server 标配取决于 BIOS/OS/NUMA/RAS/telemetry/tenant isolation/GPU direct path 的成熟度。

### 6.2 对硬件设计的启发

图 14 来自 NVIDIA Dynamo 技术博客，展示了推理系统中 KVCache 的四级存储层级及其特征[\[22\]](https://developer.nvidia.com/blog/full-stack-optimizations-for-agentic-inference-with-nvidia-dynamo/)。

![[KVCache：从推理工作负载到系统设计范式-15.png]]

*图 14：GPU HBM（~ns，最快最小）→ CPU pinned DRAM（~μs，10-100x GPU 容量）→ Local NVMe（~ms，TBs 规模）→ Remote Storage/NIXL（~ms RDMA，集群共享）。每一级的延迟、容量和共享属性要求芯片和系统联合设计。来源：NVIDIA Dynamo 技术博客[\[22\]](https://developer.nvidia.com/blog/full-stack-optimizations-for-agentic-inference-with-nvidia-dynamo/)。*

这张层级图对芯片和系统架构师的启发可以分三点收束。

**片上层面**，attention decode 不能再用一句”都是 memory-bound”打平。先看传统 GQA：以 64 query heads、8 KV heads、head\_dim 128 为例，decode 1 token 对 100K 历史做 attention，单层需要读取约 100K × 8 × 128 × 2（K+V）× 2B ≈ 400MB KV；QKᵀ 与 ·V 两个矩阵乘按 query heads 计，合计约 2 × 100K × 64 × 128 ≈ 1.64B MAC（≈3.3GFLOP）。**arithmetic intensity 约 8 FLOP/byte，仍远低于 H100 BF16 roofline 拐点（约 990 TFLOPS ÷ 3.35 TB/s ≈ 295 FLOP/byte），所以 GQA decode attention 明确卡在 HBM 带宽上。**加大 batch 也很难改变这个结论：每条序列的 KV 是私有的，batch 让计算和读取同比例增长，和权重矩阵乘靠 batch 摊薄权重读取的行为不同。

**（v1.8 更新）** MLA 的判断要更谨慎。Kimi K2.6 / DeepSeek-V3.2 这类 MLA 配置缓存的是 `kv_lora_rank=512` 加 `qk_rope_head_dim=64` 的压缩状态，而非完整 K/V；读取量降到约 576 × 2B = 1,152 bytes/token/layer。可是优化后的 MLA decode 通常会把 QK 和 AV 尽量放在 latent 空间里做，计算量随 query heads 放大：以 DeepSeek-V3.2 config 的 128 attention heads 为例[\[48\]](https://huggingface.co/deepseek-ai/DeepSeek-V3.2/resolve/main/config.json)，每个历史 token 每层粗略有 `(512 + 64)` 的 QK 路径和 `512` 的 AV 路径，约 128 × 1,088 MAC，折算 **arithmetic intensity 约 240 FLOP/byte**。这个数已经接近 H100 BF16 roofline 拐点，不能再说”仍远低于拐点”；LW 对这一点的怀疑是成立的。更准确的表述是：**GQA decode attention 是显著 HBM-bound；DeepSeek 这类 128-head MLA 会把瓶颈推到 HBM bandwidth 与 tensor-core compute 的交界处，实际是否打满 H100 取决于 FlashMLA/FlashInfer 等 kernel 的 layout、融合和调度效率。**Kimi K2.6 因为 attention heads 是 64，同样估算约 120 FLOP/byte，仍低于 DeepSeek 口径；这也说明”MLA”本身不是一个固定 roofline 点，必须读具体 config。

FlashAttention 解决了 HBM 往返次数；未来 hybrid attention 和 sparse attention 还会要求硬件更好地支持 gather/scatter、top-k metadata、compressed KV decode。

**数据搬运层面**，专用 KVCache DMA Engine 的价值会越来越高，但需要区分两类不同的硬件需求。**第一类是片上 layout transform**：KV block 在 HBM 内部的重排——例如 GPU 的 layer-first 布局与存储/传输要的 page-first 页连续布局之间的转换（HiCache 的 page-first host layout 就是为了避免反复做这类转换），或者 SWA/Full Attention 双池之间的 block 迁移。这类操作的瓶颈是片上 HBM 带宽和 copy engine 的并发能力，理想状态是与 compute kernel overlap 执行。**第二类是跨接口 offload/onboard**：KV block 通过 PCIe（CPU DRAM↔GPU）、CXL（CXL memory pool↔host）、NVLink（GPU↔GPU）或 RDMA（跨节点）搬运。这类操作的瓶颈是 I/O 接口带宽和 DMA queue 管理，需要 zero-copy pinned buffer、异步 completion notification 和可编程的 scatter/gather descriptor。KVBM、HiCache、Mooncake、NIXL 都暴露出同一个需求：**搬运已经带上 metadata、layout、lifecycle 和调度语义，通用 memcpy 不够用。**

**（v1.11 更新）** 取数路径的 SM 干扰也是硬边界条件，不是靠调度优化可缓解的问题。ShadowServe 实测了 KV 解压/反量化 kernel 与 decode 计算在 GPU 上并发的结果：arithmetic decoding 场景下，**无论使用 CUDA streams、MPS 还是 Green Context，都无法把双方 slowdown 同时压到 30% 以下**；dequantization 最好的机制也仍让双方各慢超过 25%；把整条取数流水线（解压 + 传输）卸载到 SmartNIC 后，loaded TPOT 最高降低 2.2 倍[\[59\]](https://arxiv.org/abs/2509.16857)。这把「取数路径必须是 SM-free 的纯 DMA」从工程建议升为系统边界条件——无论 KV 从哪一层取回，取数路径占用 GPU SM 就等于把网络层的干扰转移到计算层，本质上换了个地方。这也是上文两类 DMA 需求要分开设计的原因：片上 layout transform 可以与 compute overlap，跨接口 offload/onboard 必须完全 DMA-offload。

**互联层面**，PD 分离、AF 分离、Elastic EP 和多 agent 并发会把 KV、activation、weight update、expert dispatch 混在同一张网络上。**芯片需要暴露可观测的 copy engine、DMA queue、RDMA completion、PCIe/NVLink/CXL path counter，让系统能区分 compute bound、HBM bound、NIC bound 还是 storage bound。**

**（v1.11 新增）** 把上述所有约束推到极致，会得到一个架构级别的终局方向。当前预取方案里，「搬运完成」是调度依赖——调度器维护预取进度状态机（policy 评估、batching、precondition await、取消边界，KVBM offload pipeline 为此写了完整一层）；query-aware demand fetch 原理上无法前置（见 §3.3）。内存语义池支持另一种模型：**decode 直接开算，KV 按需从池里 fault in，HBM 退化为远端池的 cache**。这个模型下搬运调度从系统里消失，early binding 的投机浪费自然归零，与稀疏 demand fetch 也天然契合——按需 fault in 就是最细粒度的取数。HyperParallel 的 HyperOffload 在 UB 类超节点（200ns 单跳延迟、~15× PCIe 带宽量级）上验证了这条路的性能可行性：多级 cache pipeline 异步预取 + 全局编译器编排，推理序列长度从 71K 扩展到 123K（+70%）——收益来自消除内存瓶颈驱动的冗余并行切分，不是算子加速[\[61\]](https://arxiv.org/abs/2603.03731)。这个实验对象是单实例推理，高并发 serving 下 fault-in 尾延迟行为尚未有公开验证。但它证明的方向价值是清楚的：**南向内存语义的长期价值是改变架构，让「搬运调度」这一类问题不再需要被解决**——这是北向 RDMA 无论怎么优化都给不出的。

**（v1.14 新增）** 上面这段写完时还只有 HyperOffload 一个单实例佐证，SAC 把同一条论证补上了 serving 尺度的实例，而且落在最该验证的那类负载上——稀疏注意力的 query-aware demand fetch，正是 §3.3 说的”原理上无法前置”的那一类。SAC 的做法是让 decode 每层只按 top-k 索引去 CXL 池里取那几条 KV，不再先把整段前缀搬回本地：DeepSeek-V3.2（AWQ4、8×H20 配 2TB CXL 池）上 decode 吞吐是 RDMA 池的 2.1 倍、TTFT 低 9.7 倍、TBT 低 1.8 倍[\[81\]](https://arxiv.org/abs/2606.19746)，并且随上下文变长优势放大——32K/64K/128K 分别到 2.0×/2.5×/3.1×，因为 RDMA 侧全量取数会先撞上带宽天花板[\[82\]](https://miraclefarms.github.io/notes/2026/06/21/sac-cxl-sparse-attention-kv-cache/)。**当模型每层只读 21% 的历史，”先整段搬回来”这件事本身才是浪费；内存语义把取数做到 cache-line 粒度之后，那 79% 就再也不必被搬。**两条边界要一起读：SAC 的 RDMA 基线是 8×ConnectX-7 100Gbps 的 loopback 理想化模拟（KV 实存本地 DRAM，没有真实跨节点跳数），真实 RDMA 只会更差，所以这组倍数是保守估计；反过来，收益完全依附于稀疏度，注意力越接近稠密，优势摊得越薄[\[82\]](https://miraclefarms.github.io/notes/2026/06/21/sac-cxl-sparse-attention-kv-cache/)。合起来它把第七章和本章缝在了一起：**attention 算法越稀疏，KV 池的底座就越该是内存语义的**——模型侧的演进正在给硬件侧的选型投票。

**AI 芯片的推理竞争，正在从单 token 算力扩展到片上和片外数据移动能力。**

## 七、Attention 算法演进对 KVCache 的影响

> **本节论点**：把视角拉回模型侧——新一代 attention 算法（Sliding Window、Hybrid、CSA/HCA、稀疏、线性）正把 KVCache 从同质线性数组，改写成多状态、多有效期、多迁移成本的状态集合；prefix cache 的匹配对象也从“token 序列”升级为“token 序列 + attention state 可用性”。量化压缩、线性注意力等支线只点名，展开见附录 9.6、9.7。 **本节核心参考**：
> 
> - ★★★ **Tair + SGLang HiCache 技术博客**[\[7\]](https://www.alibabacloud.com/blog/alibaba-cloud-tair-partners-with-sglang-to-build-hicache-constructing-a-new-cache-paradigm-for-agentic-inference_602767) — 本节核心图（图 15 Shadow Radix）来源，给出多态 KV 的逻辑坐标方案
> - ★★★ **MiMo V2.5 官方博客**[\[5\]](https://mimo.xiaomi.com/zh/blog/mimo-v2-5-inference) — Hybrid SWA 把 KV 压到约 1/7、窗口 128 的一手生产数据
> - ★★★ **DeepSeek V4 技术解读与 SGLang Day-0 实现**[\[44\]](https://huggingface.co/blog/deepseekv4)[\[45\]](https://www.lmsys.org/blog/2026-04-25-deepseek-v4/) — CSA/HCA、sliding window 融合与 ShadowRadix / HiSparse 工程语义
> - ★★ MiniMax M3 官方博客[\[8\]](https://www.minimax.io/blog/minimax-m3) — MSA 稀疏 attention 面向 1M context 的另一条路线
> - ★ 前期调研文章《KV Cache 不再只有一种增长公式》[\[6\]](https://miraclefarms.github.io/notes/2026/06/01/kvcache-growth-attention-variants-essay/) — 图 5 横向对比图（已前移至 §2.3）来源

前面六章把 KVCache 从业务模式一路追到硬件形态，背后都假设“KV 是一段随上下文线性增长的连续数组”。但模型侧的 attention 算法正在动摇这个假设——SWA 让窗口有了有效期，CSA/HCA 让同一段 token 有了多种压缩形态。这一章回到模型侧，看这些演进如何反过来改写 KV 的存储与缓存语义。

### 7.1 Sliding Window 与 Hybrid Attention：窗口有效期进入缓存语义

Sliding Window Attention（SWA）把每层可见历史限制在固定窗口内，理论上把该层 KV 从 O(N) 压到 O(W)。当 SWA 与 Full Attention 混合使用时，同一个 token 在不同层会有不同的 KV 有效性。这给 prefix cache 带来了根本性挑战：传统前缀树的隐含假设是 token 序列相同则 KV 有效；**Hybrid Attention 下，这个假设会产生幽灵命中。**

图 15 展示了 Tair + SGLang 联合设计的 Shadow Radix 布局，用来解决这一问题[\[7\]](https://www.alibabacloud.com/blog/alibaba-cloud-tair-partners-with-sglang-to-build-hicache-constructing-a-new-cache-paradigm-for-agentic-inference_602767)。

![[KVCache：从推理工作负载到系统设计范式-16.png]]

*图 15：Shadow Radix 用 Source 逻辑坐标承接 token 序列，再把 SWA/C4/C128 投影到不同 shadow pool。SWA 可以 tombstone，压缩 KV 仍保留。来源：Tair 技术博客[\[7\]](https://www.alibabacloud.com/blog/alibaba-cloud-tair-partners-with-sglang-to-build-hicache-constructing-a-new-cache-paradigm-for-agentic-inference_602767)。*

MiMo V2.5 是公开材料里非常完整的生产样本：Hybrid SWA 把 KV 存储压到 Full Attention 的约 1/7，并围绕双池、SWA-aware prefix tree、GCache 和 KV-aware routing 做了全链路工程[\[5\]](https://mimo.xiaomi.com/zh/blog/mimo-v2-5-inference)。MiniMax M3 的 MSA（MiniMax Sparse Attention）也面向 1M context 和 agentic coding 场景做了类似方向的稀疏 attention 架构[\[8\]](https://www.minimax.io/blog/minimax-m3)。MiMo 的窗口安全长度、Tair/SGLang 的 Shadow Radix、DeepSeek V4 的 tombstoning 都在重写同一份合同：**prefix cache 的匹配对象从 token 序列，升级为 token 序列加 attention state 的可用性。**

### 7.2 DeepSeek V4：CSA/HCA 把”压缩表示”变成独立范式

**（v1.8 新增）** DeepSeek V4 值得在这里单独拎出来，因为它不是普通的 MLA 小 KV，也不是只在 full KV 上做 top-k 的稀疏注意力。V4 的核心变化是把每层 attention 拆成两类压缩表示：CSA（Compressed Sparse Attention）先把每 4 个 token 压成 1 个 C4 KV 条目，再在压缩域上用 lightning indexer 做 top-k；HCA（Heavily Compressed Attention）把每 128 个 token 压成 1 个 C128 KV 条目，然后在压缩域上做 dense MQA[\[44\]](https://huggingface.co/blog/deepseekv4)。这意味着”压缩”不再只是 KV 量化/低秩化的附属优化，而是一种新的 attention state 形态：模型先改变历史表示的数量，再决定是稀疏看还是全局看。

更关键的是 **sliding + CSA/HCA 的融合**。DeepSeek V4 的每层除了在压缩域里工作，还同时保留一个覆盖最近 128 个原始 token 的 SWA 分支；CSA/HCA 负责长程压缩视野，SWA 负责局部精确性。这个组合把系统语义拆成三类池：SWA 只对最近窗口有效，C4/C128 压缩 KV 可以跨窗口长期复用，compress state 还要处理正在形成但尚未封口的压缩块。SGLang Day-0 实现因此把 ShadowRadix 设计成 Source 逻辑坐标到 SWA / C4 / C128 多个 shadow pool 的投影；当 SWA 过期 tombstone 时，压缩池仍然可以保留并服务后续 prefix hit[\[45\]](https://www.lmsys.org/blog/2026-04-25-deepseek-v4/)。这正是图 15 想表达的系统难点：同一个 token 序列命中，不能说明所有物理状态都可用。

CSA 也改变了 offload 判断。对 C4 层来说，indexer 每步只触达 top-k 压缩块，大多数 C4 KV 在当前 decode step 是 inactive 的，适合放在 CPU/host tier 里按需调回；C128 是 dense over compressed stream，SWA 又只有最近窗口，二者的 offload 收益和风险完全不同。HiSparse 只扩展 C4 池而不是一视同仁地 offload 所有 KV，说明下一代 KV runtime 的管理对象已经细到 **SWA / C4 / C128 / compress state 各自的生命周期、命中语义和迁移成本**，”一段 KV block 放哪层”这个问法本身太粗。这也是 CSA 作为范式的系统含义：它把 KVCache 从线性数组，进一步推成了多状态图。**（v1.14 补充）** 这个多状态图对底座是有偏好的：C4 池按 top-k 被随机触达，正是 §6.2 里 SAC 证明”CXL cache-line 取数远优于 RDMA 全量搬运”的那种访问形态[\[81\]](https://arxiv.org/abs/2606.19746)——**模型侧每往稀疏走一步，KV 池的南向内存语义就多一分必要性。**

线性注意力和 recurrent state 对 KVCache 的影响（block 不再是唯一单位）、KV 量化与压缩的分支路线（FP8/INT8/NVFP4 及其 kernel/调度约束），详见附录 9.6 和 9.7。这两条路线的共同系统含义是：**Attention 算法正在把 KVCache 从同质线性数组，变成由多类状态、多种有效期、多种精度和多种迁移成本组成的状态集合。**

**（v1.13 更新）** 这个”状态集合”判断在过去一个月从论文语言变成了框架默认行为，主战场是线性注意力的 prefix caching。vLLM 让 Mamba/GDN 混合模型的前缀缓存进了 Model Runner V2 主路径[\[67\]](https://github.com/vllm-project/vllm/pull/42406)，并引入 Marconi 式的准入策略——recurrent state 快照不可切片、只能整段命中，值不值得缓存要按预期复用价值先过一道准入[\[68\]](https://github.com/vllm-project/vllm/pull/37898)；SGLang 把混合模型的 HiCache 分层默认打开（UnifiedTree）[\[69\]](https://github.com/sgl-project/sglang/pull/27759)，还给线性注意力的状态快照上了 int8 量化池，把不可切片的定长快照压小以扩大可缓存条数[\[70\]](https://github.com/sgl-project/sglang/pull/28185)。**recurrent state 第一次在框架里拿到了 state group 待遇——有专属的匹配粒度、准入规则和量化池**，附录 9.6 的预判成为产品现实，落地明细见该节。**（v1.14 补充）** 7 月这条线延伸到了框架外：LMCache v0.5.2 让 CacheBlend 跑在 vLLM 的混合 KV cache manager 之下，并为 SWA 模型补上 dual-RoPE 支持[\[87\]](https://github.com/LMCache/LMCache/releases/tag/v0.5.2)——窗口有效期这件事，外置管理层现在也得自己理解，不能再假装所有 token 的 KV 一样长命。

## 八、回到开篇：五个变量同时变了，开放问题该追哪些

> **本节论点**：开篇那条判断——”让已经算过、仍然有价值的状态，以低于重算的成本被再次使用”——在工作负载、模型结构、框架边界、部署拓扑、硬件形态五个变量上同时被改写；本节回到这条主线收口，并按紧迫度排出值得追的开放问题。本节是综述收尾，不引入新核心图，证据沿用前七章各节核心图。 **本节核心参考**：
> 
> - ★★★ vLLM x Mooncake[\[1\]](https://vllm.ai/blog/2026-05-06-mooncake-store) / NVIDIA Dynamo[\[22\]](https://developer.nvidia.com/blog/full-stack-optimizations-for-agentic-inference-with-nvidia-dynamo/) — 工作负载与部署拓扑两条结论的数据锚点
> - ★★ MiMo V2.5[\[5\]](https://mimo.xiaomi.com/zh/blog/mimo-v2-5-inference) / Beluga[\[18\]](https://arxiv.org/abs/2511.20172) — 调度与硬件两条结论的数据锚点
> - ★ 前期调研文章《KV Cache 不再只有一种增长公式》[\[6\]](https://miraclefarms.github.io/notes/2026/06/01/kvcache-growth-attention-variants-essay/) — 模型结构结论的汇编来源

KVCache 的系统现状可以用五句话概括。

第一，**工作负载变了**。Agentic 负载是状态积累型负载，NVIDIA Dynamo 数据显示读写比 11.7:1，vLLM Mooncake 数据显示输入输出比 131:1[\[22\]](https://developer.nvidia.com/blog/full-stack-optimizations-for-agentic-inference-with-nvidia-dynamo/)[\[1\]](https://vllm.ai/blog/2026-05-06-mooncake-store)。

第二，**模型结构变了**。GQA、MLA、DSA、Hybrid SWA 等路线让 100K token 下 KV 体积差异超 10 倍[\[6\]](https://miraclefarms.github.io/notes/2026/06/01/kvcache-growth-attention-variants-essay/)。系统必须理解 attention-derived KV slope 和 state group。

第三，**框架边界变了**。vLLM、SGLang、TRT-LLM 都把 KVCache 管理推进 scheduler 主链路；KVBM、Mooncake、NIXL 把缓存管理、分层存储和数据移动拆成更清楚的系统层。

第四，**部署拓扑变了**。PD 分离让 KV 变成网络 payload，分布式 KV pool 在 round-robin 路由下仍保持 95% 以上 hit rate[\[1\]](https://vllm.ai/blog/2026-05-06-mooncake-store)。KV-aware routing 带来 Long Request P90 TTFT 30.5% 改善[\[5\]](https://mimo.xiaomi.com/zh/blog/mimo-v2-5-inference)。

第五，**硬件问题变了**。CXL KV pool 在 cache-hit 轮次把 TTFT 降低 89.6%[\[18\]](https://arxiv.org/abs/2511.20172)；Penguin MemoryAI 等商业产品已经出现[\[20\]](https://ir.penguinsolutions.com/news/news-details/2026/Penguin-Solutions-Introduces-Industrys-First-Production-Ready-CXL-Based-KV-Cache-Server/default.aspx)。**（v1.14 更新）**过去两个月这条线的证据从”warm tier 更快”扩展到两类结构性收益：稀疏 attention 的按需取数（SAC，DeepSeek-V3.2 上吞吐 2.1×、TTFT 低 9.7×）[\[81\]](https://arxiv.org/abs/2606.19746)，以及 rack 内 P→D 交接绕开 NIC（TraCT，平均 TTFT 最高低 9.8×）[\[83\]](https://arxiv.org/abs/2512.18194)——**内存语义的价值主张已经从”再加一层更大的 warm tier”，变成”让某几类搬运彻底不必发生”。**

按紧迫度排，下面四个开放问题最先需要答案。

**最紧迫：KV-aware scheduler 的输入标准化。** 这是当前已有多个团队独立解决但没有统一接口的问题——MiMo、SGLang、vLLM Mooncake、Dynamo 都实现了各自的 KV-aware routing，但 prefix locality signal、cache state query、load metric 的格式和语义各不相同。统一这层接口的投入相对可控（协议定义 + adapter），但收益高——让调度层可以跨框架复用，避免每换一个 runtime 就重写 scheduler。**（v1.13 更新）**这个方向一个月内出现三个实质动向：Mooncake Conductor 以框架中立目录服务开源[\[75\]](https://github.com/kvcache-ai/mooncake/pull/2833)、vLLM 的 KV 事件改为 self-describing 格式[\[64\]](https://github.com/vllm-project/vllm/releases/tag/v0.24.0)、TRT-LLM 把 block-hash chain 与 cache\_salt 暴露进 connector 和事件流[\[72\]](https://github.com/NVIDIA/TensorRT-LLM/pull/14806)——事件与目录接口正在各自标准化，但跨框架共识仍未出现。

**第二：跨框架 KV state identity 标准。** 随着分布式 KV pool 成为生产组件，identity 不一致会导致 silent corruption（如前述 LoRA mismatch 场景）。当前的 blocker 是各框架对 KV block 的 hash 组成、精度标记、attention state 标记没有共识。这个问题的紧迫度会随多框架混部（如 vLLM prefill + TRT-LLM decode）场景增加而快速上升。

**第三：CXL memory pool 在真实 GPU server 拓扑中能否给出可预测 P99。** Beluga 论文的数据令人鼓舞，但实验拓扑相对简单。生产环境里 BIOS NUMA policy、IOMMU 开关、CXL switch 级联、多 tenant 隔离、RAS（reliability/availability/serviceability）都会影响延迟分布。这个问题的 blocker 是硬件成熟度和生态支持，短期内更适合作为验证课题而非生产依赖。**（v1.14 更新）**问题的重心这两个月往软件挪了一格：SAC 与 TraCT 都把 CXL 从 warm tier 推到了关键路径上（前者是逐层 top-k 取数，后者是 P→D 交接），而 TraCT 明确记录了非 coherent 共享内存带来的三道软件负担——跨节点互斥、可见性、以及可移植的共享内存操作，它自己是用一套两级同步机制补的[\[83\]](https://arxiv.org/abs/2512.18194)。**于是待验证清单要加一项：当 CXL 从”缓存下一层”变成”多个 GPU 同时读写的一致性域”，正确性成本会不会吃掉延迟收益。**这一项目前只有单篇论文的答案。

**第四：AI 芯片是否应该显式提供 KV DMA / layout transform / compressed KV engine。** 这是最长期的问题，取决于 KVCache 的管理复杂度是否会成为推理芯片的持久瓶颈。当前证据显示有需求（KVBM/NIXL/Mooncake 都暴露了通用 memcpy 的不足），但具体的硬件抽象该暴露什么层级的接口（ISA 指令 vs 可编程 DMA descriptor vs 固定功能加速器）还没有共识。

## 九、附录

### 9.1 前期调研阅读路线与核心判断

| 主题                  | 前期调研文章                                                                                                                          | 核心判断                                                                                                                     | 原始来源方向                                                                  |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------- |
| Agentic workload    | [Agentic 推理的性能重构](https://miraclefarms.github.io/notes/2026/05/28/agentic-inference-kv-cache-benchmark-survey/)                 | Agentic 负载是跨轮状态积累型负载，cache 反复命中又反复失效；系统优化不能只盯 GPU，tool call 失败引发的上下文膨胀同样重要                                               | MARS / Helium / HexAGenT 等十篇 arXiv 论文                                   |
| Claude Code context | [生产级 Agent 的 Token 经济学](https://miraclefarms.github.io/notes/2026/05/08/claude-code-context-kvcache-engineering/)               | 应用层 harness 通过拆分 static/dynamic 区来最大化 cache hit；cache breakpoint 的精确匹配使得上下文组织成为产品能力                                      | Anthropic prompt caching / context management 文档                        |
| Attention 变体        | [KV Cache 不再只有一种增长公式](https://miraclefarms.github.io/notes/2026/06/01/kvcache-growth-attention-variants-essay/)                 | 100K token 下不同 attention 结构导致 KV 体积差异超 10 倍（2.4GB vs 25.4GB）；调度器不能只按 token 数估算资源                                         | Qwen/Kimi/DeepSeek/GLM/MiniMax 模型文档与技术报告                                |
| Prefix matching     | [KV Cache 前缀匹配的设计分野](https://miraclefarms.github.io/notes/2026/05/13/kvcache-prefix-matching-design/)                           | 三大框架从不同起点（block-first / prefix-first / engine-integrated）走向 KVCache 一等公民；匹配粒度、锁策略、哈希方式构成关键设计差异                           | vLLM / SGLang / TRT-LLM 源码与文档                                           |
| vLLM runtime        | [vLLM 如何管理 KV Cache](https://miraclefarms.github.io/notes/2026/03/12/vllm-kvcache-runtime-architecture/)                        | PagedAttention 把 KV 碎片管理推成 OS 风格页表；V1 架构把 scheduler→KVCacheManager→BlockPool→connector 串成主控链路                            | PagedAttention 论文与 vLLM 源码                                              |
| SGLang runtime      | [SGLang 如何管理 KVCache](https://miraclefarms.github.io/notes/2026/03/14/sglang-kvcache-runtime-architecture/)                     | RadixTree 天然服务多轮/多分支 prefix 复用；HiCache 分层从 GPU→host→disk→remote 四级统一管理                                                   | RadixAttention / HiCache 文档                                             |
| TRT-LLM runtime     | [TensorRT-LLM 的 KVCache 架构](https://miraclefarms.github.io/notes/2026/05/09/trtllm-kvcache-runtime-architecture/)               | Engine-integrated 路线的优势在 kernel/runtime 深度协同；跨框架共享需要 connector 和外部数据面                                                    | TensorRT-LLM KV cache docs                                              |
| Mooncake            | [从 KV Cache 底座到推理统一数据面](https://miraclefarms.github.io/notes/2026/06/02/mooncake-unified-data-plane-architecture/)              | Mooncake 从 KV cache 传输引擎演化为统一数据面（KV/权重/EP），已进入 PyTorch 生态；Store + Transfer Engine 是分布式 KV pool 的基础设施                     | PyTorch Mooncake blog / Mooncake docs                                   |
| NIXL                | [NVIDIA NIXL：推理超节点里的数据移动层](https://miraclefarms.github.io/notes/2026/05/26/nvidia-nixl-inference-transfer-layer/)               | NIXL 统一了 KV transfer / long-context storage / weight transfer / RL streaming / elastic EP 的数据移动接口；不替代调度和 cache value 判断  | NVIDIA NIXL 技术博客                                                        |
| KVBM                | [NVIDIA Dynamo KVBM 源码笔记](https://miraclefarms.github.io/notes/2026/06/02/nvidia-dynamo-kvbm-architecture-field-note/)          | KVBM 把 KV 生命周期（G1-G4）建模为独立内存子系统；强弱引用、可取消 offload、connector 接口是关键抽象                                                       | NVIDIA Dynamo KVBM docs/source                                          |
| Tair + SGLang       | [同一个 Token，五种物理形态](https://miraclefarms.github.io/notes/2026/06/02/tair-sglang-deepseek-v4-hicache-hisparse/)                   | DeepSeek V4 的混合 attention 使同一 token 有 SWA/C4/C128/indexer/compress 等多种物理形态；Shadow Radix 通过 Source→Shadow pool 映射解决多态缓存管理 | Alibaba/Tair 技术文章、SGLang issue                                          |
| MiMo                | [MiMo V2.5 推理系统](https://miraclefarms.github.io/notes/2026/06/02/mimo-v2-5-inference-optimization-reading/)                     | Hybrid SWA 把 KV 压到 ~1/7；生产路由结合 prefix\_match\_percentage + load 实测 L2 命中率 +25%、输入吞吐 +30%                                 | MiMo 官方博客                                                               |
| CXL                 | [CXL + KVCache 现状调研报告](https://miraclefarms.github.io/notes/2026/05/13/cxl-kvcache-survey/)                                     | CXL 定位是 warm tier（HBM 之下），core 优势是 byte-addressable + rack 内 memory pooling；边界在 GPU direct path、P99、隔离和 RAS 成熟度          | CXL Consortium / Beluga / Penguin 等公开资料                                 |
| Beluga              | [Beluga：CXL 内存池为什么会改变 KV Cache Offload 的工程重心](https://miraclefarms.github.io/notes/2026/05/13/beluga-cxl-kvcache-memory-pool/)  | CXL load/store 语义替代 RDMA 消息语义，cache-hit TTFT↓89.6% / QPS↑7.35x；实际路径会经过 root complex、IOMMU、DMA engine                     | Beluga 论文                                                               |
| KV pool 南向通道        | [KV Cache 池化的南向选择：scale-up 内存语义的收益边界](https://miraclefarms.github.io/notes/2026/06/10/kv-cache-pool-scale-up-fabric/)           | 流量语义拆分（异步池访问→内存语义 / 同步 P→D 交接→RDMA）；CloudMatrix384 三平面生产判例；demand fetch 约 20% 在关键路径；SM-free 是系统边界条件；HBM-as-cache 终局方向    | CloudMatrix-Infer / DualPath / ShadowServe / KVDrive / HyperParallel 论文 |
| 稀疏 KV × CXL         | [SAC：用 CXL 给稀疏注意力做一个只取 top-k 的分离式 KV Cache](https://miraclefarms.github.io/notes/2026/06/21/sac-cxl-sparse-attention-kv-cache/) | 稀疏注意力下远端池的瓶颈是访问语义而非带宽；cache-line 取数让逐层 top-k 不必前置，DeepSeek-V3.2 上吞吐 2.1×、TTFT 低 9.7×，但收益完全依附于稀疏度                         | SAC 论文                                                                  |
| PD 路由博弈             | [P/D 分离推理的失控点：Dynamo 路由博弈阅读](https://miraclefarms.github.io/notes/2026/07/03/dynamo-disaggregated-inference-price-of-anarchy/)  | Planner / Smart Router / KV Block Manager 组成耦合博弈系统；饱和之前缓存亲和几乎白拿，越过拐点才开始与负载均衡互相拉扯                                         | The Price of Anarchy in Disaggregated Inference 论文                      |

### 9.2 面向 2 小时分享的建议节奏

| 时间          | 内容                                                            | 目标听众          |
| ----------- | ------------------------------------------------------------- | ------------- |
| 0-15 min    | 开场与工作负载变化：Claude Code / Codex traces / prompt caching         | 全员            |
| 15-40 min   | Attention 结构与 KV 增长公式：GQA、MLA、DSA、SWA、linear、量化               | 芯片架构师、推理系统    |
| 40-70 min   | 框架管理：vLLM、SGLang、TRT-LLM、Prefix、HiCache、KVBM                  | 推理系统、软件架构     |
| 70-95 min   | 部署系统：PD/AF 分离、KV-aware routing、Mooncake/NIXL/LMCache/UCM/Tair | AI Infra、网络通信 |
| 95-115 min  | 硬件展望：CXL、HBM/DDR/NVMe、片上 DMA、RDMA/NVLink 拓扑                   | 芯片、网络、系统      |
| 115-120 min | 讨论：标准化接口、产品路线、内部验证问题                                          | 全员            |

### 9.3 关键判断清单

| 判断                                        | 证据锚点                                                                                                                                                                                                                                                                  | 需要内部验证的问题                                                 |
| ----------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------- |
| Agentic 负载是状态积累型负载                        | vLLM Codex trace 33 turns / 80K token / 131:1[\[1\]](https://vllm.ai/blog/2026-05-06-mooncake-store)；NVIDIA Dynamo R/W 比 11.7:1[\[22\]](https://developer.nvidia.com/blog/full-stack-optimizations-for-agentic-inference-with-nvidia-dynamo/)                         | 内部 coding agent trace 是否有类似 prefix 复用率                    |
| KVCache pool 比 session stickiness 更可扩展    | Mooncake Store round-robin 下 >95% hit[\[1\]](https://vllm.ai/blog/2026-05-06-mooncake-store)                                                                                                                                                                          | 内部网络拓扑能否承受跨节点 load-back                                   |
| Attention 结构决定 KV slope                   | Qwen/Kimi/DeepSeek/MiniMax KV 差异 2.4-25.4GB[\[6\]](https://miraclefarms.github.io/notes/2026/06/01/kvcache-growth-attention-variants-essay/)                                                                                                                          | 自研模型的 bytes/token/layer 如何随版本变化                           |
| Hybrid attention 要求多 state group 管理       | MiMo SWA-aware tree[\[5\]](https://mimo.xiaomi.com/zh/blog/mimo-v2-5-inference) / Tair Shadow Radix[\[7\]](https://www.alibabacloud.com/blog/alibaba-cloud-tair-partners-with-sglang-to-build-hicache-constructing-a-new-cache-paradigm-for-agentic-inference_602767) | 当前 runtime 是否能表达 SWA/压缩 KV 生命周期                           |
| 分层存储收益取决于隐藏搬运延迟                           | HiCache layer-wise load overlap[\[10\]](https://docs.sglang.io/advanced_features/hicache_design.html) / KVBM G1-G4[\[13\]](https://docs.nvidia.com/dynamo/design-docs/component-design/kvbm-design)                                                                   | H2D/D2H、SSD、RDMA copy 是否能与 compute overlap                |
| CXL 是 warm tier，不是 HBM 替代                 | Beluga cache-hit TTFT↓89.6% / QPS↑7.35x[\[18\]](https://arxiv.org/abs/2511.20172)                                                                                                                                                                                     | GPU-CXL direct path、P99 和隔离能否过生产门槛                        |
| 芯片需要 KV-aware DMA 和 telemetry             | KVBM/NIXL/Mooncake 都暴露 layout/transfer 复杂度[\[13\]](https://docs.nvidia.com/dynamo/design-docs/component-design/kvbm-design)[\[15\]](https://developer.nvidia.com/blog/enhancing-distributed-inference-performance-with-the-nvidia-inference-transfer-library/)        | 片上 RDMA IP 是否能承接 KV block transfer primitive              |
| KV 搬运通道应按流量语义拆分而非 CXL vs RDMA 二选一         | CloudMatrix384 UB 池访问 + RDMA P→D 交接，跨节点延迟 <1μs[\[56\]](https://arxiv.org/abs/2506.12708)                                                                                                                                                                              | 内部 scale-up fabric 是否支持 CPU DRAM P2P 寻址；南向 QoS 工具是否具备     |
| demand fetch 约 20% 在 decode 关键路径，前置搬运无法覆盖 | KVDrive KV 选择+取数近 50% decode 时间；滑动窗口 ~80% 命中，剩余 ~20% 为 demand miss[\[60\]](https://arxiv.org/abs/2605.18071)                                                                                                                                                          | query-aware 稀疏负载比例是否超出计算掩盖可吸收的阈值                          |
| 取数路径必须 SM-free，否则 GPU 干扰不可消除              | ShadowServe：arithmetic decoding 并发 slowdown 无法同时 <30%；SmartNIC 卸载后 TPOT 最高 ↓2.2×[\[59\]](https://arxiv.org/abs/2509.16857)                                                                                                                                            | 当前 KV offload 路径是否占用 SM；SmartNIC/DPU 是否纳入路线图              |
| 稀疏 demand fetch 的解法是取数变便宜，不是预取变聪明         | SAC：CXL 取数为本地 DRAM 的 1.04–1.64×，RDMA 为 4.0–19.7×；DeepSeek-V3.2 吞吐 2.1×、TTFT ↓9.7×[\[81\]](https://arxiv.org/abs/2606.19746)                                                                                                                                           | 自研模型的稀疏度是否高到能兑现这条收益；近稠密时会摊薄                               |
| P→D 走北向还是南向由 fabric 半径决定，与流量语义无关          | TraCT：rack 内 CXL 共享内存直接承接 P→D，平均 TTFT 最高 ↓9.8×、P99 最高 ↓6.2×[\[83\]](https://arxiv.org/abs/2512.18194)                                                                                                                                                                 | 内部 rack 是否有 GPU 可直接 load/store 的共享内存域；非 coherent 下的软件同步成本 |
| 缓存亲和与负载均衡的冲突存在饱和拐点                        | PoA：两个模型的拐点后第一个网格点均为并发 128，自适应路由在 70B 1P/5D 上把无政府代价估计量从 66.4 压到 21.5、代价约 13% 吞吐[\[84\]](https://arxiv.org/abs/2606.17081)；DualMap 双哈希在同 TTFT SLO 下有效容量 ↑2.25×[\[86\]](https://arxiv.org/abs/2602.06502)                                                               | 现有压测是否跑到过饱和拐点另一侧                                          |

### 9.4 术语对照

| 术语                | 含义                                            |
| ----------------- | --------------------------------------------- |
| TTFT              | Time To First Token，首 token 延迟                |
| TPOT              | Time Per Output Token，decode 阶段每 token 延迟     |
| Prefill           | 输入 prompt 的并行处理阶段，生成历史 KV                     |
| Decode            | 自回归逐 token 生成阶段，反复读取历史 KV                     |
| PD 分离             | Prefill 和 Decode 分离部署                         |
| KV block          | 一段 token 对应的一组 K/V 缓存块                        |
| Prefix cache      | 多请求共享相同前缀时复用已生成 KV                            |
| Offload / Onboard | KV 从 HBM 下沉到低层，或从低层加载回 HBM                    |
| G1-G4             | Dynamo KVBM 中 GPU、host、disk、remote storage 四级 |
| SWA               | Sliding Window Attention，仅看固定窗口内历史            |
| MLA/DSA/CSA/HCA   | 多种结构性 KV 压缩或稀疏 attention 机制                   |
| CXL Type 3        | CXL memory expansion / pooling 设备             |

### 9.5 分支材料：Workload 对 KVCache 的五类设计要求

从工作负载出发，可以把 KVCache 系统需求归纳成五类。每类对应一个子系统设计决策。

**第一是身份（Identity）。** 系统要知道某段 KV 对应哪段 token、哪个模型版本、哪个 LoRA / multimodal / attention 结构、哪个位置和哪些安全隔离条件。vLLM 链式哈希、SGLang RadixTree、TRT-LLM 多维 BlockKey、Dynamo KVBM 的 sequence\_hash 都在回答这个问题。没有 identity，跨实例共享就有正确性风险——举一个具体场景：实例 A 用 LoRA adapter α 做 prefill 后将 KV block 写入分布式 pool，实例 B 用 LoRA adapter β 服务同一 session 的下一轮请求时，如果 identity 只校验 token hash 而不校验 adapter 版本，就会用 α 的 KV 拼上 β 的 decode，**产出 silent corruption——不会报错，但 attention 读到的 K/V 与当前权重不匹配，输出质量下降且难以定位。**类似的风险还包括：Hybrid Attention 下 SWA 窗口过期的 KV block 被当作有效 Full Attention KV 命中（幽灵命中）、FP8 量化 KV 被 BF16 kernel 直接读取导致精度异常、不同 model checkpoint 版本的 KV 被跨版本复用等。

**第二是生命周期（Lifecycle）。** KV 不是生成后永久有用。system prompt 可能长期复用；thinking token 在 reasoning loop 结束后价值迅速下降；subagent KV 在子任务终止后可能整体废弃；SWA 旧窗口 KV 过期后可以释放；compressed KV 可能仍有长期价值。**生命周期管理比单纯 LRU 更重要。**KVBM 的强弱引用边界、WeakBlock upgrade、可取消 offload pipeline 是这个方向的底层实现（源码层细节见前期调研 KVBM 源码笔记[\[28\]](https://miraclefarms.github.io/notes/2026/06/02/nvidia-dynamo-kvbm-architecture-field-note/)）。

**第三是位置（Placement）。** KV 可能在 HBM、host pinned memory、NVMe、远端对象存储、CXL pool、Mooncake Store、KVBM G4 remote storage 中。**位置决定加载延迟、带宽、是否可跨实例共享，以及是否值得重算。**

**第四是调度（Scheduling）。** **请求路由必须知道哪个 worker 拿到这段 prefix 的成本最低。**MiMo 的 prefix\_match\_percentage - load 类 score[\[5\]](https://mimo.xiaomi.com/zh/blog/mimo-v2-5-inference)、SGLang cache-aware load balancer、vLLM Mooncake scheduler、Dynamo KV-aware routing 都是同一类趋势。

**第五是可观测性（Observability）。** **没有 per-request matched tokens、effective hit length、offload latency、transfer queue、G2/G3/G4 occupancy、RDMA/CXL path utilization，就很难定位 TTFT 和 P99 的根因。**

### 9.6 分支材料：线性注意力与 recurrent state 的系统影响

Linear Attention、Gated DeltaNet、KDA 等路线把一部分历史表示折叠成 recurrent state，代表模型包括纯 SSM/线性的 Mamba、RWKV、RetNet，以及把线性层与 full attention 交错的混合架构（Qwen3.5 的 GDN+GQA、Kimi Linear 的 KDA、MiniMax、Jamba、Nemotron-H 等，生产上以混合形态为主）。对系统来说，这类状态和传统 KV block 差异巨大。

**核心观点：recurrent state 不是 block。** 它通常是 request-level state，不能像 full attention KV 那样按 block 做 prefix 复用；它的大小可能与序列长度弱相关或无关；它的 checkpoint、迁移和一致性语义更像 RNN hidden state。这直接改写了 prefix cache 的实现：线性层不缓存 per-token KV，而是在选定边界（如 system prompt 后、每轮后）缓存定长的状态快照、载入即从该点 resume；但递归是有损的，状态快照无法像 KV 那样切到更短前缀，只能在存过快照的确切边界复用（要任意位置 resume 就得载最近的早快照、再前向重算几个 token）。混合模型于是要同时维护 full attention 的 token 级 KV 与线性层的边界级状态快照，有效命中长度取两者的下界。

**系统含义：** 过去的 KV manager 可以假设 block 是主要单位，prefix match 返回连续 block 列表，offload/load 也按 block 执行。Hybrid/linear 模型要求同时管理 block-level full attention KV、request-level recurrent state、压缩域 KV、indexer state 和模型特定 sidecar pool。**未来 runtime 更稳的方向是把”可复用状态”抽象成多种 state group，每组声明自己的匹配粒度、生命周期、迁移成本和 correctness guard。**

**（v1.13 更新）** 上面这段写于 6 月初的预判，一个月内在 vLLM 和 SGLang 里同时落地。vLLM 侧：Model Runner V2 支持 Mamba 混合模型的前缀缓存对齐[\[67\]](https://github.com/vllm-project/vllm/pull/42406)（v0.25 起 MRv2 已是全部 dense 模型的默认路径[\[65\]](https://github.com/vllm-project/vllm/releases/tag/v0.25.0)）、线性注意力层的 prefix-cache 保留区间生效[\[64\]](https://github.com/vllm-project/vllm/releases/tag/v0.24.0)、NixlConnector 新增 Mamba prefix-caching 模式让 recurrent state 也能跨节点交接[\[63\]](https://github.com/vllm-project/vllm/releases/tag/v0.23.0)；准入端引入 Marconi 式策略——正因为快照只能在确切边界整段复用，缓存决策从”来了就存”改为按预期复用价值筛选[\[68\]](https://github.com/vllm-project/vllm/pull/37898)。SGLang 侧：SWA/Mamba 混合模型默认经 UnifiedTree 启动 HiCache，分层 offload 对混合模型开箱即用[\[69\]](https://github.com/sgl-project/sglang/pull/27759)；线性注意力 prefix cache 的 checkpoint 池支持 int8 存储，同容量下可保留的状态快照条数近乎翻倍（BF16→int8）[\[70\]](https://github.com/sgl-project/sglang/pull/28185)。这组变化的共同点是：**“按 state group 声明匹配粒度、生命周期与准入规则”已经是两大框架的现行实现。**

### 9.7 分支材料：KV 量化与压缩

KV 量化和压缩是容量优化的重要手段，但收益必须穿过 kernel 和调度才能兑现。

**核心观点一：量化收益不能只看 bytes/token。** vLLM FP8 KV cache 的收益会受 attention backend 限制；TRT-LLM 的 FP8 KV 吞吐收益主要来自 effective batch size 放大，而非 attention 计算本身加速；NVFP4 的 TTFT 改善来自更高 cache hit 和更大可缓存 prefix，而不是每次 matmul 更快[\[9\]](https://miraclefarms.github.io/notes/2026/05/27/vllm-trtllm-kv-cache-quantization/)。

**核心观点二：agent 场景增加了精度敏感性。** 工具 schema、函数签名、observation、reasoning token 的误差容忍度不同，统一 2-bit/4-bit 量化容易在工具调用上造成任务级失败。TriAxialKV 这类工作把 token 按时间、模态、语义角色分轴分配 bitwidth，**背后的系统含义是 KV 量化策略会进入 harness 和 tokenizer 模板层。**

**代表方法：** FP8（vLLM/TRT-LLM）、INT8、NVFP4（Blackwell）、TurboQuant（3-bit）、OCTOPUS、Runtime-Certified KV、TriAxialKV、KVServe[\[25\]](https://arxiv.org/abs/2605.13734)。

### 9.8 分支材料：Attention 与 FFN/MoE 分离（AFD）

AF 分离（Attention-FFN Disaggregation，AFD）指把每个 Transformer 层内的 attention 模块与 FFN/MoE 模块拆到不同 GPU、资源池乃至不同芯片上执行。动机来自两者相反的计算画像：GQA attention decode 明显受 HBM 带宽约束，而 DeepSeek 这类 MLA 会把瓶颈推向 HBM 与 tensor-core compute 的交界处（见 6.2）；dense FFN 是 compute-bound，MoE 的稀疏激活又会因每 token 只激活少数专家、权重读取摊不开而把 FFN 推回 memory-bound。两类模块同置一卡谁都喂不饱；分离后各自可以独立 sizing、用各自合适的并行策略、甚至异构硬件。

**核心观点：AFD 沿”层内模块”切，PD 分离沿”请求阶段”切，两者正交可叠加。**ByteDance 的 MegaScale-Infer 是代表系统：把每层的 attention 与 expert 模块分到不同 GPU（disaggregated expert parallelism），用 ping-pong pipeline 把一个 batch 拆成 micro-batch 在两侧往返流水，并配一个 M2N 通信库压低 token dispatch 开销，单 GPU 吞吐最高约 1.90 倍[\[42\]](https://arxiv.org/abs/2504.02263)。

**对 KVCache 的系统含义（也是它进本文的理由）：PD 分离搬的是 KV（prefill→decode），AFD 搬的是 activation/token——KV 始终留在 attention 池。**于是 AFD 把 KV 局部化到 attention tier，FFN/expert 池对 KV 是无状态的；KV 的 placement、命中和调度因此与 FFN/专家的扩缩容解耦。代价是 A↔F 边界变成一条高频的 activation/token-dispatch 传输链（M2N、all-to-all），这正是 6.2 互联层面所说”KV、activation、expert dispatch 混在同一张网”里的 activation/dispatch 那一股。

**硬件专门化是 AFD 的自然延伸。**attention（带宽密集）与 FFN/MoE（算力与路由密集）画像相反，为”每一侧用不同芯片”留出空间。一个信号是 NVIDIA 2025 年底通过”技术授权 + 团队并入（acqui-hire）”方式拿下以片上 SRAM、超低延迟著称的推理芯片公司 Groq——双方未披露交易金额，媒体报道规模约 200 亿美元[\[43\]](https://www.fool.com/investing/2025/12/28/nvidia-groq-deal-acquisition-ai-inference-lpu/)——这类带宽优先的专用硅片，正对应 attention 这一侧的 memory-bound 特征。这条路能走多远，取决于 A↔F 之间高频 activation 传输的网络与封装能力；和第五章的 PD 分离一样，支点仍是”跨边界传输贵不贵”。

### 9.9 分支材料：弹性与成本优化

KVCache 系统的成本优化不只是”省显存”。至少包括四类成本。

**重算成本：** cache miss 后重新 prefill 长 prefix 会消耗 GPU compute 和用户 TTFT。

**保留成本：** 把冷 KV 留在 HBM 会挤掉 active batch；留在 DRAM/SSD/CXL 也有容量和管理成本。

**搬运成本：** RDMA、PCIe、NVLink、CXL、SSD I/O 都不是免费，搬运还可能干扰 compute。

**复杂性成本：** 每引入一个层级，就要多一套 metadata、一致性、监控、回收和故障恢复。

生产系统要做的是比较这四类成本，而不是机械追求最大 hit rate。NIXL 统一数据面、KVBM 统一内存层、Mooncake 统一 Store/Transfer、HiCache 统一 L1/L2/L3，本质上都在降低复杂性成本。**未来的 KVCache 控制面应该像数据库优化器一样，把请求、模型、拓扑、cache 状态和 SLO 共同输入，输出一个可解释的执行计划。**

### 9.10 分支材料：推理系统三层抽象与网络通信设计

**推理系统架构的三层抽象。** 推理系统的组织原则正在从 compute-centric 转向 data-centric。一个更合理的系统抽象可以分三层：

- **KV Identity**：统一描述 model id、version、adapter、attention state group、token lineage、position、tenant salt、precision、layout。没有 identity，跨实例共享就有正确性风险。
- **KV Placement**：统一描述 block/state 位于 HBM、DRAM、CXL、SSD、remote store 中的哪一层、哪台机器、哪条访问路径，成本是多少，是否有 lease，是否可被驱逐。
- **KV Action**：load、offload、prefetch、pin、unpin、evict、compress、decompress、quantize、validate、replicate、transfer、cancel 都应该成为 runtime event，而不是散落在各个 connector 里的副作用。

Mooncake 更偏数据面和 Store；KVBM 更偏内存子系统；NIXL 更偏传输库；HiCache 更偏框架内分层策略。未来可能出现类似 “KVFS” 或 “KV memory OS” 的层次。这个趋势很像传统分布式系统里 memcache/redis 的演进路径，**但关键差异是 KVCache 的正确性与模型、位置、attention 结构和精度强绑定，不能简单照搬。**

**网络和通信架构的启发。** KVCache 传输的流量特征和训练通信不同：请求到来不均匀，block 大小随模型和上下文变化，命中位置不固定，tail latency 敏感，metadata lookup 和小块传输很多，读写比在 agent 场景高度偏读。

定量估算有助于网络架构师做 capacity planning。以 PD 分离场景为例：一条 100K context 的 GQA 模型（8 KV heads, 80 layers, head\_dim 128, BF16）请求，单次 prefill→decode KV transfer 的 payload 约为 100K × 8 × 128 × 2 × 80 × 2B ≈ 32GB。如果 TTFT SLO 要求 KV 传输在 200ms 内完成，**所需点对点带宽为 160GB/s——这已经超过单端口 400Gbps RDMA NIC（理论 ~50GB/s）的能力，需要多 NIC 或 NVLink 辅助。**MLA 模型因为 cache state 体积更小（按 512 + 64 = 576 元素计，约 100K × 576 × 80 × 2B ≈ 9.2GB），同样 SLO 下单 NIC 可能足够。burst pattern 方面，PD transfer 是大块顺序写（prefill 完成后一次性推送），而 cache miss load-back 更可能是散列的小块读取（匹配到部分 prefix 后按 block 拉回），两种流量的 NIC 调度需求不同。metadata lookup（block hash query、identity validation）则是大量小包（< 1KB），对 message rate 而非带宽敏感。

**传输路径的选择直接影响网络拓扑设计。** 当前主要有三条路径：(1) GPUDirect RDMA（GPU HBM → NIC → 网络 → NIC → 远端 GPU HBM），延迟最低但要求 NIC 与 GPU 在同一 PCIe root complex 或通过 NVSwitch 直连，对 PCIe topology 敏感；(2) D2H + Host RDMA（GPU → 本机 CPU DRAM via PCIe → NIC → 远端 host DRAM → 远端 GPU），多一跳 PCIe 拷贝但对 NIC placement 要求更宽松，Mooncake 的实现中 worker 注册 GPU memory 为 RDMA buffer 走的是第一条路径[\[1\]](https://vllm.ai/blog/2026-05-06-mooncake-store)；(3) 经 CXL memory pool 的路径（GPU → PCIe → CXL switch → pooled memory），同 rack 内多 host 可共享，不需要 RDMA 协议栈但延迟更高。网络架构师在选择时需要权衡：GPUDirect RDMA 吞吐最高但 NIC 数量可能受 PCIe lane 限制；Host RDMA 灵活但增加一跳延迟和 CPU 内存带宽占用；CXL 路径适合 rack 内共享但跨 rack 不可用。

网络平面设计需要分层考虑。HBM 内和单节点内，NVLink/NVSwitch 适合 tensor parallel、expert parallel、GPU-GPU KV handoff。Rack 内，RDMA 和 CXL memory pool 承担 warm KV transfer。更冷的层级走 NVMe-oF、对象存储或分布式文件系统。

**KV transfer 与其他推理流量混合时的 QoS 是一个尚未被充分讨论的问题。** 推理集群的网络上同时跑着多类流量：KV transfer（大块、burst、延迟敏感）、expert dispatch（MoE all-to-all，高频、带宽敏感）、weight update/RL streaming（大块、周期性、可容忍延迟）、metadata/control plane（小包、低带宽）。训练集群可以用 NCCL ring/tree 做确定性调度，但推理流量的到达不可预测。当前的实践主要有两种：物理隔离——部分部署为 KV transfer 和 EP 分别使用不同的 NIC 或不同的网络平面（如 NVIDIA Dynamo 文档中 PD transfer 和 EP dispatch 可以走不同 NIXL backend[\[15\]](https://developer.nvidia.com/blog/enhancing-distributed-inference-performance-with-the-nvidia-inference-transfer-library/)）；逻辑隔离——通过 RDMA QoS class（如 DSCP-based SL mapping）或 NIC-level traffic shaping 做优先级。目前没有哪个框架提供了开箱即用的 KV-aware QoS 配置，这是一个值得系统团队投入的方向。**（v1.13 更新）**这个空白在 2026 年 6–7 月被填上第一块砖：Mooncake TENT 合入 deadline-aware 准入（MLU 可行性判据 + `LOCAL_DECODE_SUGGESTED` 降级信号）与 TrafficClass 通道映射（KV\_PUT / KV\_GET / EP\_DISPATCH / CTRL → SL/TC/QP 池）[\[76\]](https://github.com/kvcache-ai/mooncake/issues/2519)[\[77\]](https://github.com/kvcache-ai/mooncake/issues/2568)，SGLang 社区 RFC 也在打通请求 priority 到传输队列的链路[\[78\]](https://github.com/sgl-project/sglang/issues/28631)。KV-aware QoS 从建议变成了可以在传输层直接配置的能力；边界在于全部 opt-in，且端到端闭环（真实带宽估计注入、上层消费降级信号触发本地重算）仍待补齐[\[79\]](https://miraclefarms.github.io/notes/2026/07/08/mooncake-tent-qos-deadline-traffic-class-reading/)。

拓扑调度也要感知 prefix locality。多个用户共享 system prompt、多个 subagent 共享环境状态、多轮会话共享历史，都会让某些 prefix 在集群里形成热点。**调度器如果能把具有共享 prefix 的请求聚到同一 locality domain 内，网络流量会更局部。**

带宽规划需要把模型 attention 结构映射为 bytes/token/layer，把 expected hit length 映射为 load-back bytes，把 session turn rate 映射为 read/write ratio，把 PD 配置映射为 prefill-to-decode transfer。**未来推理集群的网络规划，需要 KV-aware traffic model，而不是只用训练集群的 bisection bandwidth 经验。**

- - -

## 参考资料

\[1\] [Serving Agentic Workloads at Scale with vLLM x Mooncake](https://vllm.ai/blog/2026-05-06-mooncake-store)

\[2\] [Efficient Memory Management for Large Language Model Serving with PagedAttention](https://arxiv.org/abs/2309.06180)

\[3\] [Anthropic Prompt Caching Documentation](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)

\[4\] [前期调研：Agentic 推理的性能重构：KV Cache 与系统调度的十个新判断](https://miraclefarms.github.io/notes/2026/05/28/agentic-inference-kv-cache-benchmark-survey/)（原始来源为文内十篇 arXiv 论文）

\[5\] [MiMo-V2.5 Inference Optimization](https://mimo.xiaomi.com/zh/blog/mimo-v2-5-inference)

\[6\] [前期调研：KV Cache 不再只有一种增长公式](https://miraclefarms.github.io/notes/2026/06/01/kvcache-growth-attention-variants-essay/)（原始来源为各模型官方 config、报告与 KVServe 论文）

\[7\] [Alibaba Cloud Tair Partners with SGLang to Build HiCache](https://www.alibabacloud.com/blog/alibaba-cloud-tair-partners-with-sglang-to-build-hicache-constructing-a-new-cache-paradigm-for-agentic-inference_602767)

\[8\] [MiniMax M3: Frontier Coding, 1M Context, Native Multimodality](https://www.minimax.io/blog/minimax-m3)

\[9\] [前期调研：vLLM vs TensorRT-LLM KV Cache 量化实操指南](https://miraclefarms.github.io/notes/2026/05/27/vllm-trtllm-kv-cache-quantization/)（原始来源为 vLLM / TensorRT-LLM / SqueezeBits 相关测试材料）

\[10\] [SGLang HiCache System Design and Optimization](https://docs.sglang.io/advanced_features/hicache_design.html)

\[11\] [TensorRT-LLM KV Cache System](https://nvidia.github.io/TensorRT-LLM/features/kvcache.html)

\[12\] [前期调研：KV Cache 前缀匹配的设计分野](https://miraclefarms.github.io/notes/2026/05/13/kvcache-prefix-matching-design/)（原始来源为 vLLM / SGLang / TensorRT-LLM 源码与 PagedAttention 论文）

\[13\] [NVIDIA Dynamo KVBM Design](https://docs.nvidia.com/dynamo/design-docs/component-design/kvbm-design)

\[14\] [Mooncake Joins PyTorch Ecosystem](https://pytorch.org/blog/mooncake-joins-pytorch-ecosystem/)

\[15\] [Enhancing Distributed Inference Performance with the NVIDIA Inference Transfer Library](https://developer.nvidia.com/blog/enhancing-distributed-inference-performance-with-the-nvidia-inference-transfer-library/)

\[16\] [Unified Cache Manager Documentation](https://ucm.readthedocs.io/en/latest/index.html)

\[17\] [前期调研：Tutti：SSD 做 KV Cache 的关键不在带宽，在控制面](https://miraclefarms.github.io/notes/2026/05/23/tutti-ssd-kv-cache/)

\[18\] [Beluga: A CXL-Based Memory Architecture for Scalable and Efficient LLM KVCache Management](https://arxiv.org/abs/2511.20172)

\[19\] [Compute Express Link: About CXL](https://computeexpresslink.org/about-cxl/)

\[20\] [Penguin Solutions Introduces CXL-Based KV Cache Server](https://ir.penguinsolutions.com/news/news-details/2026/Penguin-Solutions-Introduces-Industrys-First-Production-Ready-CXL-Based-KV-Cache-Server/default.aspx)

\[21\] [NVIDIA Dynamo KVBM Overview](https://docs.nvidia.com/dynamo/components/kvbm)

\[22\] [Full-Stack Optimizations for Agentic Inference with NVIDIA Dynamo](https://developer.nvidia.com/blog/full-stack-optimizations-for-agentic-inference-with-nvidia-dynamo/)

\[23\] [DeepSeek-V4 Technical Report](https://huggingface.co/deepseek-ai/DeepSeek-V4-Pro/blob/main/DeepSeek_V4.pdf)

\[24\] [Predictive Multi-Tier Memory Management for KV Cache](https://arxiv.org/abs/2604.26968)

\[25\] [KVServe: Service-Aware KV Cache Compression for Communication-Efficient Disaggregated LLM Serving](https://arxiv.org/abs/2605.13734)

\[26\] [前期调研：从 KV Cache 底座到推理统一数据面：Mooncake 的架构演化、生态落地与选型](https://miraclefarms.github.io/notes/2026/06/02/mooncake-unified-data-plane-architecture/)

\[27\] [前期调研：NVIDIA NIXL：推理超节点里的数据移动层](https://miraclefarms.github.io/notes/2026/05/26/nvidia-nixl-inference-transfer-layer/)

\[28\] [前期调研：NVIDIA Dynamo KVBM 源码笔记：KV cache 开始拥有自己的内存子系统](https://miraclefarms.github.io/notes/2026/06/02/nvidia-dynamo-kvbm-architecture-field-note/)

\[29\] [前期调研：CXL + KVCache 现状调研报告](https://miraclefarms.github.io/notes/2026/05/13/cxl-kvcache-survey/)

\[30\] [前期调研：Beluga：CXL 内存池为什么会改变 KV Cache Offload 的工程重心](https://miraclefarms.github.io/notes/2026/05/13/beluga-cxl-kvcache-memory-pool/)

\[31\] [SGLang: Efficient Execution of Structured Language Model Programs](https://arxiv.org/abs/2312.07104)

\[32\] [NVIDIA Dynamo: Disaggregated Serving Design（Conditional Disaggregation / Runtime-Reconfigurable xPyD）](https://docs.nvidia.com/dynamo/v-0-7-1/design-docs/disaggregated-serving)

\[33\] [DeepSeek-V3/R1 Inference System Overview（官方，prefill EP32 / decode EP144）](https://github.com/deepseek-ai/open-infra-index/blob/main/202502OpenSourceWeek/day_6_one_more_thing_deepseekV3R1_inference_system_overview.md)

\[34\] [Master KV Cache Aware Routing with llm-d（Red Hat，KV-Cache Indexer / Endpoint Picker）](https://developers.redhat.com/articles/2025/10/07/master-kv-cache-aware-routing-llm-d-efficient-ai-inference)

\[35\] [SGLang v0.4: Cache-Aware Load Balancer（LMSYS）](https://www.lmsys.org/blog/2024-12-04-sglang-v0-4/)

\[36\] [NVIDIA Dynamo: KV Cache Aware Routing（KVIndexer / RadixTree）](https://docs.nvidia.com/dynamo/latest/user-guides/kv-cache-aware-routing)

\[37\] [DistServe: Disaggregating Prefill and Decoding for Goodput-optimized LLM Serving](https://arxiv.org/abs/2401.09670)

\[38\] [Splitwise: Efficient Generative LLM Inference Using Phase Splitting](https://arxiv.org/abs/2311.18677)

\[39\] [Taming Throughput-Latency Tradeoff in LLM Inference with Sarathi-Serve（OSDI’24，chunked prefill）](https://arxiv.org/abs/2403.02310)

\[40\] [vLLM PR #18242：基于 P2P NCCL 的 xPyD 原生实现（1P3D / 3P1D）](https://github.com/vllm-project/vllm/pull/18242)

\[41\] [When Open Source Meets Open Source: A Joint Effort Between LMCache and Mooncake](https://blog.lmcache.ai/en/2026/05/26/when-open-source-meets-open-source-a-joint-effort-between-lmcache-and-mooncake/)

\[42\] [MegaScale-Infer: Serving Mixture-of-Experts at Scale with Disaggregated Expert Parallelism（ByteDance，attention/FFN 分离）](https://arxiv.org/abs/2504.02263)

\[43\] [Nvidia’s “Aqui-Hire” of Groq Marks Its Entrance Into the Non-GPU AI Inference Chip Space（约 200 亿美元，2025-12）](https://www.fool.com/investing/2025/12/28/nvidia-groq-deal-acquisition-ai-inference-lpu/)

\[44\] [DeepSeek-V4: a million-token context that agents can actually use（Hugging Face）](https://huggingface.co/blog/deepseekv4)

\[45\] [DeepSeek-V4 on Day 0: From Fast Inference to Verified RL with SGLang and Miles](https://www.lmsys.org/blog/2026-04-25-deepseek-v4/)

\[46\] [Kimi-K2.6 config.json](https://huggingface.co/moonshotai/Kimi-K2.6/resolve/main/config.json)

\[47\] [Kimi-K2.6 modeling\_deepseek.py](https://huggingface.co/moonshotai/Kimi-K2.6/raw/main/modeling_deepseek.py)

\[48\] [DeepSeek-V3.2 config.json](https://huggingface.co/deepseek-ai/DeepSeek-V3.2/resolve/main/config.json)

\[49\] [前期调研：主流 Attention 算法全景：从全连接到稀疏、线性与 KV 压缩](https://miraclefarms.github.io/notes/2026/04/20/mainstream-attention-algorithms-overview/)（原始来源为各 attention 论文与模型技术报告）

\[50\] [前期调研：Claude Code 论文再读：Agent Loop 很小，真正的系统在 Loop 外面](https://miraclefarms.github.io/notes/2026/06/01/claude-code-design-space-reading/)（原始来源为 Claude Code design-space 论文与 Anthropic 官方文档）

\[51\] [llm-d: KV-Cache Wins You Can See](https://llm-d.ai/blog/kvcache-wins-you-can-see)

\[52\] [vLLM Router: A High-Performance and Prefill/Decode Aware Load Balancer](https://vllm.ai/blog/2025-12-13-vllm-router-release)

\[53\] [Lodestar: An Online-Learning LLM Inference Router](https://arxiv.org/abs/2606.00946)

\[54\] [前期调研：大规模分布式推理调度层（上）——KV cache 如何把负载均衡变成状态局部性问题](https://miraclefarms.github.io/notes/2026/06/08/inference-scheduling-kv-locality-survey-part1/)

\[55\] [前期调研：大规模分布式推理调度层（下）——从相位、区域到学出来的路由](https://miraclefarms.github.io/notes/2026/06/08/inference-scheduling-kv-locality-survey-part2/)

\[56\] [Serving Large Language Models on Huawei CloudMatrix384](https://arxiv.org/abs/2506.12708)

\[57\] [Insights into DeepSeek-V3: Scaling Challenges and Reflections on Hardware for AI Architectures](https://arxiv.org/abs/2505.09343)

\[58\] [DualPath: Breaking the Storage Bandwidth Bottleneck in Agentic LLM Inference](https://arxiv.org/abs/2602.21548)

\[59\] [ShadowServe: Interference-Free KV Cache Fetching for Distributed Prefix Caching](https://arxiv.org/abs/2509.16857)

\[60\] [KVDrive: A Holistic Multi-Tier KV Cache Management System for Long-Context LLM Inference](https://arxiv.org/abs/2605.18071)

\[61\] [HyperParallel: A Supernode-Affinity AI Framework](https://arxiv.org/abs/2603.03731)

\[62\] [Quest: Query-Aware Sparsity for Efficient Long-Context LLM Inference](https://arxiv.org/abs/2406.10774)

\[63\] [vLLM v0.23.0 Release Notes（多级 offload 框架、HMA 默认、NIXL Mamba prefix-caching 模式）](https://github.com/vllm-project/vllm/releases/tag/v0.23.0)

\[64\] [vLLM v0.24.0 Release Notes（self-describing KV 事件、NIXL P→D push、Mooncake connector 异步/零拷贝查询）](https://github.com/vllm-project/vllm/releases/tag/v0.24.0)

\[65\] [vLLM v0.25.0 Release Notes（Model Runner V2 默认化、Mamba 混合前缀缓存、Mooncake GDN/MLA 支持）](https://github.com/vllm-project/vllm/releases/tag/v0.25.0)

\[66\] [vLLM PR #41968：对象存储作为多级 KV offload 的二级层（经 NIXL）](https://github.com/vllm-project/vllm/pull/41968)

\[67\] [vLLM PR #42406：Model Runner V2 支持 Mamba 混合模型前缀缓存对齐](https://github.com/vllm-project/vllm/pull/42406)

\[68\] [vLLM PR #37898：混合模型缓存的 Marconi 式准入策略](https://github.com/vllm-project/vllm/pull/37898)

\[69\] [SGLang PR #27759：SWA/Mamba 混合模型默认经 UnifiedTree 启动 HiCache](https://github.com/sgl-project/sglang/pull/27759)

\[70\] [SGLang PR #28185：线性注意力 prefix cache 的 int8 checkpoint 池](https://github.com/sgl-project/sglang/pull/28185)

\[71\] [SGLang PR #26227：decode 侧 HiCache 预取与 PD 增量 KV 传输](https://github.com/sgl-project/sglang/pull/26227)

\[72\] [TensorRT-LLM PR #14806：向 KV cache connector 暴露存量 block-hash chain](https://github.com/NVIDIA/TensorRT-LLM/pull/14806)

\[73\] [TensorRT-LLM PR #15106：BaseResourceManager 级 KV cache 压缩管理框架](https://github.com/NVIDIA/TensorRT-LLM/pull/15106)

\[74\] [LMCache v0.5.1 Release Notes（MP 模式成熟、Device-DAX 缓存介质、L2→L1 warm-prefetch）](https://github.com/LMCache/LMCache/releases/tag/v0.5.1)

\[75\] [Mooncake PR #2833：Conductor 落地为 mooncake-store 内的 C++ KV 目录服务](https://github.com/kvcache-ai/mooncake/pull/2833)

\[76\] [Mooncake RFC #2519：TENT 传输的 deadline-aware 准入与自适应降级](https://github.com/kvcache-ai/mooncake/issues/2519)

\[77\] [Mooncake RFC #2568：TransferEngine 的 Traffic Class hint API（KV/EP/CTRL → SL/TC）](https://github.com/kvcache-ai/mooncake/issues/2568)

\[78\] [SGLang RFC #28631：跨后端 KV 传输 QoS——把请求 priority 打通到 P→D 传输与 HiCache](https://github.com/sgl-project/sglang/issues/28631)

\[79\] [前期调研：Mooncake TENT 的服务质量三轴：KV 传输从优先级排序走向 deadline 契约](https://miraclefarms.github.io/notes/2026/07/08/mooncake-tent-qos-deadline-traffic-class-reading/)

\[80\] [前期调研：vLLM 前缀缓存与分层卸载：一条被刻意解耦的两段式管线](https://miraclefarms.github.io/notes/2026/06/16/vllm-prefix-cache-offload/)

\[81\] [SAC: Disaggregated KV Cache System for Sparse Attention LLMs with CXL](https://arxiv.org/abs/2606.19746)

\[82\] [前期调研：SAC——用 CXL 给稀疏注意力做一个只取 top-k 的分离式 KV Cache](https://miraclefarms.github.io/notes/2026/06/21/sac-cxl-sparse-attention-kv-cache/)

\[83\] [TraCT: Disaggregated LLM Serving with CXL Shared Memory KV Cache at Rack-Scale](https://arxiv.org/abs/2512.18194)

\[84\] [The Price of Anarchy in Disaggregated Inference](https://arxiv.org/abs/2606.17081)

\[85\] [前期调研：P/D 分离推理的失控点——Dynamo 路由博弈阅读](https://miraclefarms.github.io/notes/2026/07/03/dynamo-disaggregated-inference-price-of-anarchy/)

\[86\] [DualMap: Enabling Both Cache Affinity and Load Balancing for Distributed LLM Serving](https://arxiv.org/abs/2602.06502)

\[87\] [LMCache v0.5.2 Release Notes（L1 全局 P2P token 匹配、pin/unpin token API、Azure Blob / Valkey / Bigtable 适配器、CacheBlend dual-RoPE）](https://github.com/LMCache/LMCache/releases/tag/v0.5.2)

\[88\] [Mooncake PR #2996：TENT 经 Transport 接口暴露 RDMA NIC 负载统计](https://github.com/kvcache-ai/Mooncake/pull/2996)

\[89\] [Mooncake PR #2690：只剩本地盘副本的热 key 走 L2→L1 提升后台重试](https://github.com/kvcache-ai/Mooncake/pull/2690)

\[90\] [Mooncake PR #2945：host 段 pin 操作纳入配额管理](https://github.com/kvcache-ai/Mooncake/pull/2945)

\[91\] [前期调研：Mooncake 与 SGLang 的结合点全景——L1/L2/L3、HiCache 与 PD 增量传输](https://miraclefarms.github.io/notes/2026/07/17/sglang-mooncake-integration-panorama/)

\[92\] [SGLang v0.5.15 Release Notes（MLA decode context parallelism、HiCache 联动、DeepSeek-V4 request state pool）](https://github.com/sgl-project/sglang/releases/tag/v0.5.15)

### 版本对齐信息（v1.13 月度更新材料）

| 材料                                           | 版本锚点                                                                       | 查询日期       |
| -------------------------------------------- | -------------------------------------------------------------------------- | ---------- |
| vLLM                                         | v0.23.0（2026-06-15）/ v0.24.0（2026-06-29）/ v0.25.0（2026-07-11）release notes | 2026-07-13 |
| vLLM PR #41968（对象存储二级层）                      | merge `6a894574`（2026-06-05，入 v0.23.0）                                     | 2026-07-13 |
| vLLM PR #42406（Mamba 混合前缀缓存）                 | merge `61ab70ec`（2026-06-29，入 v0.25.0）                                     | 2026-07-13 |
| vLLM PR #37898（Marconi 式准入）                  | merge `dc66e01a`（2026-06-10，入 v0.24.0）                                     | 2026-07-13 |
| SGLang                                       | v0.5.13（2026-06-13）/ v0.5.14（2026-06-26）/ v0.5.15（2026-07-10）release notes | 2026-07-13 |
| SGLang PR #27759（UnifiedTree 默认）             | merge `5e027153`（2026-06-11，入 v0.5.13）                                     | 2026-07-13 |
| SGLang PR #28185（int8 checkpoint 池）          | merge `3340f4e3`（2026-06-18，入 v0.5.14）                                     | 2026-07-13 |
| SGLang PR #26227（decode 侧增量传输）               | merge `3e993f61`（2026-06-02，入 v0.5.13）                                     | 2026-07-13 |
| TensorRT-LLM PR #14806（block-hash chain）     | merge `0edbbfe2`（2026-06-09，入 v1.3.0rc19）                                  | 2026-07-13 |
| TensorRT-LLM PR #15106（KV 压缩管理框架）            | merge `ca81b2a5`（2026-06-25，入 v1.3.0rc20）                                  | 2026-07-13 |
| LMCache                                      | v0.5.1（2026-07-06）release notes                                            | 2026-07-13 |
| Mooncake PR #2833（Conductor）                 | merge `d3ebf384`（2026-07-12，main）                                          | 2026-07-13 |
| Mooncake RFC #2519 / #2568、SGLang RFC #28631 | 开放 RFC（TENT 部分实现已合入 main，逐 PR 锚点见前期调研文章 \[79\]），不视为稳定 release API          | 2026-07-13 |

### 版本对齐信息（v1.14 更新材料，窗口 2026-07-13 至 07-22）

| 材料                                       | 版本锚点                                                        | 查询日期       |
| ---------------------------------------- | ----------------------------------------------------------- | ---------- |
| LMCache                                  | v0.5.2（2026-07-22）release notes                             | 2026-07-22 |
| Mooncake PR #2996（TENT 暴露 RDMA NIC 负载统计） | merge `ac01083`（2026-07-20，main）                            | 2026-07-22 |
| Mooncake PR #2690（L2→L1 提升后台重试）          | merge `3853f43`（2026-07-20，main）                            | 2026-07-22 |
| Mooncake PR #2945（host 段 pin 配额）         | merge `12df58c`（2026-07-22，main）                            | 2026-07-22 |
| SGLang                                   | v0.5.15（2026-07-10）/ v0.5.15.post1（2026-07-14）release notes | 2026-07-22 |
| vLLM                                     | v0.25.1（2026-07-14）为稳定性补丁，窗口内无新增 KV / connector 能力          | 2026-07-22 |
| TensorRT-LLM                             | 窗口内未见可核对的新 release，本版不做新增论断                                 | 2026-07-22 |
| SAC 论文                                   | arXiv:2606.19746 v1（2026-06-18）                             | 2026-07-22 |
| TraCT 论文                                 | arXiv:2512.18194 v1（2025-12-20）                             | 2026-07-22 |
| Price of Anarchy 论文                      | arXiv:2606.17081                                            | 2026-07-22 |
| DualMap 论文                               | arXiv:2602.06502 v1（2026-02-06）                             | 2026-07-22 |
