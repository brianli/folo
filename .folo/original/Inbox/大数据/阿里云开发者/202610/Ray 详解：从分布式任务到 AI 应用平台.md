---
title: "Ray 详解：从分布式任务到 AI 应用平台"
source: "阿里云开发者"
category: "Inbox"
group: "大数据"
author: "Tiny点"
url: "https://mp.weixin.qq.com/s/2269l0dKhvMGMPafAtLfqQ"
published: 2026-10-09T21:43:54+08:00
saved: 2026-10-09T21:44:12+08:00
folo_key: "url::https://mp.weixin.qq.com/s/2269l0dKhvMGMPafAtLfqQ"
tags:
  - "folo"
  - "Inbox"
  - "阿里云开发者"
---

# Ray 详解：从分布式任务到 AI 应用平台

> [!info] 阿里云开发者 · Inbox · 2026-10-09 21:43 · [原文](https://mp.weixin.qq.com/s/2269l0dKhvMGMPafAtLfqQ)

训练一个模型、运行一批数据处理任务，或者上线一个大模型服务，表面上是三类不同问题，底层却经常需要面对相同的工程挑战：任务如何分发到多台机器，数据如何在进程之间传递，失败后如何重试，资源不足时如何排队，服务扩容时如何复用已经加载的模型。

如果每个项目都从线程池、消息队列、RPC、节点管理和故障恢复开始搭建，系统很快就会变得复杂。Ray 的目标，就是提供一套统一的分布式运行时，把这些能力封装成简单的 Python API，并在此基础上提供数据处理、分布式训练、超参数搜索、强化学习和在线服务等 AI 组件。

一句话概括：

> Ray 是一个面向 Python 和 AI 工作负载的分布式计算框架。Ray Core 提供任务、Actor、对象和资源调度等基础能力，Ray Data、Ray Train、Ray Tune、RLlib 和 Ray Serve 则把这些能力组合成端到端的 AI 应用平台。

## 1\. Ray 是什么

Ray 最初由加州大学伯克利分校 RISELab 团队开发，后来形成了开源社区和商业化生态。它不是某一种模型训练框架，也不是 Kubernetes 的替代品，而是位于应用代码与底层机器资源之间的分布式运行时。

从职责上看，可以把相关组件分成三层：

| 层次                                      | 主要职责                           |
| --------------------------------------- | ------------------------------ |
| Kubernetes、云平台或裸机                       | 提供节点、容器、网络、磁盘和 GPU 等基础资源       |
| Ray Core                                | 提供任务、Actor、对象存储、资源调度、故障恢复和集群管理 |
| Ray Data / Train / Tune / RLlib / Serve | 面向数据、训练、调参、强化学习和在线推理的高级能力      |

Ray 的一个重要设计是“用普通 Python 表达分布式程序”。开发者通常不需要为每个函数手写 RPC 接口，也不需要手动管理每个 Worker 的生命周期。只要给函数或类增加 @ray.remote，就可以把它注册为可远程执行的任务或 Actor。

-
-
-
-
-
-
-
-
-
-
- 

```
import ray
``````

``````
ray.init()
``````

``````
@ray.remote
``````
def square(x):
``````
    return x * x
``````

``````
refs = [square.remote(i) for i in range(8)]
``````
results = ray.get(refs)
``````
print(results)
```

这里的 square.remote(i) 不会立即返回计算结果，而是返回一个对象引用。任务可以被调度到集群中的任意节点执行，调用方稍后通过 ray.get 获取结果。

## 2\. 为什么 AI 工作负载需要 Ray

传统分布式系统通常围绕固定的数据处理作业设计，而 AI 应用具有更强的动态性：

- 一个训练任务可能需要同时占用多张 GPU，并且要求 GPU 之间低延迟通信；

- 超参数搜索会并发启动成百上千个相互独立的试验；

- 强化学习会反复创建环境、采样轨迹、更新策略和评估模型；

- 大模型服务既要管理长生命周期的模型副本，又要处理短生命周期的请求；

- 推理流水线中，不同阶段可能运行在不同 GPU、不同节点甚至不同硬件上；

- 数据预处理、训练、评估和部署往往需要共享数据和中间结果。

仅靠进程池，通常只能解决单机并发；仅靠批处理系统，任务之间的数据传递和低延迟交互又不够灵活；仅靠 Kubernetes，则需要应用自己处理任务编排、对象传输、Actor 生命周期和训练拓扑。

Ray 把这些需求归纳为几种通用原语：

- Task：一次性的无状态远程函数调用；

- Actor：带状态、可被远程调用的长期运行对象；

- Object：任务和 Actor 之间传递的不可变数据对象；

- Resource：CPU、GPU、内存以及用户自定义资源；

- Placement Group：为多进程、多 GPU 工作负载预留一组资源。

高级组件都建立在这些原语之上，因此训练、调参和在线服务可以使用相同的集群调度和故障处理机制。

## 3\. Ray Core 的基本编程模型

### 3.1 Task：适合无状态并行计算

Task 是最轻量的分布式执行单元。函数调用会被序列化并提交到 Ray 集群，调度器根据资源需求选择 Worker 执行。

-
-
-
-
-
-
-
-
-
-
- 

```
import ray
``````

``````
ray.init()
``````

``````
@ray.remote(num_cpus=2)
``````
def preprocess(path):
``````
    # 读取文件、清洗数据并返回中间结果
``````
    return {"path": path, "rows": 1000}
``````

``````
refs = [preprocess.remote(path) for path in paths]
``````
results = ray.get(refs)
```

Task 适合以下场景：

- 数据切分和批量预处理；

- 独立的模型评估任务；

- 图片、音频或文本的并行转换；

- 不需要保留内部状态的 CPU/GPU 计算。

Task 默认可以在失败后重试，但是否安全重试取决于函数是否具有幂等性。如果任务已经向外部数据库写入数据，再自动重试可能产生重复副作用，因此需要在业务层设计去重或事务机制。

### 3.2 Actor：适合有状态的长期对象

Actor 把一个 Python 类实例放到 Ray 集群中。类的状态保留在远程进程内，调用者只传递方法请求。

-
-
-
-
-
-
-
-
-
-
-
- 

```
@ray.remote
``````
class Counter:
``````
    def __init__(self):
``````
        self.value = 0
``````

``````
    def add(self, n):
``````
        self.value += n
``````
        return self.value
``````

``````
counter = Counter.remote()
``````
print(ray.get(counter.add.remote(3)))
``````
print(ray.get(counter.add.remote(5)))
```

Actor 很适合：

- 常驻内存的模型实例；

- 数据库连接池、缓存和状态聚合器；

- 强化学习中的环境或策略对象；

- 需要串行处理请求、但又希望横向复制的服务副本。

Actor 方法默认按顺序执行。如果需要同一个 Actor 并发处理多个请求，可以配置并发组（concurrency groups）或采用异步 Actor，但必须自行确认共享状态是否线程安全。

### 3.3 Object：任务之间传递数据

Ray 使用对象引用表示远程数据。任务返回值会被放入 Ray 的对象存储中，调用方拿到的是 ObjectRef，而不是数据本身。

-
-
-
-
-
-
-
- 

```
large_data = ray.put(load_large_dataset())
``````

``````
@ray.remote
``````
def consume(data_ref, shard_id):
``````
    data = ray.get(data_ref)
``````
    return process_shard(data, shard_id)
``````

``````
refs = [consume.remote(large_data, i) for i in range(8)]
```

将大对象先放入对象存储，再把引用传给多个任务，可以避免在每次提交任务时重复序列化和传输数据。对象在本地节点可直接读取；如果任务被调度到其他节点，Ray 会通过对象管理机制获取数据。

对象引用本身很轻量，但对象数据并不是无限期保存的。对象存储容量不足时可能发生溢出到本地磁盘，磁盘、网络和序列化开销都会影响性能。因此，应该尽量使用分片、流式处理和列式格式，避免把整个数据集一次性放入对象存储。

### 3.4 资源声明与自定义资源

远程函数和 Actor 可以声明 CPU、GPU、内存等需求：

-
-
- 

```
@ray.remote(num_gpus=1, num_cpus=4)
``````
def infer(batch):
``````
    return model(batch)
```

Ray 也支持自定义资源。例如，可以在节点启动时声明某节点拥有特定型号的 GPU、某种本地存储或某个业务标签，然后在任务中请求该资源：

-
-
- 

```
@ray.remote(resources={"accelerator_a": 1})
``````
def run_on_special_device(job):
``````
    return run(job)
```

资源声明是调度约束，而不是完整的隔离机制。它能帮助 Ray 做容量管理和放置决策，但不等价于 MIG、容器安全边界或硬件级 QoS。对于 GPU 显存、NUMA、网络拓扑等要求更严格的工作负载，还需要结合运行时配置和底层平台能力。

## 4\. Ray 的集群架构

一个典型 Ray 集群包含 Head 节点和 Worker 节点。Head 节点运行集群控制服务，也可以同时执行任务；Worker 节点负责实际计算。

-
-
-
-
-
-
-
-
-
-
-
- 

```
                    Ray 集群
``````
    ┌─────────────────────────────────────────┐
``````
    │ Head Node                                │
``````
    │ ├── Global Control Store                 │
``````
    │ ├── 集群级调度与元数据                   │
``````
    │ ├── Dashboard / Job 管理                 │
``````
    │ └── Raylet + Object Manager              │
``````
    │                                         │
``````
    │ Worker Node 1 ─ Raylet ─ Object Store   │
``````
    │ Worker Node 2 ─ Raylet ─ Object Store   │
``````
    │ Worker Node 3 ─ Raylet ─ Object Store   │
``````
    └─────────────────────────────────────────┘
```

### 4.1 Raylet

每个节点都有一个 Raylet，负责本节点的资源管理、任务执行协调、Worker 进程管理以及对象传输。Raylet 会向集群报告本节点可用资源，并根据调度决策启动或复用 Python Worker。

### 4.2 Global Control Store

Global Control Store（GCS）保存集群级的控制状态和元数据，例如节点、任务、Actor、对象位置以及部分调度信息。它让多个节点能够看到一致的集群视图，并为故障检测、任务恢复和 Dashboard 提供基础信息。

### 4.3 Object Store 与数据传输

每个节点都拥有本地对象存储。任务产生的返回值通常先写入本地对象存储，其他任务需要时再通过节点间网络拉取。对于大对象，数据传输往往比调度本身更容易成为瓶颈。

因此，在设计 Ray 应用时应关注：

- 对象大小和生命周期；

- 任务是否会反复跨节点读取同一对象；

- 数据是否可以按分片放置在计算附近；

- 是否需要使用 Ray Data 的流式执行来控制内存；

- 节点之间是否有足够的带宽和低延迟网络。

### 4.4 调度流程

一次远程任务大致经历以下步骤：

-
-
-
-
-
-
-
-
-
-
-
-
-
- 

```
Python Driver
``````
    │ task.remote()
``````
    ▼
``````
任务提交与资源需求登记
``````
    ▼
``````
Ray 调度器选择节点
``````
    ▼
``````
目标节点 Raylet 获取或启动 Worker
``````
    ▼
``````
Worker 拉取输入对象并执行函数
``````
    ▼
``````
结果写入对象存储
``````
    ▼
``````
Driver 通过 ObjectRef 获取结果
```

Ray 会尽量利用本地性：如果输入对象已经位于某个节点，调度器可以优先选择该节点，减少网络传输。不过，本地性只是调度目标之一，还要和资源可用性、负载均衡、Actor 放置和故障恢复等因素一起考虑。

## 5\. Placement Group：为分布式训练预留资源

很多 AI 工作负载不是“要一张 GPU”这么简单，而是要求一组进程同时获得多张 GPU，甚至要求固定的节点拓扑。例如，一个 8 卡训练进程组可能需要 8 个 GPU；模型并行还可能要求特定数量的 CPU、GPU 和内存组合。

Placement Group 可以把多个资源请求捆绑成一组 bundle，并以某种放置策略提交：

- `PACK`

  ：尽量把资源放在较少的节点上，适合减少跨节点通信；

- `SPREAD`

  ：尽量分散到不同节点，适合提高容错或利用多台机器；

- `STRICT_PACK`

  ：所有 bundle 必须放在同一个节点；

- `STRICT_SPREAD`

  ：每个 bundle 必须位于不同节点。

分布式训练框架通常会使用 Placement Group 管理 Worker 组。资源预留成功后，训练任务才能启动，从而避免已经启动一部分进程、剩余进程长期等待的“半启动”状态。

## 6\. Ray Data：面向 AI 的分布式数据处理

Ray Data 用于读取、转换、批处理和写出大规模数据。它的设计重点不是替代所有传统数仓，而是为机器学习流水线提供可流式、可扩展、能直接连接训练和推理的执行引擎。

-
-
-
-
-
-
-
-
- 

```
import ray
``````

``````
ds = ray.data.read_parquet("s3://bucket/train/")
``````
ds = ds.map_batches(
``````
    tokenize,
``````
    batch_size=1024,
``````
    num_cpus=2,
``````
)
``````
ds.write_parquet("s3://bucket/tokenized/")
```

Ray Data 的常见能力包括：

- 从 Parquet、CSV、JSON、图片和对象存储读取数据；

- 按行或按批次执行 Python 函数；

- 并行 Tokenize、图片解码和特征生成；

- 对数据进行 shuffle、repartition、filter 和 sort；

- 以流式方式向训练器或推理服务提供 batch；

- 与 PyTorch、TensorFlow、NumPy、Arrow 等生态衔接。

它通常采用流水线执行：读取、转换和写出可以重叠进行，不必等待整个阶段全部完成后再进入下一阶段。对于数据规模大、单条样本处理成本高或需要连接 GPU 推理的任务，这种方式比简单地把全部数据收集到 Driver 更稳健。

需要注意的是，Ray Data 并不自动解决数据治理、事务一致性和数据质量问题。生产环境仍然需要设计 schema、分区、检查点、输出幂等性和失败重跑策略。

## 7\. Ray Train：分布式训练编排

Ray Train 负责启动和协调分布式训练 Worker，并把 checkpoint、指标、故障恢复和资源申请接入 Ray 集群。它可以封装 PyTorch、DeepSpeed、Hugging Face、XGBoost 等训练代码。

一个简化的训练函数如下：

-
-
-
-
-
-
- 

```
from ray import train
``````

``````
def train_loop(config):
``````
    model = build_model(config)
``````
    for epoch in range(config["epochs"]):
``````
        loss = train_one_epoch(model)
``````
        train.report({"loss": loss})
```

Ray Train 主要解决的是“如何启动和管理一组训练进程”，而不是替代 PyTorch Distributed、NCCL 或模型并行库。GPU 之间的梯度同步、通信拓扑和算子执行仍由底层训练框架负责。

生产训练任务通常还需要关注：

- checkpoint 是否保存到高可靠的对象存储；

- Worker 故障后能否从最近的 checkpoint 恢复；

- GPU 和节点是否满足拓扑要求；

- 数据读取是否成为训练瓶颈；

- 训练指标和系统指标是否可以统一观测。

## 8\. Ray Tune：超参数搜索与实验管理

Ray Tune 用于并行运行大量试验，并提供搜索算法、早停策略和资源管理。例如，可以同时搜索学习率、batch size 和模型层数：

-
-
-
-
-
-
-
-
-
-
-
-
-
-
-
-
- 

```
from ray import tune
``````

``````
search_space = {
``````
    "lr": tune.loguniform(1e-5, 1e-2),
``````
    "batch_size": tune.choice([32, 64, 128]),
``````
}
``````

``````
tuner = tune.Tuner(
``````
    train_loop,
``````
    param_space=search_space,
``````
    tune_config=tune.TuneConfig(
``````
        num_samples=20,
``````
        metric="loss",
``````
        mode="min",
``````
    ),
``````
)
``````
results = tuner.fit()
```

当试验数量较多时，真正影响效率的并不只是搜索算法，还包括资源利用率和无效试验的淘汰。ASHA 等早停策略会根据中间指标终止表现较差的试验，把资源让给更有潜力的配置。Tune 还可以和 Train 结合，使每个超参数试验内部运行多 GPU 分布式训练。

## 9\. RLlib：强化学习平台

强化学习通常包含环境交互、轨迹采样、策略更新和评估等环节，并且这些环节具有不同的计算特征。RLlib 基于 Ray 的 Task 和 Actor 模型，把环境、Rollout Worker、Learner 和评估器组织成可扩展的执行图。

-
-
-
-
-
-
- 

```
环境 Actor ──采样轨迹──▶ Rollout Worker
``````
                              │
``````
                              ▼
``````
                         Learner 更新策略
``````
                              │
``````
                              ├──▶ 广播新权重
``````
                              └──▶ Evaluator 评估
```

RLlib 支持常见的策略梯度、价值函数和多智能体训练场景，但复杂项目仍然需要自行处理环境性能、样本吞吐、策略同步频率和检查点管理。Ray 提供的是分布式执行基础，算法效果仍取决于模型、奖励函数和训练配置。

## 10\. Ray Serve：在线推理与模型服务

Ray Serve 是构建 Python 模型服务和多模型推理流水线的组件。它把模型封装为 Deployment，并负责副本管理、请求路由、并发控制、批处理和扩缩容。

-
-
-
-
-
-
-
-
-
-
-
-
- 

```
from ray import serve
``````

``````
@serve.deployment(num_replicas=2)
``````
class Model:
``````
    def __init__(self):
``````
        self.model = load_model()
``````

``````
    async def __call__(self, request):
``````
        payload = await request.json()
``````
        return self.model(payload["inputs"])
``````

``````
app = Model.bind()
``````
serve.run(app)
```

Ray Serve 的关键能力包括：

- 以 Actor 形式长期持有模型，避免每次请求重复加载；

- 根据副本数量和并发配置扩展服务；

- 支持多个 Deployment 组成推理 DAG；

- 对请求进行异步转发和批处理；

- 将模型服务与 Ray 集群资源调度统一起来；

- 支持滚动更新、版本切换和故障恢复。

对于大模型服务，Ray Serve 可以作为模型编排层，但并不等于 vLLM、SGLang 或 TensorRT-LLM。更常见的组合是：由 vLLM 等引擎负责单实例内的高性能推理，由 Ray Serve 负责 Python 层的多模型编排、业务逻辑、动态批处理和副本管理。对于极高吞吐、强 SLO 的生产系统，还需要结合专门的网关、缓存、流量治理和 GPU 调度方案。

## 11\. Ray on Kubernetes：KubeRay

在生产环境中，Ray 集群通常运行在 Kubernetes 上。KubeRay 是 Ray 社区提供的 Kubernetes Operator 和相关 CRD，用于创建、升级和管理 Ray 集群及 Ray 作业。

KubeRay 常见资源包括：

| 资源         | 作用                            |
| ---------- | ----------------------------- |
| RayCluster | 声明 Head、Worker、资源规格和 Ray 集群配置 |
| RayJob     | 提交一个 Ray 作业，并管理作业生命周期         |
| RayService | 部署 Ray Serve 应用，并提供服务级滚动更新和恢复 |

典型部署关系如下：

-
-
-
-
-
-
-
- 

```
Kubernetes API
``````
      │
``````
      ▼
``````
KubeRay Operator
``````
      │
``````
      ├── RayCluster：Head + Worker Pod
``````
      ├── RayJob：提交和监控一次性任务
``````
      └── RayService：管理 Ray Serve 应用
```

Kubernetes 负责 Pod 调度、节点故障恢复、GPU 设备分配、网络和存储；Ray 负责集群内部的任务、Actor、对象和应用级资源调度。两层职责需要明确区分：Kubernetes 的 HPA 不理解 Ray 任务队列，Ray 的资源调度也不应绕过 Kubernetes 的设备和安全边界。

在 GPU 集群中，部署 KubeRay 时还要考虑：

- GPU Operator、驱动和容器运行时是否正常；

- Worker Pod 是否有足够的共享内存和本地磁盘；

- Head 节点是否被错误地安排到高成本 GPU 节点；

- 节点亲和性、拓扑和网络带宽是否满足训练需求；

- Ray 的自动扩缩容与 Cluster Autoscaler 是否会相互等待；

- Ray Dashboard、日志和指标是否纳入统一观测系统。

## 12\. Ray 的容错与可观测性

Ray 可以检测节点、Worker、Task 和 Actor 的部分故障，并根据配置重试任务或重建 Actor。但它不是事务系统，不能保证任意副作用都能自动恢复。

设计可靠 Ray 应用时，建议遵循以下原则：

- GPU Operator、驱动和容器运行时是否正常；

- Worker Pod 是否有足够的共享内存和本地磁盘；

- Head 节点是否被错误地安排到高成本 GPU 节点；

- 节点亲和性、拓扑和网络带宽是否满足训练需求；

- Ray 的自动扩缩容与 Cluster Autoscaler 是否会相互等待；

- Ray Dashboard、日志和指标是否纳入统一观测系统。

Ray Dashboard 可以查看节点、任务、Actor、对象和资源使用情况。生产环境通常还会把 Ray 指标接入 Prometheus，把日志接入集中式日志系统，并将训练指标、推理延迟和业务指标关联起来。

## 13\. Ray 与其他系统的区别

### 13.1 Ray 与 Kubernetes

Kubernetes 是通用容器编排平台，擅长管理 Pod、节点、网络、存储和权限。Ray 是面向分布式 Python 和 AI 工作负载的应用运行时，擅长任务、Actor、对象和 AI 资源调度。

二者通常是上下层关系，而不是互相替代：Kubernetes 管理 Ray 集群，Ray 管理集群内的 AI 应用。

### 13.2 Ray 与 Apache Spark

Spark 以大规模结构化数据处理和 SQL 分析见长，拥有成熟的数据湖、批处理和流处理生态。Ray 更强调通用 Python 计算、动态任务、Actor 状态和 AI 工作负载。

如果任务主要是 SQL、ETL 和数仓处理，Spark 往往更自然；如果任务包含模型训练、在线推理、强化学习或大量 Python 自定义逻辑，Ray 通常更灵活。两者也可以组合使用，例如用 Spark 生成数据集，再由 Ray Train 训练模型。

### 13.3 Ray 与 Dask

Dask 同样提供 Python 分布式计算和任务图执行，适合 NumPy、Pandas 等科学计算。Ray 的差异在于更强调 Actor、动态资源、GPU、故障恢复和完整 AI 应用栈。对于简单数组和表格计算，Dask 可能更轻量；对于训练、服务和复杂有状态工作流，Ray 的组件更完整。

## 14\. 常见误区与性能陷阱

### 14.1 在 Driver 中频繁 ray.get

下面的代码会把循环变成串行执行：

-
- 

```
for item in items:
``````
    result = ray.get(process.remote(item))
```

更好的方式是先提交一批任务，再统一获取结果：

-
- 

```
refs = [process.remote(item) for item in items]
``````
results = ray.get(refs)
```

### 14.2 把大对象重复捕获进闭包

如果一个巨大模型或数据集被闭包反复序列化，任务提交和内存占用都会恶化。应该显式使用 ray.put、Actor 或分片数据集，并检查对象是否发生了不必要的跨节点复制。

### 14.3 用一个 Actor 承担所有请求

单个 Actor 很容易成为吞吐瓶颈。应根据模型加载成本、请求并发和 GPU 资源创建多个副本，并结合批处理、异步方法或 Serve 的 Deployment 配置控制并发。

### 14.4 盲目增加并行度

并行任务越多不一定越快。过高的并发会导致对象存储爆满、网络拥塞、上下文切换和下游限流。应根据计算、内存、网络和外部系统容量设置合理的 batch、并发和队列长度。

### 14.5 把资源声明当成硬隔离

Ray 的 num\_gpus 和自定义资源主要用于调度，不自动提供显存配额、带宽隔离或安全隔离。多租户环境仍需要 Kubernetes、容器运行时、MIG、配额和权限系统共同保障。

## 15\. 什么时候适合使用 Ray

Ray 特别适合以下场景：

- 需要用 Python 快速构建多节点分布式应用；

- 训练、数据处理、调参和服务需要共享同一个资源池；

- 工作负载具有动态任务、长生命周期状态或异构资源需求；

- 需要同时使用 CPU、GPU、自定义资源和多节点拓扑；

- 希望在本地单机和生产集群之间尽量复用代码。

以下场景则不一定需要 Ray：

- 只有一个简单的单机训练脚本；

- 主要需求是成熟 SQL 数仓和离线 ETL；

- 已经有稳定的专用推理平台，且不需要 Python 层编排；

- 团队无法承担分布式系统的监控、容量规划和故障排查成本。

## 16\. 总结

Ray 的核心价值不在于某一个 API，而在于它把分布式 AI 应用中反复出现的能力统一起来：

-
-
-
-
- 

```
Task       ── 无状态并行计算
``````
Actor      ── 有状态长期运行对象
``````
Object     ── 跨任务的数据传递
``````
Resource   ── CPU、GPU 和自定义资源调度
``````
Placement  ── 多进程、多 GPU 的资源编排
```

在这些基础原语之上，Ray Data 负责数据流水线，Ray Train 负责分布式训练，Ray Tune 负责实验搜索，RLlib 负责强化学习，Ray Serve 负责在线模型服务，KubeRay 则把 Ray 集群接入 Kubernetes。

因此，Ray 更适合被理解为“AI 应用的分布式运行时和平台层”，而不是单纯的任务队列或训练库。它可以从一个本地 Python 脚本开始，逐步扩展到多节点 GPU 集群；但当系统进入生产阶段后，仍然需要认真处理数据本地性、网络、对象存储、故障恢复、资源隔离和可观测性等问题。
