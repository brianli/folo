---
title: "Apache Flink Agents：流式智能体操作系统"
source: "Flink中文社区"
category: "Inbox"
group: "AI基础设施"
author: "谢文晋"
url: "https://mp.weixin.qq.com/s/9YZkZsDbid_FFrSTVOC4-g"
published: 2026-10-12T19:08:25+08:00
saved: 2026-10-12T19:08:39+08:00
folo_key: "url::https://mp.weixin.qq.com/s/9YZkZsDbid_FFrSTVOC4-g"
updated: 2026-10-12T19:10:41+08:00
tags:
  - "folo"
  - "Inbox"
  - "Flink中文社区"
---

# Apache Flink Agents：流式智能体操作系统

> [!info] Flink中文社区 · Inbox · 2026-10-12 19:08 · [原文](https://mp.weixin.qq.com/s/9YZkZsDbid_FFrSTVOC4-g)

Agent 可以说是过去一年中技术圈最炙手可热的词汇。但相信许多人有一个共同感受：在本地跑一个 Agent Demo 相对容易，而在面向大规模工作负载的生产环境中，把 Agent 跑稳定仍然存在诸多挑战。Flink Agents 的目的，正是为 Agent 在企业级生产场景的落地提供一套解决方案。

接下来的内容将从我们认为 Agent 落地存在的问题出发，讲清楚为什么 Flink 是企业级 Agent 生产落地最合适的底座，它的设计是什么，解决了哪些问题，最后再介绍当前的社区状态。

# 01. Why Flink Agents

首先探讨为什么 Flink 是最适合作为企业级 Agent 生产落地的底座。从这张全景图出发：从使用场景来看，当前的 Agent 可分为两个场景。第一个是面向终端用户的 Agent 场景，例如当下广泛使用的 Coding Agent、Computer Use Agent。

![[Apache Flink Agents：流式智能体操作系统-01.jpg]]

这类场景的特点在于，Agent 的触发由用户请求驱动，采用一问一答的形式，能够容忍一定的不稳定性。目前该场景已有诸多落地实践，例如 Codex、Claude Code、Cursor 等产品。另一个场景是面向大规模工作负载的生产场景，例如智能运维、实时风控、物联网和实时多模态处理。该场景的特点是，Agent 的运行并非由用户请求驱动，而是由连续不断的事件日志、系统事件来触发，因此要求 Agent 具备 7×24 小时连续运行、持续响应、实时决策的能力。该场景目前的探索仍处于比较早期的阶段。

![[Apache Flink Agents：流式智能体操作系统-02.jpg]]

那么生产场景需要什么样的 Agent？我们认为需要一种流式 Agent，它具备以下三个特点。首先是事件驱动，即触发 Agent 运行的并非用户的请求输入，而是连续不断的系统事件，这要求 Agent 必须具备连续运行、持续响应的能力。第二个特点是分布式：面对海量工作负载，单机无法承载，必须支持 Agent 的分布式调度与部署，通过水平扩展来处理海量数据。第三个特点是可靠：该场景的 Agent 必须 7×24 小时不间断运行，而在分布式部署下单点故障不可避免，因此必须具备容错能力，以保证 Agent 系统的稳定运行。这些问题并不是简单地把一个 Agent 复制几份就能解决的——单 Agent 多副本无法解决状态协同、容错恢复以及端到端一致性语义等问题，因为它们本质上都是分布式系统需要解决的难题。因此，需要为 Streaming Agent 找到一套真正的生产级底座，也就是一套 Streaming Agent 的 OS，而我们认为 Flink 正是最合适的选择。

# 02. 什么是 Flink Agents

为什么是 Flink？在过去十多年的生产建设中，Flink 已经沉淀出流式消费、分布式部署协同、状态协调等一整套完整的流处理能力。与其为大规模 Agent 的生产部署去构建一套全新的系统，不如直接把 Agent 引入到 Flink 当中，让 Agent 像其他算子一样，天然继承 Flink 的流式消费、分布式运行和容错能力。那么究竟什么是 Flink Agents？其核心理念就是让 Agent 成为 Flink 实时流处理中的一等公民，也就是把 Agent 插入到 Flink 的 Pipeline 中。

![[Apache Flink Agents：流式智能体操作系统-03.jpg]]

大家熟悉的 Flink Pipeline 通常是这样的：有一个数据源，经过一些结构化算子做过滤或聚合，然后输出给下游。Flink Agents 要做的，是在这个 Pipeline 中插入一个 Agent 算子。这个 Agent 算子内部可能调用大模型并处理大模型返回的工具调用，也可能使用一些 Skills 或者查询外部的向量数据库；它可以对接其他算子，例如结构化算子、多模态算子，也可以对接另一个 Agent 算子。 Agent 算子被引入 Flink 的 Pipeline，直接参与到实时数据流的处理中。

![[Apache Flink Agents：流式智能体操作系统-04.jpg]]

这张图描述了 Agent 算子在 Flink 中的位置。它与其他多模态算子、结构化算子一样，建立在久经打磨的 Flink 分布式 Runtime 之上，从而为构建生产级流式 Agent 提供事件驱动、分布式部署和可靠性等能力。

![[Apache Flink Agents：流式智能体操作系统-05.jpg]]

进一步看 Agent 算子本身的执行模型。上游的输入在算子内被转化为 Input Event，算子根据事件进行内部的动态编排与运行时选择：一个 Event 可以触发不同的 Action，每个 Action 又可以发送新的事件，最终发送为一个 Output Event，输出给下游。这里的 Action 既可以是用户自定义的业务逻辑，也可以是框架本身提供的，例如调用大语言模型、处理大语言模型返回的工具调用以及查询向量数据库等内置 Action。

# 03. 核心特性

将 Agent 引入实时数据流在设计上比较直观，但要让 Agent 算子成为 Flink 的原生算子，并满足大规模工作负载的要求，实际上仍面临许多挑战。下面介绍 Flink Agents 在设计和开发过程中的一些核心特性，以及背后的设计思考。

![[Apache Flink Agents：流式智能体操作系统-06.jpg]]

首先是 API 层。要在 Flink 中支持 Agent 成为原生算子，首先要解决的问题是：用户如何定义 Agent 的逻辑、如何构建一个 Agent。Flink Agents 本身提供了基于 Event-Driven 的编排框架，并支持用户声明 Agent 所需的资源，例如 LLM、向量数据库，以及需要对接的 MCP。那么应以什么样的 API 把这套 DSL 暴露给用户？传统的大数据开发者可能习惯了 Java，而 AI 生态以及数据科学家可能更偏爱 Python 生态。因此在一开始，我们就提供了 Python 和 Java 两套对齐的 API，为不同用户提供兼容的选择。此外，随着 AI 辅助编程能力的飞速发展，可以判断今后 Agent 的构建可能大多由 Coding Agent 来实现——人无需真正动手开发，只需给出一些自然语言输入，由 Claude Code 这类工具给出实现。因此，在最新发布的 0.3 版本中，我们还提供了一种声明式的 YAML API。由于它显式约束了 Schema，并且能够将作业的拓扑结构与具体业务逻辑进行分离，有助于提升 Coding Agent 基于 Flink Agents 构建 Agent应用的表现。后续我们也计划在社区提供一些 Flink Agents 的开发 Skills，来进一步提升使用 Coding Agent 构建 Flink Agents 作业的体验。

![[Apache Flink Agents：流式智能体操作系统-07.jpg]]

多语言 API 为兼容现有用户提供了便利，用户可以更灵活地定义和构建 Agent，但也可能给框架本身带来双倍的开发成本。例如，若想在 Flink Agents 中支持调用 OpenAI 的 LLM，可能需要在 Python API 和 Java API 中各做一遍；另一种情况是，用户此前已用 Java 实现了一个 Action，但在开发新的 Agent 时，由于某些库只提供了 Python SDK，需要改用 Python API 重新构建 Agent，此时若想复用之前的 Action，就必须用 Python 重写之前用 Java 实现的 Action。为解决这一问题，Flink Agents 借助在 PyFlink 建设过程中沉淀下来的跨语言调用框架 pemja，实现了对跨语言调用的支持。也就是说，用户可以在一种语言中调用使用另一种语言实现的 Resource（如 Chat Model、Vector Store、Embedding Model）和 Actions。由多种语言构建出来的 Agent，在编译期会编译为统一的 Agent Plan，底层运行则根据 Agent Plan 进行运行时解析与动态触发，从而为不同语言的开发者提供一致的开发与运行体验。

![[Apache Flink Agents：流式智能体操作系统-08.jpg]]

关于 API 兼容性的规划：我们在今年 6 月发布了 0.3 版本，也理解 API 的兼容与稳定性是 Flink Agents 生产落地的迫切需求。从社区和早期用户的反馈中，我们观察到现有 API 存在一些易用性问题，因此在下一个版本 0.4 中，我们计划集中精力对现有 API 进行打磨；0.4 发布之后会观察一段时间，如果没有太多问题，就发布 1.0，并在 1.0 中对 API 兼容性做出正式承诺。另外，如前所述，后续也会在社区提供一些 Dev Skills，帮助用户使用 Coding Agent 开发基于 Flink Agents 构建的 Agent；这部分预计在 7 月就可以提供，并不绑定到某一个具体版本。

![[Apache Flink Agents：流式智能体操作系统-09.jpg]]

下面介绍对 Agent 原语的支持。用户在声明或构建一个 Agent 时，该 Agent 本身可能需要调用 LLM、访问 Vector Store、使用一些 Skills。针对这些 Agent 领域常用的资源或概念，框架提供了内置抽象，从而降低用户构建 Agent 的工作量。这些抽象本身也遵循了业界主流框架的设计，避免给用户带来额外的理解成本，为用户提供在不同框架之间开发的一致性体验。以 Agent Skills 为例，我们在 0.3 版本新增了对 Skills 的支持，其实现参考了 Claude 开放的 Skills 协议与规范。同时支持用户为 Chat Model、Embedding Model、Vector Store 等抽象扩展自定义实现，并提供了一些内置实现，例如 Ollama、DashScope、OpenAI 等。

![[Apache Flink Agents：流式智能体操作系统-10.jpg]]

Memory 是 Agent 原语中比较特殊的一个。由于大模型的上下文并非无限，在 Agent 构建实践中，越来越多的人认识到记忆的重要性。Flink Agents 提供了一套完备的 Memory 框架，包括感官记忆（Sensory Memory）、短期记忆（Short-Term Memory）和长期记忆（Long-Term Memory）。其中感官记忆和短期记忆都基于 Flink State 实现，因此天然具备容错能力，区别在于感官记忆会进行自动清理。关于长期记忆，我们在 0.2 版本曾基于 Vector Store 手搓了一版，但后来发现长期记忆的管理本身是一个较大的话题，并且并不是 Streaming Agent 专属的问题，因此在 0.3 版本中，我们选择基于开源记忆框架 Mem0 来实现 Long-Term Memory；后续也可能进一步扩展，以支持其他 Memory Backend。

![[Apache Flink Agents：流式智能体操作系统-11.jpg]]

在原语方面的规划上，下个版本将探索 Sub Agent。我们在一些用户场景中观察到了对 Sub Agent 的需求：由于它具备上下文隔离和弹性扩缩的能力，在某些场景中是不可或缺的。因此在下个版本我们将探索如何在 Flink Agents 中原生支持 Sub Agent 能力，包括利用 Flink 最新在社区提出的 RpcOperator 能力。

![[Apache Flink Agents：流式智能体操作系统-12.jpg]]

下面介绍运行时的核心特性。第一个是对异步执行的支持。熟悉 Flink 线程执行模型的人可能知道，Flink 的 Mailbox 线程模型本质上是单线程执行模型。在 Agent 这种大量调用外部服务、IO 操作较重的场景中，这种模型可能导致作业吞吐率受到严重影响。因此我们对 Agent 算子的异步执行做了支持，这一支持包含两个层面：其一，算子本身天然支持不同 Key 之间数据的并发执行；其二，还提供了异步执行接口，允许用户将一些较重的 IO 操作直接 Offload 到异步线程，从而避免主线程对其他数据处理的阻塞。

![[Apache Flink Agents：流式智能体操作系统-13.jpg]]

另一个是对一致性的保证。基于 Flink 的 Checkpoint 机制，一致性一直是 Flink 的传统强项。但在 Agent 场景中，仅有 Checkpoint 还不够，因为从 Checkpoint 恢复时仍可能重新消费数据，导致 Action 的重复执行，从而带来副作用，例如重复调用大模型造成 Token 浪费、订单重复支付或运维命令重复执行等。因此，Flink Agents 针对 Agent 算子的这种特殊情况，提供了一系列持续演进的解决方案。在 0.1 版本中，支持了 Action State Store，以一种类似 Write-Ahead Log 的形式来追踪每个 Action 的执行状态，这样从 Checkpoint 恢复时可以跳过已经完成的 Action，从而把不一致性从 Checkpoint 缩小到单个 Action。在 0.2 版本中，提供了 Durable Execution 能力，允许用户将一段关键代码通过 Durable Execution 接口提交，由框架负责这段代码结果的持久化；这样从 Checkpoint 恢复时，对于已完成的代码片段不会重新执行，而是直接获取其持久化结果，从而把不一致性从单个 Action 缩小到代码片段。但如果这段代码片段内部会调用外部服务，并且执行到一半挂掉，由于涉及与外部系统的交互，Flink Agents 本身无法保证端到端的精确一致（Exactly-Once）语义。因此在 0.3 版本中，我们支持了一种 Reconciler 机制，允许用户为 Durable Execution 注册一个 Reconciler，在其中自定义这段代码故障恢复时的执行逻辑，例如直接重试，或者先查看外部系统的状态再决定后续决策。

![[Apache Flink Agents：流式智能体操作系统-14.jpg]]

最后介绍可观测性。可观测性是任何系统在生产落地时的必要条件。Flink Agents 建立在 Flink 之上，除了复用 Flink 本身提供的一些可观测能力外，还提供了一些针对 Agent 算子的专门观测手段。首先是 Event Log。由于 Flink Agents 算子本质上是一个基于 Event-Driven 的编排架构，Event 是算子的中心，通过记录每条 Event 的信息，可以完整重建一个 Agent 的完整执行轨迹。我们支持将 Event 以 JSON 形式输出到文件，或输出到 Log4j 并在 Flink Web UI 上进行展示；在 0.3 版本中，还支持了 Per-Event-Type 的日志级别配置，允许用户只观测一些关键事件。另一个手段是 Metrics。Flink Agents 的 Metrics 继承了 Flink 的 Metrics 体系，因此可以无缝衔接到现有的 Flink 观测大盘中。在 Flink Metrics 的基础上，增加了一些针对 Agent 算子的专门指标，例如处理的输入 Event 数量、大模型调用的次数、请求次数、失败次数与重试次数，以及每次调用的 Token 消耗量等。用户通过 Event Log 和 Metrics 这两种观测手段，可以了解 Agent 的运行过程、运行轨迹与运行状态。

![[Apache Flink Agents：流式智能体操作系统-15.jpg]]

在可观测性方面同样有一些规划，下一阶段将做一些可观测能力的提升。首先是支持对 Event Log 做结构化检索。目前用户依据 Event Log 排查问题或调优 Agent 效果时，需要手动在文件中搜索关键字或按日志时间来筛选数据；在下个版本，期望内置对 Event Log 进行结构化检索的能力，让用户可以通过一些结构化的搜索语句直接获取想要的信息，提高易用性。第二个是 Per-Run Trace，期望以可视化的方式，从 Event Log 重建一条数据在 Agent 算子内部的完整执行轨迹，这种可视化方式对于用户排查问题和调优单条数据的执行效果非常有帮助。第三个是支持对 Memory 的可观测。Memory 是 Agent 上下文的重要组成部分，对于使用了 Memory 的 Agent 来说，理解 Memory 中存放了什么、以及在读取 Memory 时读到了什么，对于调优 Agent 效果和 Memory 的使用非常有帮助。因此在下个版本，将探索如何对 Memory 进行观测。

# 04. 社区建设

聊完技术，下面介绍社区当前的状态。Flink Agents 于 2025 年 10 月，也就是去年 10 月，在巴塞罗那的 Flink Forward 大会上，由阿里云、Confluent、LinkedIn 和 Ververica 四家流处理领域的顶级公司联合发起。截至今年 6 月，已经发布了三个版本。

![[Apache Flink Agents：流式智能体操作系统-16.jpg]]

0.1 版本支持了用户定义 Agent 算子，并以原生的方式运行在一个 Flink 作业中。0.2 版本一方面扩大了生态集成，另一方面补充了一些运行时能力，例如跨语言调用、Durable Execution 等。0.3 版本则在功能、容错以及运行时等各方面做了持续演进，例如支持了 YAML API、Agent Skills、基于 Mem0 的 Long-Term Memory，以及运行时的 Reconciler。

目前社区已有四十多位贡献者，他们来自阿里云、LinkedIn、Confluent、AWS、腾讯、美团等各大公司，也包括一些创业公司，遍布世界各地。这里也列出了一些与 Flink Agents 相关的链接，包括 GitHub、文档站和 Slack；其中 Slack 有面向开发者和面向用户的两个频道。

欢迎对 Flink Agents 感兴趣的开发者了解并参与 Flink Agents 社区。
