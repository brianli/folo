---
title: "云栖2026｜湖生万物，助力 AI — 面向 Agent 的全模态数据平台"
source: "阿里云开发者"
category: "Inbox"
group: "大数据"
url: "https://mp.weixin.qq.com/s/Ms9n_Nrf2XcC0uQfwW-FRA"
published: 2026-10-09T20:03:30+08:00
saved: 2026-10-09T20:03:48+08:00
folo_key: "url::https://mp.weixin.qq.com/s/Ms9n_Nrf2XcC0uQfwW-FRA"
updated: 2026-10-09T20:10:22+08:00
tags:
  - "folo"
  - "Inbox"
  - "阿里云开发者"
---

# 云栖2026｜湖生万物，助力 AI — 面向 Agent 的全模态数据平台

> [!info] 阿里云开发者 · Inbox · 2026-10-09 20:03 · [原文](https://mp.weixin.qq.com/s/Ms9n_Nrf2XcC0uQfwW-FRA)

近日，在 2026 云栖大会"Agent 驱动的全模态大数据计算创新"论坛上，阿里云智能集团计算平台事业部开源大数据平台负责人王峰（莫问）发表主题演讲，系统阐述了阿里云开源大数据平台面向 Agentic AI 时代的演进新范式。阿里云以 DLF、Flink、EMR Spark/Ray/Daft、EMR StarRocks、Milvus 五大核心产品协同构建了"Agentic Lake + Agentic Streaming"技术体系，为自动驾驶、具身智能、大模型数据预处理、企业智能运营等场景提供下一代全模态数据基础设施。

![[云栖2026｜湖生万物，助力 AI — 面向 Agent 的全模态数据平台-01.jpg]]

阿里云智能集团计算平台事业部 开源大数据平台负责人王峰

# 01AI 时代，开源大数据平台正在经历四重变革

王峰用"四个转变"概括了大数据平台的演进趋势：数据结构从文本扩展到音视频等全模态；分析算子从 BI 统计升级为 AI 推理；管理方式从封闭数仓走向开放湖仓；用户类型从自然人分析师转变为 AI Agent。

这意味着大数据平台不再只是"给人用的分析工具"，而是需要同时服务人与 Agent 的智能基础设施。围绕这一判断，阿里云开源大数据平台确立了面向 AI 场景设计、开源生态开放架构、全模态数据统一管理三大原则，形成清晰的架构方案——DLF 管理全模态数据存储与元数据，Flink 承载流式处理与 Streaming Agent，EMR Ray/Daft 提供离线全模态 AI 计算，StarRocks 实现实时混合检索，Milvus 作为向量引擎——五者协同覆盖文本、图像、音视频、传感器信号、3D 几何空间数据等完整处理链路，并通过 MCP、Skills 与智能运维让 Agent 与自然人均可便捷使用。

# 02 DLF：构建 AI 时代的全模态数据湖

阿里云数据湖构建服务 DLF 以 Apache Paimon 为核心湖格式，通过 Omni Catalog，实现统一 Schema、目录与权限管理，支持 OSS 对象存储（图片、音频、视频）、RDS 关系数据库、SLS 日志服务、实时视频流（RTSP/RTMP/HLS）及文档知识库（钉钉语雀、PDF）等全模态数据实时和批量写入。存储层面提供缓存加速、冷热分层、生命周期管理、自适应分桶与文件合并，配合血缘管理、索引管理与排序优化，让全模态数据在湖中实现高效可查。

![[云栖2026｜湖生万物，助力 AI — 面向 Agent 的全模态数据平台-02.jpg]]

阿里云智能集团计算平台事业部 DLF负责人李劲松

DLF 创新引入"湖流一体"架构：湖表层基于 Paimon 提供分钟级自动入湖同步，实时层基于 Fluss 提供毫秒级 Changelog 写入即可消费，两层通过 Union Read 统一读取边界——历史快照与最新增量无缝衔接，一份数据同时服务离线与实时工作负载。

在场景演进上，Paimon 1.0 以流批一体服务电商、零售、物流的大规模离线湖仓；Paimon + Fluss 为金融、游戏行业提供秒级实时能力；Paimon 2.0 全面升级为全模态数据管理，原生支持 LeRobot、RLDS、HDF5 等主流格式，重点服务智能驾驶与具身智能等 AI 原生行业。实时计算 Flink 则承担"流式入湖"管道角色，确保数据湖始终保持数据的新鲜度。

# 03 EMR Ray + Daft：分布式全模态 AI 计算数据管线

全模态数据入湖后，如何高效完成向量化、标注、质量评估与格式转换，基于 EMR Ray + Daft 构建了新一代分布式全模态数据处理管线。

EMR Ray 集群提供 CPU 与 GPU 资源池独立弹性扩缩，Daft DataFrame 作为全模态 AI 算子执行引擎，支持向量化执行、Locality 调度与坏卡监测隔离。上层接口同时支持 Python DataFrame 和 SQL 两种范式。平台内置面向具身智能（场景重建、手部重建、动作标注）、自动驾驶（图片标注、向量化、索引创建）、大模型数据处理（语义去重、质量打分、Tokenizer）等行业的算法库，深度集成 Paimon 全模态湖格式，AI 算子与关系算子可联合优化执行计划，实现 Data + AI 一体化。

![[云栖2026｜湖生万物，助力 AI — 面向 Agent 的全模态数据平台-03.jpg]]

阿里云智能集团计算平台事业部 EMR负责人李钰

EMR Ray 助力舞肌科技构建 Ego-Centric 手部三维重建管线，该管线从第一视角单目视频重建双手三维运动轨迹，涵盖视频预处理与标定、深度与手部重建、长视频合并与轨迹清洗、数据导出四个阶段，涉及 GeoCalib 内参估计、MoGe-2 深度估计、HaWoR 双手重建、DROID-SLAM 相机跟踪等十余个步骤。基于 EMR Ray 实现模型常驻复用、CPU/GPU 分工独立弹性、多视频流水线执行与断点续跑四大工程优势，最终产出 LeRobot v3 格式数据集与三栏可视化结果，可直接用于具身智能模型训练。

04

EMR StarRocks：全模态数据实时混合检索

数据湖解决"存"，AI 管线解决"加工"，实时检索解决"用"——如何让 Agent 在海量全模态数据中精准定位所需信息。

EMR StarRocks 围绕"结构化分析极致性能"与"全模态混合检索"两条主线演进，同时提供标量分析、向量检索、全文检索与 AI 推理能力，支持StarRocks 仓表与DLF湖表（Paimon/Lance/Iceberg）联邦查询，覆盖从实时 BI 到 AI RAG 与 AI Memory 的全场景需求。

在千问大模型数据链路中，阿里巴巴集团 ALake 以 Paimon 宽表存储全模态数据——同一行关联 ID、属性、文本、图片、音频、视频与向量等。EMR StarRocks 贯穿三大环节：采集阶段通过混合检索与 Agent 多轮迭代完成 Top-K 抽样评测，输出相关性与覆盖度报告；训练准备阶段将合格数据集以 Top-M 大规模检索导出为专项训练集，供千问全模态训练使用，实现端到端闭环。

![[云栖2026｜湖生万物，助力 AI — 面向 Agent 的全模态数据平台-04.png]]

在智能驾驶领域，EMR StarRocks 赋能卓驭科技构建智驾数据准备链路。卓驭的全模态数据湖存储着视频 Clip、驾驶帧与目标抠图、LiDAR 点云、车机日志、地理信息、历史故障与视觉向量等数据，通过 DLF 管理元数据、Lance 湖表存储于 OSS。StarRocks 提供标量圈选（时空/天气/标签/故障条件）、向量检索（整图与抠图相似场景）与混合检索（全文+向量多路召回、RRF 融合）三类能力，支撑"输入故障图片向量 + 雨夜条件 → 检索相似驾驶帧 → 输出视频 Clip 与命中时间戳"等场景圈选工作流，为智驾模型训练补样与故障回溯提供高效数据准备。

05

阿里云 Flink：从流式计算到 Streaming Agent

实时计算 Flink 展现了最大的范式跨越——从流式数据处理引擎升级为企业级 Streaming Agent 运行时平台，通过湖流一体 构建 Agentic Lake。

传统流式计算处理"数据流"，而 Streaming Agent 处理"事件流"：数据持续到达，Agent 状态随事件增量演进，关键时刻按需触发大模型推理，输出结构化结果与业务行动。Flink 构建了三层 Streaming Agent 架构：底层提供 Streaming Pipeline、状态记忆管理、自适应弹性扩缩容与全链路故障自愈；中间算子层覆盖多源接入（媒体流/日志流/存储系统）、媒体处理（解码/抽帧/切片）、感知理解（ASR/OCR/VLM/目标检测）、检索生成（RAG/LLM/TTS）及 Agent 算子（Workflow/ReAct、MCP/Tools 扩展）；上层业务层沉淀可复用的行业解决方案。Flink 统一对接阿里云百炼推理服务，支持专用感知模型与通用大模型混合调用，Agent 全局上下文跨事件、跨模态持续演进。

![[云栖2026｜湖生万物，助力 AI — 面向 Agent 的全模态数据平台-05.jpg]]

阿里云智能集团计算平台事业部 流存储Fluss负责人伍翀

实时体育赛事解说 Agent 已成功助力央视亚运会转播，基于直播视频流实时解说，支持风格与方言调整，已适配游泳、射击等项目。管线六步执行：赛事信号接入 → 视频预处理（格式标准化、关键帧采样）→ 赛事观察（独立插件提取画面动作、比分图卡、人物目标）→ 赛事理解（要素组合为赛事语义，识别关键事件与进程）→ 解说策划（关键事件优先、事实可靠性判断）→ 生成播控（口播生成、数字校验、字幕 TTS 实时播控）。Flink内置AI服务统一承载 CV/OCR/VLM 视觉理解与 TTS 语音生成，Flink 持续运行确保全程状态不丢失。

门店智能运营 Agent 基于摄像头视频流实时识别业务事件并入湖存储，已适配顾客结账、店员理货、空货架、店内争端等场景。其特殊性在于：感知阶段只提取可见事实而不直接判定事件，保留反证与证据帧；理解阶段由 Flink 按摄像头维护跨窗口状态，静默或结束动作自动闭合事件；计划阶段按证据充分性分层复核——规则确认 → 复核 ，在成本与准确性间取得平衡；执行阶段将结构化事件与证据画面写入 Paimon 湖表，支撑大屏、告警与复盘，形成可追溯的完整数据链。

06

开源开放，全模态数据平台服务 AI 原生应用

当 Agent 成为数据平台的新用户，当全模态成为主流数据形态，基础设施必须全面重构。

DLF 以 Paimon 2.0 + Fluss 湖流一体统一全模态管理；EMR Ray + Daft 打通从数据加工到模型训练的 AI 数据管线；StarRocks 凭借混合检索能力让 Agent 实现精准混合检索；Flink 以 Streaming Agent 驱动实时业务行动——四者协同配合 Milvus 向量引擎，形成完整的 Agentic Lake + Agentic Streaming 技术体系。阿里云开源大数据平台提供全托管服务与深度优化，让企业既享受开源灵活性，又获得企业级稳定性与性能。从"湖生万物"到"助力 AI"，阿里云开源大数据平台正在成为构建 Agent 时代全模态数据基础设施的必然选择。

<!-- folo:annotations -->
## 批注

- ==Agentic Lake + Agentic Streaming==
  > Lake 代表存储，Streaming 代表计算
- ==DLF、Flink、EMR Spark/Ray/Daft、EMR StarRocks、Milvus==
- ==下一代全模态数据基础设施==

<!-- folo:data [{"id":8,"rel_path":"","kind":"highlight","color":"red","quote":"Agentic Lake + Agentic Streaming","prefix":"y/Daft、EMR StarRocks、Milvus 五大核心产品协同构建了\"","suffix":"\"技术体系，为自动驾驶、具身智能、大模型数据预处理、企业智能运营等场景提供下一代","note":"Lake 代表存储，Streaming 代表计算","created_at":1791547510,"updated_at":1791547720},{"id":9,"rel_path":"","kind":"highlight","color":"green","quote":"DLF、Flink、EMR Spark/Ray/Daft、EMR StarRocks、Milvus","prefix":"述了阿里云开源大数据平台面向 Agentic AI 时代的演进新范式。阿里云以 ","suffix":" 五大核心产品协同构建了\"Agentic Lake + Agentic Stre","note":"","created_at":1791547555,"updated_at":1791547650},{"id":10,"rel_path":"","kind":"highlight","color":"pink","quote":"下一代全模态数据基础设施","prefix":"ing\"技术体系，为自动驾驶、具身智能、大模型数据预处理、企业智能运营等场景提供","suffix":"。\n\n阿里云智能集团计算平台事业部 开源大数据平台负责人王峰\n01\nAI 时代，","note":"","created_at":1791547755,"updated_at":1791547755}] -->
<!-- /folo:annotations -->
