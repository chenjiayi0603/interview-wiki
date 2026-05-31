# Multi-Agent 面试系统学习资料

> 整理时间：2026年4月 | 目标：为面试准备提供系统性学习资源  
> **阶段一可执行细化**：见同目录 `阶段一详细路径.md`（读法、自测、框架体验步骤）。

---

## 一、理论基础

### 1.1 Multi-Agent 架构原理

#### 多智能体系统基础
| 资源名称 | 类型 | 链接 | 简介 | 优先级 |
|---------|------|------|------|--------|
| 多智能体系统百科 | 文章 | https://m.baike.com/wiki/%E5%A4%9A%E6%99%BA%E8%83%BD%E4%BD%93%E7%B3%BB%E7%BB%9F/10291068 | 百科级系统性介绍，包含网络结构、联盟结构、黑板结构、集中式、分布式、层次化、混合体系结构等架构详解 | ⭐⭐⭐⭐⭐ |
| 多智能体架构深度拆解 | 文章 | https://blog.csdn.net/lvaolan168/article/details/156113926 | 详细解析6种核心协作模式与8大主流框架，适合AI产品经理和开发者 | ⭐⭐⭐⭐⭐ |
| 多智能体系统设计指南 | 文章 | https://juejin.cn/post/7531573928247410730 | 从架构设计原则、通信协议、冲突解决机制到集体智能涌现现象的深度分析 | ⭐⭐⭐⭐ |

#### 协作模式详解
| 资源名称 | 类型 | 链接 | 简介 | 优先级 |
|---------|------|------|------|--------|
| 六大核心协作模式 | 文章 | https://blog.csdn.net/Gaga246/article/details/151399049 | Sequential/Router/Parallel/Generator/Network/Autonomous Agents 六大模式详解 | ⭐⭐⭐⭐⭐ |
| 五大经典设计模式 | 文章 | https://blog.csdn.net/2403_88078374/article/details/159435841 | 分层协作、对等协作、工具调用、责任链、共享状态五大模式 | ⭐⭐⭐⭐⭐ |
| 谷歌八种设计模式 | 文章 | https://www.infoq.com/news/2026/01/multi-agent-design-patterns/ | 顺序流水线、协调器/分发器、并行扇出/聚合、层次分解、生成器与评判器、迭代精进、人工介入、复合模式 | ⭐⭐⭐⭐ |

### 1.2 Agent 通信协议

#### 核心通信协议
| 资源名称 | 类型 | 链接 | 简介 | 优先级 |
|---------|------|------|------|--------|
| MCP模型上下文协议 | 文章 | https://cloud.tencent.com/developer/article/2521826 | Anthropic发起的标准化协议详解，MCP/ACP/A2A/ANP四大协议对比分析 | ⭐⭐⭐⭐⭐ |
| AI Agent协议深度研究 | 文章 | https://blog.csdn.net/asce1885/article/details/154976509 | MCP、A2A、AG-UI三大核心协议的架构与实践深度解析 | ⭐⭐⭐⭐⭐ |
| A2A协议详解 | 文章 | https://cloud.tencent.com/developer/article/2521826 | Google主导的跨厂商Agent互操作协议 | ⭐⭐⭐⭐ |

### 1.3 任务编排与调度

#### 编排模式
| 资源名称 | 类型 | 链接 | 简介 | 优先级 |
|---------|------|------|------|--------|
| Agent编排三种模式 | 文章 | https://github.com/Quriosity-agent/articles/blob/main/2026-03-01/multi-agent-orchestration-three-modes.md | 单Agent（独行侠）、主从（Master-Worker）、对等网络（P2P）三种模式实战分析 | ⭐⭐⭐⭐⭐ |
| Microsoft编排模式指南 | 官方文档 | https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns | Sequential/Concurrent/Handoff/Supervisor/Magentic五种编排模式官方指南 | ⭐⭐⭐⭐⭐ |
| LangChain Agent编排 | 文档 | https://docs.langchain.org.cn/oss/python/langchain/quickstart | LangChain官方Agent开发指南，包含工具创建、记忆管理等 | ⭐⭐⭐⭐ |

