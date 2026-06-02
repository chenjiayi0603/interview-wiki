# Interview Pro 英语面试分层反馈技术实现方案

## 概述

本文档为 Interview Pro 英语面试模块设计"智能选择性反馈模式"的技术实现方案，核心目标是解决面试场景中"及时反馈"与"不打断用户思路"之间的矛盾。通过三层级反馈架构，实现既不干扰用户表达流畅性，又能提供有效学习反馈的平衡体验。

---

## 一、整体架构设计

### 1.1 系统模块划分

```
┌─────────────────────────────────────────────────────────────────────┐
│                          用户对话层                                  │
│     语音输入 → ASR转文字 → 用户回答文本                              │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────────┐
│                      实时分析层（异步非阻塞）                          │
│  ┌────────────────┐    ┌────────────────┐    ┌────────────────┐     │
│  │   流利度检测   │    │   语法错误检测  │    │   发音评分     │     │
│  └────────────────┘    └────────────────┘    └────────────────┘     │
│  ┌────────────────┐    ┌────────────────┐                           │
│  │   内容质量评估 │    │   关键词追踪   │                           │
│  └────────────────┘    └────────────────┘                           │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────────┐
│                      反馈决策层（状态机）                              │
│   实时分析结果 → 判断层级 → 维护用户状态：信心值/流利度曲线/错误积累   │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────────┐
│                        反馈生成层                                    │
│  Level 1: 表情/符号反馈（无声）                                       │
│  Level 2: 一句话微点评（不展开）                                      │
│  Level 3: 完整深度复盘报告                                            │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 技术选型

| 模块 | 推荐技术方案 | 备选方案 |
|------|-------------|---------|
| ASR语音转文字 | OpenAI Whisper（实时版） | 阿里云语音识别、讯飞语音转写 |
| 语法检测 | LanguageTool API | Grammarly API、自建语法纠错模型 |
| 发音评分 | 讯飞语音评测API | Google Speech-to-Text + 自建评分模型 |
| LLM推理 | vLLM + GPT-3.5-turbo | Claude API、ChatGLM |
| 实时状态存储 | Redis | Memcached、SQLite（单机） |
| 对话记忆 | LangChain ConversationBufferMemory | 自建会话管理 |
| PDF报告生成 | ReportLab / WeasyPrint | fpdf2 |

### 1.3 数据流向图

```
用户说话
    ↓
[Whisper流式识别] ───→ 实时文本片段
    ↓
[异步分析管道]
    ├── 流利度检测（规则+模型）
    ├── 语法错误检测（LanguageTool/LLM）
    └── 关键词/主题匹配
    ↓
[反馈决策状态机]
    ├── 分析结果 → 错误队列
    ├── 评估置信度
    └── 判断反馈时机
    ↓
[反馈执行]
    ├── Level 1: 发送表情符号（WebSocket推送）
    ├── Level 2: 生成一句话（LLM调用）
    └── Level 3: 汇总分析（批量LLM调用）
    ↓
用户感知
```

---

## 二、分层反馈的具体实现

### 2.1 Level 1：实时轻微反馈（不打断）

#### 2.1.1 触发时机

用户正在说话的过程中，采用流式分析，仅在特定条件下无声反馈。

#### 2.1.2 反馈类型定义

```python
from enum import Enum
from dataclasses import dataclass
from typing import Optional

class Level1FeedbackType(Enum):
    """Level 1 反馈类型枚举"""
    APPROVAL = "✅"           # 回答流畅完整
    MINOR_WARNING = "⚠️"     # 检测到小问题但不打断
    CLARIFICATION = "❓"     # 未听清需要确认
    ENCOURAGEMENT = "💪"     # 检测到信心不足
    TIME_REMAINDER = "⏱️"    # 时间提醒

@dataclass
class Level1Feedback:
    """Level 1 反馈数据结构"""
    feedback_type: Level1FeedbackType
    confidence: float  # 置信度 0-1
    reason: Optional[str] = None
    show_to_user: bool = True  # 是否展示给用户
```

#### 2.1.3 核心检测逻辑

```python
import re
from collections import deque
from threading import Thread
from typing import Generator, List
import asyncio

class RealtimeAnalyzer:
    """实时分析器"""
    
    # 犹豫词列表
    HESITATION_WORDS = {'um', 'uh', 'ah', 'er', 'like', 'you know', 'basically', 
                        '那个', '就是说', '就是'}
    
    # 流畅完成的标志
    SENTENCE_COMPLETION_MARKERS = {'.', '!', '?', '...'}
    
    def __init__(self, websocket_manager):
        self.ws_manager = websocket_manager
        self.error_buffer = deque(maxlen=50)  # 错误积累缓冲区
        self.hesitation_count = 0
        self.last_hesitation_time = 0
        self.sentence_buffer = ""
        
    async def process_stream(self, text_stream: Generator[str, None, None]) -> None:
        """处理流式输入"""
        async for text_chunk in text_stream:
            # 1. 检测犹豫词
            hesitation_detected = self._detect_hesitations(text_chunk)
            if hesitation_detected:
                self._log_for_later(text_chunk, 'hesitation')
            
            # 2. 检测语法错误（轻量级）
            errors = await self._quick_grammar_check(text_chunk)
            for error in errors:
                if error.severity == 'minor':
                    # 轻微错误：只记录，不反馈
                    self._log_for_later(error, 'minor_error')
                elif error.severity == 'critical':
                    # 严重错误：给提示但不打断
                    await self._send_subtle_warning(error)
            
            # 3. 检测句子完整性
            if self._is_sentence_complete(text_chunk):
                await self._send_approval_signal()
    
    def _detect_hesitations(self, text: str) -> bool:
        """检测犹豫词"""
        text_lower = text.lower().strip()
        for word in self.HESITATION_WORDS:
            if re.search(rf'\b{re.escape(word)}\b', text_lower):
                self.hesitation_count += 1
                return True
        return False
    
    def _detect_pause(self, pause_duration: float) -> bool:
        """检测停顿超过阈值"""
        PAUSE_THRESHOLD = 2.0  # 2秒
        if pause_duration > PAUSE_THRESHOLD:
            self._log_for_later({'type': 'long_pause', 'duration': pause_duration}, 'pause')
            return True
        return False
    
    def _is_sentence_complete(self, text: str) -> bool:
        """判断句子是否完整"""
        text = text.strip()
        if not text:
            return False
        # 检查结尾标点
        return any(text.endswith(marker) for marker in self.SENTENCE_COMPLETION_MARKERS)
    
    def _log_for_later(self, item, category: str) -> None:
        """记录待后续处理"""
        self.error_buffer.append({
            'item': item,
            'category': category,
            'timestamp': asyncio.get_event_loop().time()
        })
    
    async def _send_approval_signal(self) -> None:
        """发送认可信号"""
        await self.ws_manager.send({
            'type': 'level1_feedback',
            'feedback': Level1FeedbackType.APPROVAL.value,
            'confidence': 0.9
        })
    
    async def _send_subtle_warning(self, error) -> None:
        """发送轻微警告（不打断）"""
        await self.ws_manager.send({
            'type': 'level1_feedback',
            'feedback': Level1FeedbackType.MINOR_WARNING.value,
            'reason': error.description,
            'confidence': error.confidence
        })
    
    async def _quick_grammar_check(self, text: str) -> List:
        """快速语法检查（轻量级规则）"""
        # TODO: 接入LanguageTool API或轻量级语法模型
        # 当前先用规则快速检测明显错误
        errors = []
        
        # 常见错误模式
        patterns = [
            (r'\bam work\b', '时态错误：am work → am working'),
            (r'\bI is\b', '主谓不一致：I is → I am'),
            (r'\bmany informations\b', '可数名词错误：many informations → much information'),
        ]
        
        for pattern, description in patterns:
            if re.search(pattern, text):
                errors.append(type('obj', (object,), {
                    'severity': 'critical',
                    'description': description,
                    'confidence': 0.95
                })())
        
        return errors
    
    def get_accumulated_errors(self) -> List:
        """获取积累的错误列表（用于Level 3报告）"""
        return list(self.error_buffer)
