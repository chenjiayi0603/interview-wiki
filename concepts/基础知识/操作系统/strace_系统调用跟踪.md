# strace - 系统调用跟踪

## 1.1 strace 简介

**strace** 是一个用于跟踪程序执行过程中系统调用的工具，可以显示程序与操作系统内核之间的交互。

**核心功能**：
- 跟踪系统调用（syscall）和信号
- 显示系统调用的参数和返回值
- 统计系统调用次数和耗时
- 分析 I/O 操作模式

**安装方法**：
```bash
# Ubuntu/Debian
sudo apt-get install strace

# CentOS/RHEL
sudo yum install strace

# 验证安装
strace --version
```

[src: raw/ingested/2技术/性能优化/内存优化-c++性能分析-strace---系统调用跟踪.md]

---

## 1.2 strace 基本用法

### 1.2.1 基础命令格式

```bash
# 跟踪新启动的程序
strace [选项] <程序> [程序参数]

# 跟踪已运行的进程
strace -p <pid> [选项]
```

### 1.2.2 常用选项

| 选项 | 说明 | 示例 |
|------|------|------|
| `-c` | 统计系统调用次数和耗时 | `strace -c ./program` |
| `-e trace=<调用>` | 只跟踪指定的系统调用 | `strace -e trace=open,read,write` |
| `-e trace=file` | 跟踪所有文件相关调用 | `strace -e trace=file ./program` |
| `-e trace=network` | 跟踪所有网络相关调用 | `strace -e trace=network ./program` |
| `-f` | 跟踪子进程 | `strace -f ./program` |
| `-p <pid>` | 跟踪指定进程 | `strace -p 1234` |
| `-o <文件>` | 输出到文件 | `strace -o trace.log ./program` |
| `-T` | 显示每个系统调用的耗时 | `strace -T ./program` |
| `-tt` | 显示时间戳（微秒精度） | `strace -tt ./program` |
| `-s <长度>` | 限制字符串参数显示长度 | `strace -s 100 ./program` |

[src: raw/ingested/2技术/性能优化/内存优化-c++性能分析-strace---系统调用跟踪.md]

---

## 1.3 strace 实战示例

### 示例 1：统计系统调用

**场景**：分析程序的系统调用分布，找出最频繁的调用。

```bash
# 统计所有系统调用
strace -c ./your_program
# 只统计系统调用时间，忽略用户空间其他时间，其它 CPU 消耗不计入 strace 统计。strace 只能看到程序和内核交互（系统调用部分）的耗时，用户代码的消耗不会被统计。
# 如果需要同时分析用户空间和内核空间的时间分布，应结合 perf、gprof 等分析工具一起使用。

# 输出示例
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 45.23    0.123456         123      1000          0 read
 30.12    0.082345          82      1000          0 write
 15.45    0.042123          42      1000          0 openat
  5.20    0.014234          14      1000          0 close
  4.00    0.010912          10      1000          0 fstat
------ ----------- ----------- --------- --------- ----------------
100.00    0.272970                   5000          0 total
```

**解读**：
- `read` 调用占用了 45.23% 的时间，是主要瓶颈
- 平均每次 `read` 耗时 123 微秒
- 总共进行了 1000 次 `read` 调用

**优化建议**：
- 考虑批量读取减少系统调用次数
- 使用更大的缓冲区
- 检查是否有不必要的频繁读取

[src: raw/ingested/2技术/性能优化/内存优化-c++性能分析-strace---系统调用跟踪.md]

---

#### 示例 2：跟踪文件操作

**场景**：分析程序的文件访问模式，排查文件 I/O 性能问题。

```bash
# 只跟踪文件相关系统调用
strace -e trace=file -T ./your_program

# 输出示例
openat(AT_FDCWD, "/etc/passwd", O_RDONLY) = 3 <0.000123>
read(3, "root:x:0:0:root:/root:/bin/bash\n"..., 4096) = 1024 <0.000045>
close(3)                                = 0 <0.000012>
openat(AT_FDCWD, "/tmp/data.txt", O_WRONLY|O_CREAT, 0644) = 3 <0.000234>
write(3, "Hello World\n", 12)          = 12 <0.000056>
close(3)                                = 0 <0.000011>
```

**解读**：
- `<0.000123>` 表示系统调用耗时（秒）
- `openat` 打开文件，返回文件描述符 3
- `read`/`write` 进行文件读写操作
- `close` 关闭文件描述符

**优化建议**：
- 如果频繁打开/关闭同一文件，考虑保持文件打开
- 使用更大的缓冲区减少 `read`/`write` 次数
- 检查是否有不必要的文件操作

[src: raw/ingested/2技术/性能优化/内存优化-c++性能分析-strace---系统调用跟踪.md]

---

#### 示例 3：跟踪网络操作

**场景**：分析网络通信性能，排查网络 I/O 问题。

