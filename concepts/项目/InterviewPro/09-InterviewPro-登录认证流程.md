# 详细登录流程分析

本文档详细分析 InterviewPro 英语面试练习系统的完整认证流程，包括注册、登录、Token 管理、会话恢复、路由保护和 WebSocket 认证。

---

## 目录

- [系统架构概览](#系统架构概览)
- [认证流程详解](#认证流程详解)
  - [1. 用户注册流程](#1-用户注册流程)
  - [2. 用户登录流程](#2-用户登录流程)
  - [3. Token 刷新流程](#3-token-刷新流程)
  - [4. 用户登出流程](#4-用户登出流程)
  - [5. 会话恢复流程](#5-会话恢复流程)
- [JWT Token 机制](#jwt-token-机制)
- [前端认证管理](#前端认证管理)
- [后端认证中间件](#后端认证中间件)
- [WebSocket 认证](#websocket-认证)
- [数据库模型](#数据库模型)
- [安全机制](#安全机制)
- [错误处理](#错误处理)
- [完整时序图](#完整时序图)

---

## 系统架构概览

```
┌─────────────────────────────────────────────────────────────┐
│                     前端 (React Native)                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────────┐  │
│  │ Login    │  │Register  │  │AuthStore │  │ API Client │  │
│  │ Screen   │  │ Screen   │  │ (Zustand)│  │ (Axios)    │  │
│  └──────────┘  └──────────┘  └──────────┘  └────────────┘  │
└──────────────────────────┬──────────────────────────────────┘
                           │ HTTP (JSON)
┌──────────────────────────▼──────────────────────────────────┐
│                   后端 (Go + Gin)                            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────────┐  │
│  │ Auth     │  │ JWT      │  │ Auth     │  │ Middleware │  │
│  │ Handler  │→ │ Generator│→ │ Service  │  │ (JWT Auth) │  │
│  └──────────┘  └──────────┘  └──────────┘  └────────────┘  │
│                           │                                  │
│                      ┌────▼────┐                            │
│                      │ SQLite  │                            │
│                      │ (GORM)  │                            │
│                      └─────────┘                            │
└─────────────────────────────────────────────────────────────┘
```

### 技术栈

**前端:**
- React Native + Expo Router (路由)
- Zustand (状态管理)
- Axios (HTTP 客户端)
- AsyncStorage (本地存储)

**后端:**
- Go + Gin (Web 框架)
- GORM (ORM)
- SQLite (数据库)
- golang-jwt/jwt/v5 (JWT 库)
- bcrypt (密码加密)

---

## 认证流程详解

### 1. 用户注册流程

#### 1.1 前端流程

**文件:** `frontend/app/(auth)/register.tsx`

```
用户输入 → 表单验证 → 调用 authStore.register() → POST /api/auth/register
                                                         ↓
                                                 注册成功 → 自动登录
                                                         ↓
                                              跳转到 /(tabs) 主页
```

**注册表单验证:**
```typescript
// 前端验证规则
1. 姓名、邮箱、密码不能为空
2. 密码和确认密码必须一致
3. 密码长度至少 6 个字符
```

**注册成功后自动登录:**
```typescript
// authStore.ts 第 45-59 行
register: async (email: string, password: string, name: string) => {
  set({ isLoading: true });
  try {
    // 1. 调用注册 API
    await api.post<RegisterResponse>(ENDPOINTS.REGISTER, {
      email,
      password,
      name,
    });
    // 2. 注册成功后自动登录
    await get().login(email, password);
  } catch (error) {
    set({ isLoading: false });
    throw error;
  }
}
```

#### 1.2 后端流程

**文件:** `backend/internal/service/auth.go`

```
POST /api/auth/register
       ↓
AuthHandler.Register() - 解析请求参数
       ↓
AuthService.Register() - 业务逻辑处理
       ↓
   1. 检查邮箱是否已存在
   2. bcrypt 加密密码
   3. 创建用户记录 (UUID, 默认 free 订阅)
   4. 返回用户信息
```

**核心代码:**
```go
// backend/internal/service/auth.go 第 62-91 行
func (s *AuthService) Register(ctx context.Context, req *RegisterRequest) (*model.User, error) {
    // 1. 检查邮箱是否已注册
    var existing model.User
    if err := s.db.Where("email = ?", req.Email).First(&existing).Error; err == nil {
        return nil, apperror.Duplicate("email already registered")
    }

    // 2. 密码加密 (bcrypt.DefaultCost = 10)
    hashedPassword, err := bcrypt.GenerateFromPassword([]byte(req.Password), bcrypt.DefaultCost)
    if err != nil {
        return nil, apperror.Internal("failed to hash password")
    }

    // 3. 创建用户
    user := &model.User{
        ID:               uuid.New(),
        Email:            req.Email,
        PasswordHash:     string(hashedPassword),
        Name:             req.Name,
        SubscriptionTier: "free", // 默认免费订阅
    }

    if err := s.db.Create(user).Error; err != nil {
        return nil, apperror.Wrap(err, "failed to create user")
    }

    return user, nil
}
```

**请求/响应格式:**
```json
// 请求: POST /api/auth/register
{
  "email": "user@example.com",
  "password": "password123",
  "name": "张三"
}

// 响应: 201 Created
{
  "success": true,
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "email": "user@example.com",
    "name": "张三",
    "subscriptionTier": "free",
    "createdAt": "2024-01-01T00:00:00Z"
  }
}
```

---

### 2. 用户登录流程

#### 2.1 前端流程

**文件:** `frontend/app/(auth)/login.tsx`

```
用户输入 → 表单验证 → 调用 authStore.login() → POST /api/auth/login
                                                    ↓
                                           保存 Token 到 AsyncStorage
                                                    ↓
                                           更新 Zustand 状态
                                                    ↓
                                        router.replace("/(tabs)")
```

**核心代码:**
```typescript
// authStore.ts 第 24-43 行
login: async (email: string, password: string) => {
  set({ isLoading: true });
  try {
    // 1. 调用登录 API
    const data = await api.post<LoginResponse>(ENDPOINTS.LOGIN, {
      email,
      password,
    });
    
    // 2. 保存 Token 到 AsyncStorage
    await api.saveTokens(data.accessToken, data.refreshToken);
    
    // 3. 更新状态
    set({
      user: data.user,
      token: data.accessToken,
      refreshToken: data.refreshToken,
      isAuthenticated: true,
      isLoading: false,
    });
  } catch (error) {
    set({ isLoading: false });
    throw error;
  }
}
```

#### 2.2 后端流程

**文件:** `backend/internal/service/auth.go`

```
POST /api/auth/login
       ↓
AuthHandler.Login() - 解析请求参数
       ↓
AuthService.Login() - 业务逻辑处理
       ↓
   1. 根据邮箱查询用户
   2. bcrypt 验证密码
   3. 生成 JWT Token Pair (Access + Refresh)
   4. 返回用户信息和 Token
```

**核心代码:**
```go
// backend/internal/service/auth.go 第 94-120 行
func (s *AuthService) Login(ctx context.Context, req *LoginRequest) (*LoginResponse, error) {
    // 1. 查询用户
    var user model.User
    if err := s.db.Where("email = ?", req.Email).First(&user).Error; err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, apperror.Invalid("invalid email or password")
        }
        return nil, apperror.Wrap(err, "failed to find user")
    }

    // 2. 验证密码
    if err := bcrypt.CompareHashAndPassword([]byte(user.PasswordHash), []byte(req.Password)); err != nil {
        return nil, apperror.Invalid("invalid email or password")
    }

    // 3. 生成 Token Pair
    tokens, err := s.jwtGen.GenerateTokenPair(user.ID, user.Email)
    if err != nil {
        return nil, apperror.Internal("failed to generate tokens")
    }

    // 4. 返回响应
    return &LoginResponse{
        User:         &user,
        AccessToken:  tokens.AccessToken,
        RefreshToken: tokens.RefreshToken,
        ExpiresAt:    tokens.ExpiresAt,
    }, nil
}
```

**请求/响应格式:**
```json
// 请求: POST /api/auth/login
{
  "email": "user@example.com",
  "password": "password123"
}

// 响应: 200 OK
{
  "success": true,
  "data": {
    "user": {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "email": "user@example.com",
      "name": "张三",
      "subscriptionTier": "free"
    },
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "expiresIn": "2024-01-01T00:15:00Z"
  }
}
```

---

### 3. Token 刷新流程

#### 3.1 自动刷新机制

**文件:** `frontend/services/api.ts`

当 API 请求返回 401 Unauthorized 时，Axios 拦截器自动触发 Token 刷新：

```
API 请求 → 401 Unauthorized → Axios 拦截器捕获
                                    ↓
                          检查是否已重试过 (_retried)
                                    ↓
                          调用 refreshToken()
                                    ↓
                    POST /api/auth/refresh
                                    ↓
                    获取新的 Access + Refresh Token
                                    ↓
                    保存到 AsyncStorage
                                    ↓
                    使用新 Token 重试原请求
```

**核心代码:**
```typescript
// api.ts 第 42-77 行
this.client.interceptors.response.use(
  (response) => response,
  async (error) => {
    const originalRequest = error.config;

    // 网络错误 (后端重启中) - 不清除 Token
    if (!error.response) {
      console.warn("Network error - backend may be restarting:", error.message);
      return Promise.reject(this.extractError(error));
    }

    // 401 - 尝试刷新 Token
    if (error.response?.status === 401 && !originalRequest._retried) {
      originalRequest._retried = true;
      try {
        await this.refreshToken();
        const newToken = await AsyncStorage.getItem(TOKEN_KEY);
        if (newToken) {
          originalRequest.headers.Authorization = `Bearer ${newToken}`;
        }
        return this.client(originalRequest); // 重试原请求
      } catch (err: any) {
        // 只有刷新返回 401 时才清除 Token
        if (err.response?.status === 401) {
          console.error("Token refresh returned 401, clearing tokens");
          await this.clearTokens();
        } else {
          console.warn("Token refresh failed (network error), keeping tokens:", err.message);
        }
        throw error;
      }
    }
    return Promise.reject(this.extractError(error));
  }
);
```

#### 3.2 防止并发刷新

```typescript
// api.ts 第 80-106 行
private async refreshToken(): Promise<void> {
  // 如果已经在刷新中，等待完成
  if (this.refreshPromise) {
    await this.refreshPromise;
    return;
  }

  this.refreshPromise = (async () => {
    const refreshToken = await AsyncStorage.getItem(REFRESH_TOKEN_KEY);
    if (!refreshToken) throw new Error("No refresh token");

    const response = await axios.post(`${API_BASE_URL}${ENDPOINTS.REFRESH}`, {
      refreshToken,
    });

    const { accessToken, refreshToken: newRefreshToken } = response.data.data;
    await AsyncStorage.setItem(TOKEN_KEY, accessToken);
    if (newRefreshToken) {
      await AsyncStorage.setItem(REFRESH_TOKEN_KEY, newRefreshToken);
    }
  })();

  try {
    await this.refreshPromise;
  } finally {
    this.refreshPromise = null;
  }
}
```

#### 3.3 后端 Token 刷新

**文件:** `backend/internal/service/auth.go`

```go
// backend/internal/service/auth.go 第 123-147 行
func (s *AuthService) RefreshTokens(ctx context.Context, refreshToken string) (*jwtpkg.TokenPair, error) {
    // 1. 验证 Refresh Token
    claims, err := s.jwtGen.ValidateRefreshToken(refreshToken)
    if err != nil {
        return nil, err
    }

    // 2. 检查 Token 是否被撤销 (黑名单)
    if s.tokenStore != nil {
        blacklisted, checkErr := s.tokenStore.IsBlacklisted(ctx, claims.TokenID)
        if checkErr != nil {
            return nil, apperror.Wrap(checkErr, "failed to check token blacklist")
        }
        if blacklisted {
            return nil, apperror.Unauthorized("token has been revoked")
        }
    }

    // 3. 生成新的 Token Pair
    tokens, err := s.jwtGen.GenerateTokenPair(claims.UserID, claims.Email)
    if err != nil {
        return nil, apperror.Internal("failed to generate tokens")
    }

    return tokens, nil
}
```

---

### 4. 用户登出流程

```
用户点击登出 → POST /api/auth/logout
                      ↓
            后端将 Token 加入黑名单 (Redis)
                      ↓
            前端清除本地 Token
                      ↓
            跳转到登录页
```

**前端:**
```typescript
// authStore.ts 第 61-67 行
logout: async () => {
  try {
    await api.post(ENDPOINTS.LOGOUT);
  } finally {
    get().clearSession();
  }
}
```

**后端:**
```go
// backend/internal/service/auth.go 第 150-161 行
func (s *AuthService) Logout(ctx context.Context, userID uuid.UUID, tokenID string) error {
    if s.tokenStore == nil {
        return nil // TokenStore 未配置 (Redis)
    }

    // 将 Refresh Token 加入黑名单，有效期 7 天
    if err := s.tokenStore.BlacklistToken(ctx, tokenID, 7*24*time.Hour); err != nil {
        return apperror.Wrap(err, "failed to blacklist token")
    }

    return nil
}
```

---

### 5. 会话恢复流程

**文件:** `frontend/app/index.tsx`

应用启动时从 AsyncStorage 恢复登录状态：

```
应用启动 → index.tsx
              ↓
    调用 authStore.restoreSession()
              ↓
    从 AsyncStorage 读取 Token
              ↓
    如果有 Token → 设置 isAuthenticated = true
              ↓
    跳转到 /(tabs) 主页
```

**核心代码:**
```typescript
// index.tsx 第 14-22 行
useEffect(() => {
  const restore = async () => {
    await restoreSession();
    setRestored(true);
    setChecking(false);
  };
  restore();
}, []);

// authStore.ts 第 107-123 行
restoreSession: async () => {
  try {
    const [token, refreshToken] = await AsyncStorage.multiGet([
      "access_token",
      "refresh_token",
    ]);
    if (token[1] && refreshToken[1]) {
      set({
        token: token[1],
        refreshToken: refreshToken[1],
        isAuthenticated: true,
      });
    }
  } catch (err) {
    console.error("Failed to restore session from storage:", err);
  }
}
```

---

## JWT Token 机制

### JWT 基本原理

**JWT (JSON Web Token)** 是一种开放标准 (RFC 7519)，用于在各方之间安全地传输声明信息。

#### 为什么使用 JWT？

| 传统 Session 方案 | JWT 方案 |
|------------------|----------|
| 服务端存储 Session 数据 | 无状态，Token 自包含所有信息 |
| 需要共享 Session 存储 (Redis/数据库) | 服务端无需存储，节省内存 |
| 水平扩展需要 Session 同步 | 天然支持分布式部署 |
| CSRF 攻击风险 | 不受 CSRF 影响 |
| 移动端 Session 管理复杂 | 跨平台友好 |

#### JWT 的三大组成部分

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX2lkIjoiNTUwZTg0MDAtZTI5Yi00MWQ0LWE3MTYtNDQ2NjU1NDQwMDAwIiwiZW1haWwiOiJ1c2VyQGV4YW1wbGUuY29tIiwidG9rZW5faWQiOiJhYmNkZWYxMjMiLCJleHAiOjE3MDQwNjcyMDAsImlhdCI6MTcwNDA2NjMwMCwiaXNzIjoiaW50ZXJ2aWV3LXBybyJ9.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
│                                    │                                                                                              │
└──────── Header (头部) ────────────┘ └────────────────────────── Payload (载荷) ────────────────────────────────────────────────────┘ └─ Signature (签名) ─┘
```

**1. Header (头部)** - 元数据
```json
{
  "alg": "HS256",  // 签名算法: HMAC SHA256
  "typ": "JWT"     // Token 类型
}
```

**2. Payload (载荷)** - 实际数据 (Claims)
```json
{
  "user_id": "550e8400-e29b-41d4-a716-446655440000",  // 自定义声明
  "email": "user@example.com",                          // 自定义声明
  "token_id": "abcdef123",                              // 自定义声明 (用于撤销)
  "exp": 1704067200,                                    // 标准声明: 过期时间 (Unix 时间戳)
  "iat": 1704066300,                                    // 标准声明: 签发时间
  "iss": "interview-pro"                                // 标准声明: 签发者
}
```

**3. Signature (签名)** - 防伪验证
```
HMACSHA256(
  base64UrlEncode(Header) + "." + base64UrlEncode(Payload),
  secret_key  // 服务端保管的密钥
)
```

#### JWT 工作流程

```
┌─────────────────────────────────────────────────────────────────────┐
│ 1. 用户登录                                                          │
│    POST /api/auth/login { "email": "...", "password": "..." }      │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 2. 服务端验证凭据                                                     │
│    - 查询数据库用户                                                   │
│    - bcrypt 验证密码                                                 │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 3. 生成 JWT Token                                                    │
│    - 创建 Header (算法 + 类型)                                        │
│    - 创建 Payload (用户信息 + 过期时间)                                │
│    - 使用密钥签名: HMACSHA256(base64(header) + "." + base64(payload), secret) │
│    - 返回: eyJhbGci... (三部分用 . 连接)                             │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 4. 客户端存储 Token                                                  │
│    - Access Token → AsyncStorage (15 分钟)                           │
│    - Refresh Token → AsyncStorage (7 天)                             │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 5. 后续请求携带 Token                                                 │
│    GET /api/interview/scenarios                                      │
│    Authorization: Bearer eyJhbGci...                                 │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 6. 服务端验证 Token                                                  │
│    - 解析 Header 和 Payload (Base64 解码)                             │
│    - 重新计算签名并比对: 验证是否被篡改                                │
│    - 检查过期时间: exp > 当前时间                                     │
│    - 提取用户信息: user_id, email                                    │
│    - 注入到请求上下文: c.Set("user_id", claims.UserID.String())      │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 7. 返回受保护资源                                                     │
│    200 OK + Data                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### JWT 的核心优势

**1. 无状态认证 (Stateless)**
```go
// 服务端不需要查询数据库或 Redis 验证 Token
func JWTAuth(jwtGen *jwtpkg.Generator) gin.HandlerFunc {
    return func(c *gin.Context) {
        // 仅通过数学运算验证签名，无需访问存储
        claims, err := jwtGen.ValidateAccessToken(tokenString)
        if err != nil {
            c.AbortWithStatusJSON(401, gin.H{"error": "invalid token"})
            return
        }
        // 直接使用 Token 中的用户信息
        c.Set("user_id", claims.UserID.String())
        c.Next()
    }
}
```

#### "仅通过数学运算验证签名" 详解

这是 JWT 最核心的设计哲学：**验证 Token 不需要查询数据库或 Redis，只需要数学计算！**

##### 传统 Session 验证 vs JWT 验证

```
┌─────────────────────────────────────────────────────────────────────────┐
│ 传统 Session 验证 (有状态)                                                │
└─────────────────────────────────────────────────────────────────────────┘

用户请求: GET /api/profile
Cookie: session_id=abc123
        ↓
┌──────────────────┐
│ 1. 提取 session_id│
└────────┬─────────┘
         ↓
┌──────────────────┐
│ 2. 查询 Redis     │  ← 网络 I/O 操作!
│    GET session:abc123 │
└────────┬─────────┘
         ↓
┌──────────────────┐
│ 3. Redis 返回数据 │
│    {user_id: 42}  │
└────────┬─────────┘
         ↓
┌──────────────────┐
│ 4. 继续处理请求   │
└──────────────────┘

性能瓶颈: 每次请求都需要访问 Redis (网络延迟 + 数据库压力)


┌─────────────────────────────────────────────────────────────────────────┐
│ JWT Token 验证 (无状态)                                                  │
└─────────────────────────────────────────────────────────────────────────┘

用户请求: GET /api/profile
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
        ↓
┌──────────────────┐
│ 1. 分割 Token     │  ← 字符串操作
│    header.payload.signature │
└────────┬─────────┘
         ↓
┌──────────────────┐
│ 2. Base64 解码    │  ← CPU 计算
│    解析 header    │
│    解析 payload   │
└────────┬─────────┘
         ↓
┌──────────────────┐
│ 3. 重新计算签名   │  ← 纯数学运算!
│    HMACSHA256(    │
│      header.payload, │
│      secret_key   │
│    )              │
└────────┬─────────┘
         ↓
┌──────────────────┐
│ 4. 比对签名       │  ← 字符串比较
│    计算值 == 原始值?│
└────────┬─────────┘
         ↓
┌──────────────────┐
│ 5. 检查过期时间   │  ← 时间比较
│    exp > now?     │
└────────┬─────────┘
         ↓
┌──────────────────┐
│ 6. 提取用户信息   │  ← 直接从 payload 读取
│    user_id, email │
└────────┬─────────┘
         ↓
┌──────────────────┐
│ 7. 继续处理请求   │
└──────────────────┘

性能优势: 零网络 I/O，纯 CPU 计算 (微秒级)
```

##### 数学运算的完整过程

让我们一步步拆解 `ValidateAccessToken()` 的数学运算：

```go
// backend/pkg/jwt/jwt.go 第 92-115 行
func (g *Generator) ValidateAccessToken(tokenString string) (*Claims, error) {
    // 输入: eyJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjoiMTIzIn0.abc123signature
    //       └── Header ──┘ └── Payload ──┘ └── Signature ─┘
    
    token, err := jwt.ParseWithClaims(tokenString, &Claims{}, func(t *jwt.Token) (interface{}, error) {
        // ============ 第 1 步: 分割 Token ============
        // 按 '.' 分割成三部分
        parts := strings.Split(tokenString, ".")
        // parts[0] = "eyJhbGciOiJIUzI1NiJ9"  (Header Base64)
        // parts[1] = "eyJ1c2VyX2lkIjoiMTIzIn0"  (Payload Base64)
        // parts[2] = "abc123signature"           (Signature)
        
        // ============ 第 2 步: 检查签名算法 ============
        if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
            return nil, fmt.Errorf("unexpected signing method: %v", t.Header["alg"])
        }
        // 防止攻击者将算法改为 "none" 来绕过验证
        
        // ============ 第 3 步: 提供密钥 ============
        return g.accessSecret, nil
        // 这个密钥用于重新计算签名
    })
    
    // ============ 第 4 步: 重新计算签名 (核心数学运算!) ============
    // 内部执行:
    // 1. 取原始 Token 的前两部分: header.payload
    //    message = "eyJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjoiMTIzIn0"
    //
    // 2. 使用密钥重新计算 HMAC-SHA256 签名:
    //    newSignature = HMAC-SHA256(message, accessSecret)
    //
    //    HMAC-SHA256 的计算过程:
    //    a. 如果密钥长度 > 块大小 (64 字节)，先对密钥做 Hash:
    //       key = SHA256(accessSecret)
    //    b. 如果密钥长度 < 块大小，用 0 填充:
    //       key = accessSecret + padding(0x00)
    //    c. 计算内层 Hash:
    //       innerHash = SHA256((key ⊕ 0x36) || message)
    //       (⊕ 表示异或运算，|| 表示拼接)
    //    d. 计算外层 Hash:
    //       finalHash = SHA256((key ⊕ 0x5c) || innerHash)
    //    e. finalHash 就是签名!
    //
    // 3. 比对签名:
    //    if newSignature == originalSignature:
    //        ✅ Token 未被篡改
    //    else:
    //        ❌ Token 被篡改或密钥错误
    
    if err != nil {
        return nil, apperror.Unauthorized(fmt.Sprintf("invalid token: %v", err))
    }
    
    // ============ 第 5 步: 提取 Claims ============
    claims, ok := token.Claims.(*Claims)
    if !ok || !token.Valid {
        return nil, apperror.Unauthorized("invalid token claims")
    }
    
    // ============ 第 6 步: 检查过期时间 ============
    if claims.ExpiresAt.Before(time.Now()) {
        return nil, apperror.Unauthorized("token expired")
    }
    
    // ============ 第 7 步: 返回用户信息 ============
    return claims, nil
    // claims.UserID = "550e8400-e29b-41d4-a716-446655440000"
    // claims.Email = "user@example.com"
    // 这些数据直接从 Token 的 payload 中读取，不需要查数据库!
}
```

##### HMAC-SHA256 数学运算可视化

```
假设:
  accessSecret = "my-secret-key"
  message = "eyJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjoiMTIzIn0"

计算过程:
┌─────────────────────────────────────────────────────────────────┐
│ 1. 密钥处理 (Key Preparation)                                     │
│    key = "my-secret-key" (13 字节)                               │
│    填充到 64 字节:                                                │
│    key = "my-secret-key" + 0x00 × 51                             │
└────────────────────────────┬────────────────────────────────────┘
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│ 2. 内层 Hash (Inner Hash)                                        │
│    a. key ⊕ 0x36 (每个字节与 0x36 异或):                          │
│       'm'(0x6d) ⊕ 0x36 = 0x5b                                    │
│       'y'(0x79) ⊕ 0x36 = 0x4f                                    │
│       ...                                                        │
│       得到: ipad_key (64 字节)                                    │
│                                                                    │
│    b. SHA256(ipad_key || message):                                │
│       将消息拼接后计算 SHA256:                                    │
│       innerHash = SHA256(ipad_key + message)                      │
│       = SHA256("my-secret-key...\x00\x00...eyJhbGci...")         │
│       = "a1b2c3d4e5f6..." (32 字节)                               │
└────────────────────────────┬────────────────────────────────────┘
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│ 3. 外层 Hash (Outer Hash)                                        │
│    a. key ⊕ 0x5c (每个字节与 0x5c 异或):                          │
│       'm'(0x6d) ⊕ 0x5c = 0x31                                    │
│       'y'(0x79) ⊕ 0x5c = 0x25                                    │
│       ...                                                        │
│       得到: opad_key (64 字节)                                    │
│                                                                    │
│    b. SHA256(opad_key || innerHash):                              │
│       finalSignature = SHA256(opad_key + innerHash)               │
│       = SHA256("\x31\x25...a1b2c3d4e5f6...")                     │
│       = "SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c"             │
└────────────────────────────┬────────────────────────────────────┘
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│ 4. Base64Url 编码                                                │
│    signature = Base64UrlEncode(finalSignature)                   │
│    = "SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c"               │
└─────────────────────────────────────────────────────────────────┘

验证:
  计算出的签名 == Token 中的签名?
  "SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c" == "SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c"
  ✅ 匹配! Token 有效!
```

##### 为什么这是"纯数学运算"?

```
❌ 不需要:
   - 数据库查询: SELECT * FROM sessions WHERE id = ?
   - Redis 查询: GET session:abc123
   - 网络请求: 无 I/O 操作
   - 磁盘读取: 无文件访问

✅ 只需要:
   - 字符串分割: strings.Split(token, ".")
   - Base64 解码: base64.RawURLEncoding.DecodeString()
   - HMAC 计算: hmac.New(sha256.New, key)
   - SHA256 哈希: sha256.Sum256(data)
   - 字节比较: bytes.Equal(sig1, sig2)
   - 时间比较: time.Before()

这些都是 CPU 指令级别的运算，在内存中完成，速度极快!
```

##### 性能对比

```
传统 Session 验证:
  - 网络 I/O: 1-5 ms (到 Redis 的往返时间)
  - Redis 查询: 0.1-1 ms
  - 总计: ~2-10 ms

JWT Token 验证:
  - 字符串分割: ~0.001 ms
  - Base64 解码: ~0.005 ms
  - HMAC-SHA256 计算: ~0.01 ms
  - 时间比较: ~0.001 ms
  - 总计: ~0.02 ms (20 微秒)

性能提升: 100-500 倍!
```

##### 实际代码追踪

让我们追踪一个完整的验证调用链:

```go
// 1. HTTP 请求到达
//    GET /api/interview/scenarios
//    Authorization: Bearer eyJhbGci...

// 2. Gin 中间件拦截
//    backend/internal/middleware/auth.go 第 16 行
func JWTAuth(jwtGen *jwtpkg.Generator) gin.HandlerFunc {
    return func(c *gin.Context) {
        // 3. 提取 Token 字符串
        tokenString := strings.TrimPrefix(authHeader, "Bearer ")
        // tokenString = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
        
        // 4. 调用验证函数 (纯数学运算!)
        claims, err := jwtGen.ValidateAccessToken(tokenString)
        //    ↓
        //    backend/pkg/jwt/jwt.go 第 93 行
        //    jwt.ParseWithClaims() 内部执行:
        //    
        //    a. 分割: parts = [header, payload, signature]
        //    b. 解码: headerJson = Base64Decode(parts[0])
        //    c. 解码: payloadJson = Base64Decode(parts[1])
        //    d. 计算: newSig = HMAC-SHA256(header + "." + payload, secret)
        //    e. 解码: oldSig = Base64Decode(parts[2])
        //    f. 比对: bytes.Equal(newSig, oldSig)?
        //    
        //    如果匹配 → token.Valid = true
        //    如果不匹配 → 返回错误 "invalid token"
        
        if err != nil {
            // Token 无效，拒绝请求
            c.AbortWithStatusJSON(401, gin.H{"error": "invalid token"})
            return
        }
        
        // 5. Token 有效，提取用户信息 (直接从 payload 读取)
        c.Set("user_id", claims.UserID.String())
        c.Set("email", claims.Email)
        
        // 6. 继续处理请求
        c.Next()
    }
}
```

##### 安全性保证

**问题: 既然只是数学运算，安全吗？**

答案: **非常安全！** 原因在于密码学哈希函数的特性:

```go
// 1. 抗原像攻击 (Pre-image Resistance)
//    已知签名 S，无法反推出密钥 K
//    HMAC-SHA256(message, K) = S
//    攻击者知道 S 和 message，但无法计算 K

// 2. 抗碰撞攻击 (Collision Resistance)
//    无法找到两个不同的消息产生相同签名
//    HMAC-SHA256(msg1, K) ≠ HMAC-SHA256(msg2, K)

// 3. 雪崩效应 (Avalanche Effect)
//    消息的 1 bit 变化 → 签名的 50% bits 变化
原始: {"user_id": "user123"}
      → signature: "abc123def456..."

篡改: {"user_id": "admin001"}  // 只改了 2 个字符
      → signature: "9f8e7d6c5b4a..."  // 完全不同的签名!

// 4. 密钥长度
//    推荐至少 256 位 (32 字节)
//    暴力破解需要 2^256 次运算 (不可能!)
```

##### 总结

"仅通过数学运算验证签名" 意味着:

✅ **零依赖**: 不需要数据库、Redis、文件系统等外部存储  
✅ **超快速**: 纯 CPU 计算，微秒级完成  
✅ **可扩展**: 无状态设计，天然支持水平扩展  
✅ **安全性**: 基于密码学哈希函数，无法伪造  
✅ **低成本**: 减少数据库压力，节省服务器资源  

这就是 JWT 在现代分布式系统中广泛使用的核心原因!

**2. 防篡改 (Tamper-Proof)**
```
攻击者尝试修改 Payload:
原始: {"user_id": "user123", "email": "user@example.com"}
修改: {"user_id": "admin001", "email": "admin@example.com"}

签名验证失败:
原始签名: HMACSHA256(base64(header.payload), secret) = SflKxwRJSMeKK...
新签名:   HMACSHA256(base64(header.modified_payload), secret) = 完全不同!

服务端检测到签名不匹配 → 拒绝请求
```

**3. 自包含 (Self-Contained)**
```json
// Token 包含所有必要信息，无需额外查询
{
  "user_id": "550e8400-e29b-41d4-a716-446655440000",
  "email": "user@example.com",
  "exp": 1704067200
}
// 服务端直接从 Token 读取 user_id，不需要:
// SELECT * FROM users WHERE id = ?
```

#### 签名算法详解 (HS256)

本项目使用 **HMAC-SHA256** (Hash-based Message Authentication Code):

```go
// backend/pkg/jwt/jwt.go 第 74 行
accessToken, err := jwt.NewWithClaims(
    jwt.SigningMethodHS256,  // HMAC-SHA256 算法
    accessClaims
).SignedString(g.accessSecret)
```

**HMAC-SHA256 工作原理:**
```
签名 = HMAC-SHA256(消息, 密钥)

其中:
- 消息 = base64UrlEncode(Header) + "." + base64UrlEncode(Payload)
- 密钥 = accessSecret (仅服务端知道)

特性:
✅ 单向性: 无法从签名反推密钥
✅ 确定性: 相同消息 + 相同密钥 = 相同签名
✅ 敏感性: 消息或密钥的任何微小变化都会导致签名完全不同
```

**验证过程:**
```go
// backend/pkg/jwt/jwt.go 第 92-98 行
token, err := jwt.ParseWithClaims(tokenString, &Claims{}, func(t *jwt.Token) (interface{}, error) {
    // 1. 检查签名算法是否匹配 (防止算法替换攻击)
    if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
        return nil, fmt.Errorf("unexpected signing method: %v", t.Header["alg"])
    }
    // 2. 提供密钥用于验证签名
    return g.accessSecret, nil
})
```

#### Token 生命周期管理

```
┌─────────────┐
│ 生成 Token   │  ← 登录时创建，包含 user_id, email, token_id, exp, iat
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ 客户端存储   │  ← Access Token (内存 + AsyncStorage)
│             │     Refresh Token (AsyncStorage)
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ 携带 Token   │  ← Authorization: Bearer <token>
│ 发起请求     │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ 验证签名     │  ← 重新计算签名并比对
│ 检查过期     │  ← exp > current_time?
└──────┬──────┘
       │
       ├─ 有效 ─→ 继续处理请求
       │
       └─ 过期 ─→ 401 Unauthorized
                  ↓
            使用 Refresh Token 刷新
                  ↓
            POST /api/auth/refresh
                  ↓
            获取新的 Token Pair
```

#### 双 Token 设计原理

**为什么需要 Access + Refresh Token？**

| 单一 Token 方案 | 双 Token 方案 |
|----------------|---------------|
| Token 有效期长 (7 天) → 泄露风险高 | Access Token 短期 (15 分钟) → 泄露影响小 |
| Token 有效期短 (15 分钟) → 频繁登录 | Refresh Token 长期 (7 天) → 自动续期 |
| 无法强制登出 (除非服务端存储) | Refresh Token 可撤销 (黑名单) |
| 用户体验差 | 用户无感知续期 |

**InterviewPro 的双 Token 策略:**
```yaml
Access Token:
  有效期: 15 分钟
  用途: API 请求认证
  存储: 内存 (Zustand) + AsyncStorage (持久化)
  泄露影响: 15 分钟后自动失效

Refresh Token:
  有效期: 7 天
  用途: 刷新 Access Token
  存储: AsyncStorage (不暴露给 API 请求)
  安全性: 仅在刷新时发送到 /api/auth/refresh
  撤销: 登出时加入黑名单 (Redis)
```

**刷新流程:**
```
用户操作 → API 请求 (Access Token 过期)
              ↓
         401 Unauthorized
              ↓
    Axios 拦截器自动触发刷新
              ↓
    POST /api/auth/refresh
    { "refreshToken": "eyJ..." }
              ↓
    后端验证 Refresh Token:
    1. 签名是否正确？
    2. 是否过期？
    3. 是否在黑名单中？
              ↓
    生成新的 Token Pair
              ↓
    返回 { accessToken, refreshToken }
              ↓
    客户端保存新 Token
              ↓
    自动重试原请求 (对用户透明)
```

### Token 结构

**文件:** `backend/pkg/jwt/jwt.go`

```go
// JWT Claims 结构
type Claims struct {
    UserID   uuid.UUID `json:"user_id"`     // 用户 ID
    Email    string    `json:"email"`       // 用户邮箱
    TokenID  string    `json:"token_id"`    // Token 唯一标识 (用于撤销)
    jwt.RegisteredClaims                    // 标准声明
}

// RegisteredClaims 包含:
// - ExpiresAt: 过期时间
// - IssuedAt: 签发时间
// - Issuer: 签发者 ("interview-pro")
```

### Token 配置

**文件:** `backend/internal/config/config.go`

```yaml
# 配置文件 (config.yaml)
jwt:
  access_secret: "your-access-secret"    # Access Token 签名密钥
  refresh_secret: "your-refresh-secret"  # Refresh Token 签名密钥
  access_expiry: 15                       # Access Token 有效期 (分钟)
  refresh_expiry: 7                       # Refresh Token 有效期 (天)
```

### Token 生成流程

```go
// backend/pkg/jwt/jwt.go 第 49-89 行
func (g *Generator) GenerateTokenPair(userID uuid.UUID, email string) (*TokenPair, error) {
    // 1. 生成唯一 Token ID
    tokenID := uuid.New().String()

    // 2. 创建 Access Token Claims
    accessClaims := Claims{
        UserID: userID,
        Email:  email,
        TokenID: tokenID,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(g.accessExpiry)), // 15 分钟
            IssuedAt:  jwt.NewNumericDate(time.Now()),
            Issuer:    "interview-pro",
        },
    }

    // 3. 创建 Refresh Token Claims
    refreshClaims := Claims{
        UserID: userID,
        Email:  email,
        TokenID: tokenID, // 相同的 Token ID
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(g.refreshExpiry)), // 7 天
            IssuedAt:  jwt.NewNumericDate(time.Now()),
            Issuer:    "interview-pro",
        },
    }

    // 4. 签名 Access Token (HS256)
    accessToken, err := jwt.NewWithClaims(jwt.SigningMethodHS256, accessClaims).SignedString(g.accessSecret)
    
    // 5. 签名 Refresh Token (HS256)
    refreshToken, err := jwt.NewWithClaims(jwt.SigningMethodHS256, refreshClaims).SignedString(g.refreshSecret)

    return &TokenPair{
        AccessToken:  accessToken,
        RefreshToken: refreshToken,
        ExpiresAt:    accessClaims.ExpiresAt.Time,
    }, nil
}
```

### Token 验证流程

```go
// backend/pkg/jwt/jwt.go 第 92-115 行
func (g *Generator) ValidateAccessToken(tokenString string) (*Claims, error) {
    // 1. 解析并验证 Token
    token, err := jwt.ParseWithClaims(tokenString, &Claims{}, func(t *jwt.Token) (interface{}, error) {
        // 验证签名算法
        if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
            return nil, fmt.Errorf("unexpected signing method: %v", t.Header["alg"])
        }
        return g.accessSecret, nil
    })

    if err != nil {
        return nil, apperror.Unauthorized(fmt.Sprintf("invalid token: %v", err))
    }

    // 2. 提取 Claims
    claims, ok := token.Claims.(*Claims)
    if !ok || !token.Valid {
        return nil, apperror.Unauthorized("invalid token claims")
    }

    // 3. 检查过期时间 (备用)
    if claims.ExpiresAt.Before(time.Now()) {
        return nil, apperror.Unauthorized("token expired")
    }

    return claims, nil
}
```

---

## 前端认证管理

### Zustand AuthStore

**文件:** `frontend/stores/authStore.ts`

```typescript
interface AuthState {
  // 状态
  user: User | null;
  token: string | null;
  refreshToken: string | null;
  isAuthenticated: boolean;
  isLoading: boolean;
  
  // 方法
  login: (email: string, password: string) => Promise<void>;
  register: (email: string, password: string, name: string) => Promise<void>;
  logout: () => Promise<void>;
  refreshSession: () => Promise<void>;
  setUser: (user: User) => void;
  setTokens: (tokens: TokenPair) => void;
  clearSession: () => void;
  setLoading: (loading: boolean) => void;
  restoreSession: () => Promise<void>;
}
```

### API Client 拦截器

**文件:** `frontend/services/api.ts`

**请求拦截器 - 附加 Token:**
```typescript
this.client.interceptors.request.use(
  async (config) => {
    const token = await AsyncStorage.getItem(TOKEN_KEY);
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);
```

**响应拦截器 - 自动刷新 Token:**
- 401 错误 → 自动刷新 Token → 重试请求
- 网络错误 → 保留 Token → 允许重试 (后端重启场景)
- 刷新失败 (401) → 清除 Token → 用户需重新登录

### 重试机制

```typescript
// api.ts 第 123-146 行
private async withRetry<T>(
  operation: () => Promise<T>,
  maxRetries = 3,
  delayMs = 1000
): Promise<T> {
  let lastError: any;
  for (let i = 0; i <= maxRetries; i++) {
    try {
      return await operation();
    } catch (error: any) {
      lastError = error;
      // 仅在网络错误或 5xx 错误时重试
      const isNetworkError = !error.response;
      const isServerError = error.response?.status >= 500;
      if ((isNetworkError || isServerError) && i < maxRetries) {
        console.warn(`Request failed, retrying ${i + 1}/${maxRetries}...`);
        await new Promise((resolve) => setTimeout(resolve, delayMs * (i + 1)));
        continue;
      }
      throw error;
    }
  }
  throw lastError;
}
```

---

## 后端认证中间件

### JWT 认证中间件

**文件:** `backend/internal/middleware/auth.go`

```go
// backend/internal/middleware/auth.go 第 16-43 行
func JWTAuth(jwtGen *jwtpkg.Generator) gin.HandlerFunc {
    return func(c *gin.Context) {
        // 1. 检查 Authorization Header
        authHeader := c.GetHeader("Authorization")
        if authHeader == "" {
            err := apperror.Unauthorized("missing authorization header")
            c.AbortWithStatusJSON(err.Status(), gin.H{"success": false, "error": err.Error()})
            return
        }

        // 2. 提取 Token
        tokenString := strings.TrimPrefix(authHeader, "Bearer ")
        if tokenString == authHeader {
            err := apperror.Unauthorized("invalid authorization format")
            c.AbortWithStatusJSON(err.Status(), gin.H{"success": false, "error": err.Error()})
            return
        }

        // 3. 验证 Access Token
        claims, err := jwtGen.ValidateAccessToken(tokenString)
        if err != nil {
            c.AbortWithStatusJSON(401, gin.H{"success": false, "error": "invalid or expired token"})
            return
        }

        // 4. 注入用户信息到 Context
        c.Set("user_id", claims.UserID.String())
        c.Set("email", claims.Email)
        c.Set("token_id", claims.TokenID)
        c.Next()
    }
}
```

### 路由保护

**文件:** `backend/internal/app/app.go`

```go
// 公开路由 (不需要认证)
auth := r.Group("/api/auth")
{
    auth.POST("/register", authHandler.Register)
    auth.POST("/login", authHandler.Login)
    auth.POST("/refresh", authHandler.Refresh)
}

// 受保护路由 (需要 JWT 认证)
interview := r.Group("/api/interview")
interview.Use(middleware.JWTAuth(jwtGen)) // 应用中间件
{
    interview.GET("/scenarios", interviewHandler.GetScenarios)
    interview.POST("/session", interviewHandler.CreateSession)
    // ...
}

user := r.Group("/api/user")
user.Use(middleware.JWTAuth(jwtGen))
{
    user.GET("/profile", userHandler.GetProfile)
    user.PUT("/profile", userHandler.UpdateProfile)
    // ...
}
```

---

## WebSocket 认证

**文件:** `backend/internal/handler/ws.go`

WebSocket 连接通过查询参数或协议头传递 Token：

```go
// backend/internal/handler/ws.go 第 108-125 行
func VerifyWSAuth() gin.HandlerFunc {
    return func(c *gin.Context) {
        // 方式 1: 查询参数 ?token=xxx
        token := c.Query("token")
        
        // 方式 2: Sec-WebSocket-Protocol 头
        if token == "" {
            token = c.GetHeader("Sec-WebSocket-Protocol")
        }
        
        if token == "" {
            err := apperror.Unauthorized("missing auth token")
            c.AbortWithStatusJSON(err.Status(), gin.H{
                "success": false,
                "error":   err.Error(),
            })
            return
        }
        c.Next()
    }
}

// 解析 WebSocket Token (第 76-106 行)
func (h *WSHandler) parseUserID(token string) uuid.UUID {
    claims, err := h.jwtGen.ValidateAccessToken(token)
    if err != nil {
        // 尝试 Refresh Token
        claims, err = h.jwtGen.ValidateRefreshToken(token)
        if err != nil {
            return uuid.Nil
        }
    }
    return claims.UserID
}
```

**前端连接示例:**
```typescript
const ws = new WebSocket(
  `ws://localhost:8080/ws/interview?token=${accessToken}`
);
```

---

## 数据库模型

### User 表

**文件:** `backend/internal/model/model.go`

```go
type User struct {
    ID               uuid.UUID          `gorm:"type:char(36);primaryKey"`
    Email            string             `gorm:"uniqueIndex;type:varchar(255);not null"`
    PasswordHash     string             `gorm:"type:varchar(255);not null"` // bcrypt 哈希
    Name             string             `gorm:"type:varchar(255)"`
    Avatar           *string            `gorm:"type:varchar(500)"`
    SubscriptionTier string             `gorm:"type:varchar(20);default:'free'"`
    Settings         *string            `gorm:"type:text"` // JSON
    CreatedAt        time.Time
    UpdatedAt        time.Time
}
```

**数据库迁移:**
```go
// backend/internal/app/app.go 第 232-245 行
if cfg.Database.AutoMigrate {
    if err := db.AutoMigrate(
        &model.User{},
        &model.InterviewSession{},
        &model.Message{},
        &model.Feedback{},
        &model.QuestionBank{},
    ); err != nil {
        return nil, fmt.Errorf("failed to auto migrate: %w", err)
    }
}
```

---

## 安全机制

### 1. 密码加密

使用 bcrypt 算法，默认 cost = 10：

```go
// 加密
hashedPassword, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)

// 验证
err := bcrypt.CompareHashAndPassword([]byte(hash), []byte(password))
```

### 2. Token 安全

- **双 Token 机制:** Access Token (15 分钟) + Refresh Token (7 天)
- **HS256 签名:** 使用独立的签名密钥
- **Token ID:** 每个 Token 有唯一标识，支持撤销
- **黑名单机制:** 登出时将 Token 加入黑名单 (需 Redis)

### 3. 错误处理安全

```go
// 登录失败时不暴露具体原因
return nil, apperror.Invalid("invalid email or password")
// 而不是:
// return nil, apperror.Invalid("user not found")
// return nil, apperror.Invalid("wrong password")
```

### 4. 前端安全

- Token 存储在 AsyncStorage (移动端安全存储)
- HTTPS 传输 (生产环境)
- 自动刷新减少 Token 泄露风险
- 网络错误时不清除 Token (防止后端重启时丢失登录状态)

---

## 错误处理

### 错误类型

**文件:** `backend/pkg/apperror/`

```go
// 错误类型映射到 HTTP 状态码
type ErrorType string

const (
    Invalid    ErrorType = "invalid"     // 400 Bad Request
    Unauthorized ErrorType = "unauthorized" // 401 Unauthorized
    Forbidden  ErrorType = "forbidden"   // 403 Forbidden
    NotFound   ErrorType = "not_found"   // 404 Not Found
    Duplicate  ErrorType = "duplicate"   // 409 Conflict
    Internal   ErrorType = "internal"    // 500 Internal Server Error
)
```

### 常见错误场景

| 场景 | HTTP 状态码 | 错误信息 |
|------|------------|---------|
| 邮箱已注册 | 409 | "email already registered" |
| 登录凭据错误 | 400 | "invalid email or password" |
| 缺少 Authorization 头 | 401 | "missing authorization header" |
| Token 格式错误 | 401 | "invalid authorization format" |
| Token 过期 | 401 | "invalid or expired token" |
| Refresh Token 无效 | 401 | "invalid refresh token" |
| Token 已被撤销 | 401 | "token has been revoked" |
| 密码长度不足 | 400 | "invalid request: password min length 6" |

---

## 完整时序图

### 注册 + 登录 + 使用流程

```
用户              前端 (React Native)              后端 (Go + Gin)              SQLite
 │                          │                            │                        │
 │ 1. 输入注册信息          │                            │                        │
 ├─────────────────────────►│                            │                        │
 │                          │                            │                        │
 │                          │ 2. POST /api/auth/register │                        │
 │                          ├───────────────────────────►│                        │
 │                          │                            │ 3. 检查邮箱是否存在    │
 │                          │                            ├───────────────────────►│
 │                          │                            │◄───────────────────────┤
 │                          │                            │                        │
 │                          │                            │ 4. bcrypt 加密密码     │
 │                          │                            │ 5. 创建用户记录        │
 │                          │                            ├───────────────────────►│
 │                          │                            │◄───────────────────────┤
 │                          │                            │                        │
 │                          │ 6. 201 Created + User      │                        │
 │                          │◄───────────────────────────┤                        │
 │                          │                            │                        │
 │                          │ 7. 自动登录                │                        │
 │                          │                            │                        │
 │                          │ 8. POST /api/auth/login    │                        │
 │                          ├───────────────────────────►│                        │
 │                          │                            │ 9. 查询用户            │
 │                          │                            ├───────────────────────►│
 │                          │                            │◄───────────────────────┤
 │                          │                            │                        │
 │                          │                            │ 10. 验证密码           │
 │                          │                            │ 11. 生成 JWT Token     │
 │                          │                            │                        │
 │                          │ 12. 200 OK + Tokens        │                        │
 │                          │◄───────────────────────────┤                        │
 │                          │                            │                        │
 │                          │ 13. 保存到 AsyncStorage    │                        │
 │                          │ 14. 更新 Zustand 状态      │                        │
 │                          │ 15. router.replace("/")    │                        │
 │◄─────────────────────────┤                            │                        │
 │                          │                            │                        │
 │ 16. 访问受保护资源       │                            │                        │
 ├─────────────────────────►│                            │                        │
 │                          │                            │                        │
 │                          │ GET /api/interview/scenarios                        │
 │                          │ Authorization: Bearer <token>                       │
 │                          ├───────────────────────────►│                        │
 │                          │                            │ 17. JWTAuth 中间件     │
 │                          │                            │ - 验证 Token           │
 │                          │                            │ - 注入 user_id         │
 │                          │                            │                        │
 │                          │                            │ 18. 查询数据           │
 │                          │                            ├───────────────────────►│
 │                          │                            │◄───────────────────────┤
 │                          │                            │                        │
 │                          │ 19. 200 OK + Data          │                        │
 │                          │◄───────────────────────────┤                        │
 │◄─────────────────────────┤                            │                        │
 │                          │                            │                        │
 │                          │ (15 分钟后 Token 过期)     │                        │
 │                          │                            │                        │
 │                          │ GET /api/user/profile      │                        │
 │                          ├───────────────────────────►│                        │
 │                          │                            │ 401 Unauthorized       │
 │                          │◄───────────────────────────┤                        │
 │                          │                            │                        │
 │                          │ 自动触发刷新               │                        │
 │                          │                            │                        │
 │                          │ POST /api/auth/refresh     │                        │
 │                          ├───────────────────────────►│                        │
 │                          │                            │ 验证 Refresh Token     │
 │                          │                            │ 生成新 Token Pair      │
 │                          │                            │                        │
 │                          │ 200 OK + New Tokens        │                        │
 │                          │◄───────────────────────────┤                        │
 │                          │                            │                        │
 │                          │ 保存新 Token               │                        │
 │                          │ 重试原请求                 │                        │
 │                          ├───────────────────────────►│                        │
 │                          │ 200 OK + Profile Data      │                        │
 │                          │◄───────────────────────────┤                        │
 │◄─────────────────────────┤                            │                        │
```

---

## 关键文件清单

### 前端文件

| 文件路径 | 作用 |
|---------|------|
| `frontend/app/(auth)/login.tsx` | 登录页面 UI |
| `frontend/app/(auth)/register.tsx` | 注册页面 UI |
| `frontend/app/index.tsx` | 应用入口，会话恢复 |
| `frontend/stores/authStore.ts` | 认证状态管理 (Zustand) |
| `frontend/services/api.ts` | API 客户端，Token 拦截器 |
| `frontend/types/user.ts` | 用户类型定义 |
| `frontend/constants/api.ts` | API 端点常量 |

### 后端文件

| 文件路径 | 作用 |
|---------|------|
| `backend/internal/handler/auth.go` | 认证 HTTP 处理器 |
| `backend/internal/service/auth.go` | 认证业务逻辑 |
| `backend/internal/middleware/auth.go` | JWT 认证中间件 |
| `backend/pkg/jwt/jwt.go` | JWT 生成与验证 |
| `backend/internal/model/model.go` | 数据库模型 |
| `backend/internal/app/app.go` | 应用初始化，路由配置 |
| `backend/internal/handler/ws.go` | WebSocket 认证 |
| `backend/internal/config/config.go` | 配置结构定义 |

---

## 配置说明

### 环境变量

```bash
# 前端 .env
EXPO_PUBLIC_API_URL=http://localhost:8080
EXPO_PUBLIC_WS_URL=ws://localhost:8080

# 后端 .env
JWT_ACCESS_SECRET=your-access-secret-key
JWT_REFRESH_SECRET=your-refresh-secret-key
JWT_ACCESS_EXPIRY=15        # 分钟
JWT_REFRESH_EXPIRY=7        # 天
```

### 配置文件

```yaml
# backend/config/config.yaml
jwt:
  access_secret: "${JWT_ACCESS_SECRET}"
  refresh_secret: "${JWT_REFRESH_SECRET}"
  access_expiry: 15
  refresh_expiry: 7

database:
  auto_migrate: true  # 开发环境自动创建表结构
```

---

## 最佳实践

### 1. Token 存储策略

- **Access Token:** 存储在内存 (Zustand) + AsyncStorage (持久化)
- **Refresh Token:** 仅存储在 AsyncStorage (安全存储)
- **不要**将 Token 存储在 localStorage (Web 端 XSS 风险)

### 2. Token 刷新策略

- 主动刷新: Token 过期前 1-2 分钟主动刷新
- 被动刷新: 收到 401 时自动刷新
- 防止并发: 使用 Promise 锁避免多次刷新

### 3. 错误处理策略

- 网络错误: 保留 Token，允许重试 (后端重启场景)
- 401 错误: 尝试刷新，失败后清除 Token
- 用户友好: 显示明确的错误提示

### 4. 安全建议

- 生产环境使用 HTTPS/WSS
- 定期轮换 JWT 签名密钥
- 启用 Redis Token 黑名单 (登出功能)
- 实施速率限制防止暴力破解
- 监控异常登录行为

---

## 常见问题

### Q1: 为什么使用双 Token 机制？

**A:** 
- Access Token 短期有效 (15 分钟)，降低泄露风险
- Refresh Token 长期有效 (7 天)，提升用户体验
- Access Token 泄露后影响时间短
- Refresh Token 可被撤销 (黑名单)

### Q2: Token 刷新时如何处理并发请求？

**A:** 
使用 Promise 锁机制，确保只有一个刷新请求在进行：
```typescript
if (this.refreshPromise) {
  await this.refreshPromise; // 等待现有刷新完成
  return;
}
```

### Q3: 后端重启时 Token 会丢失吗？

**A:** 
不会。Token 存储在客户端 AsyncStorage，后端重启不影响。API 客户端的网络错误处理会保留 Token 并自动重试。

### Q4: 如何强制用户登出？

**A:** 
1. 后端将用户的 Token ID 加入黑名单 (Redis)
2. 前端调用 `clearSession()` 清除本地状态
3. 下次请求时中间件会拒绝黑名单 Token

### Q5: WebSocket 认证为什么使用查询参数？

**A:** 
WebSocket API 不支持自定义 Header (浏览器限制)，因此通过查询参数 `?token=xxx` 或 `Sec-WebSocket-Protocol` 头传递 Token。

---

## 总结

InterviewPro 的认证系统实现了完整的双 Token JWT 认证流程：

✅ **注册:** 邮箱验证 + bcrypt 密码加密  
✅ **登录:** 密码验证 + Token 生成  
✅ **Token 管理:** 自动刷新 + 并发控制 + 重试机制  
✅ **会话恢复:** AsyncStorage 持久化 + 应用启动恢复  
✅ **路由保护:** JWT 中间件 + Context 注入  
✅ **WebSocket 认证:** 查询参数/协议头传递 Token  
✅ **安全机制:** 双 Token + 黑名单 + 错误处理  

整个系统兼顾了安全性、用户体验和系统韧性，为面试练习应用提供了可靠的认证基础。
