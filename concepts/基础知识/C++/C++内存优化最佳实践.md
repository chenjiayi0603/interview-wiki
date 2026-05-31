# C++ 内存优化最佳实践

## 3. 内存优化最佳实践

[来源：低延迟C++系统分析.md + 瓶颈症状与治理手段.md]

### 3.1 内存池与对象池

**原理**：预分配内存，避免频繁的`malloc/free`系统调用。

```cpp
// 简单对象池实现（线程不安全，示意代码）
class MyObjPool {
    static const size_t OBJ_SIZE = 64;
    static const size_t OBJ_COUNT = 1024;
    char*  buf_;
    void*  free_list_;

public:
    MyObjPool() {
        buf_ = new char[OBJ_SIZE * OBJ_COUNT];
        free_list_ = nullptr;
        for (size_t i = 0; i < OBJ_COUNT; ++i) {
            void* slot = buf_ + i * OBJ_SIZE;
            *reinterpret_cast<void**>(slot) = free_list_;
            free_list_ = slot;
        }
    }
    ~MyObjPool() { delete[] buf_; }

    void* alloc() {
        if (!free_list_) return nullptr;
        void* obj = free_list_;
        free_list_ = *reinterpret_cast<void**>(free_list_);
        return obj;
    }
    
    void free(void* p) {
        *reinterpret_cast<void**>(p) = free_list_;
        free_list_ = p;
    }
};
```

**优势**：通常比malloc/free快10-100倍，基本无碎片。

### 3.2 缓存行对齐与False Sharing避免

**问题**：多个线程访问同一缓存行的不同变量，导致缓存行在CPU间频繁同步。

```cpp
// 问题代码：False Sharing
struct Counter {
    int count1;  // 线程1频繁修改
    int count2;  // 线程2频繁修改
    // 两者在同一缓存行（64字节），导致性能下降
};

// 优化：缓存行对齐
struct alignas(64) AlignedCounter {
    int count;
    char padding[64 - sizeof(int)];  // 填充到64字节
};

struct OptimizedCounters {
    AlignedCounter counter1;  // 独占一个缓存行
    AlignedCounter counter2;  // 独占另一个缓存行
};
```

### 3.3 热冷数据分离与SoA布局

**原理**：将频繁访问的数据（热数据）和偶尔访问的数据（冷数据）分离，提高缓存命中率。

```cpp
// 优化前：热冷数据混合
struct Entity {
    int id;                    // 热数据：频繁访问
    float position[3];         // 热数据：频繁访问
    std::string description;   // 冷数据：偶尔访问
    std::vector<std::string> tags;  // 冷数据：偶尔访问
};

// 优化后：热冷数据分离
struct EntityHot {
    int id;
    float position[3];
    // 只包含热数据，提高缓存命中率
};

struct EntityCold {
    std::string description;
    std::vector<std::string> tags;
    // 冷数据单独存储
};

// 使用SoA（Structure of Arrays）布局
struct EntitySystem {
    std::vector<int> ids;              // 所有实体的ID
    std::vector<float> positions_x;    // 所有实体的X坐标
    std::vector<float> positions_y;    // 所有实体的Y坐标
    std::vector<float> positions_z;    // 所有实体的Z坐标
    // 顺序访问时缓存友好
};
```

### 3.4 内存预取（Prefetching）

**原理**：提前将数据加载到缓存，减少内存访问延迟。

```cpp
#include <xmmintrin.h>

// 软件预取
void process_array(int* data, size_t n) {
    for (size_t i = 0; i < n; ++i) {
        // 预取未来要访问的数据（提前8个元素）
        if (i + 8 < n) {
            _mm_prefetch((char*)&data[i + 8], _MM_HINT_T0);
        }
        // 处理当前数据
        process(data[i]);
    }
}

// 预取提示类型：
// _MM_HINT_T0: L1缓存（时间局部性）
// _MM_HINT_T1: L2缓存（空间局部性）
// _MM_HINT_T2: L3缓存
// _MM_HINT_NTA: 非临时访问（不污染缓存）
```

### 3.5 自定义分配器

**问题**：默认glibc malloc在高并发、多尺寸分配下容易碎片和锁争用。

**解决方案**：
```bash
# 使用jemalloc
LD_PRELOAD=/usr/lib/libjemalloc.so ./myservice

# 使用mimalloc
g++ main.cpp -o myapp -lmimalloc

# 使用tcmalloc
-ltcmalloc
```

**jemalloc统计**：
```cpp
// 查询碎片率等统计信息
mallctl("stats.arenas.0.small.allocated", ...);
mallctl("stats.arenas.0.muzzy_purged", ...);
```

### 3.6 内存优化 - GitHub项目实践案例

#### 3.6.1 jemalloc - 高性能内存分配器
**GitHub**: https://github.com/jemalloc/jemalloc  
**Stars**: 8k+  
**简介**: 通用malloc实现，专注于减少碎片和提升并发性能

**应用场景**：高并发服务器内存分配优化

**关键代码示例**:
```cpp
// jemalloc/src/jemalloc.c
JEMALLOC_EXPORT void * JEMALLOC_NOTHROW
je_malloc(size_t size) {
    void *ret;
    tsd_t *tsd;
    
    // 线程本地缓存快速路径
    if (likely(size <= arena_maxclass)) {
        tsd = tsd_fetch();
        if (likely(tsd != NULL)) {
            // 从线程本地缓存分配
            ret = iallocztm(tsd, size, false, false, true);
            if (likely(ret != NULL)) {
                return ret;
            }
        }
    }
    // 慢速路径：全局分配
    return imalloc(size);
}
```

