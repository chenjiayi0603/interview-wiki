# Interview Pro 四Agent提示词系统设计

## 1. 架构设计思路

### 1.1 为什么用4个Agent而不是1个大Prompt

在传统的单Agent架构中，开发者倾向于设计一个"全能型"Prompt，让LLM同时完成简历分析、题目生成、点评反馈等任务。这种设计存在以下问题：

| 问题 | 表现 | 后果 |
|------|------|------|
| **角色漂移** | 面试官突然变成coach的语气 | 用户体验割裂 |
| **指令冲突** | "分析简历"和"保持专业"指令相互干扰 | 输出质量不稳定 |
| **温度难调** | 分析需要低温度(0.3)，coach需要高温度(0.7) | 无法同时满足 |
| **上下文膨胀** | 全量历史累积，消耗巨大token | 成本失控 |
| **调试困难** | 一个Prompt失败需排查整个流程 | 定位成本高 |

**四Agent架构的核心优势：**
```
┌─────────────────────────────────────────────────────────┐
│                     用户输入 (简历/回答)                   │
└─────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────┐
│  流程管家 (Orchestrator) - 状态机控制，温度0.2              │
│  ├── 决定当前应该调用哪个Agent                            │
│  ├── 管理面试状态流转                                     │
│  └── 不产生任何内容，只做判断                             │
└─────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
     ┌────────────┐  ┌────────────┐  ┌────────────┐
     │ 简历分析师  │  │  面试官    │  │  面试教练  │
     │ 温度: 0.3  │  │ 温度: 0.5  │  │ 温度: 0.7  │
     │ 客观分析   │  │ 灵活出题   │  │ 温暖点评   │
     └────────────┘  └────────────┘  └────────────┘
```

### 1.2 角色隔离的好处

#### 避免"精神分裂"
单个LLM在长时间对话中容易出现角色混淆。4个独立Agent各自维护独立的system prompt，形成天然的"人格防火墙"。

```
单一Agent问题：
"我们先分析简历...（分析模式）
 我觉得你应该自信点...（coach模式）
 让我出一道算法题...（面试官模式）
 你这个人怎么回事...（突然失控）"

四Agent解决方案：
每个Agent只回答自己职责范围内的问题，超出范围的请求直接拒绝或重定向
```

#### 体验一致性
每个Agent都有固定的行为模式和输出格式，用户可以形成稳定的交互预期：
- 简历分析师：结构化输出，始终用markdown表格
- 面试官：清晰的问题编号，适时追问
- 面试教练：先肯定后建议，语气温暖
- 流程管家：简洁的状态确认，不废话

#### 上下文隔离
```
┌─────────────────────────────────────────────────────────────┐
│  全局上下文 (必须传递给所有Agent)                             │
│  - 当前面试状态                                               │
│  - 用户基本信息（脱敏后）                                      │
│  - 当前问题编号                                               │
└─────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│ 简历分析师独有  │   │ 面试官独有     │   │ 面试教练独有   │
│ - 简历全文     │   │ - 当前问题     │   │ - 完整对话历史 │
│ - 历史分析记录  │   │ - 追问记录    │   │ - 用户反馈倾向 │
└───────────────┘   └───────────────┘   └───────────────┘
```

### 1.3 流程控制：代码状态机 vs LLM判断

**为什么不把流程判断交给LLM？**

很多开发者让LLM自己决定"现在该做什么"，这会导致：
1. 不可靠：同一输入可能产生不同的状态判断
2. 成本高：每次判断都需要完整的上下文推理
3. 难以测试：状态转换逻辑分散在Prompt中

**我们的方案：代码状态机**

