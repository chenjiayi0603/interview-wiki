# libev

libev 是一个高性能、轻量级的事件驱动库，常用于网络服务器和高并发系统。其主要功能是基于 I/O 多路复用机制（如 epoll/kqueue）管理多种事件。

## 事件循环接口

- `struct ev_loop *ev_default_loop(unsigned int flags);`
  - 获取（或新建）默认事件循环。
- `struct ev_loop *ev_loop_new(unsigned int flags);`
  - 创建一个新的事件循环实例（非默认）。
- `void ev_run(struct ev_loop *loop, int flags);`
  - 启动事件循环，开始事件调度。
- `void ev_break(struct ev_loop *loop, int how);`
  - 安全地中断事件循环，参数可指定立即退出或待当前事件处理完。

## 事件 Watcher 类型和常用接口

libev 中各种事件（I/O、定时器等）由不同的 watcher 类型管理。每种 watcher 的基本结构体都以 `ev_` 为前缀。常见有：

### (1) I/O 事件
- 结构体：`ev_io`
- 用法：
  - `void ev_io_init(ev_io *w, callback, int fd, int events);`
    - 初始化 watcher，fd 是被监控的文件描述符，events 指定可读/可写(EV_READ/EV_WRITE)。
  - `void ev_io_start(struct ev_loop *loop, ev_io *w);`
    - 开始监控 I/O 事件。
  - `void ev_io_stop(struct ev_loop *loop, ev_io *w);`
    - 停止监控。

### (2) 定时器事件
- 结构体：`ev_timer`
- 用法：
  - `void ev_timer_init(ev_timer *w, callback, ev_tstamp after, ev_tstamp repeat);`
    - after 表示延迟多少秒后触发，repeat 为周期性重复时间（0 表示一次性）。
  - `void ev_timer_start(struct ev_loop *loop, ev_timer *w);`
  - `void ev_timer_stop(struct ev_loop *loop, ev_timer *w);`

### (3) 信号事件
- 结构体：`ev_signal`
- 用法：
  - `void ev_signal_init(ev_signal *w, callback, int signum);`
  - `void ev_signal_start(struct ev_loop *loop, ev_signal *w);`
  - `void ev_signal_stop(struct ev_loop *loop, ev_signal *w);`

### (4) 空闲事件（无 I/O 时调用）
- 结构体：`ev_idle`
- 用法类似 init/start/stop。

### (5) 其他常见 watcher
- `ev_prepare`, `ev_check`, `ev_fork`, `ev_async`，用于特殊时机（循环前/后、进程fork、跨线程唤醒等）。

## 回调函数定义

所有 watcher 回调函数原型均为：
```c
void callback(struct ev_loop *loop, ev_watcher *w, int revents);
```
- loop：事件循环指针
- w：当前事件 watcher 指针
- revents：触发原因（如 EV_READ）

## 常用事件类型(flags)宏
- `EV_READ`  ：可读事件
- `EV_WRITE` ：可写事件
- `EV_TIMEOUT`：定时器触发
- `EV_SIGNAL` ：信号
- `EV_IDLE`   ：空闲

## 典型用法举例

```c
struct ev_loop *loop = ev_default_loop(0);
ev_io stdin_watcher;
void stdin_cb(struct ev_loop *loop, ev_io *w, int revents) {
    // 可读回调
    char buf[100];
    ssize_t n = read(w->fd, buf, sizeof(buf));
    // ... 处理数据
}
ev_io_init(&stdin_watcher, stdin_cb, 0, EV_READ);
ev_io_start(loop, &stdin_watcher);

ev_run(loop, 0);
```

## 总结表

| Watcher类型   | 结构体      | 事件类型         | 用途                |
|---------------|-------------|------------------|---------------------|
| I/O           | ev_io       | EV_READ/WRITE    | 监控fd读写         |
| 定时器        | ev_timer    | EV_TIMEOUT       | 定时回调           |
| 信号          | ev_signal   | EV_SIGNAL        | 信号处理           |
| 空闲          | ev_idle     | EV_IDLE          | 空闲时回调         |
| 循环前/后操作 | ev_prepare/ev_check |            | 定制循环逻辑       |
| 进程相关      | ev_fork     |                  | fork相关操作       |
| 跨线程唤醒    | ev_async    |                  | 线程通知、唤醒主循环 |

libev 设计简明高效，适合构建高性能网络服务器和异步事件驱动编程。

[src: raw/ingested/2技术/网络协议/网络库-libev.md]