---
title: "AgentX 读后：agentic 推理把 CUDA 护城河从 kernel 挪到了会话状态"
source: "Readings"
category: "AI Infra"
group: "MiracleFarms"
url: "https://miraclefarms.github.io/notes/2026/09/09/agentx-agentic-inference-cuda-moat/"
published: 2026-09-09T15:40:00+08:00
saved: 2026-09-27T16:26:37+08:00
folo_key: "mf-readings::https://miraclefarms.github.io/notes/2026/09/09/agentx-agentic-inference-cuda-moat/"
tags:
  - "folo"
  - "AI_Infra"
  - "Readings"
---

# AgentX 读后：agentic 推理把 CUDA 护城河从 kernel 挪到了会话状态

> [!info] Readings · AI Infra · 2026-09-09 15:40 · [原文](https://miraclefarms.github.io/notes/2026/09/09/agentx-agentic-inference-cuda-moat/)

![[AgentX 读后：agentic 推理把 CUDA 护城河从 kernel 挪-01.jpg]]

自 2025 年 11 月的 Claude Code 拐点[\[92\]](https://newsletter.semianalysis.com/p/claude-code-is-the-inflection-point)以来，长上下文多轮 agentic 负载迅速成为生产推理的流量主体，2026 年 4 月 OpenAI 的企业 agentic 支出已经超过 ChatGPT 支出。SemiAnalysis 在 AgentX 1.0 里问了一个直白的问题：真实 agentic 负载下，CUDA 护城河还在不在[\[1\]](https://newsletter.semianalysis.com/p/agentx-inferencexv3-does-cuda-moat)。他们花了 300 万美元采集 Claude Code 与 Codex 的真实会话，用约 2 兆瓦、1000 多颗芯片跑完全矩阵，答案是”在”，但护城河的位置整个挪了一层。GLM 5.3 那条对比最说明问题：在 150 tok/s/user 的 p90 交互度上，NVIDIA 的成本效率领先到即便竞品芯片白送、只付电费和托管，每 token 成本仍然是 NVIDIA 更便宜。

真正让这篇报告有阅读价值的，是它解释了这个差距落在哪里。**agentic 推理的成本大头已经不在 attention 的 FLOPs 上，而是会话状态的保存、搬运、路由、重建和反复处理这一整条路径。** 这条路径上的每一环——DCP/PCP 的注意力后端支持、批量 GPU→CPU 拷贝的 runtime API、GPUDirect RDMA 的显存注册路径、能直接 pip 装进上游镜像的 wheel——在 NVIDIA 侧大多已是既成事实，在 AMD 侧还在一个一个补。所谓护城河，此刻的具体形态是”这条状态路径上还剩几个没修的坑”。

这也意味着它是工程债，不是物理定律。AgentX 上线几个月催生的 70 多个上游 PR，本身就是这条债正在被偿还的证据。这篇笔记按报告的原始结构逐层拆开：先看数据集和方法，再看五个前沿模型的成绩单，然后把 70+ PR 按”缓存活下来 / 状态搬得动 / 路由认得路”三条线归类，最后回到 AMD 的真实差距和这套基准自己的边界。

## 一、AgentX 到底测了什么

AgentX 1.0 是首个完整开源（Apache 2.0）的百万上下文多轮 agentic 编码推理基准[\[2\]](https://github.com/SemiAnalysisAI/InferenceX)，完整仪表盘公开可查[\[3\]](https://inferencex.semianalysis.com/agentx)。SemiAnalysis 把它作为一个新场景加进 InferenceXv3，与原有的固定序列长度场景（8k1k、1k1k、1k8k）并列，而不是替换掉后者。

规模值得先记住：完整矩阵跑在约 2 兆瓦持续运行的算力上，覆盖 1000 多颗芯片，SKU 包括 MI355X、GB300 NVL72、GB200 NVL72、B300、B200、MI325、MI300X、H200 和 RTX Pro 服务器；Rubin、TPU 和 MI455X UALoE72 排在后面。这个量级本身就是这类基准过去做不出来的原因——一次完整扫描的电费和机时，远超单个实验室愿意为”benchmark”付出的成本。

开源的程度也超出这个词平时的用法：开放前端、可公开消费的 REST API（已有多个一线实验室的容量规划团队在用）、公开的 GitHub Actions CI 溯源、每个数据点的日志和精度校验。更关键的一条是配置来源——benchmark 配置主要跟随 `recipes.vllm.ai`[\[17\]](http://recipes.vllm.ai/) 和 SGLang cookbook[\[18\]](https://docs.sglang.io/cookbook/intro) 的上游镜像。**用上游镜像而不是厂商调优镜像，测的才是客户实际拿得到的性能，而不是为跑分特调的版本。**

![[AgentX 读后：agentic 推理把 CUDA 护城河从 kernel 挪-02.png]]

*图 1：InferenceX 的基本图形语言——横轴是每用户 tok/s（交互度），纵轴是按 TCO 归一化的吞吐，每个点代表一组完整的软硬件配置，连成的上包络就是 Pareto 前沿。读 AgentX 的所有结论都要落回这张图：任何一个”A 比 B 快”的说法，都只在曲线的某一段成立。图片来自 InferenceX 开源仓库 README（Apache 2.0）[\[2\]](https://github.com/SemiAnalysisAI/InferenceX)。*

## 二、Agentic 负载的四个特征，以及它为什么变成系统问题

报告把 agentic 负载归纳为四条特征，每一条都直接对应一类系统压力。

多轮：一个会话包含几十到上百次用户/助手交互，聊天场景里通常只有三五轮。长上下文：系统提示、工具定义加上轮次累积，上下文迅速膨胀。高前缀复用：对话线性推进，第 n−1 轮的输出被拼接到第 n 轮，绝大部分上下文可以从 KV 缓存供给而不必重算；随着 n 增长，已缓存输入与未缓存输入的比值趋近于 1。子代理突发：一次会话会拉起多个短命子代理，各带全新上下文，制造出突发的 KV 缓存访问模式。

第三条和第四条合起来解释了为什么这类负载”是个系统问题”。前缀复用率极高，意味着 KV 张量必须能在节点与 rank 之间高效搬运（NIXL、MORI-IO、Mooncake）；不同会话应当按照前缀所在位置路由到对应节点，才能把命中率吃满（llm-d、Dynamo、vLLM/SGLang router）；长上下文会话压满 HBM 容量，逼出 DRAM 和 SSD 的分层卸载，而卸载本身要做得足够快（Mooncake Store、LMCache、vLLM Simple Offloading、SGLang HiCache）。

**四条特征里，前两条只是把负载变大，后两条把它变成了一个分布式状态管理问题——这才是 agentic 推理与长上下文推理的分界线。** 固定序列长度的单轮负载完全不碰这些环节。它测出来的基本就是芯片和 kernel 的基线性能，这也是它仍然有用的原因——把系统复杂度剥掉之后，底层优化的进展看得最清楚，同时给 AgentX 结果提供参照基线。两套数据回答的是不同问题，报告没有把旧矩阵作废。

这一点对读过前期调研的人来说不算意外。此前对 agentic 批推理的观察就指出，性能瓶颈正在从”模型生成速度”迁移到”冗余重计算占比”，CONCUR 在 256-agent 批次上测出 middle 阶段有 49.1% 的端到端延迟来自额外重计算而非原生生成。AgentX 相当于把这个观察从单篇论文的实验推进到跨十几种 SKU 的持续测量。

## 三、300 万美元的 trace 是怎么采到、脱敏和公平回放的

方法论部分是这篇报告里工程含量最高的一段，也是判断结果可信度的前提。

采集用的是一个 HTTP 代理：愿意上传 trace 的员工把 Claude / Codex 的 base URL 指向代理即可。截至发稿，累计 8000 多个会话、340 万次请求、6100 亿 token，折合超过 300 万美元的 API 花费。代理同时抽取时间戳、conversation ID、subagent ID 等元数据，用来还原请求顺序、并发分支和会话的父子结构。为了让这套采集成立，SemiAnalysis 和 Anthropic 一起给 Claude Code 上了两个特性[\[9\]](https://github.com/anthropics/claude-code/issues/49207)[\[10\]](https://github.com/anthropics/claude-code/issues/66761)。AgentX v1.0 公开的是其中 393 个会话的代表性子集，托管在 HuggingFace[\[4\]](https://huggingface.co/datasets/semianalysisai/cc-traces-weka-062126)，另有一个 256k 上下文的截断版[\[5\]](https://huggingface.co/datasets/semianalysisai/cc-traces-weka-062126-256k)。

脱敏的思路借鉴了 Qwen-Bailian 数据集[\[8\]](https://github.com/alibaba-edu/qwen-bailian-usagetraces-anon)：把每条请求的内容 token 化后切成 64-token 块，每块替换成一个会话内链式哈希。相同的提示前缀因此产生相同的哈希前缀，KV 复用模式被完整保留，原始内容一个字都不泄露。回放时再用编码/工具使用语料池按确定性采样把哈希块填回真实 token。

数据集在发布前还做了三步清洗：移除 Claude Code 的安全监控（auto mode）请求和标题生成请求，因为它们是 Claude Code 特有而非通用 agentic 流量；移除重建输入长度超过 99 万 token 的请求（这是近似算法高估的产物）；去掉代理在连接断开时收到的重复请求。每个会话按 WEKA trace 格式存储，这个格式来自 Callan Fox 的 kv-cache-tester 项目[\[13\]](https://github.com/callanjfox/kv-cache-tester)；报告自己也说格式相当任意，同一份代理数据集完全可以映射到 Mooncake 等其他格式。

这套方法有它承认的天花板。前沿模型的 thinking/reasoning 内容在 HTTP 请求里已被加密并替换成确定性哈希（本意是防蒸馏），服务端的 chat 模板、Anthropic 自有的 tokenizer、服务端工具引入的额外上下文，代理都看不到；图片和文档的线上表示与模型实际处理的 token 数也没有直接对应。他们用确定性占位符加上按模型经验标定的 padding 来逼近真实长度，并公开了哈希 token 数与真实 API token 数的比值分布。这是诚实的做法，但读者应当记住：**AgentX 复现的是流量形状，不是流量内容。**

回放没有自己造轮子，而是基于 NVIDIA 的 AIPerf[\[6\]](https://github.com/ai-dynamo/aiperf) 维护了一个更中立的 fork[\[7\]](https://github.com/SemiAnalysisAI/aiperf)。一个 agentic 会话被建模为有向无环图：请求是节点，边表示依赖，边上还带一个延迟，代表工具执行这类客户端本地耗时。最简单的会话退化成一条线，边上只有轮间延迟；子代理出现后就成了真正的 DAG——AIPerf 用 (spawning request, join request) 这对键给每个子代理分组，同一个节点分出去的四条流，如果两两在不同时刻汇合，会被折叠成两个分支。HTTP 时间戳只能揭示时序而非因果，所以 AIPerf 同时保留两个约束（记录到的主路径延迟 + 子代理完成），复现观察到的时序和拓扑，而不假装知道 harness 内部隐藏的依赖。

预热和计时的设计针对的是一个具体陷阱：如果所有会话都从第 0 轮开始，系统会遭遇 thundering herd，测到的是冷启动而不是稳态。AgentX 用固定随机种子在每个会话 25%–75% 的墙钟位置取一个切点，把该点之前每条活跃请求流（主代理加所有活跃子代理）的最近一条请求发出去重建状态，等它们排空；第二阶段每条回放通道再推进 10 条请求让 KV 缓存充分materialize。所有预热请求不带轮间延迟、最大输出长度设为 1 token。之后正式采样一小时。

两个细节透露出他们对 agentic 负载的理解深度。一是每条流有 5 分钟空闲上限，避免长轮间间隙把 worker 通道占满整场——这个数字取自 Anthropic 默认的 KV 缓存 TTL[\[11\]](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)，不是随手拍的。二是会话回放完毕后从采样器取下一个会话时，会给每条独立前缀链（主代理链、新上下文子代理链、一次性链）加一个确定性的 cache-bust 标记；fork 出来的子代理继承父标记。同一次回放内标记不变，保住 KV 复用模式；回放之间标记改变，防止人为拉高命中率。这一步还顺带让并发数可以超过数据集里的 393 个会话。

投机解码的公平性单独处理。合成数据会让草稿模型的接受长度失真——它没在合成 token 上训练过。他们推动主流开源引擎都提供了强制指定接受长度的机制，然后对每个 (模型, 草稿器, 草稿长度, thinking 模式) 组合，在 SPEED-Bench 的 agentic 编码数据集上[\[12\]](https://huggingface.co/datasets/nvidia/SPEED-Bench)采集平均接受长度，运行时把这个真实值套回 AgentX。

数据集本身的统计值得单独抄下来，因为它是目前公开可得的最好的 agentic 流量画像：中位 ISL 142k token，中位 OSL 444 token，中位轮间延迟（主要是工具执行时间）3.84 秒，只有约 10% 的轮间延迟超过 1 分钟；393 个会话里 175 个（约 44%）至少含一个子代理，共 1697 次子代理 rollout，每会话中位 4 次，单个子代理从首请求到末请求的中位墙钟时长 2.27 分钟。

这组数字和前期调研中的生产 trace 特征基本对得上：Codex/SWE-bench Pro 的 610 条 trace 呈现中位 33 轮、第 30 轮约 80K token 的形态，DualPath 的 500 条编码轨迹平均 157 轮、每轮仅追加 429 token、KV 命中率 98.7%。AgentX 的中位 ISL 142k 明显高于前者，中位 OSL 444 则和 DualPath 的”每轮小增量”高度一致——**agentic 流量的稳定特征是输入巨大、输出很短、绝大部分输入已在缓存里**，三份独立来源在这一点上互相印证。

## 四、五个前沿模型的成绩单

报告选了五个开放权重前沿模型，每个模型的结论都只在曲线的特定段成立，这也是它反复提醒读者自己去看数据的原因。

### 4.1 DeepSeek V4 Pro 0813：最接近的一场

1.6 万亿总参数、490 亿激活参数。所有 DeepSeek V4 运行的请求分布为 ISL p50=88k、p90=272k、p95=404k、p99=675k，OSL p50=413、p90=2.2k、p95=3.7k、p99=8.6k。

报告顺手给了一条实用标尺：服务 agentic 负载的生产系统，p90 TTFT 一般落在 200–5000ms；超过 5–10 秒就已经在”在线推理”的边界之外了。曲线上确实有 SKU 在很高吞吐、尚可交互度下运行，但 TTFT 已严重劣化——这段仍有实际用途（批处理、超长运行的 agent），只是不能拿来对比在线服务。

单节点上，MI355X 的开源路径（vLLM）落后于 AMD 自家的 ATOM（对标 TensorRT-LLM 的厂商专用引擎）。AMD 的分布式推理团队在 8k1k 场景过去半年进步明显，但在真实负载上还有距离：1xDEP8+1xDEP8 的 PD 分离配置只在高吞吐段拿到轻微收益，低延迟段反而更差；中高交互度配置上那点吞吐增益，又被 p90 TTFT 的显著尖峰盖过。原因之一是并发 64 以上启用了 SGLang 的 `--enable-prefill-delayer`[\[19\]](https://github.com/sgl-project/sglang/blob/v0.5.17/python/sglang/srt/server_args.py#L3180-L3194)，它推迟 prefill 准入（最多 30 个 forward pass）好让 DP rank 攒出更满的批次，同时把 chunked prefill 尺寸从 8192 拉到 65536。

端到端延迟上 ATOM MI355X 能赢 B200 vLLM，赢不了 B300 和 B200 SGLang。报告在这里补了一句杀伤力更大的判断：中西方绝大多数 AI 实验室都不愿在生产里用 ATOM，因为缺失的功能太多，只有阿里巴巴一个小的广告业务单元在用——**阿里 Qwen 主力 LLM 团队自己的生产环境也不跑 ATOM。** 这句话把”ATOM 赢了某段曲线”这件事的意义压得很低：赢的那个引擎没人用。

时间线是这场对比里最有意思的部分。2026 年 8 月 21 日之前，MI355X 的 SGLang 路径在端到端性能的每美元指标上追平了 B200 vLLM；8 月 21 日之后，Inferact 和 NVIDIA 在 vLLM 上的一批优化落地，B200 的每美元性能反超 MI355X。B300 vLLM 和 B200 SGLang 则始终在 MI355X 之上。这条排序随上游 commit 每周移动，任何一次快照都只对当天有效。

NVIDIA 侧最有竞争力的是 GB300 Dynamo TRT-LLM 和 GB200 Dynamo vLLM，都靠 PD 分离在合理交互度下拿高吞吐，GB300 配置还用了 DEP32 的宽 EP 解码实例。一个细节能帮助理解 TTFT 的行为：GB200 的 2xDEP8+1xDEP12 与 GB300 的 3xDEP8+1xDEP16 在 TPS 上很接近，在 TTFT 上差得多——GB300 点跑到了更高的总并发，因而遭遇更多子代理流量和更多冷 prefill。TTFT 对负载的”尖刺程度”比 TPS 敏感得多。

### 4.2 Kimi K3：模型太大带出的组合问题

2.8 万亿总参数，报告拿它当 Claude Mythos/Fable5 架构的开放权重代理。这个体量在单台 B200 服务器上放不下，必须用宽 EP / 宽 TP 或流水并行才能装下权重。而 vLLM 上投机解码（DSpark）直到不久前都无法与流水并行组合——**B200 因为用不了”投机解码 + 流水并行”的组合，在 Kimi K3 上被 MI355X 反超，这个差距来自功能组合的缺口，不是算力差距。**

AMD 侧的表现也不平坦：MI355X vLLM 在短上下文单轮负载上第 0 天开箱可用，但长上下文多轮负载第一周里 AITER 和 Triton kernel 大面积崩溃，上游 vLLM 在真实负载上完全不可用。Hopper 则整体吃力——K3 体量太大，vLLM 维护者和 NVIDIA 都没把优化重心放在 Hopper 上，SM90 需要为 K3 定制调优 kernel，还需要 TP32/EP32 的调优形状才能在高交互度下服务。

在端到端延迟 40 到 60 秒这一段，MI355X ATOM 的每美元性能能赢过 GB300 NVL72 vLLM。这是 AMD 在这份报告里最亮的一个点，前提仍是那个”没人在生产里用 ATOM”的限定。

### 4.3 MiniMax M3：NVIDIA 完胜，以及一个路由反直觉

432B。报告的原话是 NVIDIA 在 M3 上碾压所有竞争者，AMD 的软件在长上下文尤其糟糕，归因是 AMD 工程领导层的激励只指向短上下文单轮负载的调优，长上下文多轮被忽略。

B300 TRT-LLM TP2 拿下 M3 的桂冠。前沿曲线上缺少 DP-attention 的点，因为 KV 缓存的局部性在 M3 上变成了路由约束。这里有一组很能说明问题的数字：GB200 并发 40 时，TP4/EP4/DPA 只有纯 TP4 吞吐的 0.60 倍，p90 TTFT 却是 3 倍以上；并发 32 时实际缓存命中率 28.8%，理论值 96.0%。原因是每个 DP rank 私有四分之一的缓存池，一个 30 万 token 的会话落到错误的 rank 上就得全部重算。**当 DP attention 把缓存切成互不可见的私有分区，路由一旦不精确，理论命中率就成了一个与现实无关的数字。**

B200/B300 在 TCO 归一化吞吐上还完胜了它们的机架级同门。AgentX 上机架级的优势没那么突出，一个原因是 Dynamo 路由器可能成为瓶颈——它的工作量随活跃前缀的数量和长度增长，而不是随生成 token 数增长；另一个原因是宽 EP、宽 DCP、宽 TP 都还没有调优好的 kernel，而 GB200/GB300 的 TCO 更高，没有宽并行就体现为更差的每 TCO 性能。

还有一个尚未被利用的空档：尽管 M3 的 p90 ISL 达到 317k，目前没有任何提交在跑上下文并行。M3 只有 4 个 KV head，即使 TP8 下 DCP 也只能切到 2，而 MSA indexer 还需要自己的上下文并行处理（vLLM 上已有 PR 开着）。

KV 卸载的使用模式在这里也分了岔：NVIDIA 所有 Pareto 最优点在并发 20 以上都用了 KV 卸载，AMD 一个都没用。原因具体到一个 API——AMD vLLM 上 GPU 到 CPU 的传输效率低下，`hipMemcpyBatchAsync` 直到 ROCm 7.14 才有；没有它，vLLM 原生的 Simple CPU Offloading 只能串行发 memcpy，无法批成大消息。

### 4.4 Qwen3.5 397B：混合注意力与 20 倍差距

Qwen3.5 每隔几层用 GatedDeltaNet 替代常规注意力。GDN 出自 MIT/NVIDIA Research，状态存储需求理论上是常数而非线性，因此比同等稠密注意力模型的存储需求低。报告对 NVIDIA Research 的评价分得很清楚：GDN、LatentMoE 这类基础研究做得好，端到端模型训练则另说。

这个模型的原生最大上下文是 262k，所以用的是截断数据集——这恰好模拟了小上下文模型上的真实用法：上限被频繁触及、压缩（compaction）频繁发生。

SGLang 对 SGLang 的同框架对比里，NVIDIA 在 90 tok/s/user 处有 20 倍以上的性能优势，AMD 在 Qwen3.5 SGLang 上没有任何竞争性提交。同一模型上另一组数字：B300 FP4 相对 H100 有 12 倍的每美元性能。报告同时指出 NVIDIA 自己也有过度优化交互度而牺牲 TTFT 的倾向，TRT-LLM 尤其明显——同一张图上所有 NVIDIA SGLang 提交的 p90 TTFT 都比 TRT-LLM 低得多。

### 4.5 GLM 5.3：白送芯片也追不上的那一段

GLM 5.3 在 GLM5.2 744B 基础上追加后训练。开源 SGLang 路径上，NVIDIA 再次胜出：150 tok/s/user 的 p90 交互度处，成本效率领先最多 5 倍。报告由此推出那个极端表述——以 AMD 软件的当前状态，在 150 tok/s/user 这个点上，即便竞品硬件白送（服务商仍要付数据中心托管、电力和其他运营成本），每 token 成本仍是用 NVIDIA 更便宜。

换到 ATOM 就翻转了：在 p90 E2E 归一化交互度的部分区间，AMD 的每美元性能好过 GB300 NVL72 SGLang，甚至好过 TRT-LLM。同一块硬件、同一个负载，换个引擎结论反向——这本身就是”护城河在软件”的最直接证据。

## 五、HBM 容量决定命中率，命中率决定一切

DeepSeek V4 的服务器指标可视化里有一组对照，把 agentic 负载的容量经济学讲得比任何一段文字都清楚。

B300 vLLM DEP8 配 3TB DRAM（走 vLLM simple offloading），在 384 条并发 agentic trace 下拿到 91% 的 HBM 缓存命中率，外加 1.36% 的 DRAM 命中率。原因是这个配置的 HBM KV 缓存工作集约 4300 万 token，而负载在任一时刻的在途 token 数只是刚刚超过这个数。

B200 在并发 196、其他参数不变时，HBM 命中率只有 73%，对 DRAM 的依赖显著上升，卸载命中率接近 20%；工作集是 2200 万 token，大约是 B300 的一半。

**同一套软件、同一份负载，仅仅因为 HBM 容量差了 50%，缓存命中率就从 73% 跳到 91%，这不是线性劣化。** 这与前期调研中”agentic 负载的命中率崩塌是相变而非线性退化”的判断吻合——当并发 agent 的聚合上下文越过显存边界，有效输入输出比会从 1.5–7.3 倍跳到 53.9–559.8 倍，prefill 负担放大百倍以上。AgentX 用一组生产级 SKU 对照，把这条论断从论文实验搬到了机架上。

DRAM 卸载的工程规则也顺带给了出来：它通常实现为写穿透缓存，写进 HBM 的前缀同时写进 DRAM，因此只有当可用 DRAM 显著大于 HBM KV 容量（1.5 到 3 倍）时才真正有效。H200 SGLang FP8 是反面教材——低并发下能服务 DeepSeek V4，每美元性能甚至能和 B200/MI355X SGLang 掰腕子，但缺 HBM 使它在高吞吐场景无法竞争，而并发上去后对 DRAM 卸载的依赖会把延迟推到不可接受的水平。

整体上 MI355X 相对 B200/B300 表现不差，最接近的区间是低吞吐低延迟那一段——那里只用张量并行和比较朴素的 kernel。高吞吐段是 AMD 需要补 DEP kernel 的地方，考虑到 MI355X 相对 B200 有 1.5 倍 HBM，这个短板更显眼。

## 六、DCP/PCP：护城河的新地基

长上下文让一些并行策略变得有意义，而 8k 的固定提示根本无法激发它们——8k 没什么可分的，TTFT 本来就短。同时，TP 和 DP attention 在长上下文下都不理想：TP 会让完整 KV 在每个 rank 上复制一份；DP attention 虽然共享了 KV，但长上下文会话的上下文长度方差更大，容易卡住。

上下文并行把 query token 切到多张 GPU 上，分两种形态。PCP（prefill context parallelism）里每个 rank 处理自己那块 query 分片，KV 以 ring 方式传递；prefill 是计算密集的，所以这等于并行了 FLOPs，prefill 更快，也不会出现某个 rank 独自承担超长提示的尖峰。DCP（decode context parallelism）里每个 rank 扫描自己的 KV 分片，部分注意力结果再按 flash-decode 的方式合并；decode 受内存带宽约束，并行读 KV 能提高 tok/s。

报告说这个技术部分由 NVIDIA Research 发明，并直接把它算进 CUDA 护城河，理由是 AMD 的 DCP/PCP 实现尚未优化——在 vLLM 的注意力后端支持矩阵里，AMD 后端全部不支持。这一条我做了独立核对：vLLM 官方文档的后端矩阵中，支持 DCP 的是 FLASHINFER 系列、FLASH\_ATTN 系列以及多数 MLA decode 后端；ROCm 侧的 ROCM\_AITER\_FA、ROCM\_AITER\_UNIFIED\_ATTN、ROCM\_ATTN、ROCM\_AITER\_MLA、ROCM\_AITER\_MLA\_SPARSE、ROCM\_AITER\_TRITON\_MLA、ROCM\_FLASHMLA\_SPARSE\_DSV4 均无 DCP 支持[\[14\]](https://docs.vllm.ai/en/latest/design/attention_backends/)。文档矩阵里没有独立的 PCP 列，报告的这半句无法在同一处核对。

**上下文并行是长上下文时代的基础设施，一个不支持它的注意力后端，在百万级上下文上就没有可用的并行策略可选。** 这就是”护城河从 kernel 挪到系统”的具体含义——差距不在单个算子快多少，而在某一类并行形态在你的后端上根本不存在。

## 七、分布式推理生态：一张图看清 KV 状态的流向

理解后面 70+ PR 的前提，是先看清这个栈的分层。

![[AgentX 读后：agentic 推理把 CUDA 护城河从 kernel 挪-03.png]]

*图 2：llm-d 的架构分层，是这类分布式推理平台的典型形态——上层网关与推理调度器负责按前缀位置和负载选择 worker，下层 prefill/decode worker 之间通过 KV 传输通道交接状态。AgentX 报告引用的正是这一层的结构；后文所有路由、亲和、传输相关的 PR 都落在这张图的某一格里。图片来自 llm-d 官方架构文档（Apache 2.0，CNCF Sandbox 项目）[\[15\]](https://llm-d.ai/docs/architecture)。*

最上层是路由器（有时叫 frontend），按策略把请求分发给不同 worker。当服务器跑数据并行注意力时，每个 DP rank 有各自的 KV 缓存，为了不把任何一个缓存搅乱，请求会按一致性哈希之类的策略路由——同一会话或同一子代理的请求按唯一 ID 走同一条路[\[16\]](https://github.com/vllm-project/router/blob/main/src/policies/consistent_hash.rs)。多数路由策略在不同实现之间没有本质差别，区别在于形态：vLLM router 和 llm-d router 是独立组件，SGLang model gateway 和 ATOM Mesh 集成在引擎里。

请求路由之后交给推理引擎的调度器。每个引擎都提供一个接口，把引擎内部的 KV 缓存连到外部 KV 缓存管理器上，形成一个可插拔生态。AgentX 当前结果里用的一种简单部署是 Mooncake 与 vLLM 同节点共存：每个 vLLM worker 内嵌一个 Mooncake Store 客户端，贡献一部分宿主 DRAM 进外部 KV 池，通过 MooncakeStoreConnector 把可复用的 KV 块载入显存、把新算出的块存回宿主内存；Store 负责放置和驱逐这类策略，Transfer Engine 负责实际的字节搬运。

不同的 KV 管理器可以用不同的传输引擎，而且多条路径能在同一个引擎里并存——比如用 Mooncake Store 把可复用块卸到宿主 DRAM，同时用 NIXL 把请求专属的 KV 从 prefill GPU 直传 decode GPU（走 UCX 和 GPUDirect RDMA）。Dynamo、llm-d、AMD Infera 这类平台把这些组件打包成完整发行版，产出的一般是一组协同容器而非单体服务，通常跑在 k8s 上。**这张分层图解释了为什么 agentic 性能问题很难归因：一次糟糕的 TTFT 可能来自路由器选错了 worker、连接器查找占住了调度器关键路径、传输描述符碎片化，或者 kernel 本身——四层的症状在端到端指标上长得一模一样。**

## 八、70+ PR 之一：让缓存活下来

把 70 多个 PR 平铺出来会变成流水账。更有解释力的分法是按”状态生命周期的三个失败点”归类：缓存活不下来、状态搬不动、路由认不得路。第一类是数量最多的。

矛盾的形态在 vLLM 和 SGLang 上完全一致：滑动窗口页面和前缀页面来自同一个池子，而窗口是更贪婪的消费者——它不停翻新，前缀却一动不动，于是压力一来，短命的分配就把耐用的那份挤掉了。

vLLM 的解法是选择性保留。改进后的混合注意力前缀缓存不再让短命的滑动窗口分配驱逐有用的长上下文检查点，选择性保留会保住稀疏回放边界，在 14 路并发、上下文最长到 100 万 token 的条件下报告了 95% 以上的前缀缓存命中率[\[20\]](https://github.com/vllm-project/vllm/pull/43447)。同一套可达性策略被应用到 Mooncake，不可达的滑动窗口查询被直接移除[\[22\]](https://github.com/vllm-project/vllm/pull/45444)；更早的工作已经停止卸载那些永远不可能被复用的滑动窗口块[\[21\]](https://github.com/vllm-project/vllm/pull/42258)，并把投机前瞻块保留在前缀里[\[23\]](https://github.com/vllm-project/vllm/pull/44082)。

SGLang 从分配器一侧攻同一个问题，三个角度：一是页面离开窗口时主动释放，不等驱逐压力去发现它们[\[36\]](https://github.com/sgl-project/sglang/pull/26907)；二是把计算锁限制在单个窗口内，约束一个在途请求最多能钉住多少池子[\[37\]](https://github.com/sgl-project/sglang/pull/27210)；三是清掉活过其有用期的陈旧完整 KV 条目[\[38\]](https://github.com/sgl-project/sglang/pull/29369)。主动释放留了一个分叉形状的盲点：从共享前缀分支出来的请求，可能还持有可复用的完整 KV，而分叉点上的窗口状态已经被释放，结果为了缺失的那便宜一半，整个前缀被重算。后续工作在分叉点上保住 SWA 状态，让 fork 继承窗口而不是重建[\[39\]](https://github.com/sgl-project/sglang/pull/34565)。

ATOM 的对应改动给出了这一整类问题里最漂亮的一组数字。DeepSeek-V4 分页滑动窗口注意力的稀疏检查点保留，让选定的窗口尾巴活下来，使分支和回放请求能在有用的边界上恢复。在同一条 AgentX trace、并发 48 的条件下，实际前缀命中率从 5.6% 升到 96.45%，而滑动窗口门处的损失从 91.35% 降到 0.16%[\[49\]](https://github.com/ROCm/ATOM/pull/1640)。第二个数字才是机制：十次前缀匹配里有九次是找到了、然后因为缺一段窗口尾巴被丢弃——缓存不是没命中，是命中之后被否决了。

混合模型还带着一份循环状态或压缩器状态，它和普通 KV 有一个决定性区别：无法从周围 token 重建。窗口尾巴可以从邻近上下文重算，循环状态是此前一切的累积结果，一旦丢弃，唯一的回头路是重放整个序列。ATOM 给这份逐请求状态设计了内容寻址的检查点生命周期，让生成出来的轮次留下可复用的恢复点，而不必为它们预留单独的受保护缓存；一次测试中，一个请求复用了 512 个生成 token，只计算了两个 token 的后缀[\[51\]](https://github.com/ROCm/ATOM/pull/1771)。

调参细节和功能本身同样重要：无条件发布检查点会让零命中流量损失 17.5% 的吞吐——这是每一个再也不回来的会话为那些会回来的会话付出的代价。按 token 间隔给检查点排布避开了这个惩罚，固定 1k1k 的吞吐保持在测量噪声内。这才是相关的安全性质：**一个瞄准 agentic 复用的特性，不应该向永远用不到它的负载征税。**

SGLang 的 HiCache 是树内一等的卸载机制，面对同样的混合难题，它用了一个不对称解法：把全注意力缓存卸出去，回来的路上重建那段很短的滑动窗口尾巴。只有昂贵的那一半值得跨总线搬运，便宜的那一半重建比取回更快。AMD 上分阶段写回让这个搬运过程不阻塞引擎[\[54\]](https://github.com/sgl-project/sglang/pull/28534)。循环状态仍是剩下的缺口，因为它没法像窗口尾巴那样从邻近 token 重建；FlashInfer GDN 检查点让它第一次能参与前缀复用，把吞吐从 47771 提到 53004 tok/s/GPU，缓存命中率 92.4%[\[40\]](https://github.com/sgl-project/sglang/pull/29735)。

高并发 agent 需要卸载，而卸载在很长一段时间里只对统一全注意力模型可用。区别在于布局：统一模型每个 token 只有一种 KV 排布，连接器用一套块几何就能描述要存什么；混合模型同时携带多个缓存组，每组形状不同、生命周期不同，一个假设单一布局的连接器无法表达某个块属于哪一组。结果是最需要卸载的那批模型——长会话模型——恰恰用不上它。通用的 SimpleCPU 连接器先落地[\[24\]](https://github.com/vllm-project/vllm/pull/37160)，随后在 ROCm 上启用[\[26\]](https://github.com/vllm-project/vllm/pull/40549)，再扩展到 DeepSeek-V4 的混合注意力[\[25\]](https://github.com/vllm-project/vllm/pull/42296)。相对”前缀装不下 HBM 后重算一次”的基线，**它报告了 81.7% 更高的输出吞吐和 46.6% 更低的平均端到端延迟。** 这个对照组的选择很关键——它比的是重算，不是比另一种卸载实现。Mooncake 也拿到了等价的混合内存分配支持。

同样的布局问题在分离式服务里再次出现：一个待合并的改动通过 MoRI-IO 把 Kimi-K3 的 conv+ssm 循环状态与注意力 KV 一起传给 1P1D 的 prefill/decode 切分，否则 decode 侧会从一个未初始化的循环状态开始；循环状态槽位搭的是已有的 remote-block-ids 通道，所以分离式路由器不需要任何模型专属改动，这条路径在 MI355X 上、每条腿 TP8、跨节点 RDMA、两腿都开 DSpark 投机解码的条件下做了端到端验证。

ATOM 侧有两个缓存管理器修复必须先落地，上面那些数字才测得出来，而它们都是”缓存自称健康却什么也没做”的样本：一个阻止空闲池命中破坏共享缓存条目，另一个（延迟输出修复）在默认调度模式下恢复了前缀哈希，把重复长提示从零缓存 token 变成复用每一个完整前缀块[\[50\]](https://github.com/ROCm/ATOM/pull/902)。另有一个改动让命中前缀的 prefill 留在优化过的 sink attention kernel 上而不是回落到通用路径——一次缓存命中不该悄悄付掉它省下来的一部分。

LMCache 从存储一侧做了同样的可达性论证：只存 DeepSeek-V4 混合分组里真正有用的那部分，每 token 存储量降到近二十分之一[\[76\]](https://github.com/LMCache/LMCache/pull/3635)；滑动窗口预取只加载活的窗口，不再加载永远不会被读的窗口状态[\[77\]](https://github.com/LMCache/LMCache/pull/3869)。

## 九、70+ PR 之二：让状态搬得动

第二类失败发生在搬运环节，典型症状是彻底停住，慢只是它的轻症形态。

LMCache 的一个修复是这类问题的教科书样本。当许多上下文超过 10 万 token 的请求各自在开始前预留整次加载所需的全部块时，池子会被一群”都在等、都没进展”的请求耗尽。分块外部缓存加载改为按块预留，让加载可以交错并排空：并发 32 时验证跑完了 120 个请求，而旧路径在第 28 个之后死锁；并发 48 时 KV 池 98.5% 满仍在继续运行[\[75\]](https://github.com/LMCache/LMCache/pull/3382)。

vLLM 维护者在剖析真实负载时发现，开了卸载之后成本转移到了存储路径上——写得太多、太频繁。三个修复分别处理三种浪费：已有一次相同传输在途时跳过重复存储，让共享同一前缀的并发会话只付一次而不是各付一次[\[27\]](https://github.com/vllm-project/vllm/pull/41289)；一次存储只覆盖新生成的 KV 区间，让延长历史的会话写增量而不是每轮重写整个前缀[\[28\]](https://github.com/vllm-project/vllm/pull/46412)；存储不再依赖同一批块是否仍在 HBM 里，避免驱逐发生在下面时已经排好的工作被丢弃[\[29\]](https://github.com/vllm-project/vllm/pull/46906)。

加载路径单独调优，因为查找发生在每次调度决策上而不只在真正搬数据时。把调度器路径上的查找改成异步，让连接器离开每一步的关键路径，一步不再等待 CPU 侧的缓存查询才能准入工作[\[30\]](https://github.com/vllm-project/vllm/pull/45659)；紧凑的零拷贝查找键、接收侧并行加载、预构建的 Mooncake 键字符串，随后清掉了剩余的 CPU 与传输开销。**这一整组改动没有一个让计算变快，它们只是不再做本来就不必做的事。**

TensorRT-LLM 的 MiniMax-M3 工作则展示了另一种失败形态——粒度爆炸。当 prefill 和 decode 在 head 布局上不一致时，一个逻辑请求的 KV 就从几段大的连续区域变成数千个小的跨步片段，每个片段变成自己的传输描述符。搬运的字节数没变，爆炸的是每描述符开销，而且恰好在最要紧的长提示上爆得最狠。修正多池映射并加上分块 NIXL 中转路径，把这些碎片经由一块有界可复用的暂存区合并，用一次额外的暂存拷贝换取数量级更少的描述符：AgentX 诊断中，请求关键路径上的 KV p99 在并发 5 时从 26.74 秒降到 125 毫秒，并发 40 时从 10.15 秒降到 288 毫秒[\[42\]](https://github.com/NVIDIA/TensorRT-LLM/pull/17518)。

同一条路径上还有一个反馈环：非阻塞的上下文传输轮询即使在调度停滞时也回收已完成的传输，避免完成的 KV 块一直被钉住、阻止新请求准入[\[47\]](https://github.com/NVIDIA/TensorRT-LLM/pull/17428)。而停滞本身也有可去除的成因——DeepSeek-V4 上下文稀疏注意力元数据里的隐式设备标量同步，每步 18 次四字节设备读，每次都强制一个 `cudaStreamSynchronize`，在 GB300 的分离式上下文 worker 上，这让执行线程在一步的大部分时间里持有 GIL，KV 传输线程和响应线程只能排在后面[\[48\]](https://github.com/NVIDIA/TensorRT-LLM/pull/16734)。

两个还开着的传输改动展示了”一个修复怎么制造出下一个瓶颈”。默认安排下，decode worker 必须等整个提示 prefill 完再传输完才能开始，两个昂贵阶段串行执行，尽管前一个是增量产出的。流水化 KV 传输在每个 prefill 块完成时就开始发送，让传输与 prefill 计算重叠，只有最后一块留在关键路径上[\[45\]](https://github.com/NVIDIA/TensorRT-LLM/pull/15727)。这个改动让块处理变得频繁，于是把原本只发生一次的工作暴露了出来；它的后续 PR 改为只取当前块所属的块 ID，而不是每次重建整个提示的块列表——对一个切成 1024-token 块的 128000-token 提示，这是”构建一次 4096 项列表”和”为每个层组重建 128 次”的区别[\[46\]](https://github.com/NVIDIA/TensorRT-LLM/pull/17526)。**一个随提示总长增长的每块成本，会把流水化刚买来的收益整个吃掉。**

SGLang 在异构 prefill/decode 拓扑上碰到的是前缀缓存与分离式服务的恶性互动。两侧分片方式不同时，KV 无法作为一条连续流拷过去，必须在传输网格上切分、再按 decode 侧期望的偏移重组。前缀命中让这件事更难而不是更容易：prefill worker 现在只发未缓存的余量，decode 侧却仍然期待一份完整且位置正确的缓存。暂存缓冲区里的 radix 缓存支持负责在网格上切分已缓存的发送并散布到正确的 decode 偏移[\[43\]](https://github.com/sgl-project/sglang/pull/30545)。

这个修复暴露出来的问题是正确性而非性能：一个 127500-token 的共享前缀测试，从 128 根针里只找对 2 根变成 128 根全对——**缓存此前一直静默地落在错误位置上，而吞吐基准会把这个结果记成”又快又自信的错误答案”。** 同一次 AgentX 对比还把每用户中位输出吞吐提高了 9.6%，每 GPU 总吞吐几乎不变。传输本身也带着无用载荷：decode 侧的 PREBUILT 批次从不进入模型前向，但每个传过来的提示仍被展平并拷进一个 CUDA 输入张量，而第一个 decode 步会从中继元数据里重建它。去掉这次无用的提示传输是这一段里最大的单项收益，在 AgentX GB300 上带来 18.0% 的每用户输出吞吐和 12.7% 的每 GPU decode 吞吐提升[\[44\]](https://github.com/sgl-project/sglang/pull/35070)。

ATOM 的 CPU 路径从一个更朴素的算术开始：独立 LMCache 卸载把 32000-token 的前缀从 CPU 取回约需 0.32 秒，重算约 2.5 秒，八倍的差额[\[52\]](https://github.com/ROCm/ATOM/pull/1318)。这个比值在这个上下文长度上成立，在短提示上不成立——跨总线是否值得，取决于上下文长度。剩下的问题是所有权和索引位置：ATOM 抄了 vLLM 的多连接器设计，让 prefill worker 能同时把 KV 发给远端 decode worker 并把同一份前缀存到 CPU，两个消费者都读完之前不释放块。而把恢复的块提升回 GPU 前缀索引则修掉一个更隐蔽的浪费——不这样做，从 CPU 载入的前缀用完之后不被登记为常驻，下一轮又跨总线取一次同样的热前缀。

AITER 层面暴露的一类失败短固定请求基本永远碰不到：地址位宽。32 位偏移在单个缓存池越过边界之前完全够用，越过之后算术回绕，kernel 寻址到错误的行，而且不报任何错。AITER 为 4GB 以上的批 prefill 加了运行时 64 位分发[\[59\]](https://github.com/ROCm/aiter/pull/2893)，为 2GB 以上的 MLA 偏移加了 64 位[\[60\]](https://github.com/ROCm/aiter/pull/4474)，并在 DeepSeek-V4 统一缓存路径上全面转 64 位寻址，后者防止的是在约 1.5 亿行的池子里静默读写到错误的行[\[61\]](https://github.com/ROCm/aiter/pull/4680)。

Mooncake 的 AMD 支持此前卡在两个地方：RDMA 注册和可安装的包。NVIDIA 上给 GPU 显存注册 RDMA 要么用 nvidia-peermem 内核模块，要么导出 dmabuf 文件描述符；AMD 没有 nvidia-peermem 的对等物，于是 GPU-direct RDMA 根本没有路径，部署只能把 KV 经宿主 DRAM 中转。HIP dmabuf 注册分支补上了 CUDA dmabuf 路径的镜像，经 ROCm 导出，并先解析出真正的分配基址（缓存分配器会把张量打包在一块更大分配的偏移处）[\[82\]](https://github.com/kvcache-ai/Mooncake/pull/2225)。另一半是”装不上的支持不算支持”：Mooncake 此前发布 CUDA 和 MUSA wheel 但没有 ROCm 包，AMD 用户只能在每个镜像里从源码构建。ROCm wheel、CI 和发布路径把 `mooncake-transfer-engine-rocm` 发到 PyPI，验证覆盖了 MI300X × MI355X 与 `vllm/vllm-openai-rocm` × `lmsysorg/sglang` 的完整叉乘，Python 3.10 到 3.13[\[83\]](https://github.com/kvcache-ai/Mooncake/pull/3184)。LMCache 走了同一条路：预构建的 gfx942/gfx950 wheel 装进上游镜像，在 MI350X 上通过全部 56 个 KV 传输 kernel 测试，并且发布到 GitHub release 而非 PyPI，好让 `pip install lmcache` 仍然拿到 CUDA 构建[\[79\]](https://github.com/LMCache/LMCache/pull/4273)。

## 十、70+ PR 之三：让路由认得路，让前端跟得上

当一个请求不携带可复用历史（比如一个子代理刚起步），随便哪个 worker 都行，唯一值得问的是负载均衡。当请求携带着上兆字节的缓存前缀，把它发给一个空闲但不持有该前缀的 worker 就是昂贵的选择。SGLang 加了 DP 缓存亲和，让会话粘在持有其缓存的 rank 上，并且在同一个 PR 里同时实现了 DP 感知的 prefill 与 decode 路由，让分离式部署的两半做出一致决策；再把缓存均衡度作为路由信号，避免亲和退化成一个热点 worker[\[53\]](https://github.com/sgl-project/sglang/pull/26245)。路由器只能依据别人告诉它的信息行动，所以混合缓存事件也变成了 radix 感知和滑动窗口感知的。

ATOM 的路由器 ATOMesh 是 SGLang router 的一个删掉大部分功能的分叉，结果发现它需要 SGLang 的缓存感知路由，只好把这个功能加回去。ATOM 随后获得了面向缓存感知路由器的 KV 生命周期事件、多节点 prefill 与 decode 路由，以及会话粘性的数据并行路由。粘性策略是一个值得说清楚的双向妥协：会话回到持有其状态的健康 worker，但闲置的分配会过期，免得集群为已经离开的会话永久失衡。

Dynamo 的系列改动说明了一件事：**引擎 kernel 变快之后，分布式服务层自己会变成瓶颈。** 路由器的工作量正比于活跃前缀的数量与长度，而不是生成的 token 数，因此”许多长的、重叠的、长寿的会话”这种负载对它的压力，是固定形状流量永远制造不出来的。第一批 PR 降低每次路由决策的成本：查找热路径减负、去掉冗余的后缀失效、最后把 KV 匹配、注册、所有权与终态解引用批量化，在并发 512 下报告了 22.2% 的中位输出吞吐提升[\[65\]](https://github.com/ai-dynamo/dynamo/pull/11095)。

第二批改动动的是底下更难的问题——所有权怎么表示。每个缓存块都要归属到依赖它的请求上，既不能在还有人用时被释放，也不能在所有人用完后仍被钉住；上千并发会话共享重叠前缀时，记账本身就成了显著开销。Dynamo 从共享块链走到 arena 级所有权计数，最后走到后端专属的请求租约，每一步都把被追踪的单位变粗。租约设计把 AgentX 回放时间在 vLLM 后端上降低 23.7%、SGLang 后端上降低 22.0%，同时降低了峰值内存——同时降内存是个信号，说明问题出在此前的表示方式而非流量本身[\[66\]](https://github.com/ai-dynamo/dynamo/pull/12329)。

后续的路由器剖析删掉了一批形状相同的成本：周期性全扫描或全量重算，之所以此前可以接受，只是因为活跃状态一直很小。分桶过期清理取代了正比于全部被追踪对象的扫描，高频变动下的 AgentX 吞吐提高 13.7%[\[67\]](https://github.com/ai-dynamo/dynamo/pull/10521)；只处理变化量的增量后缀清理在同一时间窗内吸收了约 28 倍的存取事件[\[68\]](https://github.com/ai-dynamo/dynamo/pull/10676)；压缩提示路径把前端 CPU 削掉 35.3%，并显著改善尾部 TTFT——这在提示又长又高度重复的负载上尤其有意义[\[69\]](https://github.com/ai-dynamo/dynamo/pull/11644)。

有一个路由改动是明确的取舍而非纯粹的胜利：Dynamo 现在可以把活跃 decode 请求计入路由评分，让一个已经承诺了大量长时 decode 的 worker 看起来比它的队列深度所显示的更贵。这在报告的调参点上改善了 AgentX 的中位延迟，代价是少量吞吐[\[70\]](https://github.com/ai-dynamo/dynamo/pull/12158)。还开着的后续 PR 把这个取舍打包成一个 agentic 路由预设，往同一方向推得更远：前缀重叠权重记 2、prefill 负载放大 4、活跃 decode 请求权重记 64。在这个调参点上取舍不再牺牲吞吐——8xH200 的 AgentX 运行中，该预设相对默认代价函数把固定窗口完成输出吞吐提高 8.26%，运行级 p95 TTFT 降低 43.1%、p95 token 间延迟降低 22.6%，并多跑完一条完整轨迹[\[71\]](https://github.com/ai-dynamo/dynamo/pull/13447)。

请求面随后也被优化，因为一条 agentic trace 不是一次请求一次响应：它发出许多携带几乎相同提示的相关请求，并把每个 token 作为独立帧流回，序列化和拷贝因此按轮、按 token 反复付费。换成 MessagePack 请求载荷让吞吐提高 8.1%、平均 TTFT 降低 9.7%[\[72\]](https://github.com/ai-dynamo/dynamo/pull/10437)。之后是一串”去掉一次拷贝而不是加快一次拷贝”的改动：不拷贝 MessagePack 事件载荷、不拷贝收到的 ZeroMQ 帧、不为每个 token 支付完整的 token 间延迟指标开销。单看每一个都平平无奇，乘以每个并发会话的每个流式 token，它们决定了前端每秒能撑住多少请求。

高并发剖析还挖出了与数据搬运无关的成本：静态日志过滤器移除了一个共享的 span 匹配锁——这是争用问题而非流量问题——把前端吞吐从 932 提到 1133 请求每秒[\[73\]](https://github.com/ai-dynamo/dynamo/pull/11820)。一个还开着的改动让去 token 化指标每个响应刷新一次，而不是每个流式块都更新累计计数器，在匹配的诊断剖析中把前端 CPU 时间大约减半[\[74\]](https://github.com/ai-dynamo/dynamo/pull/12999)。**这是这一类里最清楚的例子：那段仪表代码单次调用很便宜，按每 token 一次调用就是灾难。**

TensorRT-LLM 的增量 token 化属于同一大类，只是发生在最前端。每一轮对话都重发全部历史再加一点，朴素实现会把全部重新 token 化——按千字节算 token 化很便宜，但同样的 10 万 token 每轮重算一遍就很致命。显而易见的修法（只 token 化新后缀）有一个容易忽略的错误：字节对编码不是位置无关的，token 可以跨接缝合并，把文本在边界切开再拼接两段 token 序列，可能产生与整串 token 化不同的结果，从而与前缀缓存所依据的序列悄悄分叉。边界感知的增量 token 化的做法是找到渲染文本的公共前缀、回滚一个完整 token 以便重算任何跨接缝的合并，再从那里只 token 化改变的后缀。在 Qwen3.5 的 AgentX trace 上，它在全部 1087 次转换中与完整 token 化结果一致——正确性主张是测出来的而不是假设的——平均处理时间从 185.1 毫秒降到 11.3 毫秒[\[41\]](https://github.com/NVIDIA/TensorRT-LLM/pull/17462)。

这条和前期调研里的一条判断精确对上了：agent 的长上下文小增量与高前缀命中会造成 CPU/GPU 工作量反转——无状态 tokenizer 每轮仍扫描完整上下文 O(N)，GPU prefill 只处理新增的 O(ΔN)，有状态的边界修复可以把 token 化降到 O(ΔN+w)。TRT-LLM 的这个 PR 就是那条判断的工程实现，而 16.4 倍的实测加速给它补上了量级。需要提醒的是，这个 PR 目前仍是开放状态，不应被当作稳定 release API。

## 十一、kernel 与调度层的长尾修复

还有一批改动既不在缓存层也不在传输层，而是被真实负载的形状逼出来的。

变长是第一个来源。 AgentX 的真实会话上下文长度连续变化，与生产流量的模式一致。一个按长度做特化的运行时，几乎会为见到的每个请求编译一个新 kernel。SGLang 维护者把上下文长度改成运行时标量，把这些编译折叠成一次，AgentX 并发 384 的输出吞吐提高 26.75%、平均 TTFT 改善 36.25%——收益来自去掉编译，而不是任何计算变快了[\[34\]](https://github.com/sgl-project/sglang/pull/30255)。同样精神下，去掉每步一次的设备到主机序列长度同步，消除了一个纯粹因为主机想知道设备已知长度而存在的 decode 气泡[\[35\]](https://github.com/sgl-project/sglang/pull/30365)。

变长也咬到 kernel 内部。GB300 上混合上下文的 decode 批次要付一笔尾部税：一次匹配剖析把 9.8 毫秒 decode 步差额中的 8.4 毫秒归给注意力，最长的那些请求拖长了持久 kernel 的共享 wave。开着的工作把 TRT-LLM MHA decode 批次按 KV 长度排序分组，让短请求不再等最长的那个[\[84\]](https://github.com/sgl-project/sglang/pull/34888)。

调度层的饥饿是第二个来源。 DP attention 下每个 rank 都加入同一个 MoE 集合通信，却在本地调度注意力工作；一个持续收到分块 prefill 续传的 rank 会不断赢下”prefill 优先”的决策，而同侪 rank 上正在运行的批次只能干等。可配置的 prefill 后 decode 间隔强制在 prefill 之间插入 decode 轮次；在 AgentX DeepSeek V4 Pro 上，输出吞吐上升 141%、p99 token 间延迟下降 97.3%，代价是中位 TTFT 从 36.5 秒升到 59 秒[\[85\]](https://github.com/sgl-project/sglang/pull/35017)。这是一次明码标价的交换：用首 token 等待换流的平滑度。

kernel 选择是第三个来源，也是错误最隐蔽的地方。 MiniMax-M3 给 MXFP8 自动调优加入 CuTeDSL 候选，扩大候选集，低并发聚合点上每 GPU 输出吞吐提高约 7% 到 10%[\[86\]](https://github.com/NVIDIA/TensorRT-LLM/pull/17316)。反方向的例子更有教育意义：TensorRT-LLM 在损坏的 split-K MoE 策略让 7 次 AgentX 运行崩掉 5 次之后，把它从候选池里禁掉，之后 7 次匹配运行零崩溃[\[87\]](https://github.com/NVIDIA/TensorRT-LLM/pull/17105)。**一个又快又错的策略比一个仅仅是慢的策略更糟，而自动调优器会热情地选中它，除非把它从池子里拿掉。** 选择之外，kernel 还能以另一种方式出错：M3 面向短 query 的旧稀疏注意力路径可能把一个非连续的 head-major 块索引视图交给 SM100 kernel，而 kernel 按连续处理，于是选中错误的 KV 页，产生错误或非有限的输出。为 q\_len ≤ 32 尊重块索引 stride 修好了索引，不需要物化张量也不需要新增 kernel，之后 GB300 上五对匹配的完整 AgentX 运行零服务错误、无非有限标记[\[88\]](https://github.com/NVIDIA/TensorRT-LLM/pull/17285)。

生命周期 bug 是第四个来源，也是长跑而非大跑的特征失败。 序列槽位余量与一致的按槽索引缓冲区尺寸，处理的是”一个正在完成的请求和一个刚被准入的请求同时需要槽位”这个瞬时重叠——稳定的到达与离开流会不断撞上它，固定批次一次都撞不上[\[89\]](https://github.com/NVIDIA/TensorRT-LLM/pull/16279)。后来的注意力数据并行 dummy 请求修复让 9 个 Qwen3.5 分离式单元活了下来，而此前多数单元几分钟内就失败[\[90\]](https://github.com/NVIDIA/TensorRT-LLM/pull/17278)。这是”能跑完 benchmark 的配置”和”能活过一个会话的配置”之间的区别。

并行度的稀缺也出现在单张 GPU 内部：batch-1 的 MLA decode 没有 head 维度也没有 query 维度可分，只有 KV 遍历，而硬编码的 16 路切分预算让这次遍历只跑在 gfx950 的 256 个 CU 中的 16 个上。一个已合并的改动停止覆盖 kernel 自己的切分推导，让 AITER 按机器实际的 cluster 数切分遍历[\[58\]](https://github.com/ROCm/ATOM/pull/1911)。分块流水并行 prefill 从内存一侧攻同一个问题，用流式的层级交接替代反复的张量并行集合通信，它在 GLM-5.2 高负载下的结果是这一节里最完整的一组：输出吞吐翻倍、中位 TTFT 从 28.6 秒降到 8.7 秒、每张 prefill GPU 持有的 KV 块数增加到 3.68 倍[\[57\]](https://github.com/ROCm/ATOM/pull/1552)。最后那个数字应该先读——**每张 prefill GPU 的容量决定了在撞上 HBM 悬崖之前，能有多少条长会话同时在飞。**

引擎层的并行策略只有在 kernel 能表达它时才是真的，所以 AITER 侧有一组对应工作。Prefill 上下文并行的进程组提供了 PCP 需要的额外 query 分片维度，同时把融合 kernel 的行索引拓宽到 13.1 万 token 以上的提示[\[62\]](https://github.com/ROCm/aiter/pull/3728)；DCP 把 KV 切到已经存在的张量并行 GPU 上，让更长的序列或更大的批次不必在每个 rank 复制整份缓存就能装下[\[63\]](https://github.com/ROCm/aiter/pull/3267)。DeepSeek-V4 的解码还拿到了面向 64 head 与 128 head MTP 打包的持久 MLA kernel——这两个 head 数正是常规解码和投机验证实际产生的形状，等于给引擎的常见形状配了专用长上下文路径，而不是把它们当成一个为短上下文写的 kernel 的偶然变体[\[64\]](https://github.com/ROCm/aiter/pull/3459)。在长上下文里，通用路径谈不上折中，它就是错的 kernel。

ATOM 的 PCP 报告了 35% 到 43% 更低的平均 TTFT，在 64000-token 输入上总吞吐增益最高约 49%——这个收益随输入长度增长而非随批大小增长[\[55\]](https://github.com/ROCm/ATOM/pull/1220)。要让它在实践中可用，还必须和会话依赖的一切共存，所以 DCP 被做成与前缀缓存、分块 prefill 和 FP8 KV 兼容，然后又扩展到 MTP[\[56\]](https://github.com/ROCm/ATOM/pull/1701)。一种无法与前缀缓存共存的并行方式，只是拿一个长上下文的胜利换掉另一个。

## 十二、AMD 的差距在哪里，以及 MiniMax M3 的第 0 天

把上面所有线索汇总，AMD 的问题画像相当清晰，而且几乎没有一条是关于芯片本身的。

第一条是 runtime API 的空洞。`hipMemcpyBatchAsync` 缺失到 ROCm 7.14，直接导致 vLLM 原生 CPU 卸载在 AMD 上只能串行 memcpy，进而导致 AMD 在所有模型上都比 NVIDIA 更少使用 KV 卸载，进而在需要卸载的高并发段失去竞争力。这是一条完整的因果链，起点是一个函数。

第二条是并行形态的缺席。DCP/PCP 在 vLLM 的 AMD 后端上全线不支持，长上下文缺一整类并行策略。

第三条是分发路径。Mooncake 没有 ROCm wheel、LMCache 没有 gfx942/gfx950 预构建 wheel 之前，AMD 用户要在每个镜像里从源码构建整个传输引擎和缓存层。这一条被 AMD 自己的工程师在用 AgentX 做 dogfooding 时发现并修掉——**装不上的支持不算支持。**

LMCache 的 ROCm 补完是这条链的另一半。CacheBlend 的非前缀复用依赖 flashinfer，而 flashinfer 只有 CUDA 版本，于是一个 Triton 块稀疏注意力后端重新实现了它需要的三个 kernel——带 CSR 索引和 log-sum-exp 输出的块稀疏注意力、因果 prefill、log-sum-exp 输出融合——并在检测到 ROCm 或缺少 flashinfer 时自动路由过去[\[78\]](https://github.com/LMCache/LMCache/pull/3092)。这一条对我特别有意义：前期调研曾指出 CacheBlend 的非前缀复用在 vLLM V1 连接器路径上完全不发生，选型必须按具体连接器版本实测；现在这条路径至少在 AMD 上从”没有实现”变成了”有实现待验证”。

另外两个 LMCache 改动展示了长跑负载特有的 bug 形态。DCP 感知的 CPU 卸载解决的是两个长上下文场景下必须同时开启的特性之间的直接冲突：开了解码上下文并行之后每个 rank 只持有 KV 的一条 stride，任何单个 rank 能存下的都不是一个可用前缀；修复的做法是保存前先聚合分片、加载后再重新分发。不修的后果是启用上下文并行会静默关掉恰恰是这两个特性共同目标的那些长前缀的 CPU 缓存命中——它的验证记录了超过三万次 CPU 命中事件，单请求加载达到几十万 token[\[80\]](https://github.com/LMCache/LMCache/pull/3561)。混合锁账务修复则阻止一个请求释放另一个请求在共享滑动窗口或循环状态块上的读锁：多个请求必须共享同一批块、账务必须按块而非按持有者、驱逐必须真的开始触发，三个条件同时具备时才会暴露。带 DRAM 卸载的持续 Kimi-K3 运行同时满足这三条，产生了数以万计的告警、损坏的生成结果，并在驱逐开始后最终导致 GPU 崩溃[\[81\]](https://github.com/LMCache/LMCache/pull/4524)。**任何不够长、不够共享、不够内存吃紧的运行，都不会把这个 bug 唤醒。**

第四条是激励与优先级。报告直言 AMD 工程领导层的激励指向短上下文单轮调优，长上下文多轮被系统性忽略；同时 ATOM 的优化没有稳定上游化到 vLLM，导致”AMD 最强引擎”和”客户实际会用的引擎”是两个东西。

这几条一起解释了为什么报告的结论既是”CUDA 护城河成立”，又是”这条护城河是工程债”。MiniMax M3 的第 0 天是这个判断的正面验证：按 SemiAnalysis 在 AMD Advancing AI 那篇里的对照[\[91\]](https://newsletter.semianalysis.com/p/can-amd-break-the-cuda-moat-amd-advancing)，AMD 首个公开的分离式配方（MI355X FP4）在一月份进入 InferenceX，比 NVIDIA 晚了数月；而 M3 FP4 的分离式在第 0 天就落地了，相比 DeepSeek-R1 时期需要数月才达到对等，这是明显的改善。

有三个 vLLM 修复卡在那条第 0 天路径上，而且每一个都是正确性失败而非性能问题，值得单独看，因为它们展示了”异构支持”在实践中的真实颗粒度：

NixlConnector 的握手断言 SPLIT 区域的 `block_len` 随 prefill 到 decode 的 TP 比例缩放，但 `block_len` 实际跟随每 rank 的 KV head 数。M3 只有 4 个 KV head，所以 TP4 prefill 配 TP8 decode 时两侧都被 GQA 上限压到每 rank 一个 head，两个长度相等，而断言要求相差 2 倍。握手被拒，没有 KV 移动，decode 从零重算一切，gsm8k 得分为 0[\[31\]](https://github.com/vllm-project/vllm/pull/45879)。

另外两个是平台分叉。M3 的稀疏注意力后端把字节支持的 FP8 缓存按 `float8_e4m3fn` 读，对所有 E4M3 配置都如此；但 gfx942 的平台 dtype 是 `e4m3fnuz`，两种编码并不相同，K 和 V 在被 kernel 消费之前就已经被改变了，而 prefill 与 decode 的包装器又在 FP8 检查里漏掉了 FNUZ 类型。改用平台 dtype 做缓存视图同时修好两半[\[32\]](https://github.com/vllm-project/vllm/pull/45720)。此外 M3 以独立的 NVIDIA 与 AMD 模型文件发布，只有 NVIDIA 那份实现了 EAGLE3 接口，于是 ROCm 上投机解码在引擎初始化时就以”模型不支持”中止；把 AMD 模型补齐到对等后恢复，MI355X 的 gsm8k 与非 EAGLE3 的 MI355X 运行和 B200 都能对上[\[33\]](https://github.com/vllm-project/vllm/pull/45546)。

这三个 bug 的症状都是”AMD 上跑出来的是错的”，与快慢无关。它们也解释了为什么前期调研里那条”设备能力向量必须超出规格表——算子、精度、runtime-role 与软件栈版本共同决定可部署性”的判断，在 agentic 场景下要求更严格：同一个 FP8 名字下的两种编码、同一个模型的两份文件、一个直到 ROCm 7.14 才出现的批量拷贝 API，都是规格表上看不见的能力向量分量。

## 十三、指标的困境：为什么端到端延迟已经不好用了

报告顺带处理了一个方法论问题。端到端延迟在当前形态下变得意义有限，因为它与 OSL 直接成正比，p90 e2e 延迟被输出序列长度最长的那 10% 尾巴主导。它仍可用于整体比较某些配置，但报告建议改看 TPS 与 TTFT 的组合。

他们提出了一个实验性指标 E2E Normalized Interactivity，定义为 OSL/E2EL。把 E2EL 展开为 TTFT 加 OSL 乘 TPOT，这个指标实质上是交互度（1/TPOT 部分）加上一个正比于 TTFT 的惩罚项。报告自己说得很清楚：这个指标是实验性的、并不完美，它对高 TTFT 惩罚过重，也没能捕捉 PD 分离这类优化的全部细微之处；AgentX v1.0 的所有提交仍然分别针对常规交互度和 TTFT 优化。

这个诚实的自我限定值得记下来，因为它对应着一个真实的空缺：agentic 场景里用户往往更关心”任务多久做完”，而不是”token 多快吐出来”或”首 token 多快到达”。目前还没有一个被广泛接受的指标能表达这件事。

前端的可视化改动也跟着这个思路走。曲线的构造方式变了：以往投机解码开与关会画成不同曲线，现在前端把允许的优化组合起来，为每个（模型, SKU, 推理引擎）组合展示最佳可用曲线，同一条曲线上不同的点可能用了不同的优化技术（投机解码、分离式、KV 卸载）。点击任一点可以看到具体配置、运行元数据、公开可查的 CI 溯源和 AgentX 专属统计；使用 KV 卸载的点在主图上多一圈虚线，选中后可看卸载类型、卸载引擎、芯片缓存命中率和 CPU 缓存命中率。还有一个请求时间线视图，可按会话或按 worker 组织，会话视图把子代理归在其根会话之下；以及一个会话火焰图，每根条代表一轮，按该会话内最大轮次缩放，条内切分为缓存前缀 token、未缓存输入 token 和生成输出 token。

**当一个 Pareto 点背后是数千条跨越增长中会话、子代理、预热与缓存状态的请求，只给一个坐标是不够的。** 把点击一个点到看见具体那条匿名请求的路径打通，是这套前端在方法论上的实质贡献。同一套 trace 另有一个独立的开源探索工具，把这些会话按 token、轮次和时序拆开来看[\[93\]](https://github.com/Semianalysis/agentx-conversation-explorer)。

## 十四、这份基准自己的边界

一篇阅读笔记如果只复述结论就不完整。以下是我认为读者在引用 AgentX 数据时应当同时记住的限制，其中多数是报告自己承认的。**这份基准最可靠的用法是比较同一天、同一场景下的不同配置，最不可靠的用法是把某个绝对数字搬到另一个 harness、另一个月份或另一段曲线上。**

流量形状而非流量内容。 前面已经展开：加密的思维链、看不见的服务端 chat 模板、专有 tokenizer、服务端工具引入的上下文、图片与文档的 token 对应关系，都只能用确定性占位符和经验标定的 padding 逼近。

harness 依赖。 请求分布随 harness 而变——报告点名 Pi 在注入上下文上极简，Claude Code 相反；ISL/OSL 分布也随模型的 tokenizer 而变。他们的辩护是全球相当大一部分 agentic 编码流量走 Claude Code，因此这份数据具有代表性。这个辩护成立，但它同时意味着 AgentX 度量的是”Claude Code 形状的流量”，而不是 agentic 流量的全体。

闭环基准的自然方差。 不同配置在一小时内完成的请求数不同、速率不同，因此遭遇的负载分布本身就有差异。这在低并发下最明显——完成的请求少，负载收敛到数据集整体分布的机会也少。同一配置、同一并发、同一服务器设置下可复现，跨配置比较则天然带一层噪声。

存储层不完整。 NVMe/SSD 卸载被推迟到后续版本，这意味着 Pareto 曲线左侧的高吞吐区间尚未被完整探索。允许的优化策略把 CPU KV 卸载列为可选：厂商可以用 vLLM connector、LMCache、SGLang HiCache、Mooncake、Dynamo KVBM 或其他 CPU DRAM connector，也可以在关闭卸载能得到更好的延迟/吞吐点时关掉它。CPU DRAM 必须按使用的 GPU 比例缩放，非标准化 DRAM 系统有 3TB 上限，标准化 DRAM 系统无硬上限但保留同样的比例规则——而本地生成器目前对所有 runner 一律套用 3TB 上限，标准化 DRAM 的例外还没实现。

PR 计数口径不一。 报告开头说 70+ 上游 PR，”AgentX Industry Impact”一节的开头写的是 50+。两个数字都出自同一篇文章，取哪个都不影响结论，但引用时最好直接引 PR 列表本身。我对文中出现的四十余个 PR 链接做了逐条核对，仓库、编号与标题描述全部对得上；其中若干条仍处于开放状态（TRT-LLM 的增量 token 化与有界块 ID 检索、vLLM 的 MoRI-IO 循环状态传输与 AITER gluon 稀疏 MLA、Dynamo 的 agentic 路由预设、LMCache 的 DCP 感知卸载与锁账务修复、Mooncake 的 MI350X 双节点 CI、SGLang 的多条 SWA 与 HiSparse 工作），引用它们的数字时应当标明这一点。

时效性。 这是一条随上游移动的曲线。8 月 21 日一批 vLLM 优化就让 B200 与 MI355X 的每美元性能排序反转。报告自己预告三到四周后会发更新文章。任何”X 比 Y 快”的引用都必须带上日期。

## 十五、结论

回到开篇的问题。CUDA 护城河在 agentic 推理下仍然成立，但它已经不是”kernel 快多少”这件事。**当高前缀复用把绝大部分输入变成缓存查找，竞争就从算子性能转移到了会话状态的保存、搬运、路由、重建和重复处理这条完整路径上，而这条路径上的每一环都是软件。** GLM 5.3 上”白送芯片也追不上”的那个极端结论，和 Kimi K3 上”B200 因为投机解码与流水并行无法组合而被反超”的例子，是同一枚硬币的两面——两边都不是算力的胜负。

这个判断的适用边界要说清楚。它在 AgentX 度量的这类负载上成立：Claude Code 形状的编码 agent、中位 142k 输入、中位 444 输出、约 44% 的会话带子代理、3.84 秒的中位工具执行时间。它在固定序列长度的单轮负载上不成立——那里测的仍然主要是芯片和 kernel，MI355X 在若干固定序列配置上是领先的。它在时间上也有边界：整篇报告是 2026 年 8 月 21 日前后的快照，而这条曲线每周都在动。

对我自己关心的问题，AgentX 补上了一个持续缺席的东西。此前的判断是，截至 2026 年 5 月，所有 agentic 推理系统的性能评估都在用私有或自建 trace，不存在可比的标准化 agentic 负载发生器，这与 ShareGPT / LMSYS-Chat-1M 之于 chat 负载的地位形成鲜明对比。AgentX 用 Apache 2.0 的数据集、可消费的 REST API、公开的 CI 溯源和上游镜像配置，把这个缺口填上了。同样被填上的还有另一条：agent benchmark 报告成功率和步数，KV cache benchmark 报告命中率和加载延迟，两者此前不在同一张表上；AgentX 的点详情页把输入输出序列长度分布、交互度与 TTFT 随时间变化、KV 缓存利用率、请求队列深度、前缀命中率、提示 token 来源拆分放在了一起。

还开着的问题也不少。E2E Normalized Interactivity 承认自己不完美，而”任务多久做完”这个 agentic 场景下最自然的指标仍然没有公认表达。NVMe 层缺席意味着高吞吐段的结论待定。v1.1 计划保留系统指令、用户与助手消息、工具调用与工具结果之间的边界，这会让”负载感知服务”第一次可评测——路由器可以把低复用的工具流量导向专用 prefill worker、在长工具调用期间保住 agent 的前缀、在子代理 fork 时预取并共享前缀。当前格式保住了请求尺寸和 KV 复用模式，但缺少评估这些策略所需的结构。

这三到四周后的更新文章，会是检验”工程债正在被偿还”这个判断的第一个节点。

- - -

## 参考资料

\[1\] [AgentX - InferenceXv3: Does CUDA Moat Hold up in Agentic Inferencing?](https://newsletter.semianalysis.com/p/agentx-inferencexv3-does-cuda-moat)

\[2\] [InferenceX：开源持续推理标准与研究平台（Apache 2.0）](https://github.com/SemiAnalysisAI/InferenceX)

\[3\] [InferenceX AgentX 公开仪表盘](https://inferencex.semianalysis.com/agentx)

\[4\] [AgentX 数据集：cc-traces-weka-062126（393 会话，1M 上下文）](https://huggingface.co/datasets/semianalysisai/cc-traces-weka-062126)

\[5\] [AgentX 数据集：cc-traces-weka-062126-256k（截断版）](https://huggingface.co/datasets/semianalysisai/cc-traces-weka-062126-256k)

\[6\] [AIPerf：NVIDIA 的厂商中立 HTTP 回放工具](https://github.com/ai-dynamo/aiperf)

\[7\] [SemiAnalysisAI/aiperf：AgentX 使用的中立 fork](https://github.com/SemiAnalysisAI/aiperf)

\[8\] [Qwen-Bailian 匿名生产 trace 数据集](https://github.com/alibaba-edu/qwen-bailian-usagetraces-anon)

\[9\] [Claude Code issue #49207：AgentX 数据采集所需特性](https://github.com/anthropics/claude-code/issues/49207)

\[10\] [Claude Code issue #66761：AgentX 数据采集所需特性](https://github.com/anthropics/claude-code/issues/66761)

\[11\] [Anthropic Prompt Caching 文档：默认 5 分钟缓存生命周期](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)

\[12\] [SPEED-Bench：投机解码接受长度评测数据集](https://huggingface.co/datasets/nvidia/SPEED-Bench)

\[13\] [kv-cache-tester：WEKA trace 格式的来源项目](https://github.com/callanjfox/kv-cache-tester)

\[14\] [vLLM 注意力后端支持矩阵（DCP 支持情况）](https://docs.vllm.ai/en/latest/design/attention_backends/)

\[15\] [llm-d 架构文档（Apache 2.0，CNCF Sandbox）](https://llm-d.ai/docs/architecture)

\[16\] [vLLM Router 一致性哈希路由策略源码](https://github.com/vllm-project/router/blob/main/src/policies/consistent_hash.rs)

\[17\] [recipes.vllm.ai：AgentX 跟随的 vLLM 上游配方](http://recipes.vllm.ai/)

\[18\] [SGLang Cookbook：AgentX 跟随的 SGLang 上游配方](https://docs.sglang.io/cookbook/intro)

\[19\] [SGLang server\_args：–enable-prefill-delayer 定义](https://github.com/sgl-project/sglang/blob/v0.5.17/python/sglang/srt/server_args.py#L3180-L3194)

\[20\] [vLLM #43447：DeepSeek V4 滑动窗口选择性前缀缓存保留](https://github.com/vllm-project/vllm/pull/43447)

\[21\] [vLLM #42258：跳过永远无法命中前缀缓存的 SWA 块](https://github.com/vllm-project/vllm/pull/42258)

\[22\] [vLLM #45444：Mooncake 跳过不可达 SWA 块的 KV 查询](https://github.com/vllm-project/vllm/pull/45444)

\[23\] [vLLM #44082：在 SWA 前缀缓存掩码中保留 EAGLE/MTP 前瞻块](https://github.com/vllm-project/vllm/pull/44082)

\[24\] [vLLM #37160：通用 CPU KV 缓存卸载连接器](https://github.com/vllm-project/vllm/pull/37160)

\[25\] [vLLM #42296：SimpleCPUOffloadBackend 支持 DeepSeek V4](https://github.com/vllm-project/vllm/pull/42296)

\[26\] [vLLM #40549：在 ROCm 上启用 SimpleCPUOffloadConnector](https://github.com/vllm-project/vllm/pull/40549)

\[27\] [vLLM #41289：跨调度步去重在途 CPU 卸载存储](https://github.com/vllm-project/vllm/pull/41289)

\[28\] [vLLM #46412：Mooncake 只检查并存储新增 KV 区间](https://github.com/vllm-project/vllm/pull/46412)

\[29\] [vLLM #46906：将存储保留与 HBM 保留解耦（开放）](https://github.com/vllm-project/vllm/pull/46906)

\[30\] [vLLM #45659：Mooncake 异步查找以降低调度器开销](https://github.com/vllm-project/vllm/pull/45659)

\[31\] [vLLM #45879：修复 GQA 复制 KV head 下的 NixlConnector 握手校验](https://github.com/vllm-project/vllm/pull/45879)

\[32\] [vLLM #45720：修复 ROCm 上 MiniMax-M3 的 FP8 KV 缓存 dtype](https://github.com/vllm-project/vllm/pull/45720)

\[33\] [vLLM #45546：为 AMD MiniMax M3 实现 EAGLE3 支持](https://github.com/vllm-project/vllm/pull/45546)

\[34\] [SGLang #30255：消除 DSV4 prefill 跨上下文长度的 Triton 重编译](https://github.com/sgl-project/sglang/pull/30255)

\[35\] [SGLang #30365：移除投机路径每步的 seqlen 设备到主机同步](https://github.com/sgl-project/sglang/pull/30365)

\[36\] [SGLang #26907：驱逐时保留 SWA 滑动窗口后缀](https://github.com/sgl-project/sglang/pull/26907)

\[37\] [SGLang #27210：将 SWA 计算锁限制在单个窗口（开放）](https://github.com/sgl-project/sglang/pull/27210)

\[38\] [SGLang #29369：清理 SWA tombstone 叶子的完整 KV（开放）](https://github.com/sgl-project/sglang/pull/29369)

\[39\] [SGLang #34565：统一树支持 SWA 分量的分叉点缓存](https://github.com/sgl-project/sglang/pull/34565)

\[40\] [SGLang #29735：FlashInfer GDN prefill 与额外缓冲 radix 缓存](https://github.com/sgl-project/sglang/pull/29735)

\[41\] [TensorRT-LLM #17462：边界感知的增量路由器 token 化（开放）](https://github.com/NVIDIA/TensorRT-LLM/pull/17462)

\[42\] [TensorRT-LLM #17518：修正缓存映射与分块 NIXL 路径优化 M3 KV 传输](https://github.com/NVIDIA/TensorRT-LLM/pull/17518)

\[43\] [SGLang #30545：分离式暂存缓冲区支持 radix 缓存](https://github.com/sgl-project/sglang/pull/30545)

\[44\] [SGLang #35070：避免 PD 分离中无用的 PREBUILT 提示张量传输](https://github.com/sgl-project/sglang/pull/35070)

\[45\] [TensorRT-LLM #15727：分离式服务的流水化 KV 缓存传输](https://github.com/NVIDIA/TensorRT-LLM/pull/15727)

\[46\] [TensorRT-LLM #17526：流水化 KV 传输的有界块 ID 检索（开放）](https://github.com/NVIDIA/TensorRT-LLM/pull/17526)

\[47\] [TensorRT-LLM #17428：把非阻塞上下文传输轮询回移到 M3](https://github.com/NVIDIA/TensorRT-LLM/pull/17428)

\[48\] [TensorRT-LLM #16734：避免 DeepSeek-V4 上下文稀疏元数据的隐式设备标量同步](https://github.com/NVIDIA/TensorRT-LLM/pull/16734)

\[49\] [ATOM #1640：DeepSeek-V4 分页 SWA 稀疏检查点前缀缓存保留](https://github.com/ROCm/ATOM/pull/1640)

\[50\] [ATOM #902：延迟哈希注册，在空闲池命中时保住缓存](https://github.com/ROCm/ATOM/pull/902)

\[51\] [ATOM #1771：内容寻址的逐请求状态，让有状态模型的前缀命中正确](https://github.com/ROCm/ATOM/pull/1771)

\[52\] [ATOM #1318：独立 LMCache CPU/NVMe KV 卸载连接器](https://github.com/ROCm/ATOM/pull/1318)

\[53\] [SGLang #26245：支持 DP 感知的 PD 路由分发](https://github.com/sgl-project/sglang/pull/26245)

\[54\] [SGLang #28534：启用 JIT 分阶段 HiCache 写回并修复 CPU 索引崩溃](https://github.com/sgl-project/sglang/pull/28534)

\[55\] [ATOM #1220：为 DeepSeek-V4 启用 Prefill Context Parallel](https://github.com/ROCm/ATOM/pull/1220)

\[56\] [ATOM #1701：优化 MLA DCP（兼容前缀缓存 / 分块 prefill / FP8 KV）](https://github.com/ROCm/ATOM/pull/1701)

\[57\] [ATOM #1552：支持分块流水并行 + PD 分离 + ATOMesh](https://github.com/ROCm/ATOM/pull/1552)

\[58\] [ATOM #1911：解码 KV 切分预算取自机器而非硬编码 16](https://github.com/ROCm/ATOM/pull/1911)

\[59\] [AITER #2893：批 prefill 支持超过 4GB 的 KV 缓存](https://github.com/ROCm/aiter/pull/2893)

\[60\] [AITER #4474：修复 \_mla\_gluon 超过 2GB 路径的 int32 KV 偏移溢出](https://github.com/ROCm/aiter/pull/4474)

\[61\] [AITER #4680：停止把行号与块号压成 32 位地址](https://github.com/ROCm/aiter/pull/4680)

\[62\] [AITER #3728：启用 Prefill Context Parallel](https://github.com/ROCm/aiter/pull/3728)

\[63\] [AITER #3267：启用 Decode Context Parallel](https://github.com/ROCm/aiter/pull/3267)

\[64\] [AITER #3459：面向 MI35x 的第一代 64/128 head MLA 解码 kernel](https://github.com/ROCm/aiter/pull/3459)

\[65\] [Dynamo #11095：降低 KV 块记账开销](https://github.com/ai-dynamo/dynamo/pull/11095)

\[66\] [Dynamo #12329：把请求 KV 所有权移入后端租约](https://github.com/ai-dynamo/dynamo/pull/12329)

\[67\] [Dynamo #10521：分桶清理过期项](https://github.com/ai-dynamo/dynamo/pull/10521)

\[68\] [Dynamo #10676：避免删除时重复的后缀清理](https://github.com/ai-dynamo/dynamo/pull/10676)

\[69\] [Dynamo #11644：压缩块追踪器的提示路径](https://github.com/ai-dynamo/dynamo/pull/11644)

\[70\] [Dynamo #12158：路由器加入活跃请求解码代价](https://github.com/ai-dynamo/dynamo/pull/12158)

\[71\] [Dynamo #13447：新增 agentic 路由预设（开放）](https://github.com/ai-dynamo/dynamo/pull/13447)

\[72\] [Dynamo #10437：请求面 MessagePack 载荷编解码](https://github.com/ai-dynamo/dynamo/pull/10437)

\[73\] [Dynamo #11820：避免动态日志过滤器的 span 锁](https://github.com/ai-dynamo/dynamo/pull/11820)

\[74\] [Dynamo #12999：每个响应只刷新一次去 token 化指标](https://github.com/ai-dynamo/dynamo/pull/12999)

\[75\] [LMCache #3382：分块 KV 加载修复高并发下的 GPU 块耗尽死锁](https://github.com/LMCache/LMCache/pull/3382)

\[76\] [LMCache #3635：优化 DSV4 存取尺寸](https://github.com/LMCache/LMCache/pull/3635)

\[77\] [LMCache #3869：只预取并加载滑动窗口内的 KV 缓存](https://github.com/LMCache/LMCache/pull/3869)

\[78\] [LMCache #3092：为 CacheBlend 添加 ROCm Triton 块稀疏注意力后端](https://github.com/LMCache/LMCache/pull/3092)

\[79\] [LMCache #4273：构建并发布预编译 gfx942/gfx950 wheel](https://github.com/LMCache/LMCache/pull/4273)

\[80\] [LMCache #3561：vLLM v1 连接器的 DCP 感知 CPU KV 卸载（开放）](https://github.com/LMCache/LMCache/pull/3561)

\[81\] [LMCache #4524：修复混合 SW/线性注意力分组的锁账务（开放）](https://github.com/LMCache/LMCache/pull/4524)

\[82\] [Mooncake #2225：为 AMD GPU 添加 HIP dmabuf MR 注册](https://github.com/kvcache-ai/Mooncake/pull/2225)

\[83\] [Mooncake #3184：添加 ROCm/HIP wheel 构建、CI 与发布对等](https://github.com/kvcache-ai/Mooncake/pull/3184)

\[84\] [SGLang #34888：按 KV 序列长度切分 TRT-LLM MHA 解码批次](https://github.com/sgl-project/sglang/pull/34888)

\[85\] [SGLang #35017：可配置的 prefill 后解码间隔](https://github.com/sgl-project/sglang/pull/35017)

\[86\] [TensorRT-LLM #17316：为 MiniMax-M3 自动调优加入 CuTeDSL 候选](https://github.com/NVIDIA/TensorRT-LLM/pull/17316)

\[87\] [TensorRT-LLM #17105：跳过损坏的 splitK2 MoE kernel](https://github.com/NVIDIA/TensorRT-LLM/pull/17105)

\[88\] [TensorRT-LLM #17285：为 q\_len ≤ 32 尊重 MSA 块索引 stride](https://github.com/NVIDIA/TensorRT-LLM/pull/17285)

\[89\] [TensorRT-LLM #16279：序列槽位池重叠余量与一致的按槽缓冲区尺寸](https://github.com/NVIDIA/TensorRT-LLM/pull/16279)

\[90\] [TensorRT-LLM #17278：容忍 ADP dummy 请求盈余而非断言失败](https://github.com/NVIDIA/TensorRT-LLM/pull/17278)

\[91\] [Can AMD Break the CUDA Moat? AMD Advancing AI](https://newsletter.semianalysis.com/p/can-amd-break-the-cuda-moat-amd-advancing)

\[92\] [Claude Code is the Inflection Point](https://newsletter.semianalysis.com/p/claude-code-is-the-inflection-point)

\[93\] [AgentX Conversation Explorer：AgentX trace 的开源探索工具](https://github.com/Semianalysis/agentx-conversation-explorer)
