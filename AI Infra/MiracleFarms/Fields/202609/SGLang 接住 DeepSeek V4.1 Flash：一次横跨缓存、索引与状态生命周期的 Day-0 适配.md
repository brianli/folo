---
title: "SGLang 接住 DeepSeek V4.1 Flash：一次横跨缓存、索引与状态生命周期的 Day-0 适配"
source: "Fields"
category: "AI Infra"
group: "MiracleFarms"
url: "https://miraclefarms.github.io/notes/2026/09/10/sglang-deepseek-v4-1-flash-day0-code-audit/"
published: 2026-09-10T20:30:00+08:00
saved: 2026-09-30T14:23:38+08:00
folo_key: "mf-fields::https://miraclefarms.github.io/notes/2026/09/10/sglang-deepseek-v4-1-flash-day0-code-audit/"
updated: 2026-09-30T15:14:03+08:00
tags:
  - "folo"
  - "AI_Infra"
  - "Fields"
---

# SGLang 接住 DeepSeek V4.1 Flash：一次横跨缓存、索引与状态生命周期的 Day-0 适配

> [!info] Fields · AI Infra · 2026-09-10 20:30 · [原文](https://miraclefarms.github.io/notes/2026/09/10/sglang-deepseek-v4-1-flash-day0-code-audit/)

![[SGLang 接住 DeepSeek V4.1 Flash：一次横跨缓存、索引与-01.png]]

> **版本声明**：本文分析基于 SGLang PR #38798 head commit `7bdebdab7db4`（2026-09-10）；该 PR 截至审阅时仍开放，以下实现细节可能继续变化。

SGLang 为 DeepSeek V4.1 Flash 提交的 Day-0 支持有 **153 个文件、15,883 行新增和 1,350 行删除**。如果只看 `DeepseekV41Config` 和模型注册，这个改动很容易被误判成一次常规适配；把 diff 顺着数据流读完，结论恰好相反：**V4.1 把模型结构直接压进了 serving runtime 的缓存布局、请求历史、CUDA Graph 边界和投机验证提交语义。**

这次代码审阅围绕一个生产问题展开：一张 V4.1 权重能被 `sglang serve` 拉起来，只能证明接口接通；要让百万 token、前缀命中、多模态输入和 DSpark 同时工作，runtime 必须知道哪些状态可以跨层共享、哪些状态只能保留最近 128 个 token，以及哪些草稿 token 在验收前不能写入请求历史。

审阅边界也要先说清楚。本文做的是固定 commit 的静态源码审计，并用官方技术报告、模型配置与上游测试互相核对；我没有在 GB300、H200 或 MI350X 上重新加载 552B 模型，也没有把 cookbook 中间开发阶段的读数当成 benchmark。PR 当时缺少 `run-ci` 标签、核心 CI 尚未运行，合入状态仍是 `open / review required / conflicting`。所以文中的“已支持”指代码路径已经出现，**不等于稳定 release 已经交付。**[\[1\]](https://github.com/sgl-project/sglang/pull/38798)

## 一、153 个文件在改什么：从模型插件扩张为运行时状态机

初始 Day-0 commit `85f8105f5d2e` 单次就带来 15,142 行新增；当前 PR head 又叠加了 V4.1 视觉检查点的 DP attention 修订与 DSA top-k v2 长上下文 kernel 重构。按路径统计，45 个 kernel 文件贡献约 4,910 行新增，attention runtime 约 2,613 行，27 个测试文件约 2,407 行；Engram 单文件接近 1,000 行，模型与多模态路径还各自新增一整套实现。[\[2\]](https://github.com/sgl-project/sglang/commit/85f8105f5d2e2e0bdea70ff50cb5449bf9567053)

这些数字透露了适配的重心。配置层当然新增了 `deepseek_v41`，聊天入口也接入了连续 `reasoning_effort`、DSML 工具调用和交错图文消息；但大部分代码落在 attention、kernel、memory pool 和请求级状态上。官方模型配置给出的 40 层主干被切成 20 层 causal encoder 与 20 层 decoder，前两层只跑 SWA，encoder 的 CSA2 压缩率为 2，decoder 为 1；`kv_source_layer_ids=[2,8,14,20]` 决定谁产生共享 main KV，`index_source_layer_ids=[2,8,14,20,24,28,32,36]` 决定谁重新打分。[\[3\]](https://huggingface.co/deepseek-ai/DeepSeek-V4.1-Flash/blob/dba1be0a40aa45a94ad051997016db3960a90277/config.json)

SGLang 没有给 40 层各建一套同构缓存。`MQALayer` 读取每层的 `compress_ratio`，只有命中 `kv_source_layer_ids` 的 c1/c2 层才实例化 `DeepseekV41Compressor`，只有 index source 层才实例化 `DeepseekV41Indexer`；夹在中间的 Reuse 层通过 attention backend 找到最近的 source layer，直接读共享 main KV、indexer K 与 Top-K 结果。[\[4\]](https://github.com/sgl-project/sglang/blob/7bdebdab7db4befb71c64ae0d6f0eb37fe7d8402/python/sglang/srt/models/deepseek_v4.py#L1003-L1065)

**模型配置在这里已经成为缓存拓扑描述。** 改错一个 layer id 不会只造成某层慢一点，它可能让后层读取错误的物理池，或让本该复用的状态重新生成。对部署者而言，V4.1 适配的第一道验收也应从“能否生成文本”升级为“跨层共享关系是否与 checkpoint 配置逐项一致”。

## 二、CED 与 CSA2：省下的是层维状态，代价是跨层依赖

官方技术报告给出的两个锚点很直接：V4.1 Flash 的 global KV 为 **890 bytes/token**，约为 V4 Flash 的四分之一；persistent KV footprint 进一步降到 V4 的约八分之一。前一个数字来自 CSA2 的跨层 KV 复用与 FP4 main KV，后一个数字还叠加了不再持久保存 SWA KV 的策略。[\[5\]](https://huggingface.co/deepseek-ai/DeepSeek-V4.1-Flash/blob/dba1be0a40aa45a94ad051997016db3960a90277/DeepSeek_V41_Tech_Report.pdf)

![[SGLang 接住 DeepSeek V4.1 Flash：一次横跨缓存、索引与-02.png]]

*图 1：V4.1 的主干把 CSA2 的 Full、Reindex、Reuse 模式分配到固定层位；decoder global KV 由 encoder 输出投影，后层只维护自己的 SWA 状态。来源：DeepSeek-V4.1 Technical Report Figure 3。*

CED 让 decoder 的 global KV 从 encoder 最终 hidden states 投影出来，于是长 prompt 的大部分 token 无需完整穿过后 20 层。CSA2 又把层间责任拆成三种：Full 层产生 main KV、indexer K 和 Top-K；Reindex 层复用前两者，只用自己的 query 重算 Top-K；Reuse 层连 indexer 都跳过，沿用最近一次选择。**从 FLOPs 视角看是跳层，从 runtime 视角看则是建立一张“谁生产、谁重索引、谁消费”的有向依赖图。**

这也是 `deepseek_v4_memory_pool.py` 改动幅度很大的原因。旧 V4 主要围绕 c4/c128 池组织状态，V4.1 加入 c1/c2 后，池注册表要按压缩率收集 source layers，再把普通 attention layer 映射到最近的 source。ratio 2 还有跨 chunk 的成对池化问题：如果两个相邻 token 跨越 prefill chunk 边界，第二个 token 必须从请求级 ring 中找回前一个 token 的 KV 与 gate score，完成 softmax pooling 后才能写入 c2。代码因此明确遵守“先读所有 partner，再写 ring”的顺序，避免环形覆盖破坏跨 chunk 配对。[\[6\]](https://github.com/sgl-project/sglang/blob/7bdebdab7db4befb71c64ae0d6f0eb37fe7d8402/python/sglang/srt/layers/attention/deepseek_v4_backend.py#L2911-L3016)

这里有一个容易漏掉的精度细节。c2 的投影与 pooling 用 FP32 累加，结果先舍入到 BF16、过 RMSNorm，再做 RoPE 与 FP4 fake quant；Blackwell 路径把这些步骤融合进专用 c1/c2 kernel，Hopper 与 ROCm 保留拆分路径。**“最终都存 FP4”并不代表中间舍入顺序可以随意调整。** 上游测试专门把 fused 路径与 unfused reference 对齐，说明这类 Day-0 kernel 的第一目标是复现 checkpoint 的数值语义，性能优化只能在该边界内发生。[\[7\]](https://github.com/sgl-project/sglang/blob/7bdebdab7db4befb71c64ae0d6f0eb37fe7d8402/python/sglang/srt/layers/attention/dsv4/dsv41_sparse.py#L93-L172)

## 三、Hierarchical Sparse Indexer：选择语义已接通，省算力还差最后一段

V4.1 的 decoder 只让 layer 20 的 Full indexer 扫描完整可见历史。它除了选本层 Top-512，还把得分最高的 2,048 个 block 收集成共享候选池；每个 block 含 8 个位置，所以后续 Reindex 层理论上只需扫描 **16,384 个候选位置**。上下文从 128K 长到 1M 后，这个上限保持不变，正是分层 indexer 的计算意义。[\[8\]](https://huggingface.co/deepseek-ai/DeepSeek-V4.1-Flash/blob/dba1be0a40aa45a94ad051997016db3960a90277/README.md)

![[SGLang 接住 DeepSeek V4.1 Flash：一次横跨缓存、索引与-03.png]]

*图 2：报告中的理想执行流。首个 Full 层扫描全历史并建立 block 级候选池，后续 Reindex 层应只对候选位置打分。来源：DeepSeek-V4.1 Technical Report Figure 5。*

当前 PR 的实现完成了候选池的发布、复用和最终 Top-K 过滤，但计算路径仍有一道明显缺口。Blackwell decode 先调用 `_fp4_paged_mqa_logits` 对 `metadata.max_c4_seq_len` 覆盖的完整位置生成 logits，随后 `two_level_decode_logits` 才发布 candidate mask 或把非候选位置写成负无穷；prefill dense 路径也是先算全宽 logits，再调用 `_publish_or_consume_candidates` 做 mask。[\[9\]](https://github.com/sgl-project/sglang/blob/7bdebdab7db4befb71c64ae0d6f0eb37fe7d8402/python/sglang/srt/layers/attention/deepseek_v4_backend.py#L3247-L3360)

所以这版 head 已经保证后层不会选到候选池以外的位置，**却还没有让后层 indexer 的 GEMM 只读取 16,384 个候选位置。** 在百万 token 场景里，selection 结果符合 Hierarchical Sparse Indexer，读取和打分成本仍随完整上下文增长。PR review 已经把后续方向指向 DeepGEMM 的 `fp8_fp4_paged_sparse_mqa_logits`：它需要新 wheel、512-byte 对齐的 index-K page stride，以及 BF16 权重和 logits 的精度验证。[\[10\]](https://github.com/deepseek-ai/DeepGEMM/pull/432)

这处差距决定了我们该怎样读 Day-0 支持。模型可以启动、答案可以正确、候选 mask 也可以通过单测，但报告声称的“后层 indexer 成本与上下文长度解耦”还需要 kernel 读取路径落地。生产验收若只看输出一致性，会完全漏掉这类性能债；至少要把 layer 20 与 24/28/32/36 的 indexer latency 分开采样，再随 context length 画曲线。

## 四、SWA Bounded Replay：两个同名开关，处理两类不同缺口

V4.1 每层仍保留 128-token SWA。global KV 能跨层复用之后，SWA 成为阻止 decoder 跳过长 prompt 的最后一段状态；把它全量写入 SSD 又会吃掉 persistent cache 的容量收益。官方报告于是给出 encoder 与 decoder 两条 bounded replay 路径，两者都只重放最近 128 个 token，也都接受近似状态。SGLang 没有把它们合成一个模糊的优化开关，而是分别实现。

Decoder replay 相对直接。启用后，模型把 `late_layer_start` 设为最后一个 `kv_source_layer` 加一；当前配置就是 layer 21。一次 extend 走到这里，hidden states、position、input ids 与 Engram hash ids 都被裁成每个请求最后 128 行，layer 21 到 39 只为 SWA 跑这段 tail。main KV 和 indexer K 已由 layer 20 生成，无需在后 19 层重放完整 prompt。[\[11\]](https://github.com/sgl-project/sglang/blob/7bdebdab7db4befb71c64ae0d6f0eb37fe7d8402/python/sglang/srt/models/deepseek_v4.py#L3410-L3425)

Encoder replay 处理 prefix cache 命中后的另一种缺口：global KV 命中了，SWA history 已经不在持久层。`run_encoder_swa_replay` 取缓存前缀末尾最多 128 个 token，构造一个 prefill-only batch，禁止 logprob、grammar、多模态与 speculation，再让模型重建请求级 SWA 窗口；如果缓存边界落在奇数 position，代码直接报错，因为 c2 的成对压缩会失去配对边界。[\[12\]](https://github.com/sgl-project/sglang/blob/7bdebdab7db4befb71c64ae0d6f0eb37fe7d8402/python/sglang/srt/model_executor/encoder_swa_replay.py#L6-L77)

**128-token 回放把灾难性的全层重算改成有界退化，但近似性被写进了模型训练假设。** 这不是一个可以随意移植到其他 SWA 模型的通用 cache trick。SGLang 也用 feature guard 把风险显式挡在启动阶段：encoder replay 暂时不能与 prefill CUDA Graph、DP attention、speculation、context parallel、HiCache、PD disaggregation、LoRA 等路径组合；decoder replay不能与 prefill CUDA Graph 和 DP attention同时打开。[\[13\]](https://github.com/sgl-project/sglang/blob/7bdebdab7db4befb71c64ae0d6f0eb37fe7d8402/python/sglang/srt/arg_groups/deepseek_v4_hook.py#L224-L339)

这些 guard 没有削弱实现的价值，反而把当前能力边界说清了。Day-0 最危险的做法是让未验证组合静默运行；这里选择 fail fast，使部署者在启动时就知道要牺牲哪条优化路径。

$$
$$

## 五、Engram：196B 条件记忆把操作系统也拉进了热路径

Engram 让本次适配出现了一个此前 KV cache 文章里很少见的状态：它有 196B 参数，却不是每 token 都走一次 dense GEMM。V4.1 在 layer 1 与 14 各放一张约 3.84 亿行的 hash embedding table，用当前 token 与最多三个前驱 token生成 2/3/4-gram hash，再稀疏查表、投影并通过 gate 注入四路 mHC residual stream。[\[14\]](https://github.com/sgl-project/sglang/blob/7bdebdab7db4befb71c64ae0d6f0eb37fe7d8402/python/sglang/srt/layers/engram.py#L121-L196)

按 checkpoint 配置估算，两张表共有 768,022,850 行。每行包含 256 bytes 的 FP8 payload 和 8 bytes 的 E8M0 block scales，裸表约 **202.76 GB，折合 188.83 GiB**。默认实现把行按 TP rank 切分到设备内存，各 rank 对自己拥有的行做 lookup、其余位置填零，再 all-reduce 拼回完整结果；可选 host-table 路径则把表放进 Linux `mmap`，支持 shared memfd 或每 rank private anonymous mapping。[\[15\]](https://github.com/sgl-project/sglang/blob/7bdebdab7db4befb71c64ae0d6f0eb37fe7d8402/python/sglang/srt/layers/engram.py#L531-L677)

这里的难点已经越过 CUDA kernel。代码会在首次触页前丢弃 checkpoint page cache，调用 `MADV_HUGEPAGE`，必要时尝试 `MADV_COLLAPSE`，并从 `/proc/self/smaps` 检查 huge-page 覆盖率；如果完全没有 huge page，日志直接警告 lookup 可能慢约 **10 倍**，原因是一行一次 TLB miss。**同一份权重在“能装进 host memory”和“能以可接受延迟查到”之间，还隔着页表、PID namespace、pinning 与 ATS。**

Engram 也迫使 scheduler 记住请求历史。哈希器为每个 request slot 保存最近三个 token；普通 decode 可以当步提交，DSpark target verify 却必须等 acceptance 结果出来后，只提交真正接受的 token。否则被拒绝的草稿会污染下一步 n-gram，模型即使没有崩溃，输出也会悄悄偏离参考实现。对应代码把 `commit_after_verify` 接到 DSpark worker 的验收尾部，这是一处小改动，却是状态正确性的关键边界。[\[16\]](https://github.com/sgl-project/sglang/blob/7bdebdab7db4befb71c64ae0d6f0eb37fe7d8402/python/sglang/srt/layers/engram.py#L199-L447)

## 六、DSpark 与多模态：低延迟配方还不能代替性能结论

V4.1 自带三阶段 DSpark draft，默认一次提出 5 个 token。SGLang 的 cookbook 把 GB300、B200、B300 的低延迟策略写成 TP/EP 加 DSpark，把高吞吐策略改为关闭 speculation、扩大并发；注释给出的理由很朴素：DSpark step 有固定成本，batch 大到一定程度后无法摊平。H200 的低延迟配方暂时改用 decoder SWA bounded replay，两组 H200 配方都没有打开 DSpark。[\[17\]](https://github.com/sgl-project/sglang/pull/38802)

这与推测解码的通用规律一致：接受长度只决定一轮能带走多少 token，吞吐还要除以 draft、verify 和状态维护的总成本。PR 的端到端测试注册在 `4-gpu-gb300` nightly runner，覆盖图像输入，并要求 DSpark 的 `avg_spec_accept_length > 1`；它证明最小链路可工作，尚未回答 batch、context、reasoning effort 改变后 DSpark 何时失去收益。[\[18\]](https://github.com/sgl-project/sglang/blob/7bdebdab7db4befb71c64ae0d6f0eb37fe7d8402/test/registered/models_e2e/test_deepseek_v41.py)

多模态路径同样不是把一个 image placeholder 塞进 tokenizer 就结束。V4.1 没有 Jinja chat template，SGLang 增加了 700 多行专用 encoder，处理图文交错、连续 reasoning effort、工具结果重排和 DSML 标签；图像预处理还分出 Python、Rust 与 GPU 路径。模型前向中，image token 会绕过 Engram gate，避免视觉占位符进入文本 n-gram 记忆。**协议编码、视觉 token 规划与模型状态必须共享同一套 position 语义，任何一层自行“修正”输入都会让后面的 hash 和路由失配。**[\[19\]](https://github.com/sgl-project/sglang/blob/7bdebdab7db4befb71c64ae0d6f0eb37fe7d8402/python/sglang/srt/entrypoints/openai/encoding_dsv41.py)

Cookbook 已在独立 PR 合入，但它明确使用 `lmsysorg/sglang:dev-dsv41` 预览镜像。GB300 与 MI350X 的启动配方标为 verified，H200、B200、B300 仍是 final verification in progress，而且页面主动不发布开发期 benchmark。**部署矩阵目前能回答“从哪组参数起跑”，不能回答“这组配置在你的 workload 上快多少”。**

## 七、我的验收顺序：先查状态边界，再追峰值吞吐

面对这类改动，我会把验收拆成三轮。第一轮只查正确性边界：固定输入分别跨过 c2 偶数/奇数 chunk、prefix hit/miss、DSpark accept/reject 与图文交错，比较 eager、decode graph 和参考实现 logits；重点盯住请求 retraction 后的 Engram 三 token history，以及 ratio-2 ring 有没有留下旧请求状态。

第二轮量每个架构承诺是否兑现。把 layer 20 的 Full indexer 和 layer 24/28/32/36 的 Reindex latency 分开，在 16K、128K、512K、1M context 上逐点记录；如果后四层仍近似线性增长，就说明 candidate-aware sparse GEMM 尚未接通。Decoder bounded replay 则要比较开关前后的 prefill layer-time，确认 layer 21–39 的处理行数从完整 prompt 收敛到每请求 128 行，同时用任务评测检查近似状态的质量代价。

第三轮才讨论系统吞吐。Engram device-shard、shared host table、private host shard 至少要分别记录 page residency、TLB miss、lookup latency 和通信占比；DSpark 要按 batch 与自然 acceptance 分桶，不能用固定接受率替代真实流量。**一旦模型把 cache、hash memory 与 speculation 绑进同一个请求生命周期，单项 kernel 峰值已经不足以解释端到端表现。**

## 八、结论：Day-0 已经接通，稳定性取决于状态组合能否收敛

这份 PR 的完成度比“模型能启动”高得多：c1/c2 跨层缓存、FP4 indexer、两类 SWA bounded replay、Engram 分片/host table、DSpark acceptance commit、多模态编码和硬件专用 kernel 都有实际代码与测试落点。它也诚实地暴露了边界——大量 feature guard、预览镜像、未运行的主 CI，以及 Hierarchical Sparse Indexer 仍在全宽打分，都说明 V4.1 支持还处于快速收敛期。

我的判断是：**SGLang 已经接住 DeepSeek V4.1 Flash 的架构语义，下一阶段的主战场是把语义正确的多状态路径压成可组合、可测量、可发布的生产路径。** 如果准备现在部署，适合从 cookbook 的单节点 verified shape 起步，固定 PR head 与镜像；HiCache、PD disaggregation、DP attention、prefill CUDA Graph 和 replay 的组合应逐个打开，不要一次把所有优化叠上去。

---

## 参考资料

[1] [SGLang PR #38798: Add DeepSeek V4.1 support](https://github.com/sgl-project/sglang/pull/38798)

[2] [SGLang commit 85f8105f: Add DeepSeek V4.1 Day 0 support](https://github.com/sgl-project/sglang/commit/85f8105f5d2e2e0bdea70ff50cb5449bf9567053)

[3] [DeepSeek-V4.1-Flash config.json](https://huggingface.co/deepseek-ai/DeepSeek-V4.1-Flash/blob/dba1be0a40aa45a94ad051997016db3960a90277/config.json)

[4] [SGLang DeepSeek V4 model integration at commit 7bdebdab](https://github.com/sgl-project/sglang/blob/7bdebdab7db4befb71c64ae0d6f0eb37fe7d8402/python/sglang/srt/models/deepseek_v4.py)

[5] [DeepSeek-V4.1-Flash Technical Report](https://huggingface.co/deepseek-ai/DeepSeek-V4.1-Flash/blob/dba1be0a40aa45a94ad051997016db3960a90277/DeepSeek_V41_Tech_Report.pdf)

[6] [SGLang DeepSeek V4 attention backend at commit 7bdebdab](https://github.com/sgl-project/sglang/blob/7bdebdab7db4befb71c64ae0d6f0eb37fe7d8402/python/sglang/srt/layers/attention/deepseek_v4_backend.py)

[7] [SGLang V4.1 low-ratio compressor and indexer](https://github.com/sgl-project/sglang/blob/7bdebdab7db4befb71c64ae0d6f0eb37fe7d8402/python/sglang/srt/layers/attention/dsv4/dsv41_sparse.py)

[8] [DeepSeek-V4.1-Flash model card](https://huggingface.co/deepseek-ai/DeepSeek-V4.1-Flash/blob/dba1be0a40aa45a94ad051997016db3960a90277/README.md)

[9] [SGLang V4.1 hierarchical indexer decode path](https://github.com/sgl-project/sglang/blob/7bdebdab7db4befb71c64ae0d6f0eb37fe7d8402/python/sglang/srt/layers/attention/deepseek_v4_backend.py#L3247-L3360)

[10] [DeepGEMM PR #432: paged sparse MQA logits](https://github.com/deepseek-ai/DeepGEMM/pull/432)

[11] [SGLang decoder SWA bounded replay layer boundary](https://github.com/sgl-project/sglang/blob/7bdebdab7db4befb71c64ae0d6f0eb37fe7d8402/python/sglang/srt/models/deepseek_v4.py#L3410-L3425)

[12] [SGLang encoder SWA bounded replay](https://github.com/sgl-project/sglang/blob/7bdebdab7db4befb71c64ae0d6f0eb37fe7d8402/python/sglang/srt/model_executor/encoder_swa_replay.py)

[13] [SGLang DeepSeek V4.1 feature compatibility guards](https://github.com/sgl-project/sglang/blob/7bdebdab7db4befb71c64ae0d6f0eb37fe7d8402/python/sglang/srt/arg_groups/deepseek_v4_hook.py#L224-L339)

[14] [SGLang Engram hash layout and lookup implementation](https://github.com/sgl-project/sglang/blob/7bdebdab7db4befb71c64ae0d6f0eb37fe7d8402/python/sglang/srt/layers/engram.py)

[15] [SGLang Engram host table implementation](https://github.com/sgl-project/sglang/blob/7bdebdab7db4befb71c64ae0d6f0eb37fe7d8402/python/sglang/srt/layers/engram.py#L531-L677)

[16] [SGLang Engram request history and verify commit](https://github.com/sgl-project/sglang/blob/7bdebdab7db4befb71c64ae0d6f0eb37fe7d8402/python/sglang/srt/layers/engram.py#L199-L447)

[17] [SGLang PR #38802: DeepSeek-V4.1 Flash cookbook](https://github.com/sgl-project/sglang/pull/38802)

[18] [SGLang DeepSeek V4.1 end-to-end test](https://github.com/sgl-project/sglang/blob/7bdebdab7db4befb71c64ae0d6f0eb37fe7d8402/test/registered/models_e2e/test_deepseek_v41.py)

[19] [SGLang DeepSeek V4.1 chat encoding implementation](https://github.com/sgl-project/sglang/blob/7bdebdab7db4befb71c64ae0d6f0eb37fe7d8402/python/sglang/srt/entrypoints/openai/encoding_dsv41.py)

### 版本对齐信息

| 对象                        | 状态                                | 固定版本                                    | 审阅日期       |
| ------------------------- | --------------------------------- | --------------------------------------- | ---------- |
| SGLang runtime PR #38798  | 开放，review required，merge conflict | head `7bdebdab7db4`；base `03e4c06589e3` | 2026-09-10 |
| SGLang cookbook PR #38802 | 已合并                               | merge commit `69777c4d36c0`             | 2026-09-10 |
| DeepSeek-V4.1-Flash       | 模型卡与技术报告已公开                       | Hugging Face revision `dba1be0a40aa`    | 2026-09-10 |
