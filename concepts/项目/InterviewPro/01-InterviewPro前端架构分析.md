# InterviewPro 前端架构分析

> 基于 english-learner/frontend 真实 React Native + Expo TypeScript 项目源码的深度架构分析。
> 覆盖 Expo Router 路由系统、WebSocket 实时通信、状态管理、录音流程、语音驱动 UI 状态机、五维评分展示等。
> 源码路径：/home/tommychen/english-learner/frontend

---

## 一、项目概览

### 1.1 定位

InterviewPro 前端是一个 **React Native (Expo) + TypeScript** 跨平台应用，支持 Web（主要）、iOS、Android 三端。用户通过实时语音/文本与 AI 面试官进行英语面试模拟。

### 1.2 技术栈

| 层级 | 技术 | 用途 |
|------|------|------|
| 框架 | React 19.1 + React Native 0.81 | UI 框架 |
| 语言 | TypeScript 5.9 | 类型安全 |
| 跨平台 | Expo SDK 54 | 开发/构建/部署 |
| 路由 | Expo Router v6 (file-based routing) | 页面路由 |
| 状态管理 | Zustand v5 | 轻量全局状态 |
| HTTP | Axios | API 请求 |
| WebSocket | 原生 WebSocket + gorilla/websocket（后端）| 实时通信 |
| UI | react-native-paper, expo-linear-gradient | 组件库 |
| 录音 | expo-audio + MediaRecorder (Web) | 语音输入 |
| 存储 | expo-secure-store + AsyncStorage | 本地持久化 |
| 样式 | StyleSheet + 设计系统 tokens | 主题体系 |
| 测试 | Jest, Playwright | 单元+E2E 测试 |
| 构建 | Nginx 多阶段 Docker | 生产部署 |

### 1.3 项目结构

`
frontend/
├── a`app/                          # Expo Router 页面（文件即路由）
│   ├── _layout.tsx               # 根布局（SafeAreaProvider + PaperProvider）
│   ├── index.tsx                 # 根入口（重定向到 (tabs)/index）
│   ├── (auth)/                   # 认证组
│   │   ├── _layout.tsx
│   │   ├── login.tsx
│   │   ├── register.tsx
│   │   ├── forgot-password.tsx
│   │   └── verify-email.tsx
│   ├── (tabs)/                   # 底部 Tab 导航
│   │   ├── _layout.tsx
│   │   ├── index.tsx             # Home 页（场景卡片+职位选择）
│   │   ├── practice.tsx          # 练习页
│   │   ├── history.tsx           # 历史记录
│   │   └── profile.tsx           # 个人中心
│   ├── interview/
│   │   ├── [scenario].tsx        # 场景配置页（参数配置+开始面试）
│   │   ├── session.tsx           # 面试会话页（核心 WebSocket 聊天）
│   │   ├── wsHandlers.ts         # WebSocket 事件处理器工厂
│   │   └── feedback.tsx          # 面试反馈页
│   ├── resume/
│   │   ├── index.tsx             # 简历列表
│   │   ├── upload.tsx            # 简历上传
│   │   └── [id].tsx              # 简历详情
│   ├── membership/index.tsx      # 会员订阅
│   ├── session-result.tsx        # 会话结果（跳转中介）
│   └── admin/                    # 管理后台
│       ├── _layout.tsx
│       ├── index.tsx
│       ├── login.tsx
│       ├── stats.tsx
│       ├── prompts.tsx
│       ├── traces.tsx
│       └── rag-traces.tsx
├── components/                   # 可复用组件
│   ├── ScoreCard.tsx
│   ├── FiveDimensionScore.tsx    # 五维评分雷达/RTL 组件
│   ├── AlertModal.tsx
│   ├── MicTest.tsx               # 麦克风测试
│   ├── MicTestCompact.tsx        # 紧凑版麦克风测试
│   ├── interview/
│   │   └── KeywordEditor.tsx     # 关键词编辑器
│   ├── resume/
│   │   ├── ResumeCard.tsx
│   │   └── ResumeForm.tsx
│   └── scenario/
│       └── ScenarioCard.tsx
├── services/                     # 服务层
│   ├── api.ts                    # Axios HTTP 客户端（拦截器+重试+刷新）
│   ├── websocket.ts              # WebSocket 客户端（重连+心跳+事件分发）
│   ├── audio.ts                  # 音频服务（录音+播放+TTS）
│   ├── billing.ts                # 计费
│   ├── resume.ts                 # 简历
│   └── admin.ts                  # 管理端
├── stores/                       # Zustand 状态
│   ├── authStore.ts              # 认证状态
│   └── interviewStore.ts         # 面试会话状态
├── hooks/                        # 自定义 Hooks
│   ├── useVoiceFlow.ts           # 录音状态机（核心）
│   ├── useVoiceRecorder.ts       # 录音控制
│   ├── usePersistedMessages.ts   # 消息持久化
│   └── useFrameworkReady.ts      # 框架初始化
├── types/                        # 类型定义
│   ├── interview.ts              # 面试类型
│   ├── websocket-events.ts       # WebSocket 事件 schema
│   ├── user.ts                   # 用户类型
│   ├── resume.ts                 # 简历类型
│   └── feedback.ts               # 反馈类型
├── constants/                    # 常量
│   ├── api.ts                    # API 配置
│   ├── scenarios.ts              # 场景定义
│   ├── colors.ts                 # 设计系统颜色
│   └── types.ts                  # 共享 UI 类型
├── src/styles/                   # 设计系统 tokens
│   ├── colors.ts
│   ├── typography.ts
│   ├── spacing.ts
│   ├── borderRadius.ts
│   ├── shadows.ts
│   └── index.ts
├── utils/                        # 工具函数
│   └── recording.ts              # 录音工具（classifyRecording 等）
├── admin-web/                    # 独立管理前端（Vite + React）
│   └── src/
│       ├── App.tsx
│       └── main.tsx
├── Dockerfile                    # 多阶段构建（node builder → nginx）
├── nginx.conf                    # SPA + API 反向代理
├── docker-entrypoint.sh          # 运行时 env 注入
├── package.json
└── tsconfig.json                 # strict: true, moduleResolution: bundler
`

---

## 二、路由系统（Expo Router）

### 2.1 文件即路由

Expo Router 使用文件系统作为路由定义，`app/ 目录下的文件自动映射为 URL：

