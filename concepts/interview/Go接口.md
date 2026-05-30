# Go接口

See also: [[Go高频面试问题]]

## 空接口
```go
var i interface{}
i = 42
i = "hello"
i = []int{1, 2, 3}
```
- 所有类型都实现空接口
- 泛型前的主要抽象方式

## 类型断言
```go
v, ok := i.(T)  // 安全断言
v := i.(T)      // 不安全，panic
```
- **switch类型断言**
```go
switch v := i.(type) {
case int:
    // v是int
case string:
    // v是string
default:
    // 未知类型
}
```

## 接口最佳实践
- **小接口**：io.Reader, io.Writer, io.Closer
- **接口定义在使用者一方**
- **组合优于继承**

[src: raw/ingested/2技术/go/Go语言知识.md]

## Related Pages
- [[Go高频面试问题]]
- [[Go语言基础]]
- [[Go并发安全]]
