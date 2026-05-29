# Slice与Map

See also: [[Go高频面试问题]]

## Slice内部结构
```go
type slice struct {
    array unsafe.Pointer  // 底层数组指针
    len   int             // 长度
    cap   int             // 容量
}
```
- **扩容规则**：容量<1024时翻倍，≥1024时1.25倍
- **拷贝**：copy(dst, src)，返回拷贝元素数
- **追加**：append(slice, elem...)
- **切片**：s[1:3]，共享底层数组

## Map内部结构
```go
type hmap struct {
    count      int
    flags      uint8
    B          uint8       // buckets数组大小 = 2^B
    noverflow  uint16
    hash0      uint32
    buckets    unsafe.Pointer
    oldbuckets unsafe.Pointer
    nevacuate  uintptr
}
```
- **负载因子**：6.5（超过会扩容）
- **渐进式扩容**：rehash分散到新bucket
- **并发安全**：需使用sync.RWMutex或sync.Map

[src: raw/ingested/2技术/go/Go语言知识.md]

## Related Pages
- [[Go高频面试问题]]
- [[Go语言基础]]
- [[Go内存管理]]
- [[Go并发安全]]
- [[Go手写代码模板]]
- [[Go接口]]