| 文件路径 | 路由 | 说明 |
|----------|------|------|
| `app/index.tsx | / | 根重定向 |
| `app/(tabs)/index.tsx | /(tabs)/ | Home 页 |
| `app/(tabs)/practice.tsx | /(tabs)/practice | 练习页 |
| `app/(tabs)/history.tsx | /(tabs)/history | 历史记录 |
| `app/(auth)/login.tsx | /(auth)/login | 登录页 |
| `app/interview/[scenario].tsx | /interview/[scenario] | **动态路由**：场景配置 |
| `app/interview/session.tsx | /interview/session | 面试会话页 |
| `app/resume/[id].tsx | /resume/[id] | **动态路由**：简历详情 |

### 2.2 路由嵌套

`
RootLayout (Stack)
├── (tabs) (Tab Navigator)
│   ├── index (Home)
│   ├── practice
│   ├── history
│   └── profile
├── (auth)
│   ├── login
│   ├── register
│   ├── forgot-password
│   └── verify-email
├── interview/[scenario] (card modal)
├── interview/session (card modal)
├── interview/feedback (card modal)
├── resume/index (card modal)
├── resume/upload (card modal)
├── resume/[id] (card modal)
├── session-result (card modal)
├── membership/index (card modal)
└── admin (Stack)
    ├── login
    ├── stats
    ├── prompts
    └── ...
`

**关键点**：(auth) 和 (tabs) 使用括号目录名，Expo Router 会将它们作为**布局组**（不产生 URL 路径段），内部页面共享同一种 Layout。

### 2.3 根布局 _layout.tsx

`	sx
export default function RootLayout() {
  useFrameworkReady();

  return (
    <SafeAreaProvider>
      <PaperProvider>
        <Stack screenOptions={{ headerShown: false }}>
          <Stack.Screen name="(tabs)" />
          <Stack.Screen name="(auth)" />
          <Stack.Screen name="interview/[scenario]" options={{ presentation: "card" }} />
          <Stack.Screen name="interview/session" options={{ presentation: "card" }} />
          {/* ... */}
        </Stack>
      </PaperProvider>
    </SafeAreaProvider>
  );
}
`

**设计要点**：
- presentation: "card"：从底部弹出的卡片式导航，保留原页面状态
- headerShown: false：所有页面自定义 Header
- SafeAreaProvider + PaperProvider：安全区域 + MD3 组件主题

---

## 三、状态管理（Zustand）

### 3.1 双 Store 架构

架构采用**最小化全局状态**策略，只将跨页面共享的状态放在 Zustand：

`
┌─────────────────────────────┐
│      Zustand Stores          │
│  ┌─────────────┐ ┌─────────┐│
│  │ authStore   │ │interview││
│  │             │ │ Store   ││
│  │ user        │ │ session ││
│  │ token       │ │ loading ││
│  │ isAuth      │ │ config  ││
│  │ login/logout│ │ create  ││
│  └──────┬──────┘ │ end     ││
│         │        └─────────┘│
│         │  (clearSession    │
│         │   on 401)         │
└─────────┼───────────────────┘
          │
          ▼
    authStore → api.ts（注册拦截器：token 过期自动清 session）
`

### 3.2 authStore 设计

