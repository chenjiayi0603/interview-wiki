# Go 第三方库总结

> Go Modules 是官方依赖管理系统，使用 `go mod init` 初始化，`go.mod` 管理依赖版本，`go.sum` 锁定依赖校验和。

## 核心要点

1. **Go Modules 是官方依赖管理系统**
   - 使用 `go mod init` 初始化
   - `go.mod` 管理依赖版本
   - `go.sum` 锁定依赖校验和

2. **导入方式**
   - 标准库直接导入
   - 第三方库需要 `go get` 安装
   - 支持别名、匿名、点导入

3. **依赖管理**
   - 使用语义化版本
   - 支持 replace 和 exclude
   - 定期更新和清理依赖

4. **最佳实践**
   - 优先使用标准库
   - 选择活跃维护的库
   - 避免过度依赖
   - 配置代理加速下载

## 常用命令速查

```bash
# 模块管理
go mod init <module>      # 初始化模块
go mod tidy               # 整理依赖
go mod download           # 下载依赖
go mod graph              # 查看依赖图
go mod verify             # 验证依赖

# 依赖操作
go get <package>          # 添加依赖
go get -u <package>       # 更新依赖
go get -u ./...           # 更新所有依赖
go list -m all            # 列出所有依赖

# 构建
go build                  # 编译
go install                # 编译并安装
go run                    # 运行
go test                   # 运行测试用例，检查代码正确性
go test 命令用于运行当前 Go 包中的测试用例。它会自动查找以 `_test.go` 结尾的测试文件，并执行文件中以 `Test` 前缀命名的测试函数，帮助开发者检测代码正确性和功能实现。常见作用如下：

- 自动化测试：通过编写测试用例验证函数和方法的行为是否符合预期。
- 回归测试：确保代码修改不会破坏已有功能。
- 测试覆盖率统计：可使用 `-cover` 参数查看测试覆盖率。
- 性能基准测试：执行以 `Benchmark` 命名的函数，评估代码性能。

例如，运行 `go test` 会自动构建并执行当前包的所有测试用例，显示测试结果，方便持续集成和代码质量保证。
```

---

> **参考资源：**
> - [Go Modules 官方文档](https://go.dev/ref/mod)
> - [Go 包管理最佳实践](https://go.dev/doc/modules/managing-dependencies)
> - [Go 标准库文档](https://pkg.go.dev/std)

[src: raw/ingested/2技术/go/第三方库-go库使用-九、总结.md]