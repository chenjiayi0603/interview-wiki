# Go高频面试问题

See also: [[C++高频面试问题]]

## Q1: Go和C++的区别？
**参考答案**：
- GC vs 手动管理
- 协程 vs 线程
- 隐式接口 vs 显式继承
- Error vs 异常
- 编译速度
- 部署简单性

## Q2: goroutine和线程的区别？
**参考答案**：
- 栈大小：goroutine 2KB（可增长），线程 1-8MB
- 创建成本：goroutine 约2KB，线程约1MB
- 切换成本：goroutine 约200ns，线程约1-2μs
- 调度：协作式 vs 抢占式
- M:N vs 1:1

## Q3: channel什么时候会阻塞？
**参考答案**：
- 无缓冲发送：无接收者时阻塞
- 无缓冲接收：无发送者时阻塞
- 有缓冲：缓冲区满时发送阻塞，缓冲区空时接收阻塞
- nil channel：永远阻塞

## Q4: context的作用？
**参考答案**：
- 传递请求范围的值
- 取消信号传播
- 超时控制
- 标准接口：Deadline、Done、Err、Value

## Q5: 如何停止一个goroutine？
**参考答案**：
- 使用channel发送停止信号
- 使用context.WithCancel
- 使用sync.WaitGroup等待
- 避免使用runtime.Goexit()

## Q6: Slice扩容源码？
**参考答案**：
```go
func growslice(et *elemType, old slice, cap int) slice {
    newcap := old.cap
    if newcap > 1024 {
        newcap += (newcap + 3*256) / 4  // 1.25倍
    } else {
        newcap *= 2  // 2倍
    }
    // 内存对齐
    var lenmem, capmem uintptr
    // 申请新内存，拷贝数据
}
```

## Q7: Map扩容过程？
**参考答案**：
- 负载因子 > 6.5 时触发
- 新bucket数组容量翻倍
- 渐进式迁移：每次操作迁移一部分
- oldbucket指向旧数组
- 迁移完成后释放

## Q8: 什么是interface{}的逃逸？
**参考答案**：
- interface{}包含两个指针（类型+数据）
- 存储interface{}类型变量时，变量可能逃逸到堆
- 影响GC压力
- 泛型可以减少interface{}使用

## Q9: sync.Map的实现原理？
**参考答案**：
- 读操作：优先从read map读取（无锁）
- 写操作：写入dirty map
- 双向指针解决miss问题
- 增量更新避免锁竞争
- dirty map为空时会重新构建

## Q10: defer的执行顺序？
**参考答案**：
- 先进后出（LIFO）
- return语句先执行，结果存入临时变量
- defer按逆序执行
- defer可以修改命名返回值

[src: raw/ingested/2技术/go/Go语言知识.md]

## Related Pages
- [[C++高频面试问题]]
- [[Go并发安全]]
- [[Goroutine调度]]
- [[Go内存管理]]
- [[Go接口]]