```bash
# 跟踪网络相关系统调用
strace -e trace=network -T ./your_program

# 输出示例
socket(AF_INET, SOCK_STREAM, IPPROTO_TCP) = 3 <0.000123>
connect(3, {sa_family=AF_INET, sin_port=htons(8080), sin_addr=inet_addr("127.0.0.1")}, 16) = 0 <0.001234>
sendto(3, "GET / HTTP/1.1\r\nHost: localhost\r\n\r\n", 35, 0, NULL, 0) = 35 <0.000456>
recvfrom(3, "HTTP/1.1 200 OK\r\nContent-Length: 1024\r\n\r\n...", 4096, 0, NULL, NULL) = 1024 <0.002345>
close(3)                                = 0 <0.000012>
```

**解读**：
- `socket` 创建套接字
- `connect` 建立连接，耗时 1.234 毫秒
- `sendto`/`recvfrom` 进行网络数据传输
- `recvfrom` 耗时 2.345 毫秒，可能是网络延迟或服务器响应慢

**优化建议**：
- 使用连接池减少连接建立开销
- 批量发送数据减少系统调用次数
- 使用非阻塞 I/O 或异步 I/O

[src: raw/ingested/2技术/性能优化/内存优化-c++性能分析-strace---系统调用跟踪.md]

---

#### 示例 4：跟踪已运行进程

**场景**：线上服务出现性能问题，需要实时跟踪系统调用。

```bash
# 1. 找到进程 PID
ps aux | grep your_program
# 输出: user  1234  0.5  1.2  ...  ./your_program

# 2. 跟踪该进程
strace -p 1234 -c -T

# 3. 运行一段时间后按 Ctrl+C 停止，查看统计
```

**注意事项**：
- 需要 root 权限或进程所有者权限
- 跟踪会影响程序性能（约 10-20% 开销）
- 生产环境谨慎使用

[src: raw/ingested/2技术/性能优化/内存优化-c++性能分析-strace---系统调用跟踪.md]

---

#### 示例 5：分析程序启动时间

**场景**：程序启动缓慢，需要找出启动过程中的瓶颈。

```bash
# 跟踪启动过程，显示时间戳
strace -tt -T -e trace=file,process ./your_program 2>&1 | head -100

# 输出示例
10:23:45.123456 execve("./your_program", ["./your_program"], 0x7fff12345678 /* 23 vars */) = 0 <0.001234>
10:23:45.124690 openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3 <0.000123>
10:23:45.124813 read(3, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\360\3\2\0\0\0\0\0"..., 832) = 832 <0.000045>
10:23:45.125234 openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3 <0.000234>
...
```

**解读**：
- `execve` 执行程序，耗时 1.234 毫秒
- 后续是动态库加载过程
- 可以识别哪些库加载耗时较长

**优化建议**：
- 使用静态链接减少动态库加载
- 预加载常用库（LD_PRELOAD）
- 优化配置文件读取

[src: raw/ingested/2技术/性能优化/内存优化-c++性能分析-strace---系统调用跟踪.md]

---

## 1.4 strace 高级技巧

#### 1.4.1 过滤特定系统调用

```bash
# 只跟踪 open、read、write
strace -e trace=open,read,write ./program

# 排除某些系统调用
strace -e trace=\!open,read ./program  # 跟踪除 open 和 read 外的所有调用
```

#### 1.4.2 跟踪子进程

```bash
# 跟踪主进程和所有子进程
strace -f ./program

# 同时跟踪线程
strace -f -p <pid>
```

#### 1.4.3 输出到文件并分析

```bash
# 输出到文件
strace -o trace.log -c ./program

# 分析文件
grep "read" trace.log | wc -l  # 统计 read 调用次数
grep "openat" trace.log        # 查看所有文件打开操作
```

#### 1.4.4 组合使用多个选项

```bash
# 跟踪文件操作，显示时间戳和耗时，输出到文件
# 只跟踪与文件相关的系统调用，显示时间戳与耗时，输出到文件
strace -e trace=file -tt -T -o file_trace.log ./program
# 其中 trace=file 表示只跟踪所有与文件操作有关的系统调用，如 open、read、write、close、stat 等
```

[src: raw/ingested/2技术/性能优化/内存优化-c++性能分析-strace---系统调用跟踪.md]

---

## 1.5 strace 性能开销

**注意事项**：
- strace 会显著影响程序性能（通常增加 10-50% 的开销）
- 生产环境谨慎使用，建议在测试环境分析
- 可以使用 `-c` 选项快速统计，减少详细跟踪时间

**减少开销的方法**：
```bash
# 只统计不显示详细信息（开销较小）
strace -c ./program

# 只跟踪特定系统调用（减少跟踪量）
strace -e trace=open,read,write ./program
```

[src: raw/ingested/2技术/性能优化/内存优化-c++性能分析-strace---系统调用跟踪.md]