### 1.4 Agent 记忆与状态管理

#### 记忆系统核心
| 资源名称 | 类型 | 链接 | 简介 | 优先级 |
|---------|------|------|------|--------|
| Agent记忆综述(学术) | 论文 | https://arxiv.org/abs/2407.20183 | NUS、人大、复旦、北大联合出品，Forms-Functions-Dynamics三角框架 | ⭐⭐⭐⭐⭐ |
| Agent记忆完全指南 | 文章 | https://blog.csdn.net/2301_76168381/article/details/159580776 | 四层架构详解：上下文、外部、情景、语义/参数记忆 | ⭐⭐⭐⭐⭐ |
| Memory系统实现 | 文章 | https://blog.csdn.net/Everly_/article/details/154483054 | 从RAG到Agentic RAG再到Agent Memory的演进 | ⭐⭐⭐⭐ |
| RAG vs Memory对比 | 文章 | https://mem0.ai/blog/rag-vs-ai-memory | RAG与Memory的核心差异，适用场景分析 | ⭐⭐⭐⭐ |
| Amazon Bedrock Memory | 官方文档 | https://aws.amazon.com/cn/blogs/china/when-ai-agents-learn-to-forget-amazon-bedrock-agentcore-memory-philosophy/ | 双层架构+智能整合，Semantic/User Preference/Summary/Episodic四种策略 | ⭐⭐⭐⭐ |

---

## 二、开源项目参考

### 2.1 MetaGPT

| 资源名称 | 类型 | 链接 | 简介 | 优先级 |
|---------|------|------|------|--------|
| MetaGPT GitHub | 开源仓库 | https://github.com/FoundationAgents/MetaGPT | 多智能体协作框架，ICLR 2024 oral，核心思想：Code = SOP(Team) | ⭐⭐⭐⭐⭐ |
| MetaGPT官方文档 | 官方文档 | https://docs.deepwisdom.ai/main/zh/ | 中文官方文档，含快速入门、核心概念、开发指南 | ⭐⭐⭐⭐⭐ |
| MetaGPT论文 | 论文 | https://arxiv.org/abs/2308.00352 | MetaGPT: Meta Programming for A Multi-Agent Collaborative Framework | ⭐⭐⭐⭐⭐ |
| MetaGPT实战解析 | 教程 | https://zhxin.blog.csdn.net/article/details/147962854 | 深度实战解析，角色机制、调度逻辑、部署实践 | ⭐⭐⭐⭐ |

### 2.2 AutoGen

| 资源名称 | 类型 | 链接 | 简介 | 优先级 |
|---------|------|------|------|--------|
| AutoGen GitHub | 开源仓库 | https://github.com/microsoft/autogen | Microsoft主导的多Agent框架，50.4k stars，正在与Semantic Kernel合并为Microsoft Agent Framework | ⭐⭐⭐⭐⭐ |
| AutoGen官方文档 | 官方文档 | https://microsoft.github.io/autogen/ | Getting Started、多Agent对话框架、增强LLM推理 | ⭐⭐⭐⭐ |
| Microsoft Agent Framework | 官方公告 | https://techcommunity.microsoft.com/blog/azuredevcommunityblog/introducing-the-microsoft-agent-framework/4458377 | AutoGen与Semantic Kernel合并后的新框架 | ⭐⭐⭐⭐ |

### 2.3 LangChain Agent

| 资源名称 | 类型 | 链接 | 简介 | 优先级 |
|---------|------|------|------|--------|
| LangChain文档 | 官方文档 | https://docs.langchain.org.cn/ | 中文官方文档，Agent开发核心模块详解 | ⭐⭐⭐⭐⭐ |
| LangChain实战教程 | 教程 | https://juejin.cn/post/7628076881032413235 | 第13章Agents基础，ReAct、Self-ask等Agent类型对比 | ⭐⭐⭐⭐ |
| LangGraph多智能体架构 | 教程 | https://blog.csdn.net/Gaga246/article/details/151399049 | 三层协作实战：底层API工具、中间层子代理、顶层Supervisor | ⭐⭐⭐⭐ |