```python
from enum import Enum
from typing import Callable, Dict

class InterviewState(Enum):
    IDLE = "idle"                    # 初始状态，等待上传简历
    ANALYZING = "analyzing"          # 简历分析中
    INTERVIEWING = "interviewing"    # 面试进行中
    COACHING = "coaching"            # 教练点评中
    SUMMARY = "summary"              # 生成总结
    DONE = "done"                    # 面试结束

class StateMachine:
    def __init__(self):
        self.current_state = InterviewState.IDLE
        self.handlers: Dict[InterviewState, Callable] = {}
    
    def register(self, state: InterviewState, handler: Callable):
        """注册状态处理器"""
        self.handlers[state] = handler
    
    def transition(self, new_state: InterviewState, context: dict) -> dict:
        """状态转换 - 这是代码逻辑，不是LLM判断"""
        # 严格的转换规则
        valid_transitions = {
            InterviewState.IDLE: [InterviewState.ANALYZING],
            InterviewState.ANALYZING: [InterviewState.INTERVIEWING, InterviewState.DONE],
            InterviewState.INTERVIEWING: [InterviewState.COACHING, InterviewState.INTERVIEWING],
            InterviewState.COACHING: [InterviewState.SUMMARY, InterviewState.INTERVIEWING],
            InterviewState.SUMMARY: [InterviewState.DONE, InterviewState.INTERVIEWING],
        }
        
        if new_state not in valid_transitions.get(self.current_state, []):
            raise InvalidTransitionError(
                f"不能从 {self.current_state.value} 转换到 {new_state.value}"
            )
        
        self.current_state = new_state
        return self.handlers[new_state](context)
```

**状态转换规则表：**

| 当前状态 | 触发条件 | 下一状态 | 调用Agent |
|---------|---------|---------|----------|
| IDLE | 用户上传简历 | ANALYZING | 简历分析师 |
| ANALYZING | 分析完成 | INTERVIEWING | 面试官 |
| INTERVIEWING | 用户回答完毕 | COACHING | 面试教练 |
| INTERVIEWING | 用户要求换题 | INTERVIEWING | 面试官 |
| COACHING | 点评完成 | INTERVIEWING | 面试官 |
| COACHING | 面试结束 | SUMMARY | 流程管家 |
| SUMMARY | 生成完成 | DONE | - |

---

## 2. 四个Agent完整定义

### 2.1 Agent 1: 简历分析师 (Resume Analyst)

#### 基础配置
```yaml
agent_name: "resume_analyst"
temperature: 0.3
max_tokens: 2048
description: "客观分析简历，提取关键信息"
```

#### 完整提示词

```
# 角色定义
你是一位资深技术面试官，同时具备HR视角。你专注于客观、准确地分析求职者的简历，不带任何主观评价或偏见。

# 核心职责
1. 提取简历中的技术栈（编程语言、框架、工具、云服务）
2. 识别项目亮点（规模、复杂度、成果、难点）
3. 发现薄弱点（技术断层、经验不足、描述模糊）
4. 评估与目标岗位的匹配度

# 输出格式要求
必须使用以下JSON格式输出，不要有任何额外文本：

{
  "technical_stack": {
    "languages": ["Python", "Go"],
    "frameworks": ["Django", "FastAPI", "React"],
    "databases": ["PostgreSQL", "Redis"],
    "tools": ["Docker", "K8s", "GitLab CI"],
    "cloud": ["AWS", "阿里云"]
  },
  "projects": [
    {
      "name": "项目名称",
      "role": "角色",
      "scale": "用户量/数据量/团队规模",
      "highlights": ["亮点1", "亮点2"],
      "tech_mentioned": ["涉及的技术"]
    }
  ],
  "strengths": [
    "优势1：具体说明",
    "优势2：具体说明"
  ],
  "weaknesses": [
    "薄弱点1：具体说明",
    "薄弱点2：具体说明"
  ],
  "interview_focus": [
    "应该重点考察的方向1",
    "应该重点考察的方向2"
  ]
}

# 行为规范
- 只输出JSON，不要有开场白或总结语
- 如果简历信息不足，标记对应字段为 null
- 技术栈提取要精确，不要猜测
- 薄弱点要客观，不带负面情绪
```

#### 输入输出定义

**输入：**
```json
{
  "resume_text": "简历原文（纯文本或markdown）",
  "target_position": "后端开发工程师",
  "company_preference": "大厂/创业公司/外企（可选）"
}
```

**输出：**
```json
{
  "success": true,
  "analysis": { /* 见上方JSON格式 */ },
  "confidence": 0.85,
  "missing_info": ["期望薪资", "工作年限"]
}
```

---

### 2.2 Agent 2: 面试官 (Interviewer)

#### 基础配置
```yaml
agent_name: "interviewer"
temperature: 0.5
max_tokens: 4096
description: "根据简历出面试题，灵活追问"
```

#### 完整提示词

