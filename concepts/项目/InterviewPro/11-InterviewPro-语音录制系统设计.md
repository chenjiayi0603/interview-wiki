# 语音录制系统设计与测试

## 架构全景

```
浏览器                                      后端
┌─────────────────────────────────┐     ┌──────────────────────┐
│ getUserMedia()                  │     │ WebSocket handler    │
│   │                             │     │   │                  │
│   ▼                             │     │   ▼                  │
│ Web Audio Graph                 │     │ AUDIO_TOO_SHORT?     │
│ ┌───────────────────────────┐   │     │   │ (blob < 500B)    │
│ │ source → preGain(6x)     │   │     │   ▼                  │
│ │   → highpass(80Hz)       │   │     │ Bailian STT          │
│ │   → lowpass(12kHz)       │───┼──┐  │   │ RMS < 0.0002?    │
│ │   → compressor           │   │ │  │   ▼                  │
│ │   → analyser ──→ dest    │   │ │  │ paraformer-realtime  │
│ └───────────────────────────┘   │ │  │   │ 指数退避 1s/3s/6s│
│   │                             │ │  │   ▼                  │
│   ▼                             │ │  │ transcription / error│
│ MediaRecorder (Opus 64kbps)     │ │  │   │                  │
│   │                             │ │  │   ▼                  │
│   ▼                             │ │  │ evaluateAndRespond   │
│ Blob ──→ sendAudioChunk()       │ │  │   ├─ LLM 评分        │
│   │ size < 500? → 拒绝          │ │  │   └─ 下一题生成      │
│   ▼                             │ │  │                      │
│ WebSocket base64 ───────────────┼─┘  └──────────────────────┘
└─────────────────────────────────┘
```

## 关键设计点

### 1. 预暖（Prewarm）

`audio.ts:43` — 页面加载时调用 `prewarm()`，提前建好 Web Audio 图 + 等待 400ms 稳定期。
- 避免首次录音时 `AudioContext` 从 `suspended` 恢复导致的冷启动延迟
- `startRecording()` 判断 `wasRunningBefore`：如果图已处于 `running` 状态，跳过第二次 warmup

### 2. 音频处理链

```
preGain(6x) → highpass(80Hz) → lowpass(12kHz) → compressor → analyser → dest
```
- **preGain 6x**：浏览器 mic 输入电平偏低，补偿增益
- **highpass 80Hz**：切除低频噪音（空调、风扇）
- **lowpass 12kHz**：保留英语擦音（s/f/sh/th 能量达 10-12kHz），高于 8kHz 的常见建议是因为英语辅音识别对高频更敏感
- **compressor**：-24dB threshold, 12:1 ratio，防止大音量削波
- **analyser**：`fftSize=256, smoothingTimeConstant=0.4`，用于实时电平显示和信号验证

### 3. Blob 大小检查（双重守卫）

| 位置 | 阈值 | 行为 |
|------|------|------|
| 前端 `session.tsx:669` | duration < 1000ms | "录音太短，请说完整句话后再松开按钮" |
| 前端 `websocket.ts:207` | blob.size < 500B | 返回 false，不发送 |
| 后端 `client.go:497` | decoded bytes < 500B | `AUDIO_TOO_SHORT` error |
| 后端 `bailian.go` | PCM RMS < 0.0002 | `STT_TOO_QUIET` error |

前端的 500 字节检查是 `sendAudioChunk` 的同步检查——静音通过 Opus VBR 编码后可能只产生容器头（~100-400B），在发送前就被拦截，不等 15s 超时。

### 4. 录音停不下来问题

`audio.ts:248` — `stopRecordingAsBlob()` 内部加了 5s safety timeout：
```
如果 MediaRecorder.onstop 在 5s 内不触发（静音不产 Opus 帧）
→ 强制 resolve，用已有 chunks 构造 blob
→ UI 不卡死
```

### 5. 实时电平监控