```

#### 2.1.4 用户体验效果

| 场景 | 用户看到的效果 |
|------|---------------|
| 回答流畅完整 | AI那边冒一个 ✅ |
| 有小错误但不严重 | AI啥也不说，默默记下来 |
| 严重语法错误 | AI冒一个 ⚠️（无声） |
| 完全听不懂 | AI说："Pardon?"（极少触发） |
| 停顿超过2秒 | 默默记录，不反馈 |
| 犹豫词过多 | 默默记录，面试结束后反馈 |

---

### 2.2 Level 2：段落后微反馈（不展开）

#### 2.2.1 触发时机

用户完整回答完一整道题目后，进入下一题之前。

#### 2.2.2 反馈内容约束

| 约束项 | 具体要求 |
|--------|---------|
| 句子数量 | 最多2句话 |
| 字数上限 | 不超过50字 |
| 核心要点 | 只说最关键的1个点 |
| 响应速度 | 说完立刻进入下一题 |
| 语言 | 英文为主（面试场景） |

#### 2.2.3 反馈生成逻辑

```python
from dataclasses import dataclass
from typing import Optional, List, Dict
import openai
from enum import Enum

class AnswerQuality(Enum):
    """答案质量等级"""
    EXCELLENT = "excellent"      # >= 90分
    GOOD = "good"               # 70-89分
    AVERAGE = "average"          # 50-69分
    NEEDS_IMPROVEMENT = "needs_improvement"  # < 50分

@dataclass
class AnswerAnalysis:
    """答案分析结果"""
    overall_score: float
    fluency_score: float
    grammar_score: float
    vocabulary_score: float
    content_score: float
    errors: List[Dict]
    strengths: List[str]
    weaknesses: List[str]
    top_priority_improvement: Optional[str] = None

class Level2FeedbackGenerator:
    """Level 2 微点评生成器"""
    
    SYSTEM_PROMPT = """You are an English interview micro-feedback assistant.

【重要 Rules】
1. Give only ONE sentence of feedback, no more than 30 words
2. Focus on the MOST critical point only
3. Always start with positive, then one small improvement
4. Be encouraging and specific
5. Use simple English that an intermediate learner can understand

【Output Format】
✅ [Compliment]! One thing: [One specific improvement]

【Examples】
✅ Great fluency! One thing: try using past tense consistently.
✅ Good content! One thing: watch out for article usage (a/an/the).
✅ Nice structure! One thing: vary your sentence starters.
"""
    
    def __init__(self, openai_api_key: str):
        openai.api_key = openai_api_key
        self.model = "gpt-3.5-turbo"
        self.temperature = 0.3  # 保持简洁客观
    
    def generate(self, answer_analysis: AnswerAnalysis) -> str:
        """根据分析结果生成微点评"""
        
        # 根据整体评分选择不同的点评策略
        if answer_analysis.overall_score >= 90:
            return self._excellent_feedback(answer_analysis)
        elif answer_analysis.overall_score >= 70:
            return self._good_feedback(answer_analysis)
        elif answer_analysis.overall_score >= 50:
            return self._average_feedback(answer_analysis)
        else:
            return self._needs_improvement_feedback(answer_analysis)
    
    def _excellent_feedback(self, analysis: AnswerAnalysis) -> str:
        """优秀回答的点评"""
        praise_messages = [
            "Excellent! Very smooth and natural.",
            "Outstanding! Your English sounds very fluent.",
            "Perfect! Well-structured and articulate.",
        ]
        praise = random.choice(praise_messages)
        
        if analysis.top_priority_improvement:
            return f"{praise} One thing: {analysis.top_priority_improvement}"
        return praise
    
    def _good_feedback(self, analysis: AnswerAnalysis) -> str:
        """良好回答的点评"""
        praise = "Good answer!"
        if analysis.top_priority_improvement:
            return f"{praise} One thing: {analysis.top_priority_improvement}"
        return f"{praise} Keep up the good work!"
    
    def _average_feedback(self, analysis: AnswerAnalysis) -> str:
        """一般回答的点评"""
        praise = "Good effort!"
        if analysis.top_priority_improvement:
            return f"{praise} Try to: {analysis.top_priority_improvement}"
        return f"{praise} Let's keep practicing!"
    
    def _needs_improvement_feedback(self, analysis: AnswerAnalysis) -> str:
        """需要改进的回答的点评"""
        # 重点鼓励，不提太多细节
        encouragements = [
            "Good try! Focus on one thing at a time 💪",
            "Nice effort! Let's practice more together! 💪",
            "You're getting there! Keep going! 💪",
        ]
        return random.choice(encouragements)
    
    async def generate_with_llm(self, user_answer: str, question: str, 
                                 analysis: AnswerAnalysis) -> str:
        """使用LLM生成更智能的微点评"""
        
        # 构建错误信息摘要
        error_summary = ""
        if analysis.errors:
            top_errors = analysis.errors[:2]  # 只取前2个
            error_summary = "\n".join([
                f"- {e['type']}: {e['example']} → {e['correction']}"
                for e in top_errors
            ])
        
        user_prompt = f"""Question: {question}

User Answer: {user_answer}

Score Summary:
- Overall: {analysis.overall_score}/100
- Fluency: {analysis.fluency_score}/100
- Grammar: {analysis.grammar_score}/100

Top Errors (if any):
{error_summary}

Your ONE sentence micro-feedback:"""
        
        response = await openai.ChatCompletion.acreate(
            model=self.model,
            messages=[
                {"role": "system", "content": self.SYSTEM_PROMPT},
                {"role": "user", "content": user_prompt}
            ],
            temperature=self.temperature,
            max_tokens=60
        )
        
        return response.choices[0].message.content.strip()
