---
title: "同一份 GGUF，两个运行时：Transformers 借用 ggml 内核后，边界怎样重画"
source: "MiracleFarms"
category: "AI Infra"
author: "author"
url: "https://miraclefarms.github.io/notes/2026/09/26/transformers-gguf-runtime-boundary/"
published: 2026-09-26T20:10:00+08:00
saved: 2026-09-27T10:57:17+08:00
folo_key: "miraclefarms::https://miraclefarms.github.io/notes/2026/09/26/transformers-gguf-runtime-boundary/"
tags:
  - "folo"
  - "AI_Infra"
  - "MiracleFarms"
---

# 同一份 GGUF，两个运行时：Transformers 借用 ggml 内核后，边界怎样重画

> [!info] MiracleFarms · AI Infra · 2026-09-26 20:10 · [原文](https://miraclefarms.github.io/notes/2026/09/26/transformers-gguf-runtime-boundary/)

![[同一份 GGUF，两个运行时：Transformers 借用 ggml 内核后，-01.png]]

> **版本声明**：本文依据 Hugging Face 2026-09-22 的技术文章与当时列出的实现范围。实验使用 Apple Silicon、特定 Qwen 检查点与预热后的短生成；不把跨工具的吞吐图解读为严格同条件性能排名。

过去，本地用户拿到一份 GGUF，通常会把它交给 llama.cpp 或基于它的应用。研究者若要在 PyTorch 里修改 forward、读中间激活、试解码策略，又会回到 Transformers 的模型定义与张量接口。两套工具能互相转换文件，但运行时并没有因此共享性能路径。

Hugging Face 在 9 月宣布的新实现，把边界推进了一步：Transformers 不只读取 GGUF 文件，还在 Apple Silicon 的打包量化路径上调用 ggml 的 Metal 内核，并修改自己的 `generate` 循环以减少每 token 的同步等待。[\[1\]](https://huggingface.co/blog/transformers-llama-cpp-quants)它提出的问题因此不是“Transformers 是否取代 llama.cpp”，而是：**模型定义、量化算子和服务运行时，哪些能力现在可以分开复用？**

## 一、读得懂 GGUF，与直接运行打包权重是两回事

GGUF 把权重、tokenizer 信息及可选 chat template 放在一个文件中，并容纳不同量化档位。Hugging Face 举的 Qwen3.5-4B 例子里，BF16 文件为 8.42 GB，Q4\_K\_M 为 2.74 GB；这首先改变的是能否装进本地机器的内存，不自动保证任务质量或推理速度。[\[1\]](https://huggingface.co/blog/transformers-llama-cpp-quants)

旧式“支持 GGUF”可以只意味着导入文件，再解量化回普通 PyTorch 权重。文件能打开，但打包权重的内存优势可能随之消失。新路径在兼容内核可用时，让量化权重保持打包状态，矩阵计算直接读取打包表示；若内核获取不到，模型可能退回解量化和通用 attention 路径，显存占用随之改变。**文件格式兼容、打包执行兼容、性能兼容，是三份不同的合同。**[\[1\]](https://huggingface.co/blog/transformers-llama-cpp-quants)

当前的入口仍是常见的 `from_pretrained(model_id, gguf_file=filename)`；同一检查点也可交给 `transformers serve` 暴露 OpenAI 兼容接口。这里的工程意义是研究和评测流程能继续使用 Transformers 的 API，同时借到部分本地推理内核。它并不意味着 Transformers 获得 llama.cpp 的全部内存管理、设备覆盖或成熟服务能力。Hugging Face 仍把高效本地推理优先推荐给 llama.cpp。[\[1\]](https://huggingface.co/blog/transformers-llama-cpp-quants)

## 二、真正搬过来的是哪几段算子

这次复用不是把 llama.cpp 引擎整体嵌入 Python。`ggml-quantization` 在打包权重上做矩阵运算，包括 MoE 被选中的专家；`ggml-norm` 融合归一化；`ggml-attn` 处理 prefill 与 decode 的注意力；`ggml-gated-delta-net` 服务 Qwen 混合架构中的线性注意力层。MoE 的 top-k 路由还配有独立 Metal 内核。每一段仍要对齐 PyTorch 模型定义、张量形状和设备约束。[\[1\]](https://huggingface.co/blog/transformers-llama-cpp-quants)

官方的消融图比跨引擎速度图更能说明这一层贡献：对同一打包 GGUF 检查点，两组都启用量化内核；只比较其余层内核是否启用。Qwen3.5-4B 从 44.2 到 70.4 tok/s，Qwen3.8-27B 从 10.5 到 15.9 tok/s，Qwen3.5-35B-A3B 从 28.8 到 60.2 tok/s。对应提升约 1.59、1.51 和 2.09 倍。这个对照说明瓶颈不止在“能否读取 4-bit 权重”，还在 attention、norm、线性注意力和专家路径是否走到合适内核。[\[1\]](https://huggingface.co/blog/transformers-llama-cpp-quants)

![[同一份 GGUF，两个运行时：Transformers 借用 ggml 内核后，-02.png]]

*图 1：两组均保留 `ggml-quantization` 与打包权重，仅改变其他层内核。因而这张图检验的是层内核组合，不是“打包量化相对 BF16”的收益。来源：Hugging Face 技术博客。*

这也解释了一个更长远的机会：内核作用于张量，不必和 GGUF 文件永久绑定。若模型定义先在 Transformers 中可用，开发者可以逐个验证并接入合适的 ggml 内核，而不必等一个新架构在 llama.cpp 中获得完整专用实现。但这仍是逐架构集成的工作；官方当前示例主要是文本生成，不能把可能性写成已覆盖多模态。[\[1\]](https://huggingface.co/blog/transformers-llama-cpp-quants)

## 三、生成循环的等待，比 Python 这个标签更具体

即使 GPU kernel 足够快，CPU 仍要逐 token 安排下一步。若每一步为判断是否停止而同步读回 GPU 结果，队列里未完成的 GPU 工作会迫使 CPU 等待。Hugging Face 因此做了两项生成循环调整：无 padding 的支持路径提前去掉不必要的全 1 attention mask；停止条件异步复制，在下一步消费，并删去超过停止条件的额外一步。这些改动作用在 `generate` 周边，理论上也惠及非 GGUF 模型。[\[1\]](https://huggingface.co/blog/transformers-llama-cpp-quants)[\[3\]](https://github.com/huggingface/transformers/pull/48814)[\[4\]](https://github.com/huggingface/transformers/pull/47975)

官方另一张消融图把所有层内核固定启用，再比较生成循环改动前后。三组模型分别从 49.6 到 70.4、13.7 到 15.9、33.7 到 60.2 tok/s。最大模型之外，MoE 的 1.79 倍变化尤其提醒我们：**把昂贵计算交给 GPU，并不足以消除每 token 的 CPU 同步税。** 这不是“Python 总是慢”或“Python 现在和 C++ 一样快”的普适结论，而是这条路径上两个明确等待点的收益。[\[1\]](https://huggingface.co/blog/transformers-llama-cpp-quants)

![[同一份 GGUF，两个运行时：Transformers 借用 ggml 内核后，-03.png]]

*图 2：两组都启用层内核，差别是 attention mask 处理与停止检查的调度方式。横向看它与图 1 的基线不同，不应把两图倍数简单相乘。来源：Hugging Face 技术博客。*

## 四、为什么不能宣布两套引擎打平

Hugging Face 用 MacBook Pro M2 Max、32 GB 统一内存比较了三份检查点，并写出完整的工具版本和命令。可是 `llama-bench -p 0 -n 128` 测的是 128 个 decode token，排除了 prompt processing；Transformers 端从 12-token prompt 生成 128 token，计时包含 prefill，且取三次预热运行中的最好一次。作者也明确说图不能视为同一 benchmark 条件。[\[1\]](https://huggingface.co/blog/transformers-llama-cpp-quants)

更重要的是，单路交互生成不是并发服务。当前打包执行是 MPS 路径，padding 与 batching 仍待完善，架构主要覆盖 Qwen3.5 dense/MoE 及兼容的 Qwen3.8。若讨论长上下文、多人并发、服务恢复、质量回归或不同 GPU，现有图都没有给出答案。**决定用哪套运行时，要先写出自己的负载合同：模型与量化档位、prompt/output 长度、并发、首 token 延迟、持续生成速率和任务质量。**[\[1\]](https://huggingface.co/blog/transformers-llama-cpp-quants)

因此，这轮更新最值得写的判断是：GGUF 正从“让文件跨工具流动”变成“让专用内核跨模型定义复用”。研究、转换校验与原型修改可更自然地留在 Transformers；追求成熟的本地推理部署仍可选 llama.cpp。边界没有消失，只是被拆成了文件、内核、生成循环和服务管理四层，今后每层都能单独演进，也必须单独验证。

- - -

## 参考资料

\[1\] [Transformers now runs llama.cpp quants](https://huggingface.co/blog/transformers-llama-cpp-quants)

\[2\] [GGUF and Transformers documentation](https://huggingface.co/docs/transformers/gguf)

\[3\] [Transformers PR #48814: Drop unnecessary attention mask early](https://github.com/huggingface/transformers/pull/48814)

\[4\] [Transformers PR #47975: Defer stopping check](https://github.com/huggingface/transformers/pull/47975)

<!-- folo:annotations -->
## 批注

- <u>Transformers 是否取代 llama.cpp</u>
  > 但是这两者是不同的定位，Transformers 只是个框架，而 llama.cpp 是一个面向端侧推理引擎，定位还是完全不一样的吧
- ==在 PyTorch 里修改 forward、读中间激活、试解码策略，又会回到 Transformers 的模型定义与张量接口==

<!-- folo:data [{"id":1,"rel_path":"","kind":"underline","color":"yellow","quote":"Transformers 是否取代 llama.cpp","prefix":"nerate 循环以减少每 token 的同步等待。[1]它提出的问题因此不是“","suffix":"”，而是：模型定义、量化算子和服务运行时，哪些能力现在可以分开复用？\n一、读得懂","note":"但是这两者是不同的定位，Transformers 只是个框架，而 llama.cpp 是一个面向端侧推理引擎，定位还是完全不一样的吧","created_at":1790488603,"updated_at":1790488644},{"id":2,"rel_path":"","kind":"highlight","color":"pink","quote":"在 PyTorch 里修改 forward、读中间激活、试解码策略，又会回到 Transformers 的模型定义与张量接口","prefix":"到一份 GGUF，通常会把它交给 llama.cpp 或基于它的应用。研究者若要","suffix":"。两套工具能互相转换文件，但运行时并没有因此共享性能路径。\nHugging Fa","note":"","created_at":1790488679,"updated_at":1790488679}] -->
<!-- /folo:annotations -->