`audio.ts:getAudioLevel()` — 从 AnalyserNode 读取 time-domain 数据，计算 RMS：
- 每 100ms 轮询一次（`useVoiceRecorder.ts:46`）
- 暴露为 `audioLevel` (0-1)
- UI 显示为 5 段电平条，阈值：[0.01, 0.04, 0.10, 0.20, 0.35]

### 6. Prewarm 信号验证

`audio.ts:verifySignalFlow()` — prewarm 完成后 5 次轮询 analyser：
- 每次间隔 100ms
- 任一次 `audioLevel > 0.0001` 即通过
- 全部为 0 → 抛出 "未检测到麦克风信号"

## 已修复的问题

| # | 问题 | 根因 | 修复 | 测试验证 |
|---|------|------|------|----------|
| 1 | 静音录音 UI 卡死 | `MediaRecorder.onstop` 永不触发（无 Opus 帧） | 5s safety timeout | `voice-silence` 4.1s 通过 |
| 2 | 小 blob 被静默忽略 | `sendAudioChunk` 返回 false 被丢弃 | 检查返回值，立即报错 | `voice-silence` 不等 15s 了 |
| 3 | 快速松手录音不停 | `isRecording` React state 在 400ms warmup 期间读到 false | 改用 `isRecordingRef`，在 async 之前设置 | tests #14 #15 从 FAIL→PASS |

## E2E 测试策略

### Chrome flag 注入机制

```
--use-file-for-fake-audio-capture=<path>.wav
```

Playwright 启动 Chrome 时注入此 flag，用 WAV 文件内容替代真实麦克风输入。`getUserMedia()` 返回的 stream 会播放该文件内容。

### Project 拆分

| Project | 注入文件 | 测试文件 | 用例数 |
|---------|----------|----------|--------|
| `chromium-normal` | `sample-en.wav`（英文语音） | `full-e2e.spec.ts` | 16 |
| `chromium-silence` | `silence-3s.wav`（全零 PCM） | `voice-silence.spec.ts` | 1 |

配置在 `frontend/playwright.config.ts`，通过 `testIgnore` / `testMatch` 分配。

### 语音相关用例清单

| 用例 | 测试文件 | 行号 | 验证内容 |
|------|----------|------|----------|
| 13. 语音录音 → STT 链路通 | `full-e2e.spec.ts` | 282 | 2.5s 英文 WAV → 转写 → 评分卡出现 |
| 14. 超短录音 < 1s 被拒 | `full-e2e.spec.ts` | 219 | 400ms → "录音太短" |
| 15. 语音错误后切文字 | `full-e2e.spec.ts` | 236 | 先触发"录音太短"→ 再发文字 → 评分出现在消息列表 |
| 16. 录音中点结束按钮 | `full-e2e.spec.ts` | 262 | 按住 0.8s → 点结束 → chat-input 消失（cleanup race） |
| 静音降级 | `voice-silence.spec.ts` | 24 | 2.5s 静音 → 不卡 UI → 返回友好错误 |

### E2E 能覆盖什么、不能覆盖什么

**能覆盖：**
- MediaRecorder 生命周期（start/stop/onstop/ondataavailable）
- Blob 大小守卫、录制时长守卫
- WebSocket 音频传输链路
- 后端 STT 转写 + 评分全链路
- 异步状态竞态（`isRecordingRef`）
- 异常场景下的 UI 不卡死

**不能覆盖（需真机手动测试）：**
- 真实麦克风硬件信号（fake audio 总是有数据的）
- `verifySignalFlow()` 检测死麦的错误路径
- 电平指示器 UI 的动态变化（fake audio 电平固定）
- AudioContext 浏览器自动挂起/恢复行为
- 网络抖动导致的 WebSocket 断连重连期间录音

### 运行方式

```bash
# 一键 E2E（自动检测后端，未启动会拉起）
bash scripts/e2e.sh

# 手动跑单个文件
cd frontend && npx playwright test ../E2E-test/voice-silence.spec.ts

# 只跑语音相关用例
cd frontend && npx playwright test --grep "语音\|voice\|silence\|Voice"
```
