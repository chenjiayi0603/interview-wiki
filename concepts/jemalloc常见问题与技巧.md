# jemalloc 常见问题与技巧

## Q1：如何确认程序使用的是 jemalloc？

```cpp
#include <jemalloc/jemalloc.h>
#include <iostream>

int main() {
    const char* version;
    size_t sz = sizeof(version);
    mallctl("version", &version, &sz, NULL, 0);
    std::cout << "jemalloc version: " << version << std::endl;
    return 0;
}
```

或使用命令行：
```bash
# 查看程序链接的 malloc 库
ldd ./your_program | grep malloc

# 运行时检查
LD_DEBUG=libs ./your_program 2>&1 | grep jemalloc
```

```cpp
// —— 使用 heapdump 生成手动内存快照（heap dump） ——
#include <jemalloc/jemalloc.h>

// 手动 dump 内存快照（heap dump），适用于 jemalloc profiling 已经开启的程序
void heapdump(const char* filename) {
    // 触发 jemalloc profiling 的 dump，生成 heap profile 文件
    mallctl("prof.dump", NULL, NULL, (void*)&filename, sizeof(filename));
}

// 示例用法：
int main() {
    // ... 程序运行逻辑 ...

    // 需要抓取 heap 快照时调用
    heapdump("myheap_snapshot.heap");

    // ... 继续程序逻辑 ...
    return 0;
}
```
说明：调用 `heapdump("文件名")` 可即时生成当前堆内存分配快照（heap profile）；配合 jeprof 工具可追踪内存泄漏或分配热点。

**jemalloc profiling 启动和分析指令**

**1. 启动 jemalloc profiling（环境变量方式）：**
```bash
export MALLOC_CONF="prof:true,lg_prof_sample:19,prof_active:true"
# 启动你的程序
./your_program
```
- `prof:true`：打开 profiling 功能
- `lg_prof_sample:19`：采样间隔设置为 512KB（2^19 字节），可根据实际需求调整
- `prof_active:true`：程序一启动即进入 profiling 状态

**2. 运行分析期间生成 heap profile 文件（手动或定时触发）：**
- 在代码中调用 `mallctl("prof.dump", ...)` 生成 profile 文件（见上方 heapdump 示例）
- 或通过发送信号、jeprof 工具定时采集

**3. 分析 profile 文件，生成火焰图（以 jeprof+gperftools 脚本为例）：**
```bash
# 假设生成的 profile 名为 jeprof.12345.0.f.heap
jeprof --show_bytes --svg ./your_program jeprof.12345.0.f.heap > alloc.svg
# 也可生成交互式报告
jeprof --web ./your_program jeprof.12345.0.f.heap
```
`alloc.svg` 为堆分配火焰图，可用浏览器查看并定位分配热点。

---

### jemalloc profiling 常见参数说明

- `MALLOC_CONF="prof:true,prof_active:true,lg_prof_sample:20"`
- `MALLOC_CONF="background_thread:true,prof:true,lg_prof_sample:22"`

> 建议：开发或测试环境下可适当减小采样间隔（如 2^19 或 2^20），分析更细致；生产环境下可增大间隔，避免 profile 文件过大。

---

### Q2：如何降低 jemalloc 内存占用？

```bash
# 启用后台线程清理，加快内存归还
export MALLOC_CONF="background_thread:true,dirty_decay_ms:1000,muzzy_decay_ms:0"
```

### Q3：Profile 文件太大怎么办？

```bash
# 增大采样间隔（减少采样频率）
export MALLOC_CONF="prof:true,lg_prof_sample:20"  # 每 1MB 采样一次

# 只在需要时启用 profiling
export MALLOC_CONF="prof:true,prof_active:false"
# 代码中手动开启
```

### Q4：如何与 Valgrind 配合使用？

```bash
# jemalloc 与 valgrind 不兼容，需禁用 jemalloc
# 方法1：编译时不链接 jemalloc
# 方法2：运行时禁用
unset LD_PRELOAD
valgrind --leak-check=full ./your_program
```

[src: raw/ingested/2技术/性能优化/内存优化-c++性能分析-8.-常见问题与技巧.md]