```

#### 2.2.4 Prompt模板（直接使用版）

```
你是英语面试的微点评助手。请用1句话点评用户的回答，不超过30词。
重点只说一个最关键的点，不要展开。
先说肯定，再说一个小改进点。

用户回答：{user_answer}
问题：{question}

点评：
```

---

### 2.3 Level 3：结束后深度复盘（完整反馈）

#### 2.3.1 触发时机

所有题目都答完，面试结束。

#### 2.3.2 报告生成流程

```python
from dataclasses import dataclass, field
from typing import List, Dict, Optional
from datetime import datetime
from enum import Enum
import json

class ReportFormat(Enum):
    MARKDOWN = "markdown"
    PDF = "pdf"
    HTML = "html"

@dataclass
class QuestionRecord:
    """题目记录"""
    question_id: int
    question_text: str
    user_answer: str
    audio_path: Optional[str] = None
    asr_text: str = ""
    analysis: Optional[AnswerAnalysis] = None
    sentence_analysis: List[Dict] = field(default_factory=list)

@dataclass
class InterviewSession:
    """面试会话"""
    session_id: str
    start_time: datetime
    end_time: Optional[datetime] = None
    mode: str = "real_interview"  # learning / real_interview
    questions: List[QuestionRecord] = field(default_factory=list)
    overall_score: float = 0.0
    user_level: str = "intermediate"  # beginner / intermediate / advanced
    
@dataclass
class ErrorStatistics:
    """错误统计"""
    error_type: str
    count: int
    examples: List[Dict]
    improvement_suggestion: str

class Level3ReportGenerator:
    """Level 3 深度复盘报告生成器"""
    
    def __init__(self, openai_api_key: str):
        openai.api.api_key = openai_api_key
        self.model = "gpt-3.5-turbo"
    
    async def generate_full_report(self, session: InterviewSession) -> Dict:
        """生成完整报告"""
        
        # 1. 计算整体评分
        overall_score = self._calculate_overall_score(session)
        
        # 2. 生成逐句分析（异步并行）
        all_sentence_analysis = await self._analyze_all_sentences(session)
        
        # 3. 统计高频错误
        error_stats = self._analyze_error_patterns(session, all_sentence_analysis)
        
        # 4. 生成改进建议
        improvement_suggestions = self._generate_improvement_suggestions(
            error_stats, overall_score
        )
        
        # 5. 组装报告
        report = {
            'session_id': session.session_id,
            'date': session.end_time or datetime.now(),
            'duration': self._calculate_duration(session),
            'overall_score': overall_score,
            'score_breakdown': self._generate_score_breakdown(session),
            'question_reviews': self._generate_question_reviews(
                session, all_sentence_analysis
            ),
            'error_statistics': error_stats,
            'improvement_suggestions': improvement_suggestions,
            'practice_recommendations': self._generate_practice_recommendations(
                error_stats
            )
        }
        
        return report
    
    def _calculate_overall_score(self, session: InterviewSession) -> float:
        """计算整体评分"""
        if not session.questions:
            return 0.0
        
        total_score = sum(
            q.analysis.overall_score if q.analysis else 0 
            for q in session.questions
        )
        return round(total_score / len(session.questions), 1)
    
    def _generate_score_breakdown(self, session: InterviewSession) -> Dict:
        """生成评分细分"""
        scores = {
            'fluency': [],
            'grammar': [],
            'vocabulary': [],
            'content': []
        }
        
        for q in session.questions:
            if q.analysis:
                scores['fluency'].append(q.analysis.fluency_score)
                scores['grammar'].append(q.analysis.grammar_score)
                scores['vocabulary'].append(q.analysis.vocabulary_score)
                scores['content'].append(q.analysis.content_score)
        
        return {
            dimension: round(sum(values) / len(values), 1) if values else 0
            for dimension, values in scores.items()
        }
    
    async def _analyze_all_sentences(self, session: InterviewSession) -> Dict[int, List[Dict]]:
        """对所有回答进行逐句分析"""
        analysis_results = {}
        
        # 并行分析所有回答
        tasks = []
        for q in session.questions:
            task = self._analyze_single_answer(q.question_text, q.user_answer)
            tasks.append((q.question_id, task))
        
        # 等待所有任务完成
        for q_id, coro in tasks:
            result = await coro
            analysis_results[q_id] = result
        
        return analysis_results
    
    async def _analyze_single_answer(self, question: str, answer: str) -> List[Dict]:
        """分析单个回答的逐句"""
        
        prompt = f"""请逐句分析这段英语面试回答，找出所有错误并给出改进建议。

问题：{question}
回答：{answer}

请按以下格式输出：

【原句】I am work in a software company for 3 years.
❌ 错误说明：时态错误，"am work" 应该是 "have been working"
✅ 改进建议：I have been working in a software company for 3 years.

【原句】I have many experience in Python development.
❌ 错误说明：可数名词错误，"experience" 是不可数名词，不能用 "many"
✅ 改进建议：I have extensive experience in Python development.

---

【总体评价】
1. 优点：内容结构清晰，表达了丰富的工作经验
2. 改进点：注意时态一致性和名词可数性
3. 地道表达推荐：可以用 "I've spent 3 years working in..." 增加自然度"""

        response = await openai.ChatCompletion.acreate(
            model=self.model,
            messages=[
                {"role": "user", "content": prompt}
            ],
            temperature=0.2,
            max_tokens=2000
        )
        
        # 解析返回结果
        result_text = response.choices[0].message.content
        return self._parse_sentence_analysis(result_text)
    
    def _parse_sentence_analysis(self, text: str) -> List[Dict]:
        """解析逐句分析结果"""
        sentences = []
        current_sentence = None
        
        for line in text.split('\n'):
            if line.startswith('【原句】'):
                if current_sentence:
                    sentences.append(current_sentence)
                current_sentence = {
                    'original': line.replace('【原句】', '').strip(),
                    'errors': [],
                    'corrections': []
                }
            elif line.startswith('❌') and current_sentence:
                current_sentence['errors'].append(line.replace('❌', '').strip())
            elif line.startswith('✅') and current_sentence:
                current_sentence['corrections'].append(line.replace('✅', '').strip())
        
        if current_sentence:
            sentences.append(current_sentence)
        
        return sentences
    
    def _analyze_error_patterns(self, session: InterviewSession, 
                                 all_analysis: Dict) -> List[ErrorStatistics]:
        """分析错误模式"""
        
        error_type_count = {}
        
        for q_id, sentences in all_analysis.items():
            for sentence in sentences:
                for error in sentence.get('errors', []):
                    # 简单分类错误类型
                    error_type = self._classify_error(error)
                    if error_type not in error_type_count:
                        error_type_count[error_type] = {
                            'count': 0,
                            'examples': []
                        }
                    error_type_count[error_type]['count'] += 1
                    error_type_count[error_type]['examples'].append({
                        'original': sentence.get('original', ''),
                        'error': error,
                        'correction': sentence.get('corrections', [''])[0] if sentence.get('corrections') else ''
                    })
        
        # 转换为统计列表
        stats = []
        for error_type, data in error_type_count.items():
            stats.append(ErrorStatistics(
                error_type=error_type,
                count=data['count'],
                examples=data['examples'][:3],  # 最多3个例子
                improvement_suggestion=self._get_suggestion_for_error_type(error_type)
            ))
        
        # 按出现次数排序
        stats.sort(key=lambda x: x.count, reverse=True)
        return stats
    
    def _classify_error(self, error_text: str) -> str:
        """分类错误类型"""
        error_lower = error_text.lower()
        
        if any(word in error_lower for word in ['tense', '时态']):
            return '时态错误'
        elif any(word in error_lower for word in ['plural', '单复数', '可数']):
            return '单复数错误'
        elif any(word in error_lower for word in ['preposition', '介词']):
            return '介词错误'
        elif any(word in error_lower for word in ['article', 'a/an/the']):
            return '冠词错误'
        elif any(word in error_lower for word in ['subject-verb', '主谓']):
            return '主谓不一致'
        elif any(word in error_lower for word in ['word order', '语序']):
            return '语序错误'
        else:
            return '其他错误'
    
    def _get_suggestion_for_error_type(self, error_type: str) -> str:
        """获取错误类型的改进建议"""
        suggestions = {
            '时态错误': '练习使用现在完成时和过去时，注意时间状语的搭配',
            '单复数错误': '复习常见不可数名词（information, experience, advice等）',
            '介词错误': '固定搭配需要单独记忆，如 "good at" vs "good in"',
            '冠词错误': '可数名词单独使用需要加冠词，抽象名词一般不加',
            '主谓不一致': '注意第三人称单数现在时动词变化',
            '语序错误': '英语陈述句语序：主语+谓语+宾语+状语'
        }
        return suggestions.get(error_type, '多加练习，注意语法规则')
    
    def _generate_improvement_suggestions(self, error_stats: List[ErrorStatistics],
                                           overall_score: float) -> List[str]:
        """生成改进建议"""
        suggestions = []
        
        # 基于错误统计
        for stat in error_stats[:3]:
            suggestions.append({
                'focus_area': stat.error_type,
                'priority': 'high' if stat.count >= 3 else 'medium',
                'suggestion': stat.improvement_suggestion,
                'practice_count': stat.count
            })
        
        # 基于整体评分
        if overall_score < 70:
            suggestions.append({
                'focus_area': '整体表达',
                'priority': 'high',
                'suggestion': '建议从基础句型开始练习，减少复杂从句的使用',
                'practice_count': 0
            })
        
        return suggestions
    
    def _generate_practice_recommendations(self, error_stats: List[ErrorStatistics]) -> List[Dict]:
        """生成练习推荐"""
        # 根据错误类型推荐练习题目
        practice_map = {
            '时态错误': [
                'Tell me about your previous work experience',
                'What did you do last weekend?',
                'Describe a time when you faced a challenge'
            ],
            '单复数错误': [
                'Tell me about your hometown',
                'What hobbies do you have?',
                'Describe your daily routine'
            ],
            '介词错误': [
                'What are you good at?',
                'Tell me about your education background',
                'How do you spend your free time?'
            ],
            '冠词错误': [
                'Tell me about yourself',
                'What is your greatest strength?',
                'Describe your personality'
            ]
        }
        
        recommendations = []
        for stat in error_stats[:3]:
            if stat.error_type in practice_map:
                recommendations.append({
                    'error_type': stat.error_type,
                    'recommended_questions': practice_map[stat.error_type]
                })
        
        return recommendations
