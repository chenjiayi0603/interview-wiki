# WSS 协议转换原理

## 3.1 WSS 与 WS 的关系

**wss（WebSocket Secure）** 本质上就是在 WebSocket 协议（ws）之上通过 SSL/TLS 实现加密，因此 wss 实际上等价于"通过 SSL/TLS 加密的 WebSocket"（类似于 https 之于 http）。

**协议对比：**

- `ws://` —— 明文 WebSocket，基于普通 TCP，无加密
- `wss://` —— 加密 WebSocket，建立在 TLS（SSL）之上，通信过程全部加密

## 3.2 Nginx 协议转换机制

Nginx 作为 WebSocket 网关的反向代理时，可以实现 wss（加密 WebSocket）与 ws（普通 WebSocket）之间的自动协议转换，主要原理如下：

#### 1. TLS/SSL 终止（Termination）

客户端与 Nginx 通信时采用 wss 协议（即 WebSocket over TLS/SSL），数据加密传输。Nginx 配置了 SSL 证书（`ssl_certificate`，`ssl_certificate_key`），负责与客户端建立安全连接（SSL 握手），数据到达 Nginx 时已被解密。

#### 2. WebSocket 协议升级

Nginx 收到请求后，通过 `proxy_set_header Upgrade "websocket"` 及 `proxy_set_header Connection "Upgrade"` 等配置，完成 HTTP 到 WebSocket 协议的升级，支持长连接和全双工通信。

#### 3. 转发为明文 ws

协议升级后，Nginx 将解密后的普通 WebSocket（ws）协议流量转发给后端 Access 网关。此时，Nginx 到 Access 通信链路不加密，协议为 ws，可以实现高性能、低延迟的数据交换。

#### 4. 自动协议转换，无需感知

对于客户端与 Access 网关而言，都无需关心对方的协议类型。Nginx 持续担当"协议桥梁"的角色，客户端发送的所有 wss 数据由 Nginx 转换为 ws，Access 返还的 ws 数据再由 Nginx 包裹为 wss 加密后发回客户端。转换是自动且透明的，连接存续期间持续生效。

**协议转换流程：**

```
客户端
  ⇅ (wss, SSL加密)
 [Nginx]                   // 负责TLS解密/加密 + 协议升级/转换
  ⇅ (ws, 明文)
Access 网关
```

**关键点：**

- 这种"转换"只存在于 Nginx 这一跳：**客户端与 Nginx 始终是 wss，Nginx 到 Access 始终是 ws，整个连接时刻都是这样**
- 后续客户端的所有数据消息，也都经过这个机制，即：先 wss 到 Nginx，然后由 Nginx 自动转发成 ws 到 Access，反之亦然
- **Nginx 持续充当协议转换角色**，直到 WebSocket 连接关闭。不是只握手或最初消息才做转换，而是连接期间所有的消息包都会一直经过 wss<->ws 的协议转换处理，无须客户端关注详情

**好处：**

- 保证公网链路全程加密，提升安全性
- 业务服务只需维护 ws，无需感知 SSL/TLS
- 提升后端扩展性与运维效率

## 3.3 SSL/TLS 握手流程

SSL（Secure Sockets Layer，现已由 TLS 取代）为 WebSocket 的 `wss://` 提供安全加密通道，其核心流程主要包括"握手（Handshake）"和"数据加密传输"两大阶段。

#### 握手阶段（SSL/TLS Handshake）

握手阶段目的是：**双方协商加密算法、完成身份验证、生成共享密钥。** 主要过程如下：

1. **客户端发起请求**
   - 客户端（如浏览器）发起 `ClientHello`，携带支持的 SSL/TLS 版本、加密算法列表、随机数、必要扩展（如 SNI）等

2. **服务端响应**
   - Nginx 作为服务器，返回 `ServerHello`，选定双方共同支持的协议版本和加密算法，返回自己的证书（`Certificate`，含公钥）和随机数

3. **证书验证**
   - 客户端校验服务端证书是否合法（由受信任 CA 签发/未过期/域名一致等）
   - 若验证失败则握手中断

4. **密钥协商与生成预主密钥**
   - 客户端生成预主密钥（pre-master secret），用公钥加密后发送给服务器（`ClientKeyExchange`）
   - 服务端用私钥解密获取预主密钥

