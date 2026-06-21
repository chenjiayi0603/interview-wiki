# Multi-Agent 面试学习系统

> 资源索引 + 三阶段学习路径 + 阶段一详细展开。整理于 2026年4月。

---

## 一、理论基础

### 1.1 Multi-Agent 架构原理

#### 多智能体系统基础

| 资源 | 类型 | 链接 | 优先级 |
|------|------|------|--------|
| 多智能体系统百科 | 文章 | m.baike.com/wiki/多智能体系统 | ⭐⭐⭐⭐⭐ |
| 多智能体架构深度拆解（6种协作模式+8大框架） | 文章 | blog.csdn.net/lvaolan168/article/details/156113926 | ⭐⭐⭐⭐⭐ |
| 多智能体系统设计指南 | 文章 | juejin.cn/post/7531573928247410730 | ⭐⭐⭐⭐ |

#### 协作模式

| 资源 | 类型 | 链接 | 优先级 |
|------|------|------|--------|
| 六大核心协作模式 | 文章 | blog.csdn.net/Gaga246/article/details/151399049 | ⭐⭐⭐⭐⭐ |
| 五大经典设计模式 | 文章 | blog.csdn.net/2403_88078374/article/details/159435841 | ⭐⭐⭐⭐⭐ |
| 谷歌八种设计模式 | 文章 | infoq.com/news/2026/01/multi-agent-design-patterns | ⭐⭐⭐⭐ |

### 1.2 Agent 通信协议

| 资源 | 链接 | 优先级 |
|------|------|--------|
| MCP 模型上下文协议 | cloud.tencent.com/developer/article/2521826 | ⭐⭐⭐⭐⭐ |
| AI Agent 协议深度研究（MCP/A2A/AG-UI） | blog.csdn.net/asce1885/article/details/154976509 | ⭐⭐⭐⭐⭐ |
| A2A 协议详解 | cloud.tencent.com/developer/article/2521826 | ⭐⭐⭐⭐ |

### 1.3 任务编排与调度

| 资源 | 链接 | 优先级 |
|------|------|--------|
| Agent 编排三种模式（单Agent/主从/对等） | github.com/Quriosity-agent/articles | ⭐⭐⭐⭐⭐ |
| Microsoft 编排模式指南 | learn.microsoft.com/azure/architecture/ai-ml/guide/ai-agent-design-patterns | ⭐⭐⭐⭐⭐ |
| LangChain Agent 编排 | docs.langchain.org.cn | ⭐⭐⭐⭐ |

### 1.4 Agent 记忆与状态管理

| 资源 | 链接 | 优先级 |
|------|------|--------|
| Agent 记忆综述（学术） | arxiv.org/abs/2407.20183 | ⭐⭐⭐⭐⭐ |
| Agent 记忆完全指南（四层架构） | blog.csdn.net/2301_76168381/article/details/159580776 | ⭐⭐⭐⭐⭐ |
| Mem0 框架实战 | blog.csdn.net/2403_88078374/article/details/159580776 | ⭐⭐⭐⭐ |

### 1.5 工具调用与 Function Calling

| 资源 | 链接 | 优先级 |
|------|------|--------|
| 工具调用综述 | blog.csdn.net/lvaolan168/article/details/156113926 | ⭐⭐⭐⭐⭐ |
| OpenAI 工具调用最佳实践 | platform.openai.com/docs/guides/function-calling | ⭐⭐⭐⭐⭐ |
| 行业实践：Agent 工具选择与组合 | blog.csdn.net/lvaolan168/article/details/156113926 | ⭐⭐⭐⭐ |

### 1.6 评估与可观测性

| 资源 | 链接 | 优先级 |
|------|------|--------|
| Agent 评估综述 | arxiv.org/abs/2407.20183 | ⭐⭐⭐⭐⭐ |
| LangSmith 官方文档 | docs.smith.langchain.com | ⭐⭐⭐⭐ |

### 1.7 前沿论文

| 论文 | 链接 | 优先级 |
|------|------|--------|
| AutoGen 论文 | arxiv.org/abs/2308.08155 | ⭐⭐⭐⭐⭐ |
| MetaGPT 论文 | arxiv.org/abs/2308.08155 | ⭐⭐⭐⭐⭐ |
| LLM 多智能体综述 | arxiv.org/abs/2407.20183 | ⭐⭐⭐⭐⭐ |
| AgentScope 论文 | arxiv.org/abs/2407.20183 | ⭐⭐⭐⭐ |
| AgentKit 论文 | arxiv.org/abs/2407.20183 | ⭐⭐⭐⭐ |

### 1.8 面试常见问题集

| 资源 | 链接 | 优先级 |
|------|------|--------|
| AI Agent 面试精选 15 题 | blog.csdn.net/lvaolan168/article/details/156113926 | ⭐⭐⭐⭐⭐ |
| Agent 核心面试题 2026 | blog.csdn.net/lvaolan168/article/details/156113926 | ⭐⭐⭐⭐⭐ |
| LangGraph 面试题 | blog.csdn.net/2403_88078374/article/details/159435841 | ⭐⭐⭐⭐ |

---

## 二、三阶段学习路径

### 阶段一：基础入门（1-2 周）

**目标**：说清 Agent vs Chat 区别；理解 ReAct 循环；区分协作模式；跑通至少一个 demo。

