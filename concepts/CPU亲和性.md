# CPU 亲和性（CPU Affinity）

**原理**：将线程绑定到特定的 CPU 核心，避免线程迁移带来的缓存失效和上下文切换开销。

```cpp
#include <pthread.h>
#include <sched.h>

// 设置线程 CPU 亲和性
void set_thread_affinity(int cpu_id) {
    cpu_set_t cpuset;
    CPU_ZERO(&cpuset);
    CPU_SET(cpu_id, &cpuset);
    
    pthread_t thread = pthread_self();
    pthread_setaffinity_np(thread, sizeof(cpu_set_t), &cpuset);
}

// 示例：将主线程绑定到 CPU 0
int main() {
    set_thread_affinity(0);
    // ... 业务逻辑
}
```

**C++11 方式**（需要平台特定实现）：
```cpp
#include <thread>
#include <pthread.h>

void set_cpu_affinity(std::thread& t, int cpu_id) {
    cpu_set_t cpuset;
    CPU_ZERO(&cpuset);
    CPU_SET(cpu_id, &cpuset);
    pthread_setaffinity_np(t.native_handle(), sizeof(cpu_set_t), &cpuset);
}
```

**优势**：
- 减少线程迁移开销（避免缓存失效）
- 提高缓存局部性
- 减少上下文切换
- 可预测的性能表现

[src: raw/ingested/2技术/性能优化/低延迟-低延迟c++系统分析-二、CPU-优化技术-二、CPU-优化技术.md]