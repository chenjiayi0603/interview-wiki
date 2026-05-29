# WebSocket 网关设计总结

## 二、系统架构

### 2.1 架构简图

```
客户端（浏览器/APP）
        |
     WSS连接
        |
        v
    Nginx (SSL, 反向代理)
        |
    WS转发
        |
        v
  Access 网关节点
        |
   用户登录/校验
        |
        v
   Login服务
        |
  认证信息/用户状态
        |
        v
  Access网关
```

**流程说明：**

1. 客户端通过 WSS（WebSocket Secure，基于 TLS/SSL 加密）发起安全 WebSocket 连接请求（如 `wss://websocket.raymannet.com`）
2. Nginx 作为反向代理服务器接收 WSS 连接，负责：
   - 终止 TLS/SSL，实现加密通信
   - 校验证书，保障数据在传输过程中的安全性与防窃听
   - 根据配置将 WebSocket 流量路由/转发到后端的 Access 网关节点（可以负载均衡、多后端冗余）
   - 维持长连接的 proxy 协议升级（HTTP 升级为 WebSocket），保证握手和数据帧的双向透明传递
3. Access 网关收到连接后，执行用户登录与鉴权校验
4. 鉴权通过调用 Login 服务，完成 Token 检查、用户有效性等校验，返回认证结果与用户状态
5. Access 网关根据认证信息建立用户长连接，并维护后续同步

**架构特点：**

- 客户端使用 WSS 与 Nginx（SSL 反代）建立安全连接，Nginx 将 WebSocket 流量根据负载分发给不同 Access 网关（支持水平扩展）
- Access 网关负责用户认证（调用 Login 服务）、消息收发/转发（调用 Msg 服务）、状态管理（依赖 Redis 缓存），并与其它服务松耦合
- Redis 用于用户/会话状态、Token、消息中继等高速缓存，数据库只做持久和冷数据读写
- 各分层通过 RPC/HTTP 或直连方式通信，实现高并发、高可用

### 2.2 主要组件

#### Nginx 反向代理

- **域名**：`websocket.raymannet.com`
- **通信端口**：443（WSS）
- **功能**：支持 wss 协议，反向代理到后端 Access
- **配置要点**：设置 `proxy_upgrade`，`ip_hash` 负载均衡

**前端连接示例：**

```javascript
WEBSOCKET_URL = 'wss://websocket.raymannet.com:443/hello/shake';
```

#### Access 网关

- **地址**：IP: `10.3.0.241`，端口: `27006`
- **功能**：处理 WebSocket 连接、消息加解密、限流
- **动态插件**：`ModuleShake.so` 处理握手协议
- **配置路径**：`/app/thunder/deploy/Access/confweb/Access.json`

**关键配置：**

```json
{
  "node_type": "ACCESS",
  "access_host": "10.3.0.241",
  "access_port": 27006,
  "access_codec": 10,
  "inner_host": "10.3.0.241",
  "inner_port": 27007,
  "access_verify_time": 30,
  "server_name": "Access_web_im",
  "module": [
    {
      "url_path": "/hello/shake",
      "so_path": "plugins/ModuleShake.so",
      "entrance_symbol": "create",
      "load": true,
      "version": 1
    }
  ]
}
```

#### Web 客户端

- **心跳间隔**：推荐 3.5 分钟
- **登录时限**：连接后 30 秒内完成登录（消息 1001）

[src: raw/ingested/3项目/分布式IM-雷漫/登录-websocket网关总结-二、系统架构.md]