```

#### 2.3.3 报告输出模板

生成的报告包含以下结构：

```markdown
# Interview Pro 英语面试深度复盘报告

## 📊 整体评分卡片

| 维度 | 分数 | 可视化 |
|------|------|--------|
| 发音 | 82/100 | ████████████░░░ |
| 语法 | 75/100 | ███████████░░░░ |
| 词汇 | 80/100 | ████████████░░ |
| 流利度 | 75/100 | ███████████░░░░ |
| 内容 | 78/100 | ████████████░░ |

**综合评分：78/100**

---

## 📝 逐题逐句分析

### 问题 1: Tell me about yourself

**你的回答:**
> "I am work in a software company for 3 years."

| 分析项 | 内容 |
|--------|------|
| ❌ 错误 | 时态错误："am work" → 应该是现在完成进行时 |
| ✅ 改进 | I have been working in a software company for 3 years. |

**你的回答:**
> "I have many experience in Python development."

| 分析项 | 内容 |
|--------|------|
| ❌ 错误 | 可数名词错误：experience是不可数名词 |
| ✅ 改进 | I have extensive experience in Python development. |

💡 **总体评价：** 内容结构很好，但时态和单复数需要注意

---

## 📈 错误统计

本次面试出现的高频错误：

| 错误类型 | 出现次数 | 示例 |
|----------|----------|------|
| 时态错误 | 5次 | am work → have been working |
| 单复数错误 | 3次 | many experience → much experience |
| 介词错误 | 2次 | good at → good in |
| 冠词错误 | 2次 | go to school → go to the school |

---

## 🎯 改进建议

### 重点练习方向

1. **现在完成时态** - 建议练习使用 have been V-ing 结构
2. **不可数名词** - information, experience, advice 等
3. **减少犹豫词** - um, ah, like 等填充词

### 推荐练习题目

- Tell me about a challenging project
- What are your strengths and weaknesses?
- Describe your career goals

---

## 💪 下一步行动

根据您的表现，推荐：
- 每日练习：5分钟语法专项
- 本周目标：减少犹豫词使用
- 下次面试前：复习本次报告中的错误
```

#### 2.3.4 PDF报告生成

```python
from reportlab.lib.pagesizes import A4
from reportlab.lib import colors
from reportlab.lib.styles import getSampleStyleSheet, ParagraphStyle
from reportlab.platypus import SimpleDocTemplate, Paragraph, Spacer, Table, TableStyle, Image
from reportlab.lib.units import inch
from reportlab.graphics.shapes import Drawing
from reportlab.graphics.charts.barcharts import VerticalBarChart