5. **对称密钥生成**
  握手阶段完成后，双方通过SSL（TLS）标准流程协商生成对称加密密钥。典型流程如下：

  ##### RSA 密钥交换流程

  ```
  客户端                                    服务端 (Nginx)
  ──────────────                            ──────────────
 | 生成 ClientHello，携带支持算法、随机数等 |  
 |----------------------------------------->| 
                                            | 
                                            | 校验证书，服务端生成 ServerHello，选择加密算法、随机数
                                            |<------------------------------------|
 | 验证证书合法性                           | 
 |----------------------------------------->| 
 | 生成 pre-master secret（预主密钥）       |
 | 用服务端证书中的公钥加密预主密钥         |
 |----------------------------------------->| 
                                            | 
                                            | 用私钥解密获得 pre-master secret
                                            | 
 |-----------------------------------------| 
 | 双方结合 ClientHello/ServerHello 的随机数|
 | 和 pre-master secret，通过密钥派生算法   |
 | 推导出对称加密的 session key            |
 |                  <--------------------->|
  ```

  ##### ECDHE 密钥交换流程（椭圆曲线临时 Diffie-Hellman 更安全，主流）

  ```
  客户端                                    服务端 (Nginx)
  ──────────────                            ──────────────
 | 生成 ClientHello，携带支持算法、随机数等 |  
 |----------------------------------------->| 
                                            | 
                                            | 校验证书，服务端生成 ServerHello，选择加密算法、随机数
                                            |<------------------------------------|
 | 验证证书合法性                           | 
 |----------------------------------------->| 
 | 生成临时 ECDHE 密钥对（私钥、公开密钥）  |
 | 发送客户端 ECDHE 公钥                    |
 |----------------------------------------->| 
                                            | 
                                            | 生成临时 ECDHE 密钥对（私钥、公开密钥）
                                            | 发送服务端 ECDHE 公钥
                                            |<------------------------------------|
 | 双方各自通过 ECDHE 协议协商出            |
 | 一致的 pre-master secret（利用对方公钥和  |
 | 自己私钥）                               |
 |-----------------------------------------| 
 | 双方结合 ClientHello/ServerHello 的随机数|
 | 和 pre-master secret，通过密钥派生算法   |
 | 推导出对称加密的 session key            |
 |                  <--------------------->|
  ```

- **RSA 密钥交换**：pre-master secret 由客户端生成并用服务端公钥加密发送；安全性依赖 RSA 公钥私钥机制。
- **ECDHE 密钥交换**：客户端和服务端各自生成临时密钥对，交换公钥，最终通过双方各自的私钥和对方的公钥算出相同的 pre-master secret，安全性和前向安全性更高。

不论采用哪种方式，最终都会得到一组仅本次会话有效的对称密钥（session key）用于后续加密通讯。

6. **握手完成、进入加密数据传输**
   - 客户端和服务端分别发出 `ChangeCipherSpec` 和 `Finished` 消息，表示后续通讯都用协商好的密钥和加密算法

**时序图：**

```
// TLS 握手完整时序图（带注释说明）
Client                                 Server(Nginx)
  // RSA 与 ECDHE 握手流程略有不同，核心区别主要在于密钥交换方式与前向安全性：
  // - RSA：服务端不会发送 ECDHE 公钥，也不会交换临时密钥对。pre-master secret 由客户端生成并用服务端证书的公钥加密。
  // - ECDHE：服务端和客户端都生成临时ECDHE密钥对，互相发送公钥，协商pre-master secret，具备前向安全性。

  // 下面以 ECDHE 握手为例，着重标明 ECDHE 公钥的交换时机：

  | ------ ClientHello ---------------> |   // 客户端发起握手，声明支持的加密算法(ECDHE等)、TLS版本、随机数
  | <----- ServerHello ---------------- |   // 服务端选择加密算法（如ECDHE）、TLS版本、并返回随机数
  | <----- Certificate ---------------- |   // 服务端发下证书，证明身份（含公钥，供客户端校验）
  | <----- ServerKeyExchange ---------- |   // 【仅ECDHE时有】服务端发送 ECDHE 公钥（临时公钥），此时客户端获取到服务端的 Diffie-Hellman 公钥（ECDHE）
  | <----- ServerHelloDone ------------ |   // 服务端发送握手消息结束
  | ------ ClientKeyExchange ---------->|   // 客户端发送 ECDHE 公钥（临时密钥），双方此时都拥有对方的临时公钥，通过各自私钥+对方公钥生成相同的 pre-master secret
  //                ▲
  //                │
  //         这里客户端和服务端（分别用自己的私钥+对方公钥），算出一致的 pre-master secret
  //                │
  //                ▼
  //   再结合 ClientHello/ServerHello 的随机数
  //   通过密钥派生算法**共同推导出 对称加密用的 session key(对称密钥)**
  //   对称密钥不在握手报文中传递，只在双方端本地生成并持有，外部无法截获
  | ------ CertificateVerify ---------->|   // 客户端可选发校验证书签名（双向认证时）
  | ------ ChangeCipherSpec ----------->|   // 客户端通知已准备好切换为对称加密（即用 session key 加密通信）
  | ------ Finished ------------------->|   // 客户端握手业务结束，并用对称密钥加密校验数据
  | <----- ChangeCipherSpec ------------|   // 服务端切换至对称加密通信（开始用 session key 加解密）
  | <----- Finished --------------------|   // 服务端握手结束，并用 session key 加密校验数据

  // 总结：
  // - 使用RSA：不会有 ServerKeyExchange 步骤，客户端直接用证书中公钥加密pre-master secret。
  // - 使用ECDHE：服务端会在 ServerHelloDone 前，通过 ServerKeyExchange 步骤明确发送 ECDHE 公钥。
```

