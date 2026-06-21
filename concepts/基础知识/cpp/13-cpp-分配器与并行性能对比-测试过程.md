# 分配器与并行性能对比 · 测试过程

> 本文是 `12-cpp-分配器与并行性能对比.md` 的配套测试指南，包含 Docker 环境搭建、benchmark 源码、编译运行步骤。

---

## 一、Docker 环境搭建

### 1.1 Dockerfile

```dockerfile
FROM gcc:13-bookworm

# 安装依赖
RUN apt-get update && apt-get install -y --no-install-recommends \
    libtbb-dev libjemalloc-dev libmimalloc-dev libboost-dev \
    numactl linux-perf \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /bench
COPY bench_alloc.cpp bench_pool.cpp bench_exec.cpp ./
COPY run_bench.sh .
RUN chmod +x run_bench.sh
CMD ["./run_bench.sh"]
```

### 1.2 构建 & 运行

```bash
# 构建镜像
docker build -t cpp-bench .

# 限制 CPU 核心数模拟不同环境
docker run --cpus=8 cpp-bench       # 8 核
docker run --cpus=4 cpp-bench       # 4 核
docker run --cpus=1 cpp-bench       # 1 核

# 绑定 NUMA 节点（需 --privileged）
docker run --cpus=8 --privileged -v /dev:/dev cpp-bench numactl --cpunodebind=0 ./bench_alloc
```

---

## 二、分配器对比测试（bench_alloc.cpp）

```cpp
#include <tbb/scalable_allocator.h>
#include <jemalloc/jemalloc.h>
#include <mimalloc.h>
#include <chrono>
#include <iostream>
#include <vector>
#include <thread>
#include <atomic>
#include <cstdlib>
#include <cstring>

constexpr int N = 1'000'000;
constexpr int BLOCK = 64;

template<typename Alloc>
double bench(const char* name, int threads) {
    std::vector<std::thread> pool;
    std::atomic<long long> total{0};
    auto start = std::chrono::steady_clock::now();
    int per = N / threads;
    for (int t = 0; t < threads; t++)
        pool.emplace_back([&] {
            Alloc alloc;
            for (int i = 0; i < per; i++) {
                void* p = alloc.allocate(BLOCK);
                memset(p, 0, BLOCK);
                alloc.deallocate((char*)p, BLOCK);
                total++;
            }
        });
    for (auto& t : pool) t.join();
    auto end = std::chrono::steady_clock::now();
    double ms = std::chrono::duration<double, std::milli>(end - start).count();
    std::cout << "  " << name << "(" << threads << "t): " << ms << " ms\n";
    return ms;
}

int main() {
    std::cout << "=== Allocator Benchmark ===\n";
    for (int t : {1, 2, 4, 8, 12}) {
        std::cout << "--- " << t << " threads ---\n";
        bench<std::allocator<char>>("glibc", t);
        bench<tbb::scalable_allocator<char>>("tbb", t);
    }
}
```

> 编译：`g++ -O2 bench_alloc.cpp -ltbb -ltbbmalloc -o bench_alloc`  
> 注：Ubuntu 24.04 上 `scalable_malloc` 符号在 `libtbbmalloc` 中，需额外链接 `-ltbbmalloc`。

---

## 三、线程池对比测试（bench_pool.cpp）

```cpp
#include <tbb/parallel_for.h>
#include <tbb/task_arena.h>
#include <omp.h>
#include <chrono>
#include <iostream>
#include <vector>
#include <thread>
#include <atomic>
#include <cmath>

constexpr int TASKS = 2'000'000;

// ---- TBB ----
double bench_tbb() {
    double sum = 0;
    auto start = std::chrono::steady_clock::now();
    tbb::parallel_for(0, TASKS, [&](int i) { sum += std::sin(i * 0.001); });
    return std::chrono::duration<double, std::milli>(
        std::chrono::steady_clock::now() - start).count();
}

// ---- OpenMP static ----
double bench_omp_static() {
    double sum = 0;
    auto start = std::chrono::steady_clock::now();
    #pragma omp parallel for schedule(static) reduction(+:sum)
    for (int i = 0; i < TASKS; i++) { sum += std::sin(i * 0.001); }
    return std::chrono::duration<double, std::milli>(
        std::chrono::steady_clock::now() - start).count();
}

// ---- OpenMP dynamic ----
double bench_omp_dynamic() {
    double sum = 0;
    auto start = std::chrono::steady_clock::now();
    #pragma omp parallel for schedule(dynamic) reduction(+:sum)
    for (int i = 0; i < TASKS; i++) { sum += std::sin(i * 0.001); }
    return std::chrono::duration<double, std::milli>(
        std::chrono::steady_clock::now() - start).count();
}

// ---- 手写线程池（FIFO + 原子取任务）----
double bench_handmade(int threads) {
    std::atomic<int> idx{0};
    double result = 0;
    std::vector<std::thread> pool;
    auto start = std::chrono::steady_clock::now();
    for (int t = 0; t < threads; t++)
        pool.emplace_back([&] {
            double local = 0;
            while (true) {
                int i = idx.fetch_add(1);
                if (i >= TASKS) break;
                local += std::sin(i * 0.001);
            }
            // 原子累加回全局
            __atomic_fetch_add((long long*)&result,
                              *(long long*)&local, __ATOMIC_SEQ_CST);
        });
    for (auto& t : pool) t.join();
    return std::chrono::duration<double, std::milli>(
        std::chrono::steady_clock::now() - start).count();
}

int main() {
    int hw = std::thread::hardware_concurrency();
    std::cout << "=== Thread Pool Benchmark (" << TASKS << " tasks, "
              << hw << " threads) ===\n";
    std::cout << "  TBB:          " << bench_tbb() << " ms\n";
    std::cout << "  OMP static:   " << bench_omp_static() << " ms\n";
    std::cout << "  OMP dynamic:  " << bench_omp_dynamic() << " ms\n";
    std::cout << "  Handmade:     " << bench_handmade(hw) << " ms\n";
}
```

