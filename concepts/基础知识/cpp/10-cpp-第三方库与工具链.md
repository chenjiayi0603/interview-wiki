# 第三方库与工具链

> C 标准库实现、常用第三方库、包管理器。

---

## 一、C 标准库实现视角

### 1.1 总体结构

C 标准库本质是目标文件 + 汇编优化 + 启动代码（crt*.o）+ 头文件的集合，打包为 `libc.a` / `libc.so`。

### 1.2 malloc/free 实现思路

```text
调用 malloc(size)
  → 参数检查 / 对齐 size
  → 在空闲链表 / bins 中查找（fast bins → small bins → large bins → top chunk）
  → 找到：从空闲块取出，必要时分裂
  → 未找到：通过 brk() 或 mmap() 向内核申请新内存
  → 返回数据区指针
```

**ptmalloc 层次**：tcache → fastbin → smallbin → largebin → top chunk（参见《内存》专题）。

### 1.3 printf/scanf 实现思路

```
printf("fmt", ...)
  → 解析格式字符串
  → 将参数转为字符串
  → 写入 stdio 缓冲区（用户态缓冲）
  → 缓冲区满或遇到 \n（行缓冲）→ 调用 write() 系统调用
```

### 1.4 程序启动流程

```
_start → __libc_start_main → 初始化 libc → 调用 main(argc, argv)
                                          → exit() → atexit 回调 → _exit()
```

---

## 二、常用第三方库（后台服务方向）

### 2.1 HTTP / Web 框架

| 库 | 定位 | 适用场景 |
|----|------|----------|
| **Drogon** | 现代 C++ Web 框架（全家桶） | HTTP/WebSocket 微服务，内置 ORM |
| **Crow** | 轻量 Web 框架（类 Flask） | 小型工具、监控面板 |
| **Boost.Beast** | HTTP/WebSocket 协议库 | 需完全掌控网络层的高性能服务 |

```cpp
// Drogon 示例
#include <drogon/drogon.h>
int main() {
    drogon::app()
        .setLogPath("./logs")
        .addListener("0.0.0.0", 8080)
        .run();
}
```

### 2.2 序列化

| 库 | 特点 |
|----|------|
| **Protobuf** | 二进制序列化，强类型，gRPC 标配 |
| **nlohmann/json** | 现代 C++ JSON，header-only，易用性好 |
| **JSON for Modern C++** | 同上，广泛使用 |

### 2.3 日志

| 库 | 特点 |
|----|------|
| **spdlog** | 现代 C++ 日志库，header-only，性能高 |
| **glog** | Google 日志库，成熟稳定 |

### 2.4 测试

| 库 | 特点 |
|----|------|
| **Google Test** | C++ 测试框架，功能完整 |
| **Benchmark** | Google 微基准测试库 |

### 2.5 网络与异步

| 库 | 特点 |
|----|------|
| **Boost.Asio** | 跨平台异步 I/O，epoll/kqueue/IOCP |
| **libcurl** | HTTP 客户端，稳定成熟 |

---

## 三、包管理器

### vcpkg（推荐）

```bash
git clone https://github.com/Microsoft/vcpkg.git
cd vcpkg && ./bootstrap-vcpkg.sh

./vcpkg install boost
./vcpkg install fmt spdlog

# CMake 集成
cmake .. -DCMAKE_TOOLCHAIN_FILE=/path/to/vcpkg/scripts/buildsystems/vcpkg.cmake
```

### Conan

```bash
pip install conan
# conanfile.txt
[requires]
boost/1.75.0
fmt/10.0.0

[generators]
CMakeDeps
CMakeToolchain

conan install . --build=missing
```

### CMake FetchContent（零额外工具）

```cmake
include(FetchContent)
FetchContent_Declare(
    googletest
    GIT_REPOSITORY https://github.com/google/googletest.git
    GIT_TAG v1.11.0
)
FetchContent_MakeAvailable(googletest)
```

---

## 四、依赖选型总结

| 需求 | 推荐库 |
|------|--------|
| HTTP 服务端 | Drogon / Crow / Boost.Beast |
| 序列化 | Protobuf / nlohmann-json |
| 日志 | spdlog / glog |
| 测试 | Google Test |
| 基准测试 | Google Benchmark |
| 异步网络 | Boost.Asio |
| HTTP 客户端 | libcurl |

---

## 五、第三方库选型原则

### 5.1 评估标准

| 维度 | 考量点 |
|------|--------|
| **许可协议** | MIT/BSD/Apache 宽松，GPL/LGPL 有传染性 |
| **依赖规模** | Header-only 最佳（nlohmann-json、spdlog），否则看链接和构建复杂度 |
| **活跃度** | GitHub stars + issue 响应 + 最近更新（3 个月内最佳） |
| **ABI 兼容性** | 库版本升级是否需要重新编译所有依赖 |
| **交叉编译** | 是否支持目标平台（ARM、RISC-V、嵌入式） |

### 5.2 版本管理策略

- **vcpkg**：Windows 首选，与 CMake 集成好，manifest mode 锁定版本
- **Conan**：跨平台，企业级，支持自定义频道和私有仓库
- **FetchContent**：零额外工具，适合小项目，每次 cmake configure 重新下载
- **Git Submodule**：简单但易过期，需手动更新

### 5.3 面试追问

**Q：vcpkg vs Conan 选谁？**
- vcpkg：简单易用，Windows 生态好，ports 覆盖广
- Conan：跨平台统一，支持复杂依赖图，企业私有仓库
- **小团队/个人项目** → vcpkg；**大企业多平台** → Conan

**Q：为什么避免使用动态链接库？**
- 启动慢（.so 解析开销）、部署复杂（需携带 .so）、依赖地狱
- **低延迟系统推荐静态链接**，减少启动抖动
