# ECDH 登录与通信流程图

本文档汇总 ECDH 登录、单聊及 Login-Token 相关流程图说明。图片文件：`ecdh登录.png`、`ecdh单聊.png`、`login-token.png`。
*说明：工作区中未找到名为 `登录流程图.png` 的独立文件；下文的「登录流程图」对应 `ecdh登录.png` 中的图示。*

[src: raw/ingested/3项目/分布式IM-雷漫/登录-ECDH登录与通信流程图.md]

---

## 1. 登录流程图（ecdh登录.png）

![登录流程图](ecdh登录.png)

描述客户端与服务器基于 **ECDH（椭圆曲线迪菲-赫尔曼）** 的安全登录认证流程，分为**服务器**与**客户端**两条纵列。

### 客户端流程

1. **生成临时密钥对 (c_priv, c_pub)**：生成临时私钥 `c_priv` 与公钥 `c_pub`。
2. **发送登录请求**：请求中包含用户名和 `c_pub`。
3. **验证服务器签名**：收到响应后验证服务器签名，确认服务器身份与完整性。
4. **计算共享密钥 K**：用本端私钥 `c_priv` 与服务器临时公钥 `s_pub` 计算共享密钥 `K`。
5. **派生会话密钥 SK**：从 `K` 派生用于加密的会话密钥 `SK`。
6. **发送用 SK 加密的认证凭证**：用 `SK` 加密认证凭证（如密码哈希/令牌）并发送。
7. **接收加密的成功响应**：接收服务器用 `SK` 加密的成功响应。
8. **用 SK 解密并确认**：解密后确认认证成功。

### 服务器流程

1. **根据用户名查用户**：收到登录请求后按用户名查用户记录。
2. **生成临时密钥对 (s_priv, s_pub)**：生成临时私钥 `s_priv` 与公钥 `s_pub`。
3. **计算 K 并签名**：用 `s_priv` 与客户端 `c_pub` 计算共享密钥 `K`，并用服务器长期私钥对 `c_pub`、`s_pub` 签名。
4. **发送 s_pub 与签名**：将 `s_pub` 和签名发给客户端。
5. **计算共享密钥 K**：用 `c_pub` 与 `s_priv` 计算（或复用已算出的）`K`。
6. **派生会话密钥 SK**：从 `K` 派生 `SK`。
7. **用 SK 解密并验证凭证**：用 `SK` 解密客户端凭证并校验。
8. **发送加密的成功响应**：校验通过后用 `SK` 加密成功响应并回传。

### 小结

通过 ECDH 协商出共享密钥 K，再派生会话密钥 SK；认证凭证与响应均用 SK 加解密。服务器用签名证明身份，客户端通过验签确认与可信服务器通信。

[src: raw/ingested/3项目/分布式IM-雷漫/登录-ECDH登录与通信流程图.md]

---

## 2. ECDH 单聊流程图（ecdh单聊.png）

![ECDH单聊流程图](ecdh单聊.png)

描述**客户端 A、客户端 B 与服务器**之间基于 ECDH 的安全登录与单聊通信流程。

### 参与者

- **客户端 B**：生成临时密钥对，将公钥发给服务器。
- **服务器**：维护用户公钥映射，转发公钥与加密消息。
- **客户端 A**：生成临时密钥对，发登录请求，加密消息、解密响应。

### 流程概要

**I. 密钥对生成与公钥交换**

- **客户端 B**：生成临时 ECC 密钥对（私钥 `dB`，公钥 `QB`），将 `QB` 发往服务器。
- **客户端 A**：生成临时 ECC 密钥对（私钥 `dA`，公钥 `QA`），登录请求中带用户 ID 与 `QA`。
- **服务器**：接收并存储 A 的 `QA`、B 的 `QB`，建立用户与公钥映射；将 A 的 `QA` 转发给 B。

**II. 共享密钥 K 与会话密钥 SK**

- **客户端 B**：收到 `QA` 后，用 `K = dB * QA` 计算共享密钥 K，再通过 KDF（含 client_random、server_random）派生会话密钥 SK。
- **客户端 A**：收到并验证 B 的 `QB`，用 `K = dA * QB` 计算 K，同样派生 SK。
  A、B 各自得到相同的 K 与 SK。

**III. 加密消息与响应**

- **客户端 A**：用 SK 加密消息 M（如 AES256-GCM），得到密文 C，将 C 与消息 ID 发往服务器。
- **服务器**：将加密消息转发给 B。
- **客户端 B**：用 SK 解密，再用 SK 加密响应，发回服务器。
- **服务器**：将加密响应转发给 A。
- **客户端 A**：用 SK 解密得到响应。

整体为 ECDH 握手 + 基于对称密钥 SK 的端到端加密单聊。

[src: raw/ingested/3项目/分布式IM-雷漫/登录-ECDH登录与通信流程图.md]

---

## 3. Login-Token 序列图（login-token.png）

![Login-Token序列图](login-token.png)

描述 **Client、Access、Router、Login、Redis** 五方协作的登录与会话建立流程（可能配合 ECDH 做密钥交换）。

### 参与者

- **Client**：发起连接与登录请求。
- **Access**：接入层，处理连接与请求转发。
- **Router**：将请求从 Access 路由到 Login，并回传响应。
- **Login**：解密数据、校验 token、生成 sessionKey、写缓存。
- **Redis**：存储与校验 token、缓存会话相关信息。

### 步骤概要

| 步骤 | 说明 |
|------|------|
| 1–2 | Client `connect()`，Access `ack()` 确认连接。 |
| 3–5 | Client 发 `Login()`，经 Access → Router → Login。 |
| 6 | Login **解密数据**。 |
| 7–8 | Login 向 Redis **获取 token**，Redis 返回。 |
| 9 | Login **compare token** 校验令牌。 |
| 10 | Login 在 Redis **创建/更新缓存**。 |
| 11 | Login **生成 sessionKey**（可为 ECDH 协商后的对称会话密钥）。 |
| 12–13 | Login 构造 **LoginRsp** 并附带 **sessionKey**，经 Router 返回。 |
| 14 | Access 将 **sessionKey 存入连接对象**。 |
| 15 | Access 向 Client 返回 **LoginRsp**。 |

### 要点

- 含「解密数据」与「对称密钥 sessionKey」，可与 ECDH 登录流程配合，保证登录与后续通信安全。
- Redis 负责 token 的获取、校验与缓存；Login 负责生成并下发给 Access 的 sessionKey。
- sessionKey 在 Access 层绑定到连接，用于后续加密通信。

[src: raw/ingested/3项目/分布式IM-雷漫/登录-ECDH登录与通信流程图.md]

---

## 图片文件索引

| 说明           | 文件名           |
|----------------|------------------|
| 登录流程图     | `ecdh登录.png`   |
| ECDH 单聊流程  | `ecdh单聊.png`   |
| Login-Token 序列图 | `login-token.png` |

*若你本地有单独的 `登录流程图.png`，可将其放在同目录并在此文档中增加引用。*

[src: raw/ingested/3项目/分布式IM-雷漫/登录-ECDH登录与通信流程图.md]