### 2.4 CrewAI

| 资源名称 | 类型 | 链接 | 简介 | 优先级 |
|---------|------|------|------|--------|
| CrewAI GitHub | 开源仓库 | https://github.com/joaomdmoura/crewAI | 44.7k+ stars，专为协作型多智能体设计的Python框架 | ⭐⭐⭐⭐⭐ |
| CrewAI官方文档 | 官方文档 | https://docs.crewai.com/ | Agent/Task/Crew/Process四大核心组件 | ⭐⭐⭐⭐ |
| CrewAI深度指南 | 教程 | https://developer.cloud.tencent.com/article/2654179 | 2026年深度指南，五大核心优势+典型应用场景 | ⭐⭐⭐⭐ |

### 2.5 其他相关项目

| 资源名称 | 类型 | 链接 | 简介 | 优先级 |
|---------|------|------|------|--------|
| OpenAI Agents SDK | 开源仓库 | https://github.com/openai/openai-agents-python | OpenAI官方多智能体SDK，19.4k stars，简洁API+工程化落地 | ⭐⭐⭐⭐⭐ |
| Google ADK | 官方工具 | https://www.infoq.com/news/2026/01/multi-agent-design-patterns/ | Agent Development Kit，八种设计模式官方实现 | ⭐⭐⭐⭐ |

---

## 三、实战教程

### 3.1 Agent开发基础

| 资源名称 | 类型 | 链接 | 简介 | 优先级 |
|---------|------|------|------|--------|
| ReAct原理详解 | 教程 | https://blog.csdn.net/pythonhy/article/details/138720102 | AutoGPT系列，ReAct = Reason + Act，核心循环解析 | ⭐⭐⭐⭐⭐ |
| AutoGPT深度解析 | 教程 | https://blog.csdn.net/m0_63679833/article/details/155991066 | 自主AI代理时代，三大核心组件+快速体验指南 | ⭐⭐⭐⭐ |

### 3.2 多Agent协同实现

| 资源名称 | 类型 | 链接 | 简介 | 优先级 |
|---------|------|------|------|--------|
| 多Agent协作实战 | 教程 | https://blog.csdn.net/2402_84764726/article/details/158261852 | 从架构设计到工业级落地的深度实战解析 | ⭐⭐⭐⭐⭐ |
| CrewAI完整教程 | 教程 | https://cloud.tencent.com.cn/developer/article/2625388 | Agent/Task/Crew/Process四组件详解+实操代码 | ⭐⭐⭐⭐ |

### 3.3 面试场景设计

| 资源名称 | 类型 | 链接 | 简介 | 优先级 |
|---------|------|------|--------|
| 面试系统设计 | 文章 | https://blog.csdn.net/lvaolan168/article/details/156113926 | 多Agent系统面试精选15题，涵盖Agent基础/ReAct/工具调用/规划执行/Multi-Agent | ⭐⭐⭐⭐⭐ |

---

## 四、论文资源

### 4.1 核心论文

| 论文名称 | 链接 | 简介 | 优先级 |
|---------|------|------|--------|
| MetaGPT: Meta Programming for A Multi-Agent Collaborative Framework | https://arxiv.org/abs/2308.00352 | ICLR 2024 Oral，多Agent软件工程框架 | ⭐⭐⭐⭐⭐ |
| ReAct: Synergizing Reasoning and Acting in Language Models | https://arxiv.org/abs/2210.03629 | ReAct原始论文，Reason+Act范式 | ⭐⭐⭐⭐⭐ |
| AutoGPT论文 | https://arxiv.org/abs/2306.02224 | AutoGPT for Online Decision Making | ⭐⭐⭐⭐ |
| Generative Agents: Interactive Simulacra of Human Behavior | https://arxiv.org/abs/2304.03442 | 斯坦福生成式Agent论文 | ⭐⭐⭐⭐ |