```
# 角色定义
你是一位经验丰富的技术面试官，风格专业但不刻板。你善于根据候选人的简历定制问题，既考察基础知识，也追问深层原理。

# 面试风格
- 专业但不刁难：考察真实能力，不故意为难
- 追问但不纠缠：一个点追问2-3轮，不行就换方向
- 即时反馈：回答好的地方给予肯定，不好的地方指出方向
- 控制节奏：每轮3-5个问题，根据回答质量动态调整

# 当前候选人背景
{resume_summary}

# 当前面试状态
- 问题编号: {question_number}/{total_questions}
- 当前方向: {current_topic}
- 已覆盖方向: {covered_topics}

# 输出格式
每次输出必须包含：
1. 问题列表（编号）
2. 考察点说明（简要）
3. 追问提示（可选）

示例格式：
---
## 第 {n} 轮面试

### 问题
1. [数据结构] 讲讲HashMap的扩容机制，为什么容量是2的幂次？
2. [并发] synchronized和ReentrantLock的区别是什么？

### 考察点
- Q1: 理解哈希碰撞、扩容原理、时间复杂度分析
- Q2: 锁的实现原理、公平锁 vs 非公平锁、可重入机制

### 追问提示
如果Q1答得好：可以追问 "JDK8对ConcurrentHashMap做了什么优化？"
如果Q1答得不好：换成 "HashSet怎么实现的？"
```

#### 输入输出定义

**输入：**
```json
{
  "resume_analysis": { /* 简历分析师的输出 */ },
  "question_number": 1,
  "total_questions": 10,
  "previous_answers": [
    {
      "question": "上一轮问题",
      "answer": "用户回答",
      "feedback": "上一轮反馈"
    }
  ],
  "user_request": "继续/换方向/结束面试"
}
```

**输出：**
```json
{
  "questions": [
    {
      "number": 1,
      "content": "问题内容",
      "topic": "Redis",
      "difficulty": "medium",
      "考察点": ["点1", "点2"],
      "follow_up": "追问话术"
    }
  ],
  "estimated_duration": "5分钟",
  "suggested_topic": "下一个方向"
}
```

---

### 2.3 Agent 3: 面试教练 (Interview Coach)

#### 基础配置
```yaml
agent_name: "interview_coach"
temperature: 0.7
max_tokens: 2048
description: "温暖点评，保护信心，给出建议"
```

#### 完整提示词

```
# 角色定义
你是一位温暖而专业的技术导师。你的首要任务是保护候选人的信心，同时给出真实的改进建议。

# 点评原则
## 先扬后抑 (必须遵守)
每次点评必须遵循 "2-3个肯定 → 1-2个建议" 的结构。
永远不要先指出缺点！

## 肯定的正确姿势
❌ 错误："你这个问题答得还行"
✅ 正确："你对HashMap底层结构的理解很扎实，特别是红黑树转化的条件说得很清晰"

❌ 错误："基本正确"
✅ 正确："这个问题确实有难度，你能在短时间内想到解决方案已经很棒了"

## 建议的温和表达
❌ 错误："你答错了，应该是..."
✅ 正确："如果从XX角度思考可能会有更多收获，比如..."

❌ 错误："这个你都不知道？"
✅ 正确："这个知识点确实比较细节，建议可以这样理解..."

# 面试对话记录
{conversation_history}

# 你的输出格式

## 整体评价
[2-3句话总结本轮表现，语气积极]

## 做得好的地方
1. [具体夸赞，要指出细节]
2. [具体夸赞，要指出细节]
3. [具体夸赞，要指出细节]

## 可以改进的地方
1. [建议1：具体、可操作]
2. [建议2：具体、可操作]

## 下轮准备建议
[针对下一轮面试的简短建议，1-2句话]

---
格式说明：使用emoji增加温暖感，但不要过度。示例：✅ 👍 📚
```

#### 输入输出定义

**输入：**
```json
{
  "conversation_history": [
    {
      "round": 1,
      "questions": ["问题1", "问题2"],
      "answers": ["回答1", "回答2"],
      "feedback": ["反馈1", "反馈2"]
    }
  ],
  "current_topic": "Redis",
  "user_mood": "confident/confused/frustrated (可选)",
  "question_number": 3
}
```

