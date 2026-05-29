# C++ I/O 优化

## 7.1 文件I/O优化

### 7.1.1 缓冲优化

```cpp
// ❌ 无缓冲，每次系统调用
FILE* file = fopen("data.txt", "w");
for (int i = 0; i < 1000000; ++i) {
    fprintf(file, "%d\n", i);  // 每次都可能触发系统调用
}

// ✅ 使用缓冲
FILE* file = fopen("data.txt", "w");
setvbuf(file, nullptr, _IOFBF, 8192);  // 8KB 缓冲区
for (int i = 0; i < 1000000; ++i) {
    fprintf(file, "%d\n", i);
}
fflush(file);  // 最后刷新

// ✅ 批量写入
std::string buffer;
buffer.reserve(100000);
for (int i = 0; i < 1000000; ++i) {
    buffer += std::to_string(i) + "\n";
    if (buffer.size() > 8192) {
        fwrite(buffer.data(), 1, buffer.size(), file);
        buffer.clear();
    }
}
```

### 7.1.2 内存映射文件

```cpp
#include <sys/mman.h>
#include <fcntl.h>
#include <unistd.h>

// 使用 mmap 进行文件I/O
void read_file_mmap(const char* filename) {
    int fd = open(filename, O_RDONLY);
    size_t file_size = lseek(fd, 0, SEEK_END);
    
    void* mapped = mmap(nullptr, file_size, PROT_READ, MAP_PRIVATE, fd, 0);
    char* data = static_cast<char*>(mapped);
    
    // 直接访问内存，无需系统调用
    for (size_t i = 0; i < file_size; ++i) {
        process(data[i]);
    }
    
    munmap(mapped, file_size);
    close(fd);
}
```

## 7.2 网络I/O优化

### 7.2.1 零拷贝

```cpp
// 使用 sendfile（Linux）
#include <sys/sendfile.h>

int send_file(int socket_fd, int file_fd, off_t offset, size_t count) {
    return sendfile(socket_fd, file_fd, &offset, count);
}

// 使用 splice（Linux）
#include <fcntl.h>
ssize_t splice(int fd_in, loff_t* off_in, int fd_out, loff_t* off_out,
               size_t len, unsigned int flags);
```

### 7.2.2 批量操作

```cpp
// ❌ 逐个发送
for (const auto& item : items) {
    send(socket, item.data(), item.size(), 0);
}

// ✅ 批量发送
std::string buffer;
for (const auto& item : items) {
    buffer += item;
}
send(socket, buffer.data(), buffer.size(), 0);

// ✅ 使用 writev（分散写入）
#include <sys/uio.h>
struct iovec iov[3];
iov[0].iov_base = header;
iov[0].iov_len = header_size;
iov[1].iov_base = body;
iov[1].iov_len = body_size;
iov[2].iov_base = footer;
iov[2].iov_len = footer_size;
writev(socket, iov, 3);
```

[src: raw/ingested/2技术/性能优化/瓶颈-C++性能优化大厂考点-7.-I-O优化.md]