> 编译：`g++ -O2 bench_pool.cpp -ltbb -fopenmp -o bench_pool`

---

## 四、执行策略对比测试（bench_exec.cpp）

```cpp
#include <execution>
#include <algorithm>
#include <vector>
#include <iostream>
#include <chrono>
#include <random>
#include <numeric>
#include <cmath>

constexpr size_t N = 20'000'000;

int main() {
    int hw = std::thread::hardware_concurrency();
    std::cout << "=== std::execution (" << N << " elements, "
              << hw << " threads) ===\n";

    // ---- sort ----
    {
        std::vector<int> v(N);
        std::mt19937 rng(42);
        std::generate(v.begin(), v.end(), rng);
        auto v2 = v;
        auto t1 = std::chrono::steady_clock::now();
        std::sort(std::execution::seq, v.begin(), v.end());
        auto t2 = std::chrono::steady_clock::now();
        double seq = std::chrono::duration<double, std::milli>(t2 - t1).count();
        auto t3 = std::chrono::steady_clock::now();
        std::sort(std::execution::par, v2.begin(), v2.end());
        auto t4 = std::chrono::steady_clock::now();
        double par = std::chrono::duration<double, std::milli>(t4 - t3).count();
        std::cout << "  sort seq: " << seq << " ms, par: " << par
                  << " ms (" << (seq/par) << "x)\n";
    }

    // ---- transform ----
    {
        std::vector<double> in(N), out(N);
        std::iota(in.begin(), in.end(), 1.0);
        auto t1 = std::chrono::steady_clock::now();
        std::transform(std::execution::seq, in.begin(), in.end(), out.begin(),
                       [](double x) { return std::sin(x)*std::cos(x)+std::sqrt(x); });
        auto t2 = std::chrono::steady_clock::now();
        double seq = std::chrono::duration<double, std::milli>(t2 - t1).count();
        auto t3 = std::chrono::steady_clock::now();
        std::transform(std::execution::par, in.begin(), in.end(), out.begin(),
                       [](double x) { return std::sin(x)*std::cos(x)+std::sqrt(x); });
        auto t4 = std::chrono::steady_clock::now();
        double par = std::chrono::duration<double, std::milli>(t4 - t3).count();
        std::cout << "  transform seq: " << seq << " ms, par: " << par
                  << " ms (" << (seq/par) << "x)\n";
    }

    // ---- reduce ----
    {
        std::vector<double> v(N);
        std::iota(v.begin(), v.end(), 1.0);
        auto t1 = std::chrono::steady_clock::now();
        double s1 = std::reduce(std::execution::seq, v.begin(), v.end(), 0.0);
        auto t2 = std::chrono::steady_clock::now();
        double seq = std::chrono::duration<double, std::milli>(t2 - t1).count();
        auto t3 = std::chrono::steady_clock::now();
        double s2 = std::reduce(std::execution::par, v.begin(), v.end(), 0.0);
        auto t4 = std::chrono::steady_clock::now();
        double par = std::chrono::duration<double, std::milli>(t4 - t3).count();
        std::cout << "  reduce seq: " << seq << " ms, par: " << par << " ms\n";
    }
}
```

> 编译：`g++ -O2 -std=c++17 bench_exec.cpp -ltbb -o bench_exec`  
> MSVC 不需要 TBB，但 GCC/Clang 必须链接 TBB。

