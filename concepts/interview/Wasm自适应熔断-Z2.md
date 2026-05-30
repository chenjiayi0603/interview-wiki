# Wasm 自适应熔断插件（Z2）

**简历原文**：自研 Envoy Wasm 自适应熔断插件：滑动窗口错误率统计 + 自动熔断 + half-open 探针恢复

## 数据支撑

| 参数 | 配置值 | 说明 |
|------|--------|------|
| 滑动窗口大小 | 10s | 统计窗口内错误率和 P99 延迟 |
| 错误率阈值 | >50% | 窗口内错误率超阈值触发熔断 |
| P99 延迟阈值 | >1s | 延迟超阈值也独立触发（OR 条件） |
| 熔断持续时间 | 3s | 期间所有请求直接返回 503 |
| half-open 探针间隔 | 1s | 周期性放行少量请求验证 |
| half-open 成功次数 | 连续 2 次 | 成功后关闭熔断 |
| 最短恢复路径 | **<5s** | 3s 熔断 + 1s 第 1 次探针 + 1s 第 2 次探针 |

Wasm vs Native C++ Filter 性能：Wasm 慢约 20%，但熔断逻辑轻量（几十行计数器统计），绝对差值约 0.5ms，被网络延迟（内网 RTT ~1ms）淹没，不是瓶颈。

[src: raw/ingested/3项目/面试准备/简历知识点论证手册-Z2：Envoy-Wasm-自适应熔断插件.md]

## 理论支撑

滑动窗口计数器：固定大小时间窗口内统计错误数、总请求数，计算错误率。与固定窗口的区别：固定窗口在窗口边界会出现计数重置导致的短暂误判；滑动窗口每次统计最近 N 秒，避免边界效应。

half-open 状态机：熔断器三态（Closed → Open → Half-open），Closed = 正常，Open = 全部失败，Half-open = 试探恢复。连续 K 次探针成功 → 回到 Closed；失败 → 回到 Open。这是 Hystrix 引入的经典模式，防止服务恢复后瞬间被大流量压垮。

选 Wasm 不选 Native C++ Filter 的三个原因：① 热更新（改阈值不重启 Envoy）；② 沙箱安全（插件 bug 不影响 Envoy 主进程）；③ 跨语言（Go/Rust/AssemblyScript 均可编写）。

[src: raw/ingested/3项目/面试准备/简历知识点论证手册-Z2：Envoy-Wasm-自适应熔断插件.md]

## 业界对标

| 框架 | 语言 | 熔断机制 | 热更新 |
|------|------|---------|--------|
| Hystrix（Netflix） | Java | 滑动窗口 + half-open | 需重部署 |
| Sentinel（阿里） | Java/Go | 滑动窗口 + half-open | 规则动态推送 |
| Envoy 内置熔断 | C++ | 连接数 / 请求数上限 | 配置热更新 |
| **本项目 Wasm 插件** | Go/C++ | 错误率 + 延迟双条件 + half-open | `istioctl apply` 秒级生效 |

[src: raw/ingested/3项目/面试准备/简历知识点论证手册-Z2：Envoy-Wasm-自适应熔断插件.md]

## 追问预案

**Q：为什么 Wasm 比 Native C++ Filter 慢 20%？**
> Wasm 运行在独立 VM 沙箱（V8 或 Wasmtime），每次插件执行需要跨越 Host↔Wasm 边界（上下文切换 ~1μs），加上 Wasm 字节码解释/JIT 执行比原生机器码略慢。绝对差值：Native ~0.8–1.5ms，Wasm ~1–2ms，差距约 0.5ms。在内网 RTT 1ms 的背景下可忽略。

[src: raw/ingested/3项目/面试准备/简历知识点论证手册-Z2：Envoy-Wasm-自适应熔断插件.md]

## 相关页面
- [[Wasm自适应熔断]]
- [[服务网格平台-即构]]
- [[Cpp_Wasm_开发]]