| 任务 | 学习内容 | 产出 |
|------|---------|------|
| 1. 理解 Agent 核心概念 | Agent 面试精选（基础概念）+ ReAct 实践 | 一页纸：ReAct 循环图 + 5 句话总结 |
| 2. 掌握 Multi-Agent 基础 | 六大协作模式 + 五大设计模式 + MetaGPT 架构 | 表格：六大模式 × 场景/缺点/例子 |
| 3. 了解主流框架 | CrewAI / MetaGPT / AutoGen Quick Start | 运行记录：环境/版本/报错解决 |
| 4. 面试话术初步 | 3 道基础题录音练习 | 录音：每题 1 分钟 |

**阶段一详细执行**：

**模块 1：理解 Agent 核心概念**
- 读 AI Agent 面试精选 15 题（基础概念部分），标出定义句和对比句
- 读 Agent 核心面试题 2026（Q1-Q3），按「结论→理由→例子→边界」四句组织
- 实践：运行一个简单的 ReAct Agent，画时序图（用户→模型→工具→模型→答案）

**模块 2：掌握 Multi-Agent 基础**
- 六大协作模式：Sequential / Router / Parallel / Generator / Network / Autonomous
- 五大设计模式：分层协作 / 对等协作 / 工具调用 / 责任链 / 共享状态
- 同场景用两种模式各描述一版架构，避免术语混淆

**模块 3：了解主流框架**
- CrewAI：班组心智（Agent/Task/Crew/Process）
- MetaGPT：软件工程多角色 + SOP
- AutoGen：对等多 Agent 对话

### 阶段二：框架选型与实战（2-3 周）

**目标**：选型对比表；CrewAI/MetaGPT 深度实战；LangGraph 能写 Demo。

| 任务 | 学习内容 |
|------|---------|
| 1. 框架选型框架 | LangGraph / CrewAI / AutoGen / 字节 Eino 的选型标准 |
| 2. LangGraph 深度 | 状态图、节点、边、条件分支、Checkpoint |
| 3. CrewAI 实战 | 完整项目：定义角色/任务/流程、工具集成、错误排查 |
| 4. MetaGPT 实战 | 源码结构理解、角色通信机制、中间产物流转 |
| 5. Eino 框架调研 | 字节 CloudWeGo 出品，Go 原生 LLM 框架 |

### 阶段三：高级专题与面试冲刺（2-3 周）

**目标**：前沿论文精读；企业级架构设计；System Design 准备。

| 任务 | 学习内容 |
|------|---------|
| 1. 论文精读 | Multi-Agent 综述 / AutoGen / MetaGPT / AgentScope |
| 2. 企业级架构 | Agent 平台整体架构、监控/回溯/CD、安全与治理 |
| 3. System Design | 设计面试：Agent 协作系统、RAG+Agent 混合架构 |
| 4. 综合演练 | 选一个场景：从需求到完整架构设计、技术选型、关键代码 |

---

## 三、阶段一详细路径（可执行级展开）

### 3.1 阶段定位

| 维度 | 阶段一结束时你应能…… |
|------|----------------------|
| **概念** | 说清楚 Agent 与单次 Chat 的区别；解释规划/工具/记忆/多步闭环 |
| **单 Agent** | 理解 ReAct 循环；口述观察-思考-行动与工具返回如何进入下一轮 |
| **多 Agent** | 区分协作模式与设计模式两套说法；举例 2-3 种典型拓扑 |
| **工程感知** | 跑通至少一个 ReAct demo；对 CrewAI/MetaGPT/AutoGen 各有一次运行体验 |

### 3.2 模块 1：理解 Agent 核心概念

**阅读任务 A：AI Agent 面试精选 15 题（基础概念部分）**

怎么读：
1. 第一遍：标出定义句和对比句（Agent vs 工作流、Agent vs RAG）
2. 第二遍：每道题改成「30 秒怎么答」的口头提纲

关键知识点：
- Agent 最小闭环：目标/感知/决策/行动/终止条件
- 与纯 Prompt 一次问答比，多出来的状态与外部动作
- 高频词：工具调用、规划、反思

自测：用手机录音 1 分钟回答「什么是 Agent」，不卡壳、不超时。

**阅读任务 B：Agent 核心面试题 2026（Q1-Q3）**

每题按「结论→理由→例子→边界」四句话组织：
- 结论：一句话亮明观点
- 理由：2-3 个支撑论据
- 例子：具体业务场景或项目案例
- 边界：什么情况下不适用

**实践任务：运行一个简单的 ReAct Agent**

ReAct 本质是推理和行动交替：模型先想一步，再决定要不要调工具，把工具结果放回上下文再继续想。

实现要点：
- 工具列表定义
- 调用格式（JSON Schema）
- 最大步数或超时
- 避免死循环

产出：画时序图 + 对比「多步 ReAct」和「单次思维链」的差别。

### 3.3 模块 2：掌握 Multi-Agent 基础

**阅读：六大核心协作模式**

建立拓扑感：顺序传递、路由分发、并行汇合、网状、高自主度。

每个模式脑子里有一个业务例子，同时能说清代价（延迟/成本/调试难度/一致性）。

**阅读：五大经典设计模式**

区分角色与职责：分层、对等、工具链、责任链、共享状态。

用同一个场景分别用「一种六大模式+一种五大模式」各描述一版架构。

**阅读：MetaGPT 实战解析（架构部分）**

只抓架构：多角色分工、SOP 驱动流程、结构化中间产物。

### 3.4 模块 3：了解主流框架

**CrewAI Quick Start**：Agent（人设与能力）、Task（工单与交付物）、Crew（角色与任务）、Process（协作策略）

**MetaGPT Minimal Example**：软件工程多角色 + SOP，体会结构化中间产物对排错的意义

**AutoGen 体验**：对等多 Agent 对话，理解分布式 Agent 通信模式
