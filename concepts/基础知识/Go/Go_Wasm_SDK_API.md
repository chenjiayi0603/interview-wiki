# Go Wasm SDK API

proxy-wasm-go-sdk功能测试结果。sdk 封装来自envoy声明的接口，参考：abi_hostcalls.go。

## 一、目前测试结果

Go Wasm Demo测试：
- 主动定时往外发送请求（通过envoy CLUSTER接口）
- 修改http body
- 修改 http header
- 在envoy进程内在多个vm之间共享数据（SharedData相关接口）
- 读取istio的yaml部分配置
- 原子计数

目前测试总结：
- 能实现一些统计功能
- 已测支持的功能：
  - json不使用反射的库是可以使用（例如github.com/buger/jsonparser）。
  - 能支持HttpContext（请求上下文，对应cpp wasm context）、PluginContext或者VmContext（全局上下文，对应cpp wasm rootcontext），参考：context.go
  - 能支持配置动态更新
  - 能部分使用时间函数（time.Now().Unix()和time.Now().UnixNano()）

目前测试出现的问题：
- 测试部分go 网络协议库（go-redis、net/http等）、配置json与yaml库（使用到反射的库，tinygo使用go反射出现问题）出现编译或运行问题；
- 不能使用（time.Now()和time.Now().Second()等）
- 有的文件系统接口不能使用（os.Stat、os.Open）

## 二、tinygo测试（限制和注意事项）

详细解释参考sdk说明文档（以下是部分中文翻译）：
https://github.com/tetratelabs/proxy-wasm-go-sdk/blob/main/doc/OVERVIEW.md#limitations-and-considerations

一些现有库不可用（可导入但运行时恐慌/不可导入）。有几个原因：
- TinyGo 的 WASI 目标不支持某些系统调用。
- TinyGo 没有实现所有的反射包。
- Proxy-Wasm C++ 主机尚不支持部分 WASI API，参考 https://github.com/proxy-wasm/proxy-wasm-cpp-host/blob/master/include/proxy-wasm/exports.h
- TinyGo 或 Proxy-Wasm 中不提供某些语言功能：示例包括recover和goroutine。

recover 未实现。

协程支持：在 TinyGo 中，Goroutine 是通过 LLVM 的协程实现的。在 Envoy 中，Wasm 模块以事件驱动的方式运行，因此一旦主函数退出，“调度程序”就不会执行。这意味着您不能像在普通主机环境中那样拥有 Goroutine 的预期行为。我们强烈建议您OnTick为任何异步任务实现该函数，而不是使用 Goroutine。

## 三、vm注意事项

单个envoy会启动多个vm（测试时2个），vm之间数据一般是隔离；如需共享数据（如进程内计数），需要使用proxywasm共享数据api（proxywasm.GetSharedData、proxywasm.SetSharedData）。

## 测试环境部署

系统centos7.9
Sdk库：proxy-wasm-go-sdk（使用v0.14.0，使用master版本会有问题）
https://github.com/tetratelabs/proxy-wasm-go-sdk
需要安装编译工具tinygo（使用version: 0.20.0，目前测试部分go 网络协议库出现编译问题）

[src: raw/ingested/3项目/服务网格-即构/zg_wasm技巧.md]