### 4.2 研究趋势

| 论文名称 | 链接 | 简介 | 优先级 |
|---------|------|------|--------|
| AAMAS 2024 多智能体研究 | https://www.ia.ac.cn/kxyj/kydt_1/202406/t20240612_7187500.html | 中科院自动化所AAMAS 2024入选成果速览 | ⭐⭐⭐⭐ |
| Agent Memory Survey | https://arxiv.org/abs/2407.20183 | 多智能体记忆系统综述 | ⭐⭐⭐⭐ |

---

## 五、面试专题资源

### 5.1 面试高频题

| 资源名称 | 类型 | 链接 | 简介 | 优先级 |
|---------|------|------|------|--------|
| AI Agent面试精选15题 | 文章 | https://blog.csdn.net/lvaolan168/article/details/156113926 | 多Agent系统篇3题，涵盖架构优势、协作通信、设计因素 | ⭐⭐⭐⭐⭐ |
| Agent核心面试题2026 | 文章 | https://blog.csdn.net/m0_57081622/article/details/157394199 | Multi-Agent协同、设计模式、反思机制高频考点 | ⭐⭐⭐⭐⭐ |
| Agent面试速成清单 | 文章 | https://blog.csdn.net/m0_67640079/article/details/160260861 | 核心闭环、ReAct、Function Calling、设计题模板 | ⭐⭐⭐⭐⭐ |
| AI Agent面试八股文100问 | 文章 | https://blog.csdn.net/2402_84764726/article/details/158314824 | 大模型智能体高频考点全解析，附简历模板 | ⭐⭐⭐⭐⭐ |

### 5.2 真实面经

| 资源名称 | 类型 | 链接 | 简介 | 优先级 |
|---------|------|------|------|--------|
| 阿里Agent一面经验 | 文章 | https://paicoding.com/article/detail/2610900008650753 | LangChain/LangGraph/Multi-Agent/A2A等实际面试问题 | ⭐⭐⭐⭐⭐ |
| Multi-Agent协同模拟面试 | 文章 | https://blog.csdn.net/2402_84764726/article/details/158261852 | 架构设计、通信协议、角色分工、冲突消解深度解析 | ⭐⭐⭐⭐⭐ |
| 牛客Agent面试题整理 | 文章 | https://m.nowcoder.com/feed/main/detail/196ae6ef850b4d9a86f17fba439b9f02 | ReAct设计、通信协调、记忆设计、安全防护、评估指标 | ⭐⭐⭐⭐⭐ |

---

## 六、学习路径推荐

> **说明**：下列三阶段为「最小可行路径」；阶段一若需逐步拆解（阅读顺序、实践验收、复盘清单），优先打开同目录 `阶段一详细路径.md`。

### 阶段一：基础入门（1-2周）

```
1. 理解Agent核心概念
   ├─ 阅读：AI Agent面试精选15题（基础概念部分）
   ├─ 阅读：Agent核心面试题2026（Q1-Q3）
   └─ 实践：运行一个简单的ReAct Agent

2. 掌握Multi-Agent基础
   ├─ 阅读：六大核心协作模式
   ├─ 阅读：五大经典设计模式
   └─ 阅读：MetaGPT实战解析（架构部分）

3. 了解主流框架
   ├─ 体验：CrewAI Quick Start
   ├─ 体验：MetaGPT Minimal Example
   └─ 阅读：AutoGen Getting Started
```

### 阶段二：核心深入（2-3周）

```
1. 深度理解协作模式
   ├─ 阅读：谷歌八种设计模式
   ├─ 阅读：Agent编排三种模式
   └─ 阅读：Microsoft编排模式指南

2. 掌握通信协议
   ├─ 阅读：MCP/A2A/ACP/ANP协议详解
   └─ 阅读：AI Agent协议深度研究

3. 理解记忆系统
   ├─ 阅读：Agent记忆综述
   ├─ 阅读：Agent记忆完全指南
   └─ 阅读：RAG vs Memory对比

4. 阅读核心论文
   ├─ MetaGPT论文
   └─ ReAct论文
```