`	sx
// stores/authStore.ts
export const useAuthStore = create<AuthState>((set, get) => ({
  user: null,
  token: null,
  refreshToken: null,
  isAuthenticated: false,

  login: async (email, password) => {
    const data = await api.post<LoginResponse>(ENDPOINTS.LOGIN, { email, password });
    await api.saveTokens(data.accessToken, data.refreshToken);
    set({ user: data.user, token: data.accessToken, isAuthenticated: true });
  },

  // 应用启动时从 AsyncStorage 恢复登录态
  restoreSession: async () => {
    const [token, refreshToken] = await AsyncStorage.multiGet(["access_token", "refresh_token"]);
    if (token[1] && refreshToken[1]) {
      set({ token: token[1], refreshToken: refreshToken[1], isAuthenticated: true });
    }
  },

  clearSession: () => set({ user: null, token: null, isAuthenticated: false }),
}));
`

### 3.3 401 自动清 Session

`	sx
// 在 authStore.ts 顶部注册回调
import("@/services/api").then(({ api }) => {
  api.setOnAuthCleared(() => useAuthStore.getState().clearSession());
});
`

**设计要点**：避免循环依赖（api → authStore → api），使用动态 import() 延迟注册回调。

---

## 四、HTTP 客户端（Axios）

### 4.1 ApiClient 架构

`	sx
// services/api.ts
class ApiClient {
  private client: AxiosInstance;
  private refreshPromise: Promise<void> | null = null;

  constructor() {
    this.client = axios.create({
      baseURL: API_BASE_URL,
      timeout: API_TIMEOUT, // 30s
    });
    this.setupInterceptors();
  }
}
`

### 4.2 拦截器链

`
Request Interceptor:
  AsyncStorage → get access_token → Authorization: Bearer <token>

Response Interceptor:
  200 → 返回 response.data.data
  401 → refreshToken() → retry original request
  401 (refresh also 401) → clearTokens() → reject
  Network Error → retry(max 3, exponential backoff) → reject
`

### 4.3 重试策略

`	sx
private async withRetry<T>(operation: () => Promise<T>, maxRetries = 3, delayMs = 1000): Promise<T> {
  for (let i = 0; i <= maxRetries; i++) {
    try {
      return await operation();
    } catch (error: any) {
      // 只重试网络错误（无 response）或 5xx
      if ((!error.response || error.response?.status >= 500) && i < maxRetries) {
        await new Promise((r) => setTimeout(r, delayMs * (i + 1)));
        continue;
      }
      throw error;
    }
  }
}
`

### 4.4 端点和超时

`	sx
// constants/api.ts
export const ENDPOINTS = {
  REGISTER: "/api/auth/register",
  LOGIN: "/api/auth/login",
  REFRESH: "/api/auth/refresh",
  CREATE_SESSION: "/api/interview/session",
  SESSION_END: (id: string) => /api/interview/session//end,
  // ...
};

export const API_TIMEOUT = 30000;           // 普通请求 30s
export const SESSION_END_TIMEOUT_MS = 180000; // 结束会话 3min（等待 AI 生成反馈）
`

---

## 五、WebSocket 实时通信（核心）

### 5.1 架构设计

`
session.tsx (React 组件)
    │
    ├── wsService.connect(sessionId, token, handlers)
    │       │
    │       ├── WebSocket(url)
    │       ├── onopen → send(start_session) → startHeartbeat(3s)
    │       ├── onmessage → JSON.parse → handleEvent(event)
    │       │       │
    │       │       ├── "ai_response_final" → onAIResponse(content, messageId)
    │       │       ├── "five_dimension_evaluation" → onFiveDimensionEvaluation(data)
    │       │       ├── "transcription" → onTranscription(text)
    │       │       ├── "audio_response" → onAudioResponse(base64, format)
    │       │       ├── "user_message" → onUserMessage(content, inputMode)
    │       │       ├── "typing_indicator" → onTyping(isTyping)
    │       │       ├── "session_ended" → router.replace(/interview/feedback)
    │       │       ├── "error" → onError(code + message)
    │       │       └── "turn_limit_reached" → onError(TURN_LIMIT)
    │       │
    │       ├── onclose(code !== 1000) → scheduleReconnect()
    │       └── onerror → 等待 onclose 统一处理
    │
    └── wsHandlers.ts (事件处理器工厂)
            └── buildInterviewWSHandlers(deps) → WSHandlers
`

### 5.2 心跳机制

`	sx
const HEARTBEAT_INTERVAL_MS = 3000;

private startHeartbeat() {
  this.heartbeatInterval = setInterval(() => {
    if (this.ws?.readyState === WebSocket.OPEN) {
      this.sendHeartbeat(); // { event_type: "heartbeat" }
    }
  }, HEARTBEAT_INTERVAL_MS);
}
`

