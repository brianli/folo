---
title: "从 On-Policy Baseline 到 Off-Policy Score Centering！深度解析 score-centering 的前世今生"
source: "大模型智能"
category: "技术文章"
group: "AI 技术"
url: "https://mp.weixin.qq.com/s?__biz=MzU3NjE4NjQ4MA==&mid=2247557366&idx=1&sn=b522c87ab97575e839033d58e5206e26"
published: 2026-09-27T00:00:00+08:00
saved: 2026-09-27T22:16:51+08:00
folo_key: "s-9d016b1124::b3c252ca0b47c301ed7a91dc7457ca6e"
tags:
  - "folo"
  - "技术文章"
  - "大模型智能"
---

# 从 On-Policy Baseline 到 Off-Policy Score Centering！深度解析 score-centering 的前世今生

> [!info] 大模型智能 · 技术文章 · 2026-09-27 00:00 · [原文](https://mp.weixin.qq.com/s?__biz=MzU3NjE4NjQ4MA==&mid=2247557366&idx=1&sn=b522c87ab97575e839033d58e5206e26)

大模型智能 2026-09-27 00:00 吉林

![[从 On-Policy Baseline 到 Off-Policy Score-01.jpg]]

Off-policy强化学习，采样与目标策略的差异会导致传统基线失效引发梯度漂移。ScoreCentering通过将Score在采样分布下中心化，恢复了基线不变性并消除了均值漂移。其梯度等价于特定的STE，可与TIS/MIS结合以修正截断偏差，但并未完全解决分布不匹配的根本偏差

![[从 On-Policy Baseline 到 Off-Policy Score-02.gif]]

大模型智能｜分享

来源 | 青稞AI

作者 | haotian

![[从 On-Policy Baseline 到 Off-Policy Score-03.png]]

***01***

**从 On-Policy Baseline 到 Off-Policy Score Centering**

### 1\. 问题设定

核心问题是：

> 在 on-policy 情况下，任意只依赖状态的  都是合法 baseline；但当样本来自 、score 来自  时， 不再自动消失。

严格来说，baseline 恒等式是一个**梯度恒等式**，而不是两个标量 surrogate loss 的数值恒等式。因此，应从 policy gradient 出发，而不是直接声称

固定状态 ，定义目标策略的 score：

定义 advantage：

以下假设在 actor 更新中， 与  都经过 stop-gradient。

### 2\. On-policy： 是合法 baseline

当 action 来自目标策略

score-function identity 给出

on-policy 的 -based policy gradient 为

代入

得到

因此

也就是

所以，在 on-policy 情况下，任意与 action 无关的  都不会改变期望梯度。

### 3\. Off-policy：引入  会改变梯度

现在假设 action 来自 behavior policy

但使用的仍然是目标策略  的 score：

定义  分布下的平均 score：

off-policy 的 -based update 为

代入 ：

因此

等价地，

当  时，一般有

所以

这说明，在 off-policy 情况下，从  换成  不再只是 variance reduction，而会真正改变梯度方向。

### 4\. 额外项与 reverse KL 梯度的关系

如果在 actor 更新时将  视为固定分布，那么

对  求梯度：

因此

代入前面的结果：

也就是说，直接使用  的 off-policy update，会额外包含一个由  缩放的 reverse-KL 梯度项。

若采用梯度上升，并且 ，这一项倾向于减小

而从  换成 ，相当于删除了这个隐式的 KL 拉回项。

### 5\. Reward shift invariance 被破坏

考虑对所有 action 的  加上同一个状态相关常数：

在 on-policy 情况下：

但是在 off-policy 情况下：

一般来说，

因此，仅仅改变 reward 或 value 的零点，就可能改变 off-policy 更新方向。这就是 mean-score drift。

### 6\. Score centering 恢复 baseline invariance

定义 centered score：

根据定义，

使用 centered score 的更新为

代入 ：

因此

score centering 在 -sampling 下重新恢复了 baseline invariance：使用原始 score 时， 在  下不是合法 baseline；使用 centered score 后，任意  又重新成为合法 baseline。

### 7\. Score centering 的协方差形式

展开 score-centering update：

因此

它也可以写成

因此，score centering 在期望意义上等价于使用特殊的 baseline：

### 8\. 普通 Advantage Update 与 Score Centering 的区别

普通 advantage update 为

score-centering update 为

展开得到

所以

二者只有在下面条件成立时才相等：

例如，如果

那么

从而

但是，如果使用目标策略的 value

一般不能推出

所以，使用  的普通 advantage update 与 score-centering update 一般并不相同。

### 9\. Exact Importance Sampling

定义 importance ratio：

使用 exact importance sampling 时：

对于状态 baseline：

因此

所以 exact importance sampling 也能恢复 baseline invariance。但是 clipped ratio、noisy ratio、approximate ratio 或 support mismatch 都可能破坏这个精确等式。

需要注意：上面的推导只修正给定状态  后的 action distribution mismatch；如果状态访问分布也来自  而不是 ，还会保留 state-distribution mismatch。

### 10\. 总结

On-policy 情况下：

Off-policy 且不使用 importance sampling 时：

使用 score centering 后：

最终结论是：

> sampling policy 从  变成  后，状态函数  不再自动成为合法 baseline，因为目标策略 score 在  下的期望不为零。  Score centering 通过强制该期望为零，恢复了 baseline invariance；但它得到的仍然是  分布下的协方差，并没有消除  与  之间的一般 off-policy bias。

***02***

**Score Centering 与 MIS/TIS 组合的完整形式化**

### 1\. 核心结论

Score Centering 与 MIS/TIS 的组合，并不是使用 MIS/TIS 来估计原始期望

正确的理解是：

- • 首先使用 MIS/TIS 将原始 score 变成 weighted score；

- • 然后计算 weighted score 在 sampler 分布下的期望；

- • 最后从 sampled weighted score 中减去这个期望。

即：

最终得到：

因此，MIS/TIS + Score Centering 在期望意义上等价于：对 MIS/TIS weighted score 使用一个精确的 prefix-level sampler value

但不需要显式训练 critic。

本文主要参考：

```
论文：Score Centering Stabilizes Off-policy Reinforcement Learning  链接：[https://arxiv.org/pdf/2609.20807](https://arxiv.org/pdf/2609.20807)
```

### 2\. 固定 Prefix 下的记号

固定一个 token prefix：

令  表示词表中的候选 token，定义 trainer policy：

定义 sampler 或 behavior policy：

定义目标策略的 token score：

目标策略满足 score-function identity：

定义 token-level importance ratio：

再引入任意非负 weight function：

不同方法对应不同的 ：

| 方法                      | Weight function |
| ----------------------- | --------------- |
| Vanilla Score Centering |                 |
| Exact IS                |                 |
| TIS                     |                 |
| MIS                     |                 |

论文实验中使用：

以及：

### 3\. 通用 Weighted Policy Gradient

令：

给定当前 token 为  后，定义：

sampler  下的 prefix value 为：

暂时将  简写为 。未经 score centering 的 weighted update 为：

定义 weighted mean score：

离散 token 空间下：

### 4\. 引入任意  时的 Residual Drift

对于任意只依赖状态的 baseline ，定义：

则：

所以：

只要：

那么  就不再是合法 baseline。从  换成  会真正改变期望梯度，而不只是降低方差。

### 5\. Exact IS 为什么不需要 Score Centering

对于 exact importance sampling：

因此：

所以：

进而：

因此，exact IS 已经恢复了 weighted score 的零均值。再做 Score Centering 时，centering correction 恰好为零。

但是 TIS/MIS 对 importance ratio 进行了 clipping 或 masking，通常有：

以及：

因此 TIS/MIS 仍然会留下 residual drift，可以继续与 Score Centering 组合。

### 6\. 通用 Weighted Score Centering

定义 generalized weighted score：

它在  下的条件期望为：

定义 centered weighted score：

于是：

Score Centering 加到通用 importance weight 后的更新为：

对于任意 ：

所以：

这说明 weighted Score Centering 对任意 weight function  都能重新恢复 baseline invariance。

### 7\. 与精确  的等价关系

展开 weighted Score Centering：

因此：

如果直接使用 Monte Carlo return ，同样有：

其中：

所以：

> TIS/MIS + Score Centering，在期望意义上等价于 TIS/MIS 配合一个精确的 prefix-level -value baseline，但不需要显式训练 critic。

### 8\. 协方差形式

由：

可得：

即：

需要注意：

一般不等于：

Score Centering 消除了 weighted update 在  下的均值漂移，但没有自动把整个 covariance measure 变成 。

### 9\. TIS + Score Centering

TIS 使用：

于是：

weighted mean score 为：

利用：

可以改写为：

因此，TIS 的 residual mean score 完全来自被 upper clipping 的 token。

TIS + Score Centering 更新为：

等价地：

TIS 部分修正 action measure，使其更接近 ；Score Centering 消除 ratio clipping 后残留的 mean-score drift。

### 10\. MIS + Score Centering

MIS 使用：

因此：

weighted mean score 为：

利用：

得到：

所以，MIS 的 residual mean score 等于所有被 mask token 的 target-policy score mass 的负值。

MIS + Score Centering 更新为：

等价地：

### 11\. 不同方法对应的 Effective Token Measure

定义未归一化的 effective token measure：

Vanilla Score Centering：

Exact IS：

TIS：

MIS：

因此：

- • Score Centering 只消除  measure 下的 drift；

- • exact IS 将当前 action measure 精确变成 ；

- • TIS/MIS 将 measure 部分推向 ，同时引入 clipping 或 masking bias；

- • TIS/MIS + Score Centering 再消除 bounded IS 剩余的 mean-score drift。

### 12\. Top- Sampler Distribution 重建

保存 sampler 的全词表概率通常代价过高。论文只保存 sampler 的 top- log-probabilities，并用 trainer distribution 重建 tail。

设：

以及：

定义 sampler 和 trainer 的 tail mass：

定义 tail scaling：

使用下面的分布近似真实 sampler：

这个构造满足：

在 tail 上，估计的 importance ratio 是常数：

因此，tail weight 也是常数：

定义：

### 13\. Weighted Mean Score 的 Head-only 形式

在重建分布  下：

因为：

所以：

最终得到：

这就是论文中 weighted Score Centering 的 head-only 形式。虽然 tail model 覆盖整个词表，但实际 correction graph 只需要 top- token。

### 14\. 不同方法对应的 

#### 14.1 Vanilla Score Centering

当：有：

#### 14.2 Exact IS

当：有：

同时，head 上：

因此：

#### 14.3 TIS

当：

有：

所以：

#### 14.4 MIS

当：

有：

因此：

对于论文使用的参数：

得到：

### 15\. 最终 Scalar Loss

通用的 MIS/TIS + Score Centering loss 可以写成：

其中：

所有作为 coefficient 出现的量都必须 stop-gradient，包括：

特别需要注意，correction coefficient 中的  也必须被 detach，否则会产生额外梯度项。

对 loss 求梯度：

sampled token 即使不在 top- head 中，也需要保存其 sampler log-probability ，以计算：

### 16\. Top- 近似误差

定义真实 weighted mean score 与 top- 近似之间的误差：

真实期望下，实际更新为：

所以：

这说明：

- • full-vocabulary expectation 下，drift cancellation 是精确的；

- • top- 版本对重建分布  的 cancellation 是精确的；

- • 对真实  是否精确，取决于 tail model 的误差 。

在 constant reward 情况下：

更新退化为：

因此，一个直接的实现诊断是：使用 constant reward 或随机打乱 reward，检查平均更新是否接近零。

### 17\. Token-level IS 的重要限制

即使使用 exact token ratio：

如果后续 rollout 仍然由  生成，那么：

它一般不是：

所以 token-level IS 只修正当前 action distribution，并没有自动修正未来 continuation distribution。

要完整变换从时间  开始的 trajectory measure，需要使用：

但 trajectory ratio 的方差通常过大，所以实践中经常只使用 token-level TIS/MIS，并接受其偏差。

### 18\. Fully Async 与 Multi-Behavior Policy

在 fully asynchronous rollout 中，不同 token 或不同 response 可能来自不同 generator：

对于 token ，应使用实际生成该 token 的 policy：

对应的 weighted mean score 应为：

正确的 centered update 是：

如果实际 generator 是 ，却使用另一个  或全局近似分布计算 correction，则残余漂移为：

对应的额外 drift 为：

因此，在 multi-version rollout 中，最稳妥的做法是保存每个 token 对应 generator 的：

- • model version；

- • sampled-token log-probability；

- • sampler top- token IDs；

- • sampler top- log-probabilities；

- • sampler tail mass。

然后对每个 token 使用自己的 generator  计算 TIS/MIS 与 Score Centering correction。

### 19\. 最终统一表达

定义：

以及：

通用的 MIS/TIS + Score Centering update 为：

它等价于：

也等价于：

最终应将方法理解为：

> TIS/MIS 负责对 action measure 进行有偏但低方差的部分修正；Score Centering 负责消除 TIS/MIS clipping 或 masking 后留下的 weighted-score mean drift。二者组合恢复了 baseline invariance，但一般仍不是无偏的 on-policy policy gradient。

***03***

**Score Centering 与 STE 的梯度等价关系**

### 1\. 核心结论

Score Centering 可以理解为一种特殊的 Straight-Through Estimator（STE）：

其中：

- •  是 trainer policy 的 logits；

- •  是 sampler / behavior policy 的 logits；

- •  表示 stop-gradient。

这个构造具有两个性质：

- • 前向传播使用 ，因此 softmax 分布为 ；

- • 反向传播沿  的计算图传递梯度。

因此，它产生的参数梯度与 full-distribution Score Centering 完全相同：

这里的“等价”是**梯度等价**，而不是前向 loss 数值相同。

### 2\. 问题设定

在某个 token prefix  上，定义：

 是 trainer policy， 是实际生成 rollout 的 sampler / behavior policy，并且：

以下省略条件 。令  表示动作  对应的 one-hot 向量，trainer logits 对参数的 Jacobian 为：

### 3\. 普通 off-policy Policy Gradient 的 drift

对于 trainer policy：

因此参数空间中的 score 为：

普通 policy-gradient 更新为：

但是动作来自 ，而不是 。因此：

进而：

只要 ，trainer score 在 behavior policy  下的均值就不再为零。

利用协方差分解：

得到：

第一项只依赖 advantage 的均值，不包含动作与 advantage 之间的相关性。它会把 trainer 朝 sampler 的方向蒸馏，是 training-inference mismatch 下的额外漂移。

### 4\. Score Centering 如何消除 drift

Score Centering 定义 centered score：

在 logit 空间中：

所以：

最终得到：

因此 Score-Centered policy-gradient 更新为：

由于 ：

所以 centered score 在每个 prefix 上都满足：

进而：

这正好消除了：

这一 drift 项。

### 5\. 为什么它等价于 STE

构造 STE logits：

#### 5.1 前向传播

前向数值为：

因此：

#### 5.2 反向传播

由于 stop-gradient 内部不传梯度：

于是：

继续对参数  求梯度：

而上一节已经得到：

因此：

这说明 full Score Centering 与该 STE 构造产生完全相同的单样本参数梯度。

### 6\. 两种 surrogate loss 的对应关系

#### 6.1 Score Centering loss

Score Centering 可以写为：

对参数求梯度：

即：

#### 6.2 STE loss

相应的 STE surrogate 可以写成：

它的梯度同样是：

因此：

但是两者的前向 loss 数值并不相同。STE 前向时 ，所以：

而  是另一个 surrogate。因而：

> 二者是梯度等价，不是目标函数数值等价；不能直接比较两者上报的 loss value。

### 7\. 最直观的理解

普通 off-policy PG 的 logit 梯度是：

样本  来自 ，所以 one-hot 向量的均值是：

但普通 PG 减去的是 ，因此残留：

Score Centering / STE 把梯度变成：

此时样本均值与被减掉的分布完全相同：

所以这个 STE 的本质不是让 sampler 可微，而是：

> 用 sampler 分布  充当 softmax 梯度中的负相，同时保留 trainer logits  对参数的 Jacobian。

可以概括为：

| 部分            | 使用的对象                             |
| ------------- | --------------------------------- |
| rollout 动作    |                                   |
| 前向 softmax 分布 | sampler q                         |
| 反向 Jacobian   | trainer 的                         |
| 最终 logit 梯度   |                                   |
| 对应方法          | full-distribution Score Centering |

### 8\. 它没有解决什么

Score Centering 消除的是：

这一 drift，而不是把 behavior distribution  修正成 target distribution 。

Score Centering 后的期望更新仍然是：

而目标 on-policy 更新对应：

其中第二个等式要求采用适当的 on-policy advantage / baseline 条件。

因此 Score Centering：

- • 能精确消除由 score 非零均值产生的 drift；

- • 不能单独消除  与  之间的 sampling-distribution mismatch；

- • 当 staleness 很大时，通常仍需与 TIS、MIS 等 importance-sampling correction 组合。

### 9\. 实现与适用边界

上述 STE 等价关系在以下条件下最直接成立：

- • 使用 sampler 的完整 token 分布 ；

- •  在反向传播中完全 detach；

- • advantage  同样作为常数处理；

- • trainer 和 sampler 的 vocabulary、mask、temperature 及 token 变换保持一致；

- • 讨论的是基础 score-centering policy-gradient 项。

实践中通常只有 sampler 的 top- log-prob。此时论文使用 trainer 分布对 tail 进行重构，得到近似分布 。对应的梯度实际上是：

也就是对重构分布  的近似 STE，而不是对完整  的精确 STE。

此外，如果加入 TIS / MIS 权重 ，正确的 Score Centering 形式是：

这不再等价于简单地把 logits 替换成：

此时需要对加权 score 本身做 centering。

### 10\. 一句话总结

它把普通 off-policy PG 中的：

改成：

从而保证在  时 centered score 的条件均值严格为零，消除 training-inference mismatch 所产生的 drift；但剩余学习信号仍然是在  下计算的 covariance，因此它不是完整的 off-policy distribution correction。

![[从 On-Policy Baseline 到 Off-Policy Score-04.webp]]

技术交流群邀请函

![[从 On-Policy Baseline 到 Off-Policy Score-05.webp]]

![[从 On-Policy Baseline 到 Off-Policy Score-06.webp]]

![[从 On-Policy Baseline 到 Off-Policy Score-07.webp]]

![[从 On-Policy Baseline 到 Off-Policy Score-08.webp]]

△长按添加小助手

扫描二维码添加小助手微信

请备注：姓名-学校/公司-研究方向-城市

（如：小夏-浙大-大模型-杭州）

即可申请加入深度学习/机器学习等技术交流群

****—**完****—******

**为您推荐**

[《跨语言大模型》最新综述](http://mp.weixin.qq.com/s?__biz=MzU3NjE4NjQ4MA==&mid=2247549597&idx=1&sn=680bfff2720cab9044f4604e70c13e6d&chksm=fd15f382ca627a94510fbc69fed4236d54a81bfb46719a46ac2f4852805aef39159aad864c7b&scene=21#wechat_redirect)

[深度学习领域，你心目中 idea 最惊艳的论文是哪篇？](http://mp.weixin.qq.com/s?__biz=MzU3NjE4NjQ4MA==&mid=2247549576&idx=1&sn=7dd1c6dab78242cc4cf0839a00927fb7&chksm=fd15f397ca627a813e6d9ab3efaf2e84f8afc89c278250551ef3c8dfdc4d5d7cf867fad146f4&scene=21#wechat_redirect)

[思考丨到底什么叫算法工程师的落地能力？](http://mp.weixin.qq.com/s?__biz=MzU3NjE4NjQ4MA==&mid=2247503046&idx=1&sn=5de61949ccad86727f8febcfeae57584&chksm=fd153dd9ca62b4cfe87f0174c94a8af5fc8880f90e9ad1dc5695f3b8f3d2ecf3186381698519&scene=21#wechat_redirect)

[Transformer模型有多少种变体？看看这篇全面综述](http://mp.weixin.qq.com/s?__biz=MzU3NjE4NjQ4MA==&mid=2247513823&idx=2&sn=df4ef9164cad147334ae85cf84e59bba&chksm=fd1547c0ca62ced65fa68323ceaaaa8b22e20da5cd5abf0d9d0a29fa3d12019caedb9d4a3c2f&scene=21#wechat_redirect)

[从SGD到NadaMax，十种优化算法原理及实现](http://mp.weixin.qq.com/s?__biz=MzU3NjE4NjQ4MA==&mid=2247496698&idx=1&sn=b424a1e3b474ae23d75ddcacbacbbc85&chksm=fd1502e5ca628bf365a4d44bceb31de00ff7b1285d124e0883a28d4f697a1313b03ab98308d0&scene=21#wechat_redirect)

[各种注意力机制的PyTorch实现](http://mp.weixin.qq.com/s?__biz=MzU3NjE4NjQ4MA==&mid=2247513642&idx=2&sn=a8e5511279b92f15ee0cad6fce63bbbd&chksm=fd154735ca62ce23ff55cdc649fe96811c9034d21523061b32d887fb05f24c29f6639732175c&scene=21#wechat_redirect)

![[从 On-Policy Baseline 到 Off-Policy Score-09.gif]]

[跳转微信打开](https://wechat2rss.bestblogs.dev/link-proxy/?k=1d424e7a&r=1&u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzU3NjE4NjQ4MA%3D%3D%26mid%3D2247557366%26idx%3D1%26sn%3Db522c87ab97575e839033d58e5206e26)
