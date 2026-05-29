# QPS 上不去（CPU 满）

## CPU 与算法

- 选择更优数据结构（cache 友好：数组/struct of arrays）、降低分支预测失败（去虚函数、少 if 长链、使用查表）。
- 向量化（SIMD intrinsics/auto-vectorization），编译器优化选项（-O2/-O3，-march=native，注意与调试符号的取舍）。
- 预分配/批处理/零拷贝（string_view、span、std::pmr）、减少拷贝和序列化开销。

**举例**：

- if-else 派发改查表：
  ```cpp
  using fn_t = void(*)(...);
  fn_t fns[256];
  fns[type]();
  ```
- AoS 改 SoA + SIMD：
  ```cpp
  // before: struct Point3d { double x,y,z; }; vector<Point3d>
  // after:
  vector<double> xs, ys, zs;
  // 用 _mm256_add_pd 做 batch-wise 加速
  ```
- 解析字符串用 string_view：
  ```cpp
  string_view s(payload + off, len);
  ```
- 批处理：
  ```cpp
  vector<Request> batch; // 一次性解析和处理一批
  ```
- g++ 编译带优化：
  ```
  g++ -O3 -march=native main.cpp -o main
  ```

**分析**：CPU 打满时优先看热点函数与数据布局；cache 友好与分支优化往往比"换更猛算法"收益更快。

## Cache 友好数据结构与分支预测

**要点**：CPU 与算法层首先看数据布局和分支。**Cache 友好**：优先连续数组、struct of arrays（SoA）便于向量化与预取，避免指针追逐和随机访问。**分支预测**：虚函数调用、长 if-else 链易导致分支失败；可减少虚调用、用查表替代多分支、对热点路径做 branchless 写法。

**举例**：

- SoA:
  ```cpp
  vector<double> xs, ys, zs;
  // 注意：for(size_t i=0; i<N; ++i) sum += xs[i] + ys[i] + zs[i]; 这种写法本身只是三数组相加，并不能自动提高 cache 命中或更好地利用预取，SoA 布局只是为 cache 友好和向量化创造了条件，实际收益依赖后续向量化（自动或显式 SIMD）。单纯这样遍历，cache line 可能各自预取 xs/ys/zs，但没有额外提升。
  // 这种写法利用 SoA 布局，同一分量（如 xs）在内存中连续存放，CPU 访问更 cache 友好，有利于进行批量预取和提升缓存命中率，同时也便于被编译器自动向量化（SIMD）加速处理。
  // SIMD 指令示例（手写 AVX 向量化）：
  // #include <immintrin.h>
  // for(size_t i=0; i+3<N; i+=4) {
  //   __m256d vx = _mm256_loadu_pd(&xs[i]); // _mm256_loadu_pd 表示“un-aligned load”，即从任意内存地址读取 4 个 double（256 bit），放进 AVX 寄存器 vx；相比 _mm256_load_pd 不要求数据地址按 32 字节对齐，适用更广，可能稍慢一点。
  //   __m256d vy = _mm256_loadu_pd(&ys[i]);
  //   __m256d vz = _mm256_loadu_pd(&zs[i]);
  //   __m256d vsum = _mm256_add_pd(vx, _mm256_add_pd(vy, vz));
  //   // 累加 vsum 到 sum
  //   double buf[4];
  //   _mm256_storeu_pd(buf, vsum);      // 将寄存器中的 4 个 double 值存到 buf
  //   for(int j=0; j<4; ++j) sum += buf[j]; // 把本轮向量加和累加进总 sum
  // }
  // 这样 SoA+SIMD 可以显著提升内存带宽利用率和计算吞吐量，远高于传统指针跳跃（AoS）方式。
  ```
- 分支查表派发：
  ```cpp
  static fn_t handlers[256]; handlers[msg.type]();
  ```
- 禁用虚函数 run-time dispatch:
  ```cpp
  template<int type> void handle() { ... }
  // if constexpr 分支在编译期分发
  ```

- branch predictor miss 性能监控:
  ```
  # 统计分支指令数及分支预测失败的次数与比例，辅助分析分支预测对性能的影响
  perf stat -e branches,branch-misses ./myservice
  ```

## 向量化与编译器优化

**要点**：**向量化**：显式 SIMD（SSE/AVX intrinsics）或依赖编译器 auto-vectorization（循环结构简单、无数据依赖时效果好）。**编译选项**：`-O2`/`-O3` 开启优化，`-march=native` 使用本机指令集；注意 `-O3` 与调试符号可能拉长链路，发布构建可剥离符号；LTO/PGO 需单独验证稳定性。

**举例**：

- 显式 SIMD：
  ```cpp
  #include <immintrin.h>
  __m256d x = _mm256_load_pd(arr);
  __m256d y = _mm256_load_pd(brr);
  __m256d z = _mm256_add_pd(x, y);
  _mm256_store_pd(crr, z);
  ```
- 自动向量化和并行循环：
  ```cpp
  #pragma omp simd
  for(int i=0; i<N; ++i) arr[i] += 1.0;
  ```
- 编译开启优化与 native 指令集:
  ```
  g++ -O3 -march=native myfile.cpp
  ```

- 单独函数向量化：
  ```cpp
  __attribute__((target("avx2")))
  void myfunc(...) { ... }
  ```

## 预分配、批处理与零拷贝

**要点**：减少临时分配和拷贝：**预分配**（启动时或首次使用时分配好 buffer/容器容量）、**批处理**（多条消息/请求一起处理，摊薄单次开销）、**零拷贝与轻量视图**（`std::string_view`、`std::span` 避免拷贝字符串和数组；`std::pmr` 与 monotonic_buffer_resource 做短生命周期批量分配）。序列化尽量选零拷贝或低拷贝格式（如 flatbuffers/capnproto）。

**举例**：

- std::vector<typename>::reserve:
  ```cpp
  vector<T> v; v.reserve(10000);
  ```
- batch:
  ```cpp
  for(int i=0;i<batch_size;++i) process(recv_msgs[i]);
  ```
- string_view 零拷贝字符串解析:
  ```cpp
  string_view key = string_view(payload + off, len);
  ```
- pmr:
  ```cpp
  std::pmr::monotonic_buffer_resource pool(65536);
  std::pmr::vector<T> v{&pool};
  ```
- flatbuffers 解析直接用 buffer:
  ```cpp
  auto user = GetUser(buffer.data());
  user->id();
  ```

## 序列化与 TLS 优化

**要点**：**序列化**：protobuf 体积和解析开销较小；flatbuffers/capnproto 支持零拷贝解析，适合高性能路径。**压缩**：只在带宽或存储是瓶颈时开启，CPU 与延迟会上升。**TLS**：会话复用（session ticket 或 session id）可避免每次建连的完整握手，显著降低 HTTPS 延迟。

**举例**：

- protobuf 序列化：
  ```cpp
  msg.SerializeToArray(buf, bufsize);
  msg.ParseFromArray(buf, len);
  ```
- flatbuffers 直接访问字段
  ```cpp
  auto obj = GetUser(buf); obj->id();
  ```
- zlib 压缩：
  ```cpp
  compress(dest, &destLen, src, srcLen);
  ```
- openssl session 复用:
  ```
  openssl s_client -reconnect -sess_out session.pem ...
  ```
- nginx / OpenResty 配置 SSL session_cache

[src: raw/ingested/2技术/性能优化/瓶颈-瓶颈症状与治理手段-QPS-上不去（CPU-满）.md]