**设计原因**：STT 转写窗口约 4s 无数据帧，若心跳间隔 > idle 阈值，连接会在转写完成瞬间被回收。3s 确保任何静默窗口都有数据帧流动。

### 5.3 重连策略

`
scheduleReconnect():
  ├── maxReconnectAttempts = 15
  ├── baseReconnectDelayMs = 1000
  ├── 指数退避 + 随机 jitter: min(30s, 1000 * 2^(attempt-1)) + random(0~400ms)
  └── 重连前刷新 token（api.refreshAccessToken()）
`

### 5.4 消息去重

`	sx
// wsHandlers.ts — 核心去重逻辑
onAIResponse: (content, messageId) => {
  // 见过的 messageId → 直接 return
  if (messageId && hasSeenId(messageId)) {
    currentAIQuestionRef.current = content;
    return;
  }
  // 旧版兜底（无 messageId）：按 content 比对
  if (!messageId && content === currentAIQuestionRef.current) return;

  // 真正新题
  if (messageId) markSeen(messageId);
  currentAIQuestionRef.current = content;
  setMessages(prev => [...stripVoicePlaceholders(prev), { id: messageId || ..., role: "ai", content, timestamp: new Date() }]);
}
`

### 5.5 音频 Base64 传输

`	sx
sendAudioChunk(audioData: Blob, pronunciationOnly = false): string | null {
  // 最小 500 字节检查（避免空音频）
  if (audioData.size < 500) return "未检测到有效语音";

  // 最大 7MB base64（后端 maxMessageSize=8MB，留 1MB 余量）
  const estBase64 = Math.ceil(audioData.size / 3) * 4;
  if (estBase64 > 7 * 1024 * 1024) return "录音过长（超过约 7 分钟）";

  // WebSocket OPEN 检查
  if (this.ws?.readyState !== WebSocket.OPEN) return "连接已断开";

  // 异步 FileReader → base64 → send
  const reader = new FileReader();
  reader.onloadend = () => {
    const base64 = (reader.result as string).split(",")[1];
    this.send({ event_type: "audio_chunk", data: { audio: base64, format } });
  };
  reader.readAsDataURL(audioData);
}
`

---

## 六、录音流程与语音状态机（核心）

### 6.1 useVoiceRecorder

`	sx
// hooks/useVoiceRecorder.ts
export function useVoiceRecorder() {
  const [isRecording, setIsRecording] = useState(false);
  const [isPreparing, setIsPreparing] = useState(false);    // 300ms 静音缓冲
  const [isReadyToSpeak, setIsReadyToSpeak] = useState(false);
  const [duration, setDuration] = useState(0);
  const [audioLevel, setAudioLevel] = useState(0);           // EMA 平滑音量

  // startRecording → audioService.startRecording → 300ms 后 isReadyToSpeak=true
  // stopRecordingAsBlob → audioService.stopRecording → return Blob
  // cancelRecording → audioService.cancelRecording → reset
}
`

**300ms 静音缓冲**：MediaRecorder 启动后等待 300ms，让用户看到指示器再说话，避免截断句首。

### 6.2 useVoiceFlow（核心状态机）

`	sx
// hooks/useVoiceFlow.ts — 录音业务状态机
// 替代 session.tsx 中散落的 6 个 ref + 6 个清理点

type VoiceFlowState = "idle" | "recording" | "transcribing" | "done";

状态机：
  beginRecording       finishRecording(ok)
  ┌── idle ──► recording ──► transcribing ──► done
  │              │              │  onTranscription / onEvaluationArrived
  │              │              │
  │  cancel /    │  onSTTTimeout / onSTTError / onRateLimit / onReconnecting
  │  recordTooShort  ▼                ▼
  └─────────────────── back to idle  ──┘
`

**占位消息机制**：

| 时间 | 占位内容 | 说明 |
|------|----------|------|
| 0ms（开始录音） | "🎤 正在听..." | 创建占位 |
| +2000ms | "📝 正在转写..." | stage1 定时器 |
| +8000ms | "⏳ 模型处理中..." | stage2 定时器 |
| STT 转写到达 | 替换为转写文本 | onTranscription |
| 五维评分到达 | 清理占位 | onEvaluationArrived |
| 超时/错误 | 删占位+显示错误提示 | onSTTTimeout / onSTTError |

### 6.3 录音流程完整链路

