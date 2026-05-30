# Redis 连接问题

See also: [[C++网络编程]]

## 现象
- Connection refused
- 连接超时

## 可能原因
- Redis 未启动或已崩溃
- `bind` 只绑定了 127.0.0.1，客户端从其他机器访问
- `protected-mode yes` 且未配置 `requirepass` 或客户端未带密码，非本机被拒绝
- 防火墙（iptables/firewalld/云安全组）未放行 Redis 端口
- 网络不通：路由、VPN、跨机房、端口被占用等
- `maxclients` 已满，新连接被拒绝
- 客户端连接池耗尽或连接泄漏，导致“假性”超时

## 排查步骤
1. **网络连通性**：检查客户端与 Redis 之间的网络是否可达
2. **服务状态**：确认 Redis 服务是否正常运行（`redis-cli ping`）
3. **配置绑定地址与保护模式**：检查 `bind`、`protected-mode` 配置
4. **最大连接数限制**：检查 `maxclients` 是否过小
5. **防火墙规则**：排查端口是否被防火墙拦截

## 解决方案
- 调整 `bind` / `protected-mode` 配置
- 优化连接池（避免连接过多或过少）
- 调整防火墙规则，放行 Redis 端口

## 根因原理深入分析

### maxclients 耗尽的根本原因
Redis 采用**单 Reactor 模型**（6.0 之前），所有客户端连接由一个 event loop 管理。每个连接对应一个 file descriptor（fd），Linux 默认单进程 fd 上限为 1024（可通过 `ulimit -n` 调整）。当连接数达到 `maxclients`（默认 10000）或操作系统 fd 上限时，新连接直接被拒绝。

**根因链条**：客户端未正确释放连接 → 连接泄漏 → fd 占满 → 新连接被拒

**关键原理**：
- Redis 的 `maxclients` 实际受限于 `min(maxclients配置, 操作系统fd上限 - 32)`，32 是 Redis 内部预留 fd
- 连接池配置不当（maxIdle 过大/过小、未设超时回收）是生产环境最常见的连接问题根因
- 短连接模式（每次请求新建+关闭连接）在高并发下会产生大量 TIME_WAIT，耗尽端口资源

**处理方式**：
```
# 查看当前连接数
redis-cli INFO clients
# connected_clients: 当前连接数
# blocked_clients: 阻塞在 BLPOP 等命令的连接数

# 查看各客户端连接详情，定位泄漏来源
redis-cli CLIENT LIST

# 调大系统 fd 上限
ulimit -n 65535
# 永久生效需修改 /etc/security/limits.conf
```

### protected-mode 的设计原理
Redis 3.2 引入 `protected-mode`，其本质是一个**安全兜底机制**：当 Redis 同时满足以下三个条件时，会拒绝外部连接：
1. `bind` 未显式配置（默认绑定所有网卡）
2. `requirepass` 未设置
3. 客户端来自非 127.0.0.1

设计目的是防止裸奔的 Redis 暴露在公网，避免被未授权访问或被挖矿攻击。

[src: raw/ingested/2技术/redis/Redis_KeyDB运维问题速查.md]

## Related Pages
- [[C++网络编程]]
- [[TCP协议]]
- [[Redis性能问题]]
- [[服务网格平台-即构]]
- [[Go网络编程]]
