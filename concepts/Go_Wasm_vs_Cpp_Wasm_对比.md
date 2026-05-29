# Go Wasm vs Cpp Wasm 对比

## 接口对比

| envoy api\语言库 | go wasm | cpp wasm |
|---|---|---|
| 定时回调 | 支持，已测试 | 支持，已测试 |
| 主动发送http | 支持，已测试，ProxyHttpCall | 支持，有声明接口，proxy_http_call |
| 主动发送grpc | 未有声明接口 | 支持，有声明接口,proxy_grpc_call |
| 修改http body | 支持，已测试 | 支持，已测试 |
| 修改 http header | 支持，已测试 | 支持，已测试 |
| 配置动态更新 | 支持，已测试 | 支持，已测试 |
| 共享数据cas接口 SharedData | 支持，已测试 | 支持，已测试 |
| 共享数据原子类型接口 Metric | 支持，已测试 | 支持，已测试 |
| 请求上下文 | 支持，已测试，HttpContext | 支持，已测试，Context |
| 全局上下文 | 支持，已测试，PluginContext或者VmContext | 支持，已测试，RootContext |
| 时间函数 | 部分支持，已测试，支持time.Now().Unix()和time.Now().UnixNano()，不能直接使用time.Now()和time.Now().Second() | 支持，已测试 |
| 读取文件 | 不支持，不能使用（os.Stat、os.Open） | 不支持读取系统文件 |
| json库 | 部分支持，已测试，支持buger/jsonparser，不支持需要反射json库encoding/json | 支持，cjson |
| 远程函数 | 支持，有声明接口 | 支持，有声明接口 |
| 兼容性 | 支持沙盒模式 | 支持沙盒模式、支持直接编译连接到envoy执行文件 |
| 生态 | 资料较少 tinygo_0.21 | istio 有proxy实践、社区有分享 emscripten 3.0 |

功能接口上，go wasm 和cpp wasm大致相同，除了go wasm未有主动发送grpc；可以处理一些类似错误熔断、自适应熔断等功能。

## 性能对比

| 性能项 | go wasm | cpp wasm | 对比结果 |
|---|---|---|---|
| 计算密集，数字运算处理（例子素数判断1到10000）使用库函数 | 200 conns 113.738 ms 1751.2 qps | 200 conns 26.453 ms 7538.1 qps | cpp wasm 比go wasm 性能高很多 |
| 计算密集，数字运算处理（例子素数判断1到10000）不使用库函数 | 200 conns 25.511 ms 7811.8 qps | 200 conns 26.202 ms 7604.9 qps | cpp wasm 和go wasm差不多 |
| 调用envoy 的api（例子原子api读写分别10000次） | 200 conns 263.305 ms 753.3 qps | 200 conns 242.102 ms 819 qps | cpp wasm 比go wasm 稍微高一点 |
| 网络数据包处理，较大数据包请求和响应（例子10000字节消息体请求并相同内容响应） | 200 conns 35.438 ms 5623.4 qps | 200 conns 33.459 ms 5953.9 qps | cpp wasm 比go wasm 稍微高一点 |

性能对比上，网络数据包处理、envoy api调用性能cpp wasm 比go wasm 稍微高一点。cpp wasm和go wasm差不多（不用库函数情况下，使用库有区别）。

[src: raw/ingested/3项目/服务网格-即构/zg_wasm技巧.md]