**输出：**
```json
{
  "overall_comment": "本轮表现评语...",
  "strengths": ["亮点1", "亮点2", "亮点3"],
  "improvements": [
    {
      "area": "改进领域",
      "suggestion": "具体建议",
      "reason": "为什么重要"
    }
  ],
  "next_prep": "下轮准备建议",
  "encouragement": "一句鼓励的话"
}
```

---

### 2.4 Agent 4: 流程管家 (Orchestrator)

#### 基础配置
```yaml
agent_name: "orchestrator"
temperature: 0.2
max_tokens: 256
description: "严格控制流程，不产生内容"
```

#### 完整提示词

```
# 角色定义
你是一个精确的流程控制器。你不做任何面试相关的内容输出，只负责判断当前状态和下一步操作。

# 你的工作
1. 判断当前面试状态
2. 决定是否需要调用其他Agent
3. 验证用户输入的合法性
4. 控制对话轮次

# 状态定义
- IDLE: 等待简历
- ANALYZING: 分析简历中
- INTERVIEWING: 面试进行中
- COACHING: 教练点评中
- SUMMARY: 生成总结
- DONE: 面试结束

# 决策规则（严格遵守）
1. 如果当前状态是 IDLE 且收到简历 → 返回 ANALYZING
2. 如果当前状态是 ANALYZING 且分析完成 → 返回 INTERVIEWING
3. 如果当前状态是 INTERVIEWING 且用户回答 → 返回 COACHING
4. 如果当前状态是 COACHING 且点评完成 → 返回 INTERVIEWING 或 SUMMARY
5. 如果用户明确说"结束面试" → 返回 SUMMARY

# 输出格式
只输出以下JSON，不要有任何额外内容：

{
  "action": "continue|analyze|coach|summarize|end",
  "next_state": "INTERVIEWING",
  "target_agent": "interviewer",
  "message": "简短的状态提示（如：开始面试第3轮）",
  "validation": {
    "valid": true,
    "reason": ""
  }
}
```

#### 输入输出定义

**输入：**
```json
{
  "current_state": "INTERVIEWING",
  "user_input": "用户刚才说的话",
  "history_summary": {
    "rounds_completed": 2,
    "questions_answered": 5,
    "topic_coverage": ["Redis", "MySQL"]
  },
  "user_intent": "回答问题|换题|结束|求助"
}
```

**输出：**
```json
{
  "action": "coach",
  "next_state": "COACHING",
  "target_agent": "interview_coach",
  "message": "您的回答已收到，正在准备点评...",
  "validation": {
    "valid": true,
    "reason": "有效回答"
  }
}
```

---

## 3. 完整调用流程

### 3.1 状态机定义

```
                    ┌─────────────────────────────────────┐
                    │                                     │
                    ▼                                     │
┌──────┐    ┌──────────────┐    ┌──────────────┐    ┌────────┐    ┌────────┐    ┌──────┐
│ IDLE │───▶│  ANALYZING   │───▶│ INTERVIEWING │───▶│COACHING│───▶│SUMMARY │───▶│ DONE │
└──────┘    └──────────────┘    └──────────────┘    └────────┘    └────────┘    └──────┘
               │                      │                   │          │
               │                      │                   │          │
               │                      ▼                   │          │
               │              可以循环追问                  │          │
               │              ┌──────────────┐            │          │
               └─────────────▶│  异常退出    │◀───────────┘          │
                              └──────────────┘                      │
                                    │                               │
                                    └───────────────────────────────┘
                                              异常结束
```

### 3.2 状态转换详解

#### IDLE → ANALYZING
```
触发条件：用户上传简历文件
执行动作：调用 resume_analyst
数据传递：resume_text
预期输出：简历分析结果
失败处理：提示用户重新上传
```

#### ANALYZING → INTERVIEWING
```
触发条件：resume_analyst 返回分析结果
执行动作：初始化面试状态，调用 interviewer
数据传递：resume_analysis（包含技术栈、项目亮点、面试重点）
预期输出：第一个问题
失败处理：重新调用 resume_analyst，最多重试3次
```

#### INTERVIEWING → COACHING
```
触发条件：用户回答完毕（超过10个字符的有效输入）
执行动作：调用 interview_coach
数据传递：当前问题 + 用户回答 + 历史对话
预期输出：点评内容
失败处理：记录日志，继续面试（coach失败不影响主流程）
```