`
用户按下录音按钮
  → useVoiceRecorder.startRecording()
    → audioService.startRecording() (expo-audio / MediaRecorder)
    → 创建 "🎤 正在听..." 占位
    → 300ms 后 isReadyToSpeak=true

用户松开/点击停止
  → stopRecordingAsBlob() → 返回 Blob
  → wsService.sendAudioChunk(blob) → base64 → WebSocket
  → enterTranscribing() → state = "transcribing"
  → stage1(2s): "📝 正在转写..."
  → stage2(8s): "⏳ 模型处理中..."

后端 STT 转写完成
  → onTranscription(text) → 替换占位为文本

后端 Eino Graph 评分完成
  → onFiveDimensionEvaluation(data) → 清理占位 → 显示评分卡
  → onAIResponse(content) → 显示下一题

TTS 音频到达（或 4s 超时用浏览器 TTS 兜底）
  → onAudioResponse(base64) → audioService.playAudio()
  → 或 speakWithBrowserTTS(content)
`

### 6.4 TTS 双轨策略

`
onAIResponse → 保存到 pendingTTSRef
  │
  ├── 4s 内 onAudioResponse 到达（edge 音频）
  │   └── clearTimeout → audioService.playAudio(audioUrl)
  │
  └── 4s 超时（后端 TTS 未就绪）
      └── speakWithBrowserTTS(content) ← Web Speech API
`

**「先发声者胜」**：如果 edge 音频到达时浏览器 TTS 尚未触发（定时器未到期），优先播 edge（质量更高）；反之如果浏览器 TTS 已经启动，忽略 edge 音频避免重复。

---

## 七、页面详解

### 7.1 Home 页 ((tabs)/index.tsx)

**职责**：场景入口 + 职位选择 + 统计概览

`
HomeScreen
├── Header（Logo + 标题 + 通知按钮）
├── Greeting（时段问候 + 用户名字）
├── Position Selector
│   ├── Hot Position Chips（横向滚动）
│   └── All Positions（展开/收起）
├── Scenario Card Grid（2×2）
│   ├── 行为面试 → HR
│   ├── 技术面试 → Tech-Lead
│   └── 领导力面试 → Boss
├── Resume CTA（上传/查看简历）
└── Stats Bar（Sessions / Avg Score / Practice Hours）
`

**关键设计**：必须先选职位才能开始面试，确保后续出题有 position context。

### 7.2 面试配置页 (interview/[scenario].tsx)

**职责**：配置面试参数并发起会话

`
ScenarioDetailScreen
├── Header + 场景标题
├── Sub-Scenario Selector（三个 Tab）
├── Role Input（Modal 弹窗）
├── Difficulty Slider（初级/中级/高级）
├── AI Character Display（自动推导：hr/tech-lead/boss）
├── Advanced Config（展开/收起）
│   ├── KeywordEditor（简历技能提取）
│   └── Use Resume Toggle
├── Mic Test
└── Start Interview Button
`

**自动推导 AI Character**：
`	sx
const derivedAICharacter =
  selectedSub === "behavioral" ? "hr"
  : selectedSub === "technical" ? "tech-lead"
  : "boss"; // leadership
`

### 7.3 面试会话页 (interview/session.tsx)

**职责**：核心面试交互 — 消息列表 + 录音/文字输入 + 评分展示

`
SessionScreen
├── Header（场景信息 + 退出按钮）
├── Message List (FlatList)
│   ├── AI Question Bubbles
│   ├── User Answer Bubbles (text/voice placeholder)
│   ├── ScoreCard (五维评分)
│   ├── Typing Indicator
│   └── Error/Turn Limit Bubbles
├── Input Area
│   ├── Voice Button (录制/停止/重录)
│   ├── Text Input + Send
│   └── Audio Level Indicator
└── Score Modal (详情弹窗)
`

**核心 useEffect 链**：
`
1. mount → load persisted messages from AsyncStorage
2. mount → wsService.connect(sessionId, token, handlers)
3. mount → init audioService
4. AppState change → foreground → wsService.resumeIfNeeded()
5. unmount → wsService.disconnect() + audioService.cleanup() + clear persistence
`

### 7.4 反馈页 (interview/feedback.tsx)

**职责**：展示会话结束后 AI 生成的综合反馈报告

`
FeedbackScreen
├── Overall Score（大数字 + 雷达图）
├── Dimension Scores（Pronunciation/Fluency/Grammar/Vocabulary/Content）
├── Strengths List
├── Areas for Improvement
├── Sample Improved Answer
└── Actions（重试 + 新面试 + 分享）
`

### 7.5 管理后台 (`dmin/)

基于 Vite + React 的独立 SPA（`dmin-web/），通过 Nginx 反向代理到 /admin 路径：

| 页面 | 路由 | 功能 |
|------|------|------|
| 登录 | /admin/login | 管理员身份验证 |
| 统计 | /admin/ | 平台运营数据 |
| 提示词管理 | /admin/prompts | 编辑 AI System Prompt |
| RAG 追踪 | /admin/rag-traces | 混合搜索效果监控 |
| API 追踪 | /admin/traces | LLM 调用明细 |

---

## 八、事件类型定义（TypeScript Schema）

### 8.1 WebSocket 事件类型

`	sx
// types/websocket-events.ts
export interface FiveDimensionEvaluationData {
  message_id?: string;           // 幂等去重 ID
  overall_score: number;          // 总分（算术均值）
  dimensions: {
    fluency?: DimensionScore;
    grammar?: DimensionScore;
    vocabulary?: DimensionScore;
    content?: DimensionScore;
    pronunciation?: DimensionScore;  // 已下线，保留兼容
  };
  overall_feedback: OverallFeedback;
  has_voice_input?: boolean;      // 是否是语音模式
  is_fallback?: boolean;          // 是否是降级评分
}

