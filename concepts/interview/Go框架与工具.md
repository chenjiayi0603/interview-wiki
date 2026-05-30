# Go框架与工具

See also: [[Go高频面试问题]]

## Web框架
- **Gin**：轻量、高性能
- **Echo**：简洁、灵活
- **Beego**：全功能、ORM
- **Fiber**：性能极好（基于fasthttp）

## 中间件
```go
func Logger() gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()
        c.Next()
        fmt.Println(time.Since(start))
    }
}
```

## ORM
- **GORM**：全功能ORM
- **XORM**：轻量
- **sqlx**：原生SQL+结构体映射

## 依赖管理
```bash
go mod init module
go mod tidy
go get package@version
go mod vendor
```

[src: raw/ingested/2技术/go/Go语言知识.md]

## Related Pages
- [[Go高频面试问题]]
- [[Go网络编程]]
- [[设计模式]]