### 阶段三：面试冲刺（1-2周）

```
1. 面试题专项
   ├─ AI Agent面试精选15题（全部）
   ├─ Agent面试速成清单
   └─ AI Agent面试八股文100问

2. 面经实战
   ├─ 阿里Agent一面经验
   ├─ Multi-Agent协同模拟面试
   └─ 牛客Agent面试题整理

3. 项目准备
   ├─ 使用CrewAI构建一个多Agent协作项目
   └─ 理解MetaGPT代码架构
```

---

## 七、资源速查表

### 按优先级排序

| 优先级 | 资源 | 用途 |
|-------|------|------|
| P0 | AI Agent面试精选15题 | 面试核心考点 |
| P0 | Agent面试速成清单 | 快速回顾 |
| P0 | 六大核心协作模式 | 架构理解 |
| P0 | 五大经典设计模式 | 架构理解 |
| P1 | 多智能体系统百科 | 系统性理论 |
| P1 | MetaGPT论文 | 核心论文 |
| P1 | ReAct原理详解 | 技术基础 |
| P1 | CrewAI官方文档 | 框架实战 |
| P2 | MCP/A2A协议详解 | 通信协议 |
| P2 | Agent记忆综述 | 记忆系统 |
| P2 | 谷歌八种设计模式 | 进阶架构 |

### 按主题分类

**架构设计**
- 多智能体系统百科 ⭐⭐⭐⭐⭐
- 六大核心协作模式 ⭐⭐⭐⭐⭐
- 五大经典设计模式 ⭐⭐⭐⭐⭐
- 谷歌八种设计模式 ⭐⭐⭐⭐

**框架使用**
- MetaGPT GitHub ⭐⭐⭐⭐⭐
- CrewAI GitHub ⭐⭐⭐⭐⭐
- AutoGen GitHub ⭐⭐⭐⭐⭐
- OpenAI Agents SDK ⭐⭐⭐⭐⭐

**协议标准**
- MCP协议详解 ⭐⭐⭐⭐⭐
- A2A协议详解 ⭐⭐⭐⭐

**记忆系统**
- Agent记忆综述 ⭐⭐⭐⭐⭐
- Agent记忆完全指南 ⭐⭐⭐⭐⭐
- RAG vs Memory对比 ⭐⭐⭐⭐

**面试准备**
- AI Agent面试精选15题 ⭐⭐⭐⭐⭐
- Agent核心面试题2026 ⭐⭐⭐⭐⭐
- Agent面试速成清单 ⭐⭐⭐⭐⭐

---

## 八、补充说明

### 学习建议（细化）

1. **理论 + 实践绑定**  
   - 每读完一类「模式/协议/记忆」文章，用**同一业务例子**（如「需求→设计→实现→评审」）画一版架构草图，避免概念漂移。  
   - 每个意向框架至少保留：**安装命令、模型配置方式、跑通的示例名、一次失败与修复记录**（面试可讲 troubleshooting）。

2. **设计模式优先于框架 API**  
   - 面试官常问「为何选 Sequential / Router / Supervisor」而非「CrewAI 的 Process 怎么写」。先能口述**拓扑与代价**（延迟、成本、一致性、调试难度），再映射到具体框架字段。

3. **项目叙事准备**  
   - 准备 1 个可演示的多 Agent 小项目：**目标、角色分工、状态存在哪（内存/DB/消息队列）、如何观测（日志/追踪）、失败怎么办（重试/人工介入）**。  
   - 能回答「若去掉多 Agent、单 Agent + 工具能否完成」——体现你对复杂度的判断。

4. **信息时效**  
   - Agent 协议与微软系框架（AutoGen → Microsoft Agent Framework）变化快，面经与博客需**对照官方文档**核对；论文以 arXiv / 会议版本为准。

5. **安全与成本意识（加分项）**  
   - 工具调用的权限边界、敏感数据进模型上下文、无限循环与 Token 爆炸、多 Agent 轮次乘数带来的费用——可在项目介绍中主动提一句。

