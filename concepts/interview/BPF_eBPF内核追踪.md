# BPF/eBPF内核追踪

## BPF/eBPF内核追踪

### Q20: 如何使用BPF/eBPF进行内核追踪？

**答案：**

```bash
# 1. 使用bpftrace（简单场景）
bpftrace -e 'tracepoint:syscalls:sys_enter_sendto { @[comm] = count(); }'

# 2. 使用BCC工具
# 追踪TCP连接
./tcpconnect.py

# 追踪TCP重传
./tcpretrans.py

# 3. 使用systemtap（复杂场景）
stap -e 'probe syscall.sendto { printf("%s\n", execname()); }'
```

**BPF程序示例：**
```c
// trace_tcp_send.c
#include <uapi/linux/bpf.h>
#include <bpf/bpf_helpers.h>

SEC("tracepoint/tcp/tcp_sendmsg")
int trace_tcp_send(void *ctx) {
    // 追踪TCP发送
    return 0;
}

char LICENSE[] SEC("license") = "GPL";
```

### Q20.1: trace_tcp_send.c 的编译和使用方法

**环境要求说明：**

1. **Kernel版本要求**  
   eBPF追踪与tracepoint挂载需Linux内核4.9或更高版本。推荐使用内核5.x及以上，获得更丰富的BPF特性与性能。

2. **所需工具包/依赖**  
   - `clang` 和 `llvm` (通常版本7以上，用于BPF程序编译)
   - `bpftool` (用于加载/卸载eBPF程序)
   - `bcc`/`bpftrace`（用于更高级追踪与脚本，部分功能需Python3支持）
   - 内核头文件（如 `/lib/modules/$(uname -r)/build/include`）

   Ubuntu/Debian 可通过以下命令安装：
   ```bash
   sudo apt-get install clang llvm libbpf-dev linux-headers-$(uname -r) bpftool bpfcc-tools bpftrace
   ```

3. **开启BPF功能支持**  
   - 内核需开启`CONFIG_BPF`、`CONFIG_BPF_SYSCALL`，绝大多数主流Linux发行版默认已开启。
   - 挂载 BPF 虚拟文件系统（如未自动挂载）：
     ```bash
     sudo mount -t bpf bpf /sys/fs/bpf/
     ```

4. **权限要求**  
   加载和挂载eBPF程序需root权限。

5. **网络追踪建议**  
   - 生产高频交易主机通常启用`kernel.perf_event_paranoid ≤ 1`，以允许更细粒度追踪；
   - 某些防护安全策略下，需确保没有 SELinux/AppArmor 拦截内核追踪行为。

6. **可选：开发/调试**  
   - 推荐使用 [libbpf-bootstrap](https://github.com/libbpf/libbpf-bootstrap) 作为用户态模板，可快速验证自定义tracepoint BPF程序与数据采集方案；
   - 用 `bpftrace` 实时调试、验证 tracepoint 表达式及输出。

**总结：**  
高频交易C++开发者在生产系统用BPF/eBPF做追踪，需保证内核新、工具全、有root权限，以及安装好必要头文件和用户态加载工具。建议开发调试阶段用容器或测试机，严禁在运营主机无备份直接装载高风险eBPF程序。

**1. 编写 eBPF 程序（trace_tcp_send.c）：**
```c
// trace_tcp_send.c
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>

SEC("tracepoint/tcp/tcp_sendmsg")
int trace_tcp_send(void *ctx) {
    // 追踪TCP发送
    return 0;
}

char LICENSE[] SEC("license") = "GPL";
```

**2. 编译 eBPF 程序：**

需要`clang`和`llvm`。编译命令如下（假设内核头文件位于`/lib/modules/$(uname -r)/build`）：
```bash
clang -O2 -g -target bpf -D__TARGET_ARCH_x86 -I/usr/include -I/usr/include/x86_64-linux-gnu -c trace_tcp_send.c -o trace_tcp_send.o
```
- `-O2`、`-g`：优化与调试信息
- `-target bpf`：目标为bpf字节码
- `-c`：只编译，不链接
- 如环境报找不到内核头，请加上：`-I/usr/include -I/lib/modules/$(uname -r)/build/include`

**3. 加载并运行 eBPF 程序：**

最常见方法有两种：

① 使用`bpftool`（内核 4.9+ 推荐）
```bash
sudo bpftool prog load trace_tcp_send.o /sys/fs/bpf/tcp_send
sudo bpftool prog attach /sys/fs/bpf/tcp_send tracepoint tcp/tcp_sendmsg
```

② 使用 `bcc` 或 `libbpf` 工具/模板加载，如 [libbpf-bootstrap](https://github.com/libbpf/libbpf-bootstrap)：

- 你可以参考 libbpf 的 userspace 加载模板（C/C++）。
- 或用 [bpftrace](https://github.com/iovisor/bpftrace) 快速验证 tracepoint：

  ```bash
  sudo bpftrace -e 'tracepoint:tcp:tcp_sendmsg { printf("TCP send from: %s\n", comm); }'
  ```

**4. 卸载 eBPF 程序：**
```bash
sudo bpftool prog detach tracepoint/tcp/tcp_sendmsg id <prog_id>
sudo bpftool prog list   # 查看所有加载程序
```

**5. 验证与调试**

- 用`bpftool prog`命令观察加载状态。
- 可搭配`trace-cmd`、`perf trace`等工具间接验证。
- 若希望记录事件计数或参数，自行加入`bpf_printk()`打印或维护全局`BPF_MAP`。

**注意事项：**
- 需 root 权限。
- 内核需开启`CONFIG_BPF`相关配置。
- 建议内核版本 4.9 及以上，部分功能需更高版本。

*参考文档和实例：*
- https://github.com/libbpf/libbpf-bootstrap
- https://www.brendangregg.com/ebpf.html

[src: raw/ingested/BPF_eBPF内核追踪.md]