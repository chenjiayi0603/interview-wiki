# Cpp Wasm 开发

## istio 拓展wasm的原理介绍

https://istio.io/latest/docs/concepts/wasm/

envoy周期调用过滤器时，会基于事件驱动触发以下类型的接口，这些也是envoy对外wasm提供的api。
- Filter Service Provider Interface (SPI) for building Proxy-Wasm plugins for filters.
- Sandbox V8 Wasm Runtime embedded in Envoy.
- Host APIs for headers, trailers and metadata.
- Call out APIs for gRPC and HTTP calls.
- Stats and Logging APIs for metrics and monitoring.

目前自定义wasm使用v8引擎来连接envoy。v8引擎：V8 JavaScript engine
其他方式参考：Wasm — envoy 1.21.0-dev-b1219e documentation (envoyproxy.io)
官方有的wasm插件是直接编译进envoy来运行，参考配置https://github.com/istio/proxy/blob/master/extensions/stats/testdata/server.yaml 参考编译https://github.com/istio/proxy/blob/master/extensions/stats/BUILD

目前使用的wasm cpp 开发sdk，参考 https://github.com/proxy-wasm/proxy-wasm-cpp-sdk
wasm cpp sdk原理，参考：https://zhuanlan.zhihu.com/p/339498540

proxy wasm host 接口，参考 https://github.com/proxy-wasm/proxy-wasm-cpp-host/blob/master/include/proxy-wasm/exports.h
host接口调用的envoy的接口 参考：https://github.com/envoyproxy/envoy/blob/main/source/extensions/common/wasm/context.h

host 接口如下：
- Wasm::Context：封装 HTTP/Network Filter 接口使得 Envoy 上层可以将其嵌入到插件链中。同时提供了对 Envoy API 的具体实现。封装了请求上下文。继承自 proxy_wasm::ContextBase。
- proxy_wasm::ContextBase：封装 proxy_wasm::WasmBase 中绑定的 WASM 沙箱 API。
- Wasm::Wasm：对 WASM 沙箱的上层封装，继承自 proxy_wasm::WasmBase。相比于其基类，增加了 Envoy 相关的一些状态，如 stats 指标监控，日志以及一些全局的 API 如 Cluster Manager 等。
- proxy_wasm::WasmBase：proxy_wasm 中对 WASM 沙箱的封装。通过组合管理 proxy_wasm::WasmVM。绑定 WASM 沙箱 API 以提供给 proxy_wasm::ContextBase 调用。
- proxy_wasm::WasmVM：对不同类型 WSAM runtime 的封装，暴露处统一的对外接口，如注册 API，获取 API 等。
- proxy_wasm::exports：名称空间。其中包含所有 Envoy 提供给 WASM 沙箱的 Envoy API 函数。

proxy-wasm-cpp-sdk 是声明和调用proxy wasm host 接口，或者作为回调被调用；也做一些封装提供api给wasm filter。

proxy-wasm-cpp-sdk 接口说明：
接口文档： https://github.com/proxy-wasm/proxy-wasm-cpp-sdk/blob/master/docs/wasm_filter.md
接口源文件：
对上声明对envoy api的引用接口：https://github.com/proxy-wasm/proxy-wasm-cpp-sdk/blob/master/proxy_wasm_externs.h
对下wasm filter提供的sandbox api具体接口实现：https://github.com/proxy-wasm/proxy-wasm-cpp-sdk/blob/master/proxy_wasm_api.h

cpp proxy wasm 使用emscripten的前端编译器，功能上使用已封装好的envoy api，也就是cpp proxy sdk api，和一些c++17 stl库、c库功能；但不能使用emscripten中的c++嵌入js的方法，这个跟nodejs服务上运行wasm不同；也不能调用系统linux指令，和不同直接访问系统文件，它有单独虚拟文件系统，但没有访问宿主文件系统的api，而nodejs服务能通过c++嵌入js的方法来访问系统文件。

## 准备环境

测试环境系统：CentOS Linux release 7.9.2009 (Core)
安装软件：
gcc （v4.8.5，系统自带）

下载cpp proxy sdk （目前使用master版本）
git clone https://github.com/proxy-wasm/proxy-wasm-cpp-sdk

安装protobuf（参考https://github.com/proxy-wasm/proxy-wasm-cpp-sdk/blob/master/sdk_container.sh）
git clone https://github.com/protocolbuffers/protobuf
cd protobuf
git checkout v3.9.1
git submodule update --init --recursive
./autogen.sh
./configure
make
make install

安装emscripten（编译工具链，官网https://emscripten.org/index.html）
git clone https://github.com/emscripten-core/emsdk.git
cd emsdk
./emsdk update-tags
./emsdk install 2.0.7
./emsdk activate 2.0.7
source ./emsdk_env.sh

[src: raw/ingested/3项目/服务网格-即构/zg_wasm技巧.md]