**技术说明**：jemalloc通过线程本地缓存（TLS）减少锁竞争，每个线程有自己的内存分配缓存，避免频繁访问全局锁。同时采用size-class分桶策略减少碎片，在高并发场景下比glibc malloc性能提升30-50%。

#### 3.6.2 RocksDB - 内存池与缓存优化
**GitHub**: https://github.com/facebook/rocksdb  
**Stars**: 27k+  
**简介**: Facebook开发的高性能嵌入式键值存储引擎

**应用场景**：数据库内存管理和缓存优化

**关键代码示例**:
```cpp
// rocksdb/memtable/arena.h
class Arena {
private:
    char* alloc_ptr_;           // 当前块分配指针
    size_t alloc_bytes_remaining_; // 当前块剩余字节
    std::vector<char*> blocks_; // 已分配的内存块
    size_t block_size_;         // 块大小
    
public:
    char* Allocate(size_t bytes) {
        // 对齐分配
        const size_t align = (bytes > 8) ? 8 : 1;
        bytes = ((bytes + align - 1) / align) * align;
        
        if (bytes <= alloc_bytes_remaining_) {
            // 快速路径：当前块有足够空间
            char* result = alloc_ptr_;
            alloc_ptr_ += bytes;
            alloc_bytes_remaining_ -= bytes;
            return result;
        }
        // 慢速路径：分配新块
        return AllocateFallback(bytes);
    }
    
    char* AllocateFallback(size_t bytes) {
        if (bytes > block_size_ / 4) {
            // 大对象直接分配独立块
            return AllocateNewBlock(bytes);
        }
        // 分配新标准块
        alloc_ptr_ = AllocateNewBlock(block_size_);
        alloc_bytes_remaining_ = block_size_;
        return Allocate(bytes);
    }
};
```

**技术说明**：RocksDB使用Arena内存池管理MemTable内存，通过预分配大块内存并按需切分，减少malloc/free调用。同时采用LRU缓存策略管理BlockCache，通过分片减少锁竞争，在SSD存储场景下实现微秒级读写延迟。

#### 3.6.3 mimalloc - 现代高性能分配器
**GitHub**: https://github.com/microsoft/mimalloc  
**Stars**: 9k+  
**简介**: Microsoft开发的高性能内存分配器，专注于低延迟和低碎片

**应用场景**：游戏服务器、实时系统内存分配优化

**关键代码示例**:
```cpp
// mimalloc/src/alloc.c
void* mi_malloc(size_t size) {
    // 快速路径：小对象分配
    if (size <= MI_SMALL_SIZE_MAX) {
        // 获取线程本地堆
        mi_heap_t* heap = mi_heap_get_default();
        // 根据大小选择size-class
        size_t idx = _mi_bin(size);
        // 从本地空闲列表分配
        mi_page_t* page = heap->pages[idx];
        if (page != NULL && page->free != NULL) {
            void* block = page->free;
            page->free = *(void**)block;
            page->used++;
            return block;
        }
        // 慢速路径：分配新页
        return mi_malloc_small(heap, size);
    }
    // 大对象分配
    return mi_malloc_large(size);
}
```

**技术说明**：mimalloc采用线程本地分配、无锁设计，每个线程有独立的内存池。通过分代垃圾回收机制自动回收空闲内存，碎片率比jemalloc低30%。在游戏服务器等高并发场景下，分配延迟比glibc malloc低60-80%。

### 3.7 避免内存分配

**策略**：
- 使用栈分配小对象
- 预分配所有资源
- 使用内存池/对象池
- 避免 `std::string` 小对象优化失效

```cpp
// 优化前：频繁堆分配
void process_message(const std::string& msg) {
    std::string buffer = "Prefix: " + msg;  // 可能触发堆分配
    // ...
}

// 优化后：预分配或栈分配
void process_message(const char* msg, size_t len) {
    thread_local char buffer[1024];  // thread_local 不是静态(static)变量，但与静态变量类似：每个线程独有 buffer（而非全局唯一），在所属线程内生命周期贯穿整个线程。它不是共享的全局静态变量，而是"线程静态变量"。
    // %.*s 是 printf 系列函数中的格式化写法，用于按照指定宽度（前面的*所给）打印字符串的前len个字符
    // 这里 (int)len 提供了需要打印的长度，只会把 msg 的前 len 个字节复制进 buffer，防止越界或多余分配
    // 如果 len 大于等于 sizeof(buffer)，snprintf 会按 (int)len 处理格式化操作，但实际拷贝到 buffer 的内容不会超过 buffer 的大小（1024 字节），因为 snprintf 本身会自动截断并保证结尾有 '\0'
    // 不过，这样输出内容会不完整（只会拷贝 buffer 能容纳的前部分内容），极端情况下甚至可能仅拷贝 "Prefix: " 和部分 msg
    // 实际使用时，建议先对 len 做边界检查，只拷贝最多 (sizeof(buffer) - 前缀长度 - 1) 个字符，避免内容截断和歧义
    snprintf(buffer, sizeof(buffer), "Prefix: %.*s", (int)len, msg);
    // ...
}
```

[src: raw/ingested/2技术/性能优化/低延迟-低延迟c++系统分析-三、内存优化策略.md]