class PDFReportGenerator:
    """PDF报告生成器"""
    
    def __init__(self, output_path: str):
        self.doc = SimpleDocTemplate(
            output_path,
            pagesize=A4,
            rightMargin=72,
            leftMargin=72,
            topMargin=72,
            bottomMargin=18
        )
        self.styles = getSampleStyleSheet()
        self.story = []
    
    def build_report(self, report_data: Dict) -> None:
        """构建PDF报告"""
        
        # 标题
        self._add_title("Interview Pro 英语面试深度复盘报告")
        self._add_spacer()
        
        # 整体评分
        self._add_heading("整体评分")
        self._add_score_card(report_data['overall_score'], report_data['score_breakdown'])
        self._add_spacer()
        
        # 逐题分析
        self._add_heading("逐题分析")
        for q_review in report_data['question_reviews']:
            self._add_question_review(q_review)
        
        # 错误统计
        self._add_heading("错误统计")
        self._add_error_statistics(report_data['error_statistics'])
        self._add_spacer()
        
        # 改进建议
        self._add_heading("改进建议")
        self._add_improvement_suggestions(report_data['improvement_suggestions'])
        
        # 生成PDF
        self.doc.build(self.story)
    
    def _add_title(self, text: str) -> None:
        """添加标题"""
        style = ParagraphStyle(
            'CustomTitle',
            parent=self.styles['Heading1'],
            fontSize=24,
            textColor=colors.HexColor('#2C3E50'),
            spaceAfter=30
        )
        self.story.append(Paragraph(text, style))
    
    def _add_heading(self, text: str) -> None:
        """添加标题"""
        style = ParagraphStyle(
            'CustomHeading',
            parent=self.styles['Heading2'],
            fontSize=16,
            textColor=colors.HexColor('#34495E'),
            spaceAfter=12
        )
        self.story.append(Paragraph(text, style))
    
    def _add_score_card(self, overall: float, breakdown: Dict) -> None:
        """添加评分卡片"""
        # 创建一个简单的评分条
        data = [
            ['发音', breakdown.get('fluency', 0), '█' * 10],
            ['语法', breakdown.get('grammar', 0), '█' * 10],
            ['词汇', breakdown.get('vocabulary', 0), '█' * 10],
            ['流利度', breakdown.get('fluency', 0), '█' * 10],
            ['内容', breakdown.get('content', 0), '█' * 10],
        ]
        
        t = Table(data, colWidths=[80, 50, 200])
        t.setStyle(TableStyle([
            ('BACKGROUND', (0, 0), (-1, -1), colors.white),
            ('TEXTCOLOR', (0, 0), (-1, -1), colors.HexColor('#2C3E50')),
            ('FONTNAME', (0, 0), (-1, -1), 'Helvetica'),
            ('FONTSIZE', (0, 0), (-1, -1), 11),
            ('ALIGN', (1, 0), (1, -1), 'CENTER'),
            ('GRID', (0, 0), (-1, -1), 1, colors.HexColor('#BDC3C7')),
            ('VALIGN', (0, 0), (-1, -1), 'MIDDLE'),
            ('ROWBACKGROUNDS', (0, 0), (-1, -1), [colors.white, colors.HexColor('#F8F9FA')]),
        ]))
        self.story.append(t)
        
        # 综合评分
        score_style = ParagraphStyle(
            'ScoreStyle',
            parent=self.styles['Normal'],
            fontSize=28,
            textColor=colors.HexColor('#27AE60'),
            alignment=1  # 居中
        )
        self._add_spacer()
        self.story.append(Paragraph(f"综合评分：<b>{overall}</b>/100", score_style))
    
    def _add_question_review(self, review: Dict) -> None:
        """添加题目复盘"""
        self.story.append(Paragraph(f"<b>问题：</b>{review['question']}", self.styles['Normal']))
        self._add_spacer(6)
        
        for sentence in review.get('sentences', []):
            if sentence.get('errors'):
                self.story.append(Paragraph(
                    f"<font color='red'>❌ {sentence['original']}</font>",
                    self.styles['Normal']
                ))
                for error in sentence['errors']:
                    self.story.append(Paragraph(f"  错误：{error}", self.styles['Normal']))
                if sentence.get('corrections'):
                    for correction in sentence['corrections']:
                        self.story.append(Paragraph(
                            f"<font color='green'>✅ {correction}</font>",
                            self.styles['Normal']
                        ))
                self._add_spacer(6)
        
        self._add_spacer(12)
    
    def _add_error_statistics(self, stats: List) -> None:
        """添加错误统计"""
        data = [['错误类型', '次数', '示例']]
        for stat in stats:
            example = stat.examples[0]['error'] if stat.examples else '-'
            data.append([stat.error_type, str(stat.count), example])
        
        t = Table(data, colWidths=[100, 50, 250])
        t.setStyle(TableStyle([
            ('BACKGROUND', (0, 0), (-1, 0), colors.HexColor('#3498DB')),
            ('TEXTCOLOR', (0, 0), (-1, 0), colors.white),
            ('FONTNAME', (0, 0), (-1, -1), 'Helvetica'),
            ('FONTSIZE', (0, 0), (-1, -1), 10),
            ('ALIGN', (1, 0), (1, -1), 'CENTER'),
            ('GRID', (0, 0), (-1, -1), 1, colors.HexColor('#BDC3C7')),
            ('VALIGN', (0, 0), (-1, -1), 'MIDDLE'),
        ]))
        self.story.append(t)
    
    def _add_improvement_suggestions(self, suggestions: List) -> None:
        """添加改进建议"""
        for i, suggestion in enumerate(suggestions, 1):
            self.story.append(Paragraph(
                f"{i}. <b>{suggestion['focus_area']}</b>：{suggestion['suggestion']}",
                self.styles['Normal']
            ))
            self._add_spacer(6)
    
    def _add_spacer(self, height: float = 12) -> None:
        """添加空白"""
        self.story.append(Spacer(1, height))
```

---

## 三、状态管理与数据流转

### 3.1 Redis存储的用户状态

```python
import redis
import json
from dataclasses import dataclass, asdict
from typing import List, Optional, Dict
from datetime import datetime

@dataclass
class UserState:
    """用户状态数据模型"""
    user_id: str
    current_mode: str  # learning / real_interview
    confidence_score: float  # 0-1
    error_count: int
    consecutive_wrong: int
    consecutive_right: int
    errors_accumulated: List[Dict]
    fluency_curve: List[float]
    current_question: int
    session_id: str
    last_update: str