export interface AIResponseFinalData {
  message_id?: string;            // 幂等去重 ID
  content: string;                // 问题文本
  type?: "question" | "answer";  // 预留字段
}

export interface TranscriptionData {
  text: string;
  engine?: string;                // "bailian" | "xunfei_iat" | "sensevoice" | "text"
}

export interface AudioResponseData {
  audio: string;                  // base64
  format: "mp3" | "wav" | "webm" | "ogg";
}

export interface ErrorData {
  code: string;                   // "STT_TIMEOUT" | "STT_EMPTY" | "RATE_LIMIT" | "TURN_LIMIT" ...
  message: string;
}
`

### 8.2 错误码分类

| Code | 含义 | 前端行为 |
|------|------|----------|
| RATE_LIMIT | 评分中，拒收 | 显示"正在评分中，请等待" |
| STT_TIMEOUT | 转写超时 | 显示"⏱️ 转写超时，请重试" |
| STT_EMPTY | 语音无有效内容 | 显示"未检测到有效语音" |
| STT_TOO_QUIET | 语音过轻 | 提示靠近麦克风 |
| STT_NO_SPEECH | 未检测到语音 | 提示重试 |
| AUDIO_TOO_SHORT | 录音<1秒 | 提示缩短或重试 |
| STT_BUSY | 服务端繁忙 | 提示稍候 |
| STT_FAILED | 转写失败 | 通用错误提示 |
| TURN_LIMIT | 免费版3轮限制 | 显示升级提示 |
| Connection failed | 首次连接失败 | 显示重新连接按钮 |
| Failed to reconnect | 重连耗尽 | 显示重新连接按钮 |

---

## 九、组件体系

### 9.1 FiveDimensionScore

五维评分雷达图组件：

`	sx
interface FiveDimensionScoreProps {
  scores: {
    pronunciation?: number;
    fluency: number;
    grammar: number;
    vocabulary: number;
    content: number;
  };
  hasVoiceInput?: boolean;
  size?: "small" | "large";
}
`

**设计**：
- 使用 react-native-svg 绘制雷达图（RTL 兼容阿拉伯语）
- 各维度独立评分 + 总分显示
- 语音模式显示 pronunciation 维度，文本模式隐藏
- 小型组件用于聊天气泡内，大型用于反馈页

### 9.2 ScoreCard

`	sx
interface ScoreCardProps {
  overallScore: number;
  dimensions: Dimensions;
  strengths: string[];
  improvements: string[];
  hasVoiceInput?: boolean;
}
`

**设计**：
- 折叠式卡片，默认显示总分和雷达图
- 展开后显示详细 strengths/improvements 列表
- 点击跳转到 FiveDimensionScore 全屏详情

### 9.3 MicTest / MicTestCompact

麦克风测试组件，面试开始前验证录音/播放链路：

`	sx
// MicTest：全尺寸，用于配置页
// MicTestCompact：紧致版，用于会话页头部
`

**流程**：
1. 开始录制 3s 测试音频
2. 播放录制内容让用户确认
3. 如果听不到，引导检查浏览器权限/系统音量

### 9.4 KeywordEditor

关键词编辑器，用于配置目标岗位关键词：

`	sx
interface KeywordEditorProps {
  keywords: string[];
  onChange: (keywords: string[]) => void;
}
`

- 支持 Chip 式添加/删除
- 自动从简历解析结果提取技能作为默认关键词
- 英文大小写自动归一化

---

## 十、设计系统

### 10.1 颜色体系

`	sx
// constants/colors.ts
export const Colors = {
  light: {
    primary: "#1a365d",        // 深蓝
    accent: "#d4a843",         // 金色
    background: "#f5f0e8",     // 米白
    surface: "#FFFFFF",
    text: "#1a365d",
    textSecondary: "#6b7280",
    error: "#DC2626",
    success: "#059669",
    warning: "#F59E0B",
    gradientBlue: ["#1a365d", "#2a4a7f"],
    gradientGold: ["#d4a843", "#e0c56e"],
    gradientNavyGold: ["#1a365d", "#d4a843"],
  },
  dark: { /* ... */ },
};
`

### 10.2 布局 Token

`	sx
// src/styles/spacing.ts
export const spacing = {
  xs: 4, sm: 8, md: 16, lg: 24, xl: 32, xxl: 48,
};

