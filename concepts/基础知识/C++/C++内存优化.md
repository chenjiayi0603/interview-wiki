# C++ 内存优化

## 3. 内存优化

### 3.1 内存分配优化

#### 3.1.1 减少动态分配

```cpp
// ❌ 频繁分配
void process_data() {
    for (int i = 0; i < 1000; ++i) {
        std::vector<int> temp;  // 每次循环都分配
        // ...
    }
}

// ✅ 复用内存
void process_data() {
    std::vector<int> temp;  // 只分配一次
    temp.reserve(1000);     // 预分配容量
    for (int i = 0; i < 1000; ++i) {
        temp.clear();       // 清空但不释放内存
        // ...
    }
}
```

#### 3.1.2 预分配和预留容量

```cpp
// ❌ 动态增长，多次重分配
std::vector<int> vec;
for (int i = 0; i < 1000000; ++i) {
    vec.push_back(i);  // 可能触发多次重分配
}

// ✅ 预分配容量
std::vector<int> vec;
vec.reserve(1000000);  // 一次性分配
for (int i = 0; i < 1000000; ++i) {
    vec.push_back(i);  // 无重分配
}

// ✅ 使用数组（栈分配）
int arr[1000];  // 栈分配，速度快

// ✅ 使用 std::array（栈分配）
std::array<int, 1000> arr;
```

#### 3.1.3 内存池

```cpp
// 简单的内存池实现
class MemoryPool {
private:
    struct Block {
        Block* next;
    };
    
    Block* free_list_;
    size_t block_size_;
    size_t pool_size_;
    char* pool_;
    
public:
    MemoryPool(size_t block_size, size_t num_blocks)
        : block_size_(block_size), pool_size_(block_size * num_blocks) {
        pool_ = new char[pool_size_];
        free_list_ = nullptr;
        
        // 初始化空闲列表
        for (size_t i = 0; i < num_blocks; ++i) {
            Block* block = reinterpret_cast<Block*>(pool_ + i * block_size);
            block->next = free_list_;
            free_list_ = block;
        }
    }
    
    ~MemoryPool() {
        delete[] pool_;
    }
    
    void* allocate() {
        if (free_list_ == nullptr) {
            return nullptr;  // 池已满
        }
        Block* block = free_list_;
        free_list_ = free_list_->next;
        return block;
    }
    
    void deallocate(void* ptr) {
        Block* block = static_cast<Block*>(ptr);
        block->next = free_list_;
        free_list_ = block;
    }
};

// 使用示例
MemoryPool pool(sizeof(MyClass), 1000);
MyClass* obj = new(pool.allocate()) MyClass();
// ...
obj->~MyClass();
pool.deallocate(obj);
```

#### 3.1.4 对象池

```cpp
// 对象池模板
template<typename T>
class ObjectPool {
private:
    std::vector<std::unique_ptr<T>> pool_;
    std::queue<T*> available_;
    
public:
    ObjectPool(size_t size) {
        pool_.reserve(size);
        for (size_t i = 0; i < size; ++i) {
            auto obj = std::make_unique<T>();
            available_.push(obj.get());
            pool_.push_back(std::move(obj));
        }
    }
    
    T* acquire() {
        if (available_.empty()) {
            return nullptr;
        }
        T* obj = available_.front();
        available_.pop();
        return obj;
    }
    
    void release(T* obj) {
        available_.push(obj);
    }
};
```

### 3.2 内存对齐优化

#### 3.2.1 结构体对齐

```cpp
// ❌ 未优化的结构体（24字节）
struct BadStruct {
    char a;      // 1字节，补7字节
    double b;    // 8字节
    char c;      // 1字节，补7字节
    double d;    // 8字节
};

// ✅ 优化的结构体（16字节）
struct GoodStruct {
    double b;    // 8字节
    double d;    // 8字节
    char a;      // 1字节
    char c;      // 1字节
    // 总共16字节，节省8字节
};

// 使用 alignas 指定对齐
struct alignas(64) CacheLineAligned {
    int data[16];  // 对齐到缓存行（64字节）
};
```

#### 3.2.2 缓存行对齐

```cpp
// False Sharing 问题
struct Counter {
    int count1;  // 线程1访问
    int count2;  // 线程2访问
    // 如果 count1 和 count2 在同一缓存行，会导致缓存行竞争
};

// ✅ 解决：缓存行对齐
struct alignas(64) Counter {
    int count1;
    char padding[64 - sizeof(int)];  // 填充到64字节
};

struct alignas(64) Counter2 {
    int count2;
    char padding[64 - sizeof(int)];
};
```

### 3.3 内存访问模式优化

#### 3.3.1 缓存友好的访问模式

```cpp
// ❌ 缓存不友好：随机访问
std::unordered_map<int, int> map;
for (int i = 0; i < n; ++i) {
    map[i] = i;  // 哈希表导致随机内存访问
}

// ✅ 缓存友好：顺序访问
std::vector<int> vec(n);
for (int i = 0; i < n; ++i) {
    vec[i] = i;  // 顺序访问，缓存命中率高
}

// ❌ 二维数组列优先访问
int arr[1000][1000];
for (int j = 0; j < 1000; ++j) {
    for (int i = 0; i < 1000; ++i) {
        arr[i][j] = i + j;  // 列优先，缓存不友好
    }
}

// ✅ 二维数组行优先访问
for (int i = 0; i < 1000; ++i) {
    for (int j = 0; j < 1000; ++j) {
        arr[i][j] = i + j;  // 行优先，缓存友好
    }
}
```

#### 3.3.2 数据局部性

```cpp
// ✅ 时间局部性：重复访问相同数据
int sum = 0;
for (int i = 0; i < n; ++i) {
    sum += arr[i];  // arr[i] 可能仍在缓存中
}

// ✅ 空间局部性：访问相邻数据
for (int i = 0; i < n; ++i) {
    process(arr[i]);     // 顺序访问
    process(arr[i + 1]);  // 相邻数据可能在缓存中
}
```

### 3.4 智能指针优化

```cpp
// ❌ 不必要的 shared_ptr 拷贝
void process(std::shared_ptr<Data> data) {
    // 如果不需要共享所有权，使用引用
}

// ✅ 使用引用或 const 引用
void process(const Data& data) {
    // 避免智能指针开销
}

// ✅ 使用 unique_ptr 代替 shared_ptr（如果不需要共享）
std::unique_ptr<Data> data = std::make_unique<Data>();

// ✅ 使用 make_shared/make_unique（减少分配次数）
auto ptr = std::make_shared<Data>();  // 一次分配
// 而不是
std::shared_ptr<Data> ptr(new Data);  // 两次分配
```

[src: raw/ingested/2技术/性能优化/瓶颈-C++性能优化大厂考点-3.-内存优化.md]