class UserStateManager:
    """用户状态管理器（基于Redis）"""
    
    STATE_KEY_PREFIX = "interview_pro:user_state:"
    SESSION_KEY_PREFIX = "interview_pro:session:"
    
    def __init__(self, redis_host='localhost', redis_port=6379, redis_db=0):
        self.redis = redis.Redis(
            host=redis_host,
            port=redis_port,
            db=redis_db,
            decode_responses=True
        )
        self.state_ttl = 3600 * 24  # 24小时过期
    
    def _get_state_key(self, user_id: str) -> str:
        return f"{self.STATE_KEY_PREFIX}{user_id}"
    
    def _get_session_key(self, session_id: str) -> str:
        return f"{self.SESSION_KEY_PREFIX}{session_id}"
    
    def get_user_state(self, user_id: str) -> Optional[UserState]:
        """获取用户状态"""
        key = self._get_state_key(user_id)
        data = self.redis.get(key)
        if data:
            return UserState(**json.loads(data))
        return None
    
    def update_user_state(self, user_id: str, state: UserState) -> None:
        """更新用户状态"""
        key = self._get_state_key(user_id)
        state.last_update = datetime.now().isoformat()
        self.redis.setex(
            key,
            self.state_ttl,
            json.dumps(asdict(state))
        )
    
    def init_user_state(self, user_id: str, mode: str = 'real_interview') -> UserState:
        """初始化用户状态"""
        state = UserState(
            user_id=user_id,
            current_mode=mode,
            confidence_score=0.5,
            error_count=0,
            consecutive_wrong=0,
            consecutive_right=0,
            errors_accumulated=[],
            fluency_curve=[],
            current_question=0,
            session_id='',
            last_update=datetime.now().isoformat()
        )
        self.update_user_state(user_id, state)
        return state
    
    def add_error(self, user_id: str, error: Dict) -> None:
        """添加错误记录"""
        state = self.get_user_state(user_id)
        if state:
            error['reported'] = False
            state.errors_accumulated.append(error)
            state.error_count += 1
            state.consecutive_wrong += 1
            state.consecutive_right = 0
            self.update_user_state(user_id, state)
    
    def add_success(self, user_id: str) -> None:
        """添加成功记录"""
        state = self.get_user_state(user_id)
        if state:
            state.consecutive_right += 1
            state.consecutive_wrong = 0
            self.update_user_state(user_id, state)
    
    def update_fluency(self, user_id: str, fluency_score: float) -> None:
        """更新流利度"""
        state = self.get_user_state(user_id)
        if state:
            state.fluency_curve.append(fluency_score)
            # 只保留最近20个数据点
            if len(state.fluency_curve) > 20:
                state.fluency_curve = state.fluency_curve[-20:]
            self.update_user_state(user_id, state)
    
    def is_fluency_decreasing(self, user_id: str, threshold: int = 3) -> bool:
        """判断流利度是否持续下降"""
        state = self.get_user_state(user_id)
        if not state or len(state.fluency_curve) < threshold + 1:
            return False
        
        recent = state.fluency_curve[-threshold:]
        # 连续下降
        return all(recent[i] > recent[i+1] for i in range(len(recent)-1))
    
    def get_unreported_errors(self, user_id: str) -> List[Dict]:
        """获取未反馈的错误"""
        state = self.get_user_state(user_id)
        if not state:
            return []
        return [e for e in state.errors_accumulated if not e.get('reported')]
    
    def mark_errors_reported(self, user_id: str, indices: List[int]) -> None:
        """标记错误已反馈"""
        state = self.get_user_state(user_id)
        if not state:
            return
        
        for idx in indices:
            if idx < len(state.errors_accumulated):
                state.errors_accumulated[idx]['reported'] = True
        
        self.update_user_state(user_id, state)
```

### 3.2 自动难度调节

```python
class DifficultyManager:
    """难度管理器"""
    
    DIFFICULTY_LEVELS = ['easy', 'medium', 'hard', 'expert']
    
    def __init__(self, state_manager: UserStateManager):
        self.state_manager = state_manager
    
    def should_adjust_difficulty(self, user_id: str) -> Optional[str]:
        """判断是否需要调整难度"""
        state = self.state_manager.get_user_state(user_id)
        if not state:
            return None
        
        # 连续答对3题 → 提升难度
        if state.consecutive_right >= 3:
            return 'increase'
        
        # 连续答错2题 → 降低难度
        if state.consecutive_wrong >= 2:
            return 'decrease'
        
        # 流利度持续下降 → 给鼓励，可能降低难度
        if self.state_manager.is_fluency_decreasing(user_id):
            return 'decrease'
        
        return None
    
    def adjust_difficulty(self, user_id: str, current_level: str) -> str:
        """执行难度调整"""
        adjustment = self.should_adjust_difficulty(user_id)
        
        if not adjustment:
            return current_level
        
        current_idx = self.DIFFICULTY_LEVELS.index(current_level)
        
        if adjustment == 'increase' and current_idx < len(self.DIFFICULTY_LEVELS) - 1:
            return self.DIFFICULTY_LEVELS[current_idx + 1]
        
        if adjustment == 'decrease' and current_idx > 0:
            return self.DIFFICULTY_LEVELS[current_idx - 1]
        
        return current_level
    
    def get_difficulty_config(self, level: str) -> Dict:
        """获取难度配置"""
        configs = {
            'easy': {
                'question_complexity': 'simple',
                'speech_speed': 'slow',
                'allow_repetition': True,
                'hint_available': True,
                'time_per_question': 120,  # 秒
            },
            'medium': {
                'question_complexity': 'moderate',
                'speech_speed': 'normal',
                'allow_repetition': True,
                'hint_available': False,
                'time_per_question': 90,
            },
            'hard': {
                'question_complexity': 'challenging',
                'speech_speed': 'normal',
                'allow_repetition': False,
                'hint_available': False,
                'time_per_question': 60,
            },
            'expert': {
                'question_complexity': 'advanced',
                'speech_speed': 'fast',
                'allow_repetition': False,
                'hint_available': False,
                'time_per_question': 45,
            }
        }
        return configs.get(level, configs['medium'])
```

---

## 四、两种模式的切换实现

### 4.1 模式定义

```python
from dataclasses import dataclass
from typing import Dict, List, Optional
from enum import Enum

class InterviewMode(Enum):
    """面试模式枚举"""
    LEARNING = "learning"       # 学习模式
    REAL_INTERVIEW = "real_interview"  # 真实面试模式

@dataclass
class ModeConfig:
    """模式配置"""
    # Level 1 配置
    level1_enabled: bool
    level1_detail_level: str  # minimal / standard
    
    # Level 2 配置
    level2_enabled: bool
    level2_detail_level: str  # brief / standard / detailed
    level2_max_sentences: int
    
    # Level 3 配置
    post_interview_report: str  # none / brief / full
    
    # 交互规则
    interrupt_allowed: bool
    hint_available: bool
    repetition_allowed: bool
    
    # 面试官行为
    interviewer_style: str  # supportive / neutral / strict
    encouragement_frequency: str  # high / normal / minimal

class ModeConfigManager:
    """模式配置管理器"""
    
    CONFIGS = {
        InterviewMode.LEARNING: ModeConfig(
            level1_enabled=True,
            level1_detail_level='standard',
            level2_enabled=True,
            level2_detail_level='detailed',
            level2_max_sentences=3,
            post_interview_report='full',
            interrupt_allowed=True,
            hint_available=True,
            repetition_allowed=True,
            interviewer_style='supportive',
            encouragement_frequency='high'
        ),
        InterviewMode.REAL_INTERVIEW: ModeConfig(
            level1_enabled=True,
            level1_detail_level='minimal',
            level2_enabled=True,
            level2_detail_level='brief',
            level2_max_sentences=2,
            post_interview_report='full',
            interrupt_allowed=False,
            hint_available=False,
            repetition_allowed=False,
            interviewer_style='neutral',
            encouragement_frequency='minimal'
        )
    }
    
    @classmethod
    def get_config(cls, mode: InterviewMode) -> ModeConfig:
        """获取模式配置"""
        return cls.CONFIGS.get(mode)
    
    @classmethod
    def switch_mode(cls, current_mode: InterviewMode) -> InterviewMode:
        """切换模式"""
        if current_mode == InterviewMode.LEARNING:
            return InterviewMode.REAL_INTERVIEW
        return InterviewMode.LEARNING
