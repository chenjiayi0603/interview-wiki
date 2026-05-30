# Envoy Sidecar性能

See also: [[C++网络编程]]

## 概述

Envoy Sidecar是服务网格中数据面的核心组件，以Sidecar模式部署在每个服务Pod中，代理所有进出流量。Sidecar的引入会带来一定的性能开销。

[src: raw/ingested/3项目/服务网格-即构/面试故事-服务网格.md]

## 性能开销分析

### 请求链路

```
客户端 → Envoy(in) → 应用 → Envoy(out) → 下游服务
```

### 开销组成

| 开销项 | 延迟增加 | 说明 |
|--------|---------|------|
| Envoy代理转发 | +0.5ms × 2 = +1ms | 进出各一次 |
| Wasm插件执行 | +0.5ms | 自定义插件逻辑 |
| 统计指标收集 | +0.2ms | 监控指标 |
| 连接池管理 | +0.3ms | 连接复用管理 |
| **总计** | **约+2ms** | QPS损失约15% |

[src: raw/ingested/3项目/服务网格-即构/面试故事-服务网格.md]

## QPS测试数据

| 场景 | QPS | P99延迟 | Sidecar开销 |
|------|-----|--------|------------|
| 无Sidecar（直连） | 80,000 | 2ms | 0% |
| 有Sidecar（无插件） | 72,000 | 3ms | 10% |
| 有Sidecar（+Wasm熔断） | 68,000 | 4ms | 15% |
| 熔断触发时 | 60,000 | 50ms | 部分请求被拦截 |

[src: raw/ingested/3项目/服务网格-即构/面试故事-服务网格.md]

## 边车延迟实测

边车（客户端、服务器增加边车）增加30%左右的延迟，降低40%左右的QPS。

| 场景 | 平均延迟 | QPS | 说明 |
|------|---------|-----|------|
| 远程直连（无策略） | 110.245 ms | 886.8 | 通过IP直接访问 |
| 远程（一个Envoy + 一次iptables） | 138.972 ms | 620.5 | 对方一个Envoy（使用UDS）、一次iptables，无策略 |
| 远程（两个Envoy + 两次iptables） | 148.748 ms | 581.1 | 经过两个Envoy（使用UDS）、两次iptables；有路由策略 |

**延迟增加**：从110ms到149ms，增加约35%
**QPS下降**：从887到581，下降约35%

[src: raw/ingested/3项目/服务网格-即构/面试故事-服务网格.md]

## 测试命令

```bash
# 直连测试
gunicorn httpbin:app -D -b '192.168.63.56:8000'
fortio load -c 100 -n 2000 -qps 0 http://192.168.63.56:8000/ip

# 一个Envoy测试
fortio load -c 100 -n 500 -qps 0 http://192.168.63.56:80/ip

# 两个Envoy测试
fortio load -c 100 -n 500 -qps 0 -H 'end-user:x' -unix-socket /tmp/gunicornout.sock labor-vm-cjy-app2.labor-vm-cjy.svc.cluster.local/ip
```

[src: raw/ingested/3项目/服务网格-即构/面试故事-服务网格.md]

## 开源代理性能对比

### Apache APISIX vs Envoy

| 进程数 | APISIX QPS | APISIX Latency | Envoy QPS | Envoy Latency |
|--------|-----------|----------------|-----------|---------------|
| 1 worker | 18608.4 | 0.96 | 15625.56 | 1.02 |
| 2 workers | 34975.8 | 1.01 | 29058.135 | 1.09 |
| 3 workers | 52334.8 | 1.02 | 42561.125 | 1.12 |

**总结**：APISIX在响应延迟和QPS层面都略优于Envoy，由于nginx的多worker协作方式在高并发场景下更有优势。Envoy的总线设计使它在处理东西向流量上有独特优势。

[src: raw/ingested/3项目/服务网格-即构/面试故事-服务网格.md]

### nginx、kong、envoy代理转发性能对比

**网关资源估算方法**（假设网关总共32核，仅做代理）：

| 网关 | TPS上限估算 |
|------|-----------|
| nginx | 338k |
| kong | 157k |
| envoy | 230k |

**估算公式**：
- nginx: 36k * (32/3.4) = 338.8k
- kong: 36k * (32/7.3) = 157.8k
- envoy: 36k * (32/5.0) = 230.4k

[src: raw/ingested/3项目/服务网格-即构/面试故事-服务网格.md]

## 面试问答

**Q: 性能开销大吗？**

> 有开销，大约增加1-2ms延迟。但对于业务服务，这个开销可以接受。我们做过压测，QPS影响在5%以内。

[src: raw/ingested/3项目/服务网格-即构/面试故事-服务网格.md]

## 相关概念

- [[服务网格平台-即构]] - 即构科技服务网格平台项目
- [[Wasm自适应熔断]] - Wasm插件实现的自适应熔断算法

## Related Pages
- [[C++网络编程]]
- [[服务网格平台-即构]]
- [[Wasm自适应熔断]]