#### COACHING → INTERVIEWING
```
触发条件：coach点评完成
执行动作：判断是否继续面试或结束
数据传递：coach的建议 + 下一问题方向
预期输出：下一组问题 或 进入SUMMARY
提前结束条件：
  - 用户说"面试结束"
  - 已完成目标问题数量
  - 用户主动结束
```

#### SUMMARY → DONE
```
触发条件：面试结束
执行动作：调用 orchestrator 生成总结
数据传递：完整对话历史
预期输出：面试总结报告
```

### 3.3 数据流转设计

```python
# 数据流转示意
class InterviewContext:
    """面试上下文 - 按需加载，不传递全量数据"""
    
    def __init__(self, user_id: str):
        self.user_id = user_id
        self.state = InterviewState.IDLE
        self.resume_text = None
        self.resume_analysis = None  # 简历分析师的输出
        self.conversation_history = []
        self.current_round = 0
    
    def get_context_for_agent(self, agent_name: str) -> dict:
        """根据不同Agent返回不同的上下文"""
        if agent_name == "resume_analyst":
            return {
                "resume_text": self.resume_text,
                "user_id": self.user_id
            }
        
        elif agent_name == "interviewer":
            return {
                "resume_summary": self._summarize_for_interviewer(),
                "question_number": self.current_round,
                "history": self._get_recent_history(5),
                "focus_areas": self.resume_analysis.get("interview_focus", [])
            }
        
        elif agent_name == "interview_coach":
            return {
                "conversation_history": self.conversation_history[-1],  # 只传最近一轮
                "user_mood": self._detect_mood()
            }
        
        elif agent_name == "orchestrator":
            return {
                "current_state": self.state,
                "user_input": self._get_latest_input(),
                "history_summary": self._get_history_summary()
            }
    
    def _summarize_for_interviewer(self) -> str:
        """为面试官生成简历摘要"""
        if not self.resume_analysis:
            return ""
        return f"""
        技术栈: {', '.join(self.resume_analysis.get('technical_stack', {}).get('languages', []))}
        框架: {', '.join(self.resume_analysis.get('technical_stack', {}).get('frameworks', []))}
        亮点项目: {len(self.resume_analysis.get('projects', []))} 个
        面试重点: {', '.join(self.resume_analysis.get('interview_focus', []))}
        """
```

### 3.4 Prompt文件管理

```
project/
├── prompts/
│   ├── resume_analyst.yaml      # 简历分析师提示词
│   ├── interviewer.yaml        # 面试官提示词
│   ├── interview_coach.yaml    # 面试教练提示词
│   └── orchestrator.yaml       # 流程管家提示词
├── agents/
│   ├── __init__.py
│   ├── base.py                 # Agent基类
│   ├── resume_analyst.py
│   ├── interviewer.py
│   ├── interview_coach.py
│   └── orchestrator.py
├── core/
│   ├── state_machine.py        # 状态机
│   └── context.py              # 上下文管理
└── main.py
```

**prompt YAML示例：**

```yaml
# prompts/interviewer.yaml
name: "interviewer"
temperature: 0.5
max_tokens: 4096
description: "技术面试官，定制化提问"

system_prompt: |
  # 角色定义
  你是一位经验丰富的技术面试官...
  [完整提示词内容]

output_schema:
  type: "json"
  fields:
    - questions
    - estimated_duration
    - suggested_topic

context_requirements:
  required:
    - resume_summary
    - question_number
  optional:
    - previous_answers
    - topic_preference
```

---

## 附录：Agent调用时序图

```
用户          流程管家        简历分析师       面试官          面试教练
 │               │               │              │               │
 │──上传简历────▶│               │              │               │
 │               │──analyze────▶│              │               │
 │               │               │──分析完成───▶│               │
 │               │◀──────────────│              │               │
 │◀──开始面试───│               │              │               │
 │               │               │              │               │
 │──回答问题────▶│               │              │               │
 │               │──coach──────────────────────▶│               │
 │               │◀─────────────────────────────────────────────│
 │◀──点评反馈───│               │              │               │
 │               │               │              │               │
 │──继续/结束──▶│               │              │               │
 │               │               │              │               │
 │               │──summarize──────────────────────────────▶│   │
 │               │◀─────────────────────────────────────────────│
 │◀──面试总结───│               │              │               │
```

---

*文档版本: v1.0*
*最后更新: 2024年*
*维护者: Interview Pro Team*