```

### 4.2 面试官Prompt模板

#### 学习模式

```
你是英语面试的学习助手。请用英语进行面试练习。

【重要规则】
1. 可以适当打断用户的错误，及时纠正
2. 用户回答完后，给出详细的一到两句点评
3. 如果用户卡住，可以给出提示
4. 多给予鼓励，营造轻松的学习氛围
5. 保持专业、耐心的态度

当前问题：{current_question}
用户回答：{user_answer}

请输出：
1. 详细点评（1-2句话）
2. 下一个面试问题
```

#### 真实面试模式

```
你是英语面试官。请用英语进行模拟面试。

【重要规则】
1. 绝对不要中途打断用户
2. 用户回答完后，只给出一句话的微点评
3. 不要给提示，让用户独立完成
4. 保持专业、客观的评分态度
5. 点评完后立刻问下一题

当前问题：{current_question}
用户回答：{user_answer}

请输出：
1. 一句话微点评（英文）
2. 下一个面试问题（英文）
```

### 4.3 模式切换UI

```python
class ModeSwitcher:
    """模式切换器"""
    
    def __init__(self, state_manager: UserStateManager, 
                 report_generator: Level3ReportGenerator):
        self.state_manager = state_manager
        self.report_generator = report_generator
    
    async def switch_mode(self, user_id: str, new_mode: InterviewMode) -> Dict:
        """切换模式"""
        state = self.state_manager.get_user_state(user_id)
        if not state:
            state = self.state_manager.init_user_state(user_id, new_mode.value)
        else:
            state.current_mode = new_mode.value
            self.state_manager.update_user_state(user_id, state)
        
        config = ModeConfigManager.get_config(new_mode)
        
        return {
            'mode': new_mode.value,
            'config': {
                'level1_enabled': config.level1_enabled,
                'level2_enabled': config.level2_enabled,
                'interrupt_allowed': config.interrupt_allowed,
                'hint_available': config.hint_available
            },
            'message': self._get_mode_switch_message(new_mode)
        }
    
    def _get_mode_switch_message(self, mode: InterviewMode) -> str:
        messages = {
            InterviewMode.LEARNING: 
                "切换到学习模式：反馈更详细，可随时提问和获取提示",
            InterviewMode.REAL_INTERVIEW: 
                "切换到真实面试模式：模拟真实面试环境，不打断，不给提示"
        }
        return messages.get(mode, "")
```

---

## 五、关键Prompt模板汇总

### 5.1 微点评Prompt

```markdown
你是英语面试的微点评助手。请用1句话点评用户的回答，不超过30词。
重点只说一个最关键的点，不要展开。
先说肯定，再说一个小改进点。

评分参考：
- 整体: {overall_score}/100
- 流利度: {fluency_score}/100
- 语法: {grammar_score}/100

用户回答：{user_answer}
问题：{question}

点评：
```

### 5.2 逐句分析Prompt

```markdown
请逐句分析这段英语面试回答，找出所有错误并给出改进建议。

问题：{question}
回答：{user_answer}

请按以下格式输出：

【原句】I am work in a software company for 3 years.
❌ 错误说明：时态错误，"am work" 应该是 "have been working"
✅ 改进建议：I have been working in a software company for 3 years.

【原句】I have many experience in Python development.
❌ 错误说明：可数名词错误，"experience" 是不可数名词，不能用 "many"
✅ 改进建议：I have extensive experience in Python development.

---

【总体评价】
1. 优点：内容结构清晰，表达了丰富的工作经验
2. 改进点：注意时态一致性和名词可数性
3. 地道表达推荐：可以用 "I've spent 3 years working in..." 增加自然度
```

### 5.3 完整报告生成Prompt

```markdown
请根据以下面试数据生成一份完整的复盘报告。

面试问题与回答：
{question_answers}

错误记录：
{error_records}

请生成包含以下部分的Markdown报告：
1. 整体评分卡片（各维度分数）
2. 逐题逐句分析（每个回答的错误和改进建议）
3. 常见错误统计（错误类型、次数、示例）
4. 改进建议（可执行的练习建议）
5. 推荐练习题目（针对本次暴露的问题）
```

### 5.4 面试官Prompt

```markdown
你是英语面试官。请用英语进行模拟面试。

【模式】{mode}  # learning / real_interview

【重要规则】
1. 绝对不要中途打断用户
2. 用户回答完后，根据模式给出相应的反馈
3. 保持专业、自然的对话感
4. 问题要清晰，语速适中

{additional_rules_based_on_mode}

当前问题：{current_question}
用户回答：{user_answer}

请输出：
1. 微点评（一句话）
2. 下一个面试问题（英语）
```

---

## 六、性能优化考虑

### 6.1 实时分析异步化

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor
from queue import Queue

class AsyncAnalyzer:
    """异步分析器"""
    
    def __init__(self):
        self.executor = ThreadPoolExecutor(max_workers=4)
        self.analysis_queue = Queue()
        
    async def analyze_realtime(self, text_chunk: str) -> Optional[Dict]:
        """实时分析（轻量级，异步执行）"""
        # 轻量级规则检测 - 同步执行
        light_result = self._quick_rule_check(text_chunk)
        
        # 如果需要深度分析，加入队列异步执行
        if light_result.get('needs_deep_analysis'):
            loop = asyncio.get_event_loop()
            deep_result = await loop.run_in_executor(
                self.executor,
                self._deep_grammar_check,
                text_chunk
            )
            return {**light_result, 'deep_analysis': deep_result}
        
        return light_result
    
    def _quick_rule_check(self, text: str) -> Dict:
        """快速规则检测"""
        result = {
            'hesitations': [],
            'has_obvious_errors': False,
            'needs_deep_analysis': False
        }
        
        # 检测犹豫词
        hesitation_pattern = r'\b(um|uh|ah|er|like|you know)\b'
        result['hesitations'] = re.findall(hesitation_pattern, text.lower())
        
        # 检测明显错误模式
        obvious_errors = [
            r'\bam work\b',
            r'\bI is\b',
            r'\bmany informations\b'
        ]
        for pattern in obvious_errors:
            if re.search(pattern, text):
                result['has_obvious_errors'] = True
                result['needs_deep_analysis'] = True
                break
        
        return result
    
    def _deep_grammar_check(self, text: str) -> Dict:
        """深度语法检查（耗时操作）"""
        # TODO: 调用LanguageTool API或LLM
        # 这是一个耗时操作，应该异步执行
        pass
```

### 6.2 错误去重