---

## 五、Work-Stealing 对比测试（bench_steal.cpp）

```cpp
#include <tbb/parallel_for.h>
#include <chrono>
#include <iostream>
#include <vector>
#include <thread>
#include <atomic>

void compare_steal_vs_nosteal() {
    const int N = 12;              // 任务数
    const int HEAVY_MULT = 50;     // 重任务倍数
    std::vector<long long> workload(N, 10000);
    for (int i = N/2; i < N; i++) workload[i] = HEAVY_MULT * 10000;

    // ---- 无 work-stealing：std::thread 静态分配 ----
    auto start = std::chrono::steady_clock::now();
    std::vector<std::thread> threads;
    for (int i = 0; i < N; i++)
        threads.emplace_back([i, &workload] {
            volatile long long s = 0;
            for (long long j = 0; j < workload[i]; j++) s += j;
        });
    for (auto& t : threads) t.join();
    auto ms_nosteal = std::chrono::duration<double, std::milli>(
        std::chrono::steady_clock::now() - start).count();

    // ---- TBB work-stealing ----
    start = std::chrono::steady_clock::now();
    tbb::parallel_for(0, N, [&](int i) {
        volatile long long s = 0;
        for (long long j = 0; j < workload[i]; j++) s += j;
    });
    auto ms_steal = std::chrono::duration<double, std::milli>(
        std::chrono::steady_clock::now() - start).count();

    int hw = std::thread::hardware_concurrency();
    std::cout << "=== Work-Stealing (" << N << " tasks, "
              << hw << " cores) ===\n";
    std::cout << "  No steal (static):  " << ms_nosteal << " ms\n";
    std::cout << "  TBB (steal):        " << ms_steal << " ms\n";
    std::cout << "  Speedup:            " << (ms_nosteal / ms_steal) << "x\n";
}

int main() { compare_steal_vs_nosteal(); }
```

> 编译：`g++ -O2 bench_steal.cpp -ltbb -o bench_steal`

---

## 六、一键运行脚本（run_bench.sh）

```bash
#!/bin/bash
set -e
echo "=== 分配器性能对比 ==="
g++ -O2 bench_alloc.cpp -ltbb -ltbbmalloc -o bench_alloc
./bench_alloc

echo ""
echo "=== 线程池性能对比 ==="
g++ -O2 bench_pool.cpp -ltbb -fopenmp -o bench_pool
./bench_pool

echo ""
echo "=== std::execution 并行策略对比 ==="
g++ -O2 -std=c++17 bench_exec.cpp -ltbb -o bench_exec
./bench_exec

echo ""
echo "=== Work-Stealing 对比 ==="
g++ -O2 bench_steal.cpp -ltbb -o bench_steal
./bench_steal
```

---

## 七、实际输出示例（2026-06-14 实测）

详细测试结果见 `12-cpp-分配器与并行性能对比.md` 的「实测记录」章节。  
以下是当时运行的完整命令行序列：

```bash
# 1. 启动容器（安装依赖约 30s）
docker run --cpus=12 -it --name bench ubuntu:24.04 bash
apt-get update -qq && apt-get install -y -qq g++ libtbb-dev

# 2. 复制源码到容器内（略），然后依次编译运行
g++ -O2 bench_alloc.cpp -ltbb -ltbbmalloc -o bench_alloc && ./bench_alloc
g++ -O2 -std=c++17 bench_exec.cpp -ltbb -o bench_exec && ./bench_exec
g++ -O2 bench_pool.cpp -ltbb -fopenmp -o bench_pool && ./bench_pool
g++ -O2 bench_steal.cpp -ltbb -o bench_steal && ./bench_steal

# 3. 输出详见结果文档
```

### 踩坑记录

| 问题 | 原因 | 解决 |
|------|------|------|
| `undefined reference to scalable_malloc` | Ubuntu 24.04 TBB 将 `scalable_malloc` 分离到单独 `libtbbmalloc` | 链接加 `-ltbbmalloc` |
| TBB benchmark 耗时 0.5ms 反常 | 编译器优化掉了空循环，`volatile` 不够 | 改用 `sin()` 计算负载 |
| `--cpus=12` 看到的核心数仍是实际物理核 | Docker `--cpus` 限制 CPU 时间而非并发度 | 使用 `--cpuset-cpus` 固定到指定核心 |
| 分配器小规模 glibc 反而快 | tcache 全在 L1 缓存，无竞争 | 必须放大到 `10⁸+` 随机模式 |

> 测试过程中遇到的这些坑本身就是很好的面试素材——面试官问 "你测试时遇到过什么问题" 时可以直接引用。