// src/styles/typography.ts
export const typography = {
  h1: { fontSize: 28, fontWeight: "800" as const },
  h2: { fontSize: 22, fontWeight: "700" as const },
  body: { fontSize: 15, fontWeight: "400" as const },
  caption: { fontSize: 12, fontWeight: "500" as const },
};

// src/styles/borderRadius.ts
export const borderRadius = { sm: 6, md: 10, lg: 16, xl: 24 };

// src/styles/shadows.ts
export const shadows = { sm: { ... }, md: { ... }, lg: { ... } };
`

---

## 十一、音频服务

### 11.1 录音实现

`	sx
// services/audio.ts
// Web 端使用 MediaRecorder（WebM/opus 格式）
// 移动端使用 expo-audio
class AudioService {
  private mediaRecorder: MediaRecorder | null = null;
  private audioChunks: Blob[] = [];

  async startRecording(onProgress?: (durationMs: number) => void) {
    const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
    this.mediaRecorder = new MediaRecorder(stream, {
      mimeType: MediaRecorder.isTypeSupported("audio/webm;codecs=opus")
        ? "audio/webm;codecs=opus" : "audio/webm",
    });
    this.mediaRecorder.ondataavailable = (e) => this.audioChunks.push(e.data);
    this.mediaRecorder.start(250); // 每 250ms 触发 dataavailable
  }

  async stopRecordingAsBlob(): Promise<Blob | null> {
    return new Promise((resolve) => {
      this.mediaRecorder!.onstop = () => {
        const blob = new Blob(this.audioChunks, { type: this.mediaRecorder!.mimeType });
        resolve(blob);
      };
      this.mediaRecorder!.stop();
      this.stream!.getTracks().forEach((t) => t.stop());
    });
  }
}
`

### 11.2 音频播放

`	sx
// 后端 TTS 返回 base64 → 转为 Object URL → 使用 <audio> 播放
async playAudio(audioUrl: string): Promise<void> {
  const audio = new Audio(audioUrl);
  await audio.play();
}

// 浏览器 TTS 兜底（Web Speech API）
speakWithBrowserTTS(text: string) {
  if (typeof window !== "undefined" && window.speechSynthesis) {
    const utterance = new SpeechSynthesisUtterance(text);
    utterance.lang = "en-US";
    utterance.rate = 0.9;
    window.speechSynthesis.speak(utterance);
  }
}
`

---

## 十二、持久化策略

### 12.1 消息持久化

usePersistedMessages hook 在每次消息变化时保存到 AsyncStorage：

`	sx
// hooks/usePersistedMessages.ts
const MESSAGES_CACHE_KEY = chat_messages_;

// 保存：setMessages 时自动写入 AsyncStorage
// 恢复：mount 时从 AsyncStorage 读取
// 清理：会话结束时 clearPersistence()
`

**用途**：配合 WebSocket 回放，确保重连时聊天记录不丢失。

### 12.2 令牌持久化

`	sx
// access_token + refresh_token 存 AsyncStorage
// 应用启动时 authStore.restoreSession() 恢复
// 401 响应时自动清除（api.clearAuthTokens() + authStore.clearSession()）
`

### 12.3 去重持久化

markedIds 集合同样存 AsyncStorage，避免重连后重新处理已见过的消息。

---

## 十三、部署架构

### 13.1 Docker 多阶段构建

`
Stage 1: node:20-alpine
  ├── npm ci
  ├── expo export --platform web
  └── 输出 dist/

Stage 2: nginx:1.26-alpine
  ├── 复制 nginx.conf
  ├── 复制 docker-entrypoint.sh（运行时注入 env）
  ├── 复制 dist/ → /usr/share/nginx/html
  ├── 端口 80/443
  ├── HEALTHCHECK → /healthz
  └── 入口: /docker-entrypoint.sh
`

### 13.2 Nginx 反向代理

`
ginx
# WebSocket 代理（长连接）
location /ws {
    proxy_pass http://backend:8080;
    proxy_http_version 1.1;
    proxy_set_header Upgrade ;
    proxy_set_header Connection "upgrade";
    proxy_read_timeout 1800s;   # 30 分钟 WS 超时
    proxy_send_timeout 1800s;
}

# API 代理
location /api/ {
    proxy_pass http://backend:8080;
}

# SPA 路由（History API fallback）
location / {
    try_files  / /index.html;
}
`

### 13.3 运行时配置注入

docker-entrypoint.sh 在容器启动时生成 untime-config.js：

`ash
cat > /usr/share/nginx/html/runtime-config.js <<EOF
window.__RUNTIME_CONFIG__ = {
  API_URL: "",
  WS_URL: ""
};
EOF
`

---

## 十四、关键设计模式总结

### 14.1 工厂模式

wsHandlers.ts 的 uildInterviewWSHandlers(deps)：将事件处理器的依赖显式注入，消除 session.tsx 中重复的回调定义。

`	sx
// 两处调用复用同一工厂
// 1. 主入口 useEffect
wsService.connect(sessionId, token, buildInterviewWSHandlers(deps));

