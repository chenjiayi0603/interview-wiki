# 共享内存 IPC

> 共享内存 IPC 是进程间通信中最高效的方式之一，通过将同一物理内存映射到多个进程的虚拟地址空间，实现零拷贝的数据交换。

See also: [[IPC进程间通信]], [[POSIX共享内存]], [[零拷贝与异步IO]]

## 共享内存环形缓冲区

```cpp
#include <sys/shm.h>
#include <sys/mman.h>

// 共享内存环形缓冲区
class SharedMemoryRingBuffer {
    static constexpr size_t SHM_SIZE = 1024 * 1024 * 64;  // 64MB
    
    struct SharedHeader {
        alignas(64) std::atomic<uint64_t> write_seq{0};
        alignas(64) std::atomic<uint64_t> read_seq{0};
    };
    
    SharedHeader* header_;
    char* data_;
    int shm_fd_;
    
public:
    bool init(const char* name) {
        shm_fd_ = shm_open(name, O_CREAT | O_RDWR, 0666);
        ftruncate(shm_fd_, SHM_SIZE);
        
        void* ptr = mmap(nullptr, SHM_SIZE,
            PROT_READ | PROT_WRITE, MAP_SHARED, shm_fd_, 0);
        
        header_ = static_cast<SharedHeader*>(ptr);
        data_ = static_cast<char*>(ptr) + sizeof(SharedHeader);
        
        return true;
    }
    
    // 零拷贝写入
    bool write(const void* data, size_t len) {
        uint64_t seq = header_->write_seq.load(std::memory_order_relaxed);
        void* dest = data_ + (seq % (SHM_SIZE - sizeof(SharedHeader)));
        memcpy(dest, data, len);
        header_->write_seq.store(seq + len, std::memory_order_release);
        return true;
    }
};
```

[src: raw/ingested/2技术/性能优化/低延迟-高频交易系统优化技术指南-六、系统架构设计.md]