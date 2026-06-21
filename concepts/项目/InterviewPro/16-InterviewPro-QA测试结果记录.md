# QA 测试结果记录

## 2026-05-08 全功能测试

执行方式：gstack `/qa` 无头浏览器自动测试 + 手动验证

### 测试前健康分：52/100
### 测试后健康分：82/100

---

## ✅ 通过的功能

| 功能 | 验证方式 |
|------|---------|
| 用户登录 / 注册 | 浏览器自动点击 |
| 开始面试（WebSocket 连接） | 实际建立连接，收到 AI 出题 |
| 文字答题提交 | 提交后收到评分 85/100，五维分项正常 |
| History 页面直接 URL 访问 | 修复前显示"未登录"，修复后正常显示会话列表 |
| History 卡片点击跳转 Feedback | 修复前弹 Alert，修复后正确跳转 |

---

## 🐛 发现并修复的 Bug

### P0 — 评分一直返回 5.0/100
- **根因**：DeepSeek 返回 ` ```json ``` ` markdown 包裹，且 max_tokens=2000 导致 JSON 被截断
- **修复**：新增 `GenerateJSONWithMaxTokens`（`response_format: json_object`），token 上限提升到 4000
- **文件**：`backend/internal/service/ai/deepseek_model.go`、`llm.go`、`provider.go`
- **提交**：`a56778f`

### P1 — History 页面直接访问显示未登录
- **根因**：`history.tsx` 未调用 `restoreSession()`，Zustand 内存状态在硬导航后丢失
- **修复**：添加 `useEffect(() => { if (!isAuthenticated) restoreSession(); }, [])`
- **文件**：`frontend/app/(tabs)/history.tsx`
- **提交**：`09e3ed1`

### P1 — History 卡片点击弹 Alert 而非跳转
- **根因**：`onPress` 用 `Alert.alert()` 占位，未接入路由
- **修复**：改为 `router.push({ pathname: "/interview/feedback", params: { sessionId: item.id } })`
- **文件**：`frontend/app/(tabs)/history.tsx`
- **提交**：`09e3ed1`

### P1 — WebSocket 连接失败（port-forward 环境）
- **根因**：WS URL 在模块加载时写死为常量，port-forward 环境 origin 不匹配
- **修复**：改为动态计算，跟随当前页面 `window.location.origin`
- **文件**：`frontend/services/websocket.ts`
- **提交**：`91080a1`

---

## ⚠️ 已知问题（未修）

| 问题 | 原因 | 优先级 |
|------|------|--------|
| TTS 语音合成不可用 | 阿里云试用到期，ElevenLabs 配额超限 | P2，需续费或换服务 |
| 旧 `.doc` 格式简历解析不稳定 | 过滤器后备方案，LLM 输出不稳定 | P2 |

---

## ❓ 未覆盖的功能（待测）

| 功能 | 备注 |
|------|------|
| 语音答题完整流程 | STT → 发音评测 → 评分，需麦克风环境 |
| 简历上传（PDF / docx） | 有代码，未在此次 QA 中触发 |
| 简历手动填写并保存 | 提交 `bda0277` 修过 POST /api/resume，未重跑 QA 验证 |
| 管理后台 | 未测 |

---

## 部署注意事项（本次测试总结）

- 前端镜像重建必须加 `--no-cache`，否则代码变更被 Docker 层缓存跳过
- `kubectl rollout restart` 后需重新运行 `bash k8s-run.sh forward`，port-forward 会断开
- 完整验证流程见 `CLAUDE.md` § "大修改后的完整验证流程"
