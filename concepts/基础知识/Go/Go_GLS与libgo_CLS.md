# Go GLS 与 libgo CLS

## 概述

Go 官方不提供 goroutine-local storage (GLS)，推荐使用 context 传递。libgo 提供协程本地存储 (Coroutine Local Storage, CLS)。

## Go 没有原生 GLS

Go 官方不提供 goroutine-local storage，推荐使用 context 传递：

```go
func handler(ctx context.Context) {
    // 从 context 获取值
    userID := ctx.Value("userID").(string)
    
    // 传递给子函数
    processData(ctx)
}

func processData(ctx context.Context) {
    userID := ctx.Value("userID").(string)
    // ...
}
```

[src: raw/ingested/2技术/go/原理-libgo和go的对比-五、协程本地存储.md]

## libgo 的 CLS（协程本地存储）

### 实现原理简述

libgo 的 GLS（协程本地存储，CLS）实现方式通常采用“协程上下文指针+模板变量”的机制，不依赖于线程本地存储 TLS，而是每个协程维护自己的本地变量副本。

核心思想如下：

1. **每个协程结构体中包含 CLS 数据的存储区**：libgo 内部为每个协程分配一块独立的内存，用于存储该协程的本地变量。

2. **模板类封装变量访问**：通过 `CLS(T, name)` 宏声明协程本地变量，生成的 `name` 实例会根据当前运行的协程自动访问其私有存储空间的数据。

3. **变量的正确隔离**：每一个协程中的 `cls_counter` 实际存储完全互不干扰，不同协程访问同名 CLS 变量，互不冲突。切换协程上下文时，libgo 运行时自动切换 CLS 存取区域。

4. **实现机制**（简化伪码）：关键点为为每个变量分配唯一 id，然后将（变量id，对象指针）映射存储到当前协程的 context 区域中。

5. **自动清理**：协程结束时，libgo 会自动回收其 context 与 CLS 占据的内存。

[src: raw/ingested/2技术/go/原理-libgo和go的对比-五、协程本地存储.md]

### CLS 的拷贝过程详解

**拷贝过程是什么？**
libgo 的 CLS（协程本地存储）变量在以下情况下会发生“副本拷贝”：

- **协程创建之初**：新协程会有自己全新的一份 CLS 变量副本（即默认构造/初始化），不会自动带入父协程的 CLS 内容。
- **协程手动切换/派生**：如果开发者需要在新协程“继承”老协程的某些状态，可以在协程函数入口处手动把需要的信息赋值给 CLS 变量。libgo 不会默认拷贝父协程 CLS 给子协程，开发者需自行传递/设置。
- **协程切换运行时**：每个协程运行时访问自己的 CLS 区域，协程切换不会发生变量内容的拷贝（因为本质是指针切换）。

**拷贝哪些？**
只拷贝你用 `CLS(Type, name);` 显式声明的变量，每个协程拥有这些变量的独立副本。其他普通的局部变量、全局变量不会受到影响，也不会自动变成协程本地存储。

**啥时候会触发拷贝？**
- 创建新协程时：给新协程独立分配一份 CLS 区域，各变量用其默认构造函数初始化。
- 如果你希望“传递”某些上下文给新协程，需要显式地通过参数或设置 CLS 变量（libgo 不会帮你自动拷贝当前协程的 CLS）。

[src: raw/ingested/2技术/go/原理-libgo和go的对比-五、协程本地存储.md]

### 使用示例

```cpp
// 定义协程本地变量
CLS(int, cls_value);

go []{
    // 每个协程有独立的 cls_value
    cls_value = 100;
    printf("coroutine 1: %d\n", cls_value.get());
};

go []{
    cls_value = 200;
    printf("coroutine 2: %d\n", cls_value.get());
};
```

[src: raw/ingested/2技术/go/原理-libgo和go的对比-五、协程本地存储.md]

### 优点

- 变量真正隔离，并非线程本地而是协程本地
- 使用方式近似 TLS，便于移植线程代码
- 性能开销极低

### 典型应用场景

- 记录协程 id、追踪链路、日志上下文
- 保存请求级状态

### 对比 Go

Go 标准库没有 GLS，推荐 context，但 libgo 的 CLS 用法更接近于线程的 thread_local。

[src: raw/ingested/2技术/go/原理-libgo和go的对比-五、协程本地存储.md]