```python
class ErrorDeduplicator:
    """错误去重器"""
    
    def __init__(self):
        self.seen_errors = {}  # {(error_type, example_pattern): count}
    
    def add_error(self, error_type: str, example: str) -> bool:
        """
        添加错误，返回True表示新错误，False表示重复
        """
        # 归一化示例文本
        normalized = self._normalize_error(example)
        
        key = (error_type, normalized)
        
        if key in self.seen_errors:
            self.seen_errors[key] += 1
            return False
        
        self.seen_errors[key] = 1
        return True
    
    def _normalize_error(self, text: str) -> str:
        """归一化错误文本"""
        # 移除具体变量，只保留模式
        # 例如: "I am work" 和 "He am work" 归一化为 "* am work"
        pattern = re.sub(r'\b(I|he|she|we|they)\b', '*', text)
        return pattern.lower()
    
    def get_error_counts(self) -> Dict:
        """获取错误计数"""
        return {
            f"{k[0]}: {k[1]}": v 
            for k, v in self.seen_errors.items()
        }
```

### 6.3 缓存策略

```python
import hashlib
from functools import lru_cache

class ResponseCache:
    """响应缓存"""
    
    def __init__(self, redis_client):
        self.redis = redis_client
        self.default_ttl = 3600  # 1小时
    
    def _generate_key(self, prefix: str, content: str) -> str:
        """生成缓存键"""
        content_hash = hashlib.md5(content.encode()).hexdigest()
        return f"cache:{prefix}:{content_hash}"
    
    def get(self, prefix: str, content: str) -> Optional[str]:
        """获取缓存"""
        key = self._generate_key(prefix, content)
        return self.redis.get(key)
    
    def set(self, prefix: str, content: str, value: str, ttl: int = None) -> None:
        """设置缓存"""
        key = self._generate_key(prefix, content)
        self.redis.setex(key, ttl or self.default_ttl, value)

# 常见问题缓存
class CommonQuestionCache(ResponseCache):
    """常见问题缓存"""
    
    COMMON_QUESTIONS = {
        'tell_me_about_yourself': [
            "I'm [Name], a [profession] with [X] years of experience in [field].",
            "Currently working at [company], I specialize in [skills].",
            "I'm passionate about [interest] and enjoy [hobby]."
        ],
        'strengths': [
            "My key strengths are [strength1] and [strength2].",
            "For example, [specific example].",
            "This helps me [positive outcome]."
        ],
        'weaknesses': [
            "One area I'm working on is [weakness].",
            "I've been addressing this by [improvement approach].",
            "I've seen progress in [evidence]."
        ]
    }
    
    def get_common_response(self, question_type: str) -> Optional[List[str]]:
        """获取常见问题的标准回答"""
        return self.COMMON_QUESTIONS.get(question_type)
    
    def get_improvement_suggestion(self, error_type: str) -> Optional[str]:
        """获取错误类型的改进建议（缓存）"""
        cached = self.get('suggestion', error_type)
        if cached:
            return cached
        
        # 从数据库/配置文件加载
        suggestions = {
            'tense': 'Practice using present perfect and past tenses correctly.',
            'plural': 'Review common uncountable nouns: information, experience, advice.',
            'preposition': 'Memorize common collocations: good at, interested in, etc.'
        }
        
        suggestion = suggestions.get(error_type)
        if suggestion:
            self.set('suggestion', error_type, suggestion, ttl=86400)  # 24小时
        
        return suggestion
```

---

## 七、MVP落地路线图

### 第一阶段：基础功能（预计1周）

**目标**：实现核心反馈功能，可运行MVP

| 功能 | 优先级 | 具体实现 |
|------|--------|----------|
| Level 2 微点评 | P0 | 接入LLM，生成一句话点评 |
| Level 3 基础报告 | P0 | 生成Markdown格式复盘报告 |
| 基础语法检测 | P0 | 接入LanguageTool API |
| 语音输入 | P1 | 接入Whisper进行ASR |

**验收标准**：
- [ ] 用户回答完一题，能得到一句话点评
- [ ] 面试结束，能生成一份包含评分和错误分析的复盘报告
- [ ] 能检测出常见的语法错误

### 第二阶段：体验优化（预计2周）

**目标**：提升用户体验，增加智能反馈

| 功能 | 优先级 | 具体实现 |
|------|--------|----------|
| Level 1 实时反馈 | P0 | 实现表情符号无声反馈 |
| 两种模式切换 | P0 | Learning/Real Interview模式切换 |
| 多维度评分 | P1 | 发音、流利度、词汇、内容分别评分 |
| 错误统计汇总 | P1 | 自动统计高频错误类型 |
| 状态记忆 | P2 | Redis存储用户状态 |

**验收标准**：
- [ ] 用户说话时，AI能给出无声的表情反馈
- [ ] 能一键切换学习模式和真实面试模式
- [ ] 报告包含多维度的评分细分
- [ ] 错误统计自动汇总，展示Top问题

### 第三阶段：持续迭代

**目标**：根据用户反馈持续优化

| 功能 | 优先级 | 说明 |
|------|--------|------|
| 发音评分API | P1 | 接入讯飞发音评分 |
| 难度自动调节 | P1 | 根据表现自动调整题目难度 |
| 听力训练模块 | P2 | 加入跟读和复述练习 |
| A/B测试反馈策略 | P2 | 测试不同反馈策略的效果 |
| PDF报告导出 | P1 | 支持导出精美PDF报告 |
| 进度追踪 | P2 | 记录多次练习的进步曲线 |

---

## 八、附录

### A. 错误类型对照表

| 错误类型 | 英文 | 示例 | 改进建议 |
|----------|------|------|----------|
| 时态错误 | Tense Error | am work | have been working |
| 单复数 | Plural Error | many experiences | much experience |
| 介词错误 | Preposition Error | good in | good at |
| 冠词错误 | Article Error | go to school | (depending on context) |
| 主谓不一致 | Subject-Verb Agreement | He don't | He doesn't |
| 词序错误 | Word Order | Yesterday I went | I went yesterday |
| 遗漏词 | Omission | I very like | I really like |

### B. 反馈时机对照表

| 情况 | Level | 具体反馈 |
|------|-------|----------|
| 句子完整流畅 | 1 | ✅ |
| 检测到犹豫词 | - | 默默记录 |
| 轻微语法错误 | - | 默默记录 |
| 严重语法错误 | 1 | ⚠️ |
| 完全听不懂 | 1 | Pardon? |
| 一题回答完毕 | 2 | 一句话微点评 |
| 面试结束 | 3 | 完整复盘报告 |

### C. 技术栈推荐

| 层级 | 技术选型 | 说明 |
|------|----------|------|
| 前端 | React + WebSocket | 实时反馈推送 |
| 后端 | FastAPI + Python | 高性能异步API |
| 数据库 | Redis + PostgreSQL | 状态+持久化 |
| AI推理 | vLLM / OpenAI API | LLM服务 |
| ASR | Whisper API | 语音转文字 |
| 语法检测 | LanguageTool | 语法检查 |
| 缓存 | Redis | 热数据缓存 |
| 部署 | Docker + K8s | 容器化部署 |

---

*文档版本：v1.0*
*最后更新：2024年*
*维护者：Interview Pro 技术团队*