### 框架选择建议（扩展）

| 场景 | 推荐框架 | 简要理由（面试可用） |
|------|---------|----------------------|
| 快速原型、角色任务清晰 | CrewAI | Agent/Task/Crew/Process 心智模型简单，适合演示协作流 |
| 偏软件工程全流程、SOP 强 | MetaGPT | Code = SOP(Team)，角色与中间产物结构化，便于对齐研发流程 |
| 微软栈、对话式多 Agent、企业集成 | AutoGen / Microsoft Agent Framework | 与 Azure/生态整合方向一致，关注合并后的官方路线 |
| 图编排、复杂状态机、可回滚 | LangGraph + LangChain | 显式图与检查点，适合复杂分支与人机在环 |
| 对外 API 简洁、官方示例路径 | OpenAI Agents SDK | 「官方标准」叙事，适合与 OpenAI 模型能力绑定讲解 |

**没有银弹时的说法**：先定「编排拓扑 + 状态与记忆 + 人机边界」，再选框架；否则容易变成「为了多 Agent 而多 Agent」。

---

## 九、全篇要点详解（面试向）

> 下表与上文资源表**互补**：资源表负责「去哪学」，本节负责「学完要能复述什么」。表述以通用业界用法为主，具体名词以你所读原文与官方文档为准。

### 9.1 一、理论基础 —— 要点展开

**Multi-Agent 架构（对应 1.1）**

- **为何要多 Agent**：单模型上下文与能力边界、任务分解、角色专精（安全/成本/并行）、人在环分工；代价是**协调开销、一致性、调试复杂度、延迟与费用**。  
- **经典结构词汇**（百科/综述类文章常见）：集中式、分布式、层次化、黑板、联盟、混合——面试能各用**一句话**说清适用与缺点即可。  
- **两套「模式」别混用**：  
  - **协作/编排拓扑**：Sequential、Router、Parallel、Supervisor、Handoff 等（偏「怎么走」）。  
  - **角色/职责设计**：分层、对等、责任链、共享状态等（偏「谁负责什么」）。  
  - 标准答法：「拓扑解决控制流，角色设计解决职责与状态归属。」

**通信协议（对应 1.2）**

- **MCP（Model Context Protocol）**：把工具、资源、上下文以**标准接口**暴露给模型/宿主，偏「单 Agent 或宿主进程」的生态对接。  
- **A2A（Agent-to-Agent）**：跨系统、跨厂商的 **Agent 互操作**，偏「多服务、多供应商」场景。  
- **面试常追问**：和 REST/消息队列有何关系？——协议多在**语义层**约定发现、认证、能力描述；底层仍可走 HTTP/gRPC/MQ。  
- **AG-UI 等**：偏人机界面与交互协议，读资源表中的「协议深度研究」时留意**边界**（不是替代 MCP，而是不同层）。

**任务编排（对应 1.3）**

- **单 Agent / Master-Worker / P2P** 与 **Sequential / Concurrent / Supervisor** 等可对照理解：前者偏**权力结构**，后者偏**执行时序**。  
- **Handoff**：会话或职责从一方转移到另一方，要能说清**上下文如何交接**（摘要、结构化票据、共享存储）。  
- **Supervisor**：中央调度决策分支，易实现但可能成为**单点瓶颈**；扩展时考虑并行与超时。

**记忆与状态（对应 1.4）**

- **分层记忆（常见四分法）**：工作记忆/上下文、情景（会话与任务轨迹）、语义（长期知识）、用户偏好；工程上常映射到「上下文窗口 + 向量库 + KV/图 + 摘要」。  
- **RAG vs Memory**：RAG 多强调**检索外部知识**；Memory 强调**跨会话持续学习与个性化**，二者可结合（Agentic RAG + 记忆写入）。  
- **综述类论文框架**（如 Forms-Functions-Dynamics）：面试可一句话——「记忆长什么样、起什么作用、如何随时间更新」。

