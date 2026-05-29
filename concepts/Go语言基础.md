# Go语言基础

See also: [[Go框架与工具]]

## 语言特性
- **静态类型**：编译时检查类型
- **并发模型**：goroutine + channel
- **垃圾回收**：三色标记法、并发GC
- **接口**：隐式实现、duck typing
- **Error处理**：显式错误返回而非异常

## 基础类型
```go
bool, string
int, int8, int16, int32, int64
uint, uint8, uint16, uint32, uint64
float32, float64
complex64, complex128
byte (= uint8), rune (= int32)
uintptr
```

## 复合类型
- 数组：`[5]int{1,2,3,4,5}`
- 切片：`[]int{1,2,3}`（动态数组）
- Map：`map[string]int{"a":1}`
- 结构体：`struct{Name string; Age int}`
- 通道：`chan int`
- 指针：`*int`

[src: raw/ingested/2技术/go/Go语言知识.md]

## Related Pages
- [[Go框架与工具]]
- [[Go接口]]
- [[Go高频面试问题]]
- [[Go并发安全]]
