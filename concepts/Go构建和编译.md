# Go 构建和编译

## 构建命令

```bash
# 编译当前包
go build

# 编译指定包
go build ./cmd/server

# 编译并指定输出文件名
go build -o myapp main.go

# 交叉编译（Linux）
# 交叉编译可以让你在当前操作系统上为其他平台（如 Linux）生成可执行文件。例如，本地是 macOS，也能编译出 Linux 下可运行的二进制。
GOOS=linux GOARCH=amd64 go build

# 交叉编译（Windows）
GOOS=windows GOARCH=amd64 go build

# 编译并自动把可执行文件安装到 $GOPATH/bin 目录下。$GOPATH 是 Go 早期的工作区路径，常用于存放源码和编译输出。
go install

# 只编译不生成文件（检查语法）
go build -o /dev/null ./...
# 其中 ./... 表示当前目录及其所有子目录下的所有包，常用于批量操作
```

## 构建标签（Build Tags）

**使用构建标签控制编译：**

```go
// +build linux darwin

package main

// 只在 Linux 和 macOS 上编译此文件
```

**使用方式：**
```bash
go build -tags=dev ./...      # 例如：只编译包含 dev 标签的代码，常用于开发环境
// 举例：
// 在 main.go 里使用构建标签：
// 文件顶部添加
// // +build dev

// 然后通过以下命令只编译带 dev 标签的文件：
// go build -tags=dev

// 例如，不同环境下打印不同信息：
// dev_only.go
// // +build dev
// package main
// import "fmt"
// func init() {
//     fmt.Println("开发环境")
// }

// prod_only.go
// // +build prod
// package main
// import "fmt"
// func init() {
//     fmt.Println("生产环境")
// }

go build -tags=prod ./...     # 包含 prod 标签
go build -tags=dev,debug ./...  # 包含多个标签
```

## 条件编译

**使用文件名后缀：**
```
go build main.go    # 默认编译
# main_linux.go 通常用于只在 Linux 系统下编译和运行的代码，可以结合 `// +build linux` 标签实现。通过以下命令只编译 main_linux.go（适用于 Linux 环境）：
go build -o app_linux main_linux.go     # 只在 Linux 平台编译 main_linux.go
main_windows.go   # 只在 Windows 编译
main_amd64.go     # 只在 amd64 架构编译
```

[src: raw/ingested/2技术/go/第三方库-go库使用-七、构建和编译.md]