### 9.2 二、开源项目参考 —— 要点展开

| 项目 | 核心抽象 | 典型优势 | 典型代价 / 注意点 |
|------|----------|----------|-------------------|
| MetaGPT | Role、SOP、结构化产出 | 强软件工程叙事，易讲清「流水线 + 文档化中间态」 | 定制非软工场景需改 SOP；依赖模型与工具质量 |
| AutoGen | 对话、多 Agent 协作、与微软生态 | 对话式协作、企业路线（关注 Agent Framework） | API 与生态演进期，需跟官方迁移说明 |
| LangChain / LangGraph | Chain、Tool、Graph、Checkpoint | 图式编排、分支与人机在环 | 概念多，学习曲线陡；注意版本与包名 |
| CrewAI | Agent、Task、Crew、Process | 上手快，demo 叙事清晰 | 极复杂状态机可评估是否换图编排 |
| OpenAI Agents SDK | 官方多 Agent 抽象 | 「官方路径」、与模型能力故事一致 | 绑定供应商与 API 形态，阅读 release 说明 |

**通用人话总结**：MetaGPT 像「**岗位 + SOP**」，CrewAI 像「**班组 + 工单**」，LangGraph 像「**流程图 + 状态机**」，AutoGen 像「**群聊协作**」。

### 9.3 三、实战教程 —— 要点展开

- **ReAct**：观察（上下文/工具结果）→ 思考（自然语言推理）→ 行动（工具调用或输出）；需配合**停止条件、最大步数、工具模式（schema）**才工程化。  
- **多 Agent 实战文**：关注**角色边界、消息格式、冲突解决、评估指标**四块，而不是只抄代码。  
- **面试系统设计**：把「15 题」类文章当** checklist**，每题准备 30～60 秒口述版。

### 9.4 四、论文资源 —— 要点展开

- **ReAct 论文**：论证「推理与行动交织」对工具使用任务的收益；面试可对比纯 CoT、纯工具循环。  
- **MetaGPT 论文**：多 Agent **元编程/软件工程**；抓住「角色、消息、可执行产出」而非死记实验数字。  
- **Generative Agents（斯坦福小镇）**：记忆流、反思、规划——适合回答「长期记忆与行为连贯性」类问题。  
- **读法**：摘要 + 图 + 方法段落 + Limitation；时间紧可只读**贡献与实验设置**。

### 9.5 五、面试专题 —— 使用顺序建议

1. **先建立骨架**：「15 题」或「速成清单」过一遍，标出完全不会的词条。  
2. **再填血肉**：用「八股 100 问」补广度，用「核心面试题 2026」补深度与表述。  
3. **最后用面经**：阿里 / 牛客 / 模拟面试类当**全真模拟**，限时答并录音回听。

**高频主题自检清单（建议默写一遍）**：ReAct 与工具、规划与重规划、反思与评判器、多 Agent 通信、记忆与 RAG、评估与安全、成本与延迟、人机协作边界。

### 9.6 六、学习路径 —— 三阶段成果定义

| 阶段 | 时间（参考） | 结束时建议能演示 / 口述 |
|------|----------------|--------------------------|
| 阶段一 | 1～2 周 | ReAct 闭环图；六大模式各一例；三框架中至少两个跑通过 |
| 阶段二 | 2～3 周 | 谷歌八种与微软五种编排的异同；MCP/A2A 场景选型；记忆分层 + RAG/Memory 辨析；ReAct/MetaGPT 论文各 3 分钟口述 |
| 阶段三 | 1～2 周 | 15 题级全覆盖；1 个完整项目故事；1～2 场面经级模拟 |

### 9.7 七、资源速查 —— 使用提示

- **P0**：保证「能讲」——协作模式 + 设计模式 + 面试题清单。  
- **P1**：保证「能深」——百科/论文/ReAct/一个主修框架文档。  
- **P2**：保证「能扩展」——协议、记忆综述、谷歌八种模式。

---

*整理自多个来源，如链接失效请搜索对应关键词获取最新资源*