rsa：
  // 下面是基于 RSA 密钥交换的典型 TLS 握手流程（无 ServerKeyExchange，ClientKeyExchange 用服务端公钥加密 pre-master secret）：
  | ------ ClientHello ---------------> | // 客户端发起握手，声明支持的加密算法、TLS版本、客户端随机数
  | <----- ServerHello ---------------- | // 服务端响应，选择加密算法、TLS版本、服务端随机数
  | <----- Certificate ---------------- | // 服务器公钥就在这里：服务端发送证书，证书里包含服务器的公钥
  | <----- ServerHelloDone ------------ |
  | ------ ClientKeyExchange ---------->| // 客户端用证书中携带的公钥加密 pre-master secret 并发送
  | ------ CertificateVerify ---------->| // （可选，双向认证时客户端提供证书证明身份）
  | ------ ChangeCipherSpec ----------->| // 客户端通知服务器即将启用加密
  | ------ Finished ------------------->| // 客户端握手结束，校验握手完整性（加密数据）
  | <----- ChangeCipherSpec ------------| // 服务器通知客户端切换加密
  | <----- Finished --------------------| // 服务器握手完成，校验握手完整性（加密数据）

wss（即 WebSocket Secure）本质上就是 WebSocket 通信通过 SSL/TLS 加密。它并不限定采用哪一种具体的 SSL/TLS 密钥交换算法，底层取决于 Nginx 和客户端协商支持的 TLS 协议与加密算法套件（Cipher Suite）。

常见情况说明：
- **协议类型**：wss 用的是 TLS（早期称 SSL，但实际现在基本都是 TLS1.2/1.3），协议和 HTTPS 完全一致。
- **加密套件**：大多数现代配置下，会优先协商使用 ECDHE（椭圆曲线 Diffie-Hellman 临时密钥交换，具备前向安全性）结合 RSA/ECDSA 证书，或者直接用 RSA（不具备前向安全性）。
- **和 HTTPs 一致**：nginx 的 wss 和 https 一样，走的是 Nginx 的 ssl_certificate 和 ssl_certificate_key，使用标准 TLS 协议进行协商和加密。
- **实际用哪种算法？** 取决于：客户端（浏览器/SDK）与 Nginx 的 TLS 支持交集以及 Nginx 配置的 ssl_ciphers。默认推荐且常见的都是 `ECDHE-RSA-AES256-GCM-SHA384`、`ECDHE-ECDSA-...` 这类即前向安全又高强度的套件。

**结论：wss 采用和 https 一致的 SSL/TLS 协议，算法协商由客户端与 Nginx 双方共同决定，并不限定为单一一种加密算法。常见推荐为 ECDHE+RSA/EC 证书套件。**

#### 加密数据传输阶段

握手完成后，双方利用上面协商得出的"对称密钥"进行数据的**加密和解密**。每次通讯的数据包经过加密算法处理，外部窃听者即使截获数据也无法直接还原明文。

- WebSocket 的 wss 通道数据同样采用此密钥加密
- 直到连接断开或会话超时，密钥和通信保持一致

#### SSL/TLS 主要目标

- **安全性**：所有数据在公网传输全程加密防窃听
- **身份认证**：服务器端通过证书向客户端证明"我是 websocket.raymannet.com"
- **数据完整性**：加入 MAC 校验，防止中间人篡改数据包

#### 关键点

- 握手阶段用**非对称加密**（公钥/私钥）保证密钥安全传递
- 传输阶段用**对称加密**（如 AES）保证高效加密
- Nginx 承担 SSL 处理的所有细节，后端 Access 无须感知

**总结：** SSL 的全部核心在于——通过短暂一次性"握手"过程安全协商出传输密钥，后续所有数据均透明加密，大幅提升了 WebSocket 通道的安全等级。这一切在 Nginx 层屏蔽，业务无须关心。

[src: raw/ingested/3项目/分布式IM-雷漫/登录-websocket网关总结-三、WSS-协议转换原理.md]