// 2. 重连按钮点击
wsService.connect(sessionId, "", buildInterviewWSHandlers(deps));
`

### 14.2 状态机模式

useVoiceFlow：用有限状态机管理录音流程的所有状态转换，替代散落的 ref + cleanup 组合。

`
状态: idle → recording → transcribing → done
事件: beginRecording, finishRecording, onTranscription, 
      onEvaluationArrived, onSTTTimeout, onRateLimit, cancel
`

### 14.3 观察者模式

WebSocketService 的事件回调注册：

`	sx
// websocket.ts: 注册各种事件回调
interface WSHandlers {
  onAIResponse?: (content: string, messageId?: string) => void;
  onFiveDimensionEvaluation?: (data: FiveDimensionEvaluationData) => void;
  onTranscription?: (text: string) => void;
  onAudioResponse?: (audioBase64: string, format: string) => void;
  onError?: (error: string) => void;
  // ...
}

// 事件分发：handleEvent 根据 event_type 分发到对应回调
private handleEvent(event: WSEvent) {
  switch (event.event_type) {
    case "ai_response_final":
      this.handlers.onAIResponse?.(d.content, d.message_id);
      break;
    case "five_dimension_evaluation":
      this.handlers.onFiveDimensionEvaluation?.(event.data as FiveDimensionEvaluationData);
      break;
    // ...
  }
}
`

### 14.4 适配器模式

`	sx
// authStore 通过回调适配 api.ts（避免循环依赖）
import("@/services/api").then(({ api }) => {
  api.setOnAuthCleared(() => useAuthStore.getState().clearSession());
});
`

### 14.5 幂等去重

WebSocket 重连回放时通过 message_id 去重：

`	sx
const seenIds = new Set<string>();

onAIResponse: (content, messageId) => {
  if (messageId && seenIds.has(messageId)) return;
  seenIds.add(messageId);
  // 处理新消息
}
`

---

## 十五、测试体系

### 15.1 测试分类

| 测试类型 | 框架 | 位置 |
|----------|------|------|
| 单元测试 | Jest + ts-jest | *.test.ts |
| 组件测试 | @testing-library/react | — |
| E2E 测试 | Playwright | playwright-report/ |

### 15.2 已有测试

`	ypescript
// hooks/__tests__/useVoiceFlow.test.ts
// hooks/__tests__/useVoiceRecorder.test.ts
// hooks/__tests__/usePersistedMessages.test.ts
// services/__tests__/billing.test.ts
// utils/__tests__/recording.test.ts
`

---

## 附：面试 Q&A

### Q1: 为什么用 Zustand 而不是 Redux？

Zustand 足够轻量（< 1KB），没有 boilerplate，TypeScript 支持原生优秀。InterviewPro 的全局状态只有认证和会话配置两个小 Store，Redux 过于重量。

### Q2: WebSocket 为什么用 3s 心跳？

STT 转写窗口约 4s 无数据帧，3s 心跳确保连接在任何静默期间都不被 NAT/代理回收。使用**数据帧**而非 ping 控制帧——部分中间层只按数据帧计算 idle。

### Q3: 录音占位消息的设计解决了什么问题？

防止用户在等待 STT 转写 + AI 评分期间界面空白。占位从"🎤 正在听..." → "📝 正在转写..." → "⏳ 模型处理中..." 渐进式更新，给用户明确的进度感知。

### Q4: TTS 双轨策略的考虑？

edge-tts（后端合成）音质好但延迟高（2-5s），浏览器 TTS（Web Speech API）延迟低但音质差。4s 窗口期让两者竞争，"先发声者胜"—— edge 先到播 edge，超时用浏览器 TTS 兜底。

### Q5: 为什么用 file-based routing？

Expo Router 文件即路由，好处：
- 没有路由配置文件，目录结构一目了然
- 动态路由 [scenario] 零配置
- Layout 嵌套通过目录名 (tabs) 自然表达
- 类型安全的路由参数

### Q6: 如何处理 401 token 过期？

请求拦截器自动附带 token；响应拦截器检测 401 → 触发 refreshToken() → 重试原请求。refresh 也 401 → 清除所有令牌 → 通知 authStore 置为未登录，Axios 请求的 Promise 继续 reject，由调用方处理。

### Q7: 重连数据一致性如何保证？

后端 session_flow.go 在重连时回放完整对话历史，每条消息带 message_id。前端通过 seenIds Set 按 ID 去重，避免重复渲染。
