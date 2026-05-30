# Go 语言语法快速入门

> Go（Golang）是一种静态类型、编译型、并发友好的语言，语法简洁高效。

## 🧩 基本结构

```go
package main

import "fmt"

func main() {
    fmt.Println("Hello, Go!")
}
```

### 📘 程序结构
| 关键字 | 含义 |
|:--|:--|
| `package main` | 定义包名，程序入口必须是 `main` |
| `import` | 导入包 |
| `func main()` | 程序主函数（入口） |

## 🧠 变量与常量

### 变量声明
```go
package main

import "fmt"

func main() {
    var a int = 10
    var b = "hello"
    c := 3.14 // 自动推断类型，只能在函数内部使用
    fmt.Println(a, b, c)
}
```

### 多变量声明
```go
package main

import "fmt"

func main() {
    var x, y, z = 1, 2, 3
    fmt.Println(x, y, z)
}
```

### 常量
```go
package main

import "fmt"

const PI = 3.14159
const (
    A = 1
    B = 2
)

func main() {
    fmt.Println(PI, A, B)
}
```

## 🔢 基本数据类型

| 类型 | 示例 | 说明 |
|:--|:--|:--|
| `int`, `int8`, `int64` | `42` | 整数 |
| `float32`, `float64` | `3.14` | 浮点数 |
| `bool` | `true` / `false` | 布尔值 |
| `string` | `"Hello"` | 字符串 |
| `byte` | `'A'` | ASCII 字符 |
| `rune` | `'中'` | Unicode 字符 |

## 🧮 运算符

```go
package main

import "fmt"

func main() {
    // 算术
    fmt.Println(10 + 3, 10-3, 10*3, 10/3, 10%3)
    // 比较
    fmt.Println(5 == 5, 5 != 3, 5 > 3)
    // 逻辑
    fmt.Println(true && false, true || false, !true)
    // 位运算
    fmt.Println(1 & 1, 1 | 0, 2 << 1, 4 >> 1)
}
```

## 🔁 流程控制

### if 判断
```go
package main

import "fmt"

func main() {
    x := 10
    if x > 10 {
        fmt.Println("大于10")
    } else if x == 10 {
        fmt.Println("等于10")
    } else {
        fmt.Println("小于10")
    }
}
```

### switch 分支
```go
package main

import "fmt"

func main() {
    switch day := 3; day {
    case 1:
        fmt.Println("Mon")
    case 2, 3, 4:
        fmt.Println("Midweek")
    default:
        fmt.Println("Other")
    }
}
```

### for 循环
```go
package main

import "fmt"

func main() {
    for i := 0; i < 5; i++ {
        fmt.Println(i)
    }
}
```

### for range 遍历
```go
package main

import "fmt"

func main() {
    arr := []string{"a", "b", "c"}
    for i, v := range arr {
        fmt.Println(i, v)
    }
}
```

## 🧱 数组与切片（Slice）

### 数组
```go
package main

import "fmt"

func main() {
    var arr [3]int = [3]int{1, 2, 3}
    fmt.Println(arr)
}
```

### 切片（Slice，动态数组）

> 切片是 Go 里比数组更常用的“动态数组”，与数组的主要区别：
>
> - **数组**是定长的（如 `[3]int`），一旦声明长度不可变，值类型。
> - **切片**是基于数组的动态视图（如 `[]int`），长度可变，引用类型，用得最多，底层自动扩容。
>
> **特性归纳**：
>
> |        | 数组                   | 切片                                  |
> |--------|----------------------|--------------------------------------|
> | 类型   | `[N]T`               | `[]T`                                |
> | 长度   | 固定                  | 可变（append 自动扩容）                |
> | 传递   | 传值（复制全部元素）    | 传引用（底层指向同一块内存）             |
> | 用途   | 较少（常用于定长数据）   | 最常用集合类型                         |
> | 支持   | 不能直接扩容/追加等     | 支持追加、截取、动态增长                 |
>
#### 容量区别

- **数组**的容量 `cap` 等于长度 `len`，且不可变。
- **切片**的容量 `cap` 可以大于当前长度 `len`，`append` 可能触发扩容。

```go
var arr = [3]int{1, 2, 3}
fmt.Println(len(arr), cap(arr)) // 3 3

s := make([]int, 2, 5)          // 长度 2，容量 5
fmt.Println(len(s), cap(s))     // 2 5
s = append(s, 3, 4, 5)          // 长度变 5，容量仍 5
s = append(s, 6)                // 触发扩容，容量变大
```

> 总结：数组定长定容；切片长度和容量可变，`append` 会自动扩容。

#### 内存区别

- **数组**变量本身直接存储所有元素；赋值/传参会复制整个数组（深拷贝）。
- **切片**变量是一个描述符（指针、长度、容量）；赋值/传参通常只复制描述符，多个切片可共享底层数组（浅拷贝语义）。

内存示意：

- 数组：`| [元素1][元素2][元素3] ... |`
- 切片：`| 指针 | 长度 | 容量 | -> 底层数组`

```go
// 数组赋值：深拷贝
a := [3]int{1, 2, 3}
b := a
b[0] = 100
fmt.Println(a) // [1 2 3]

// 切片赋值：共享底层数组（浅拷贝语义）
s := []int{1, 2, 3}
t := s
t[0] = 100
fmt.Println(s) // [100 2 3]
```

> 总结：数组操作的是完整数据副本；切片操作的是底层数组的视图。


```go
package main

import "fmt"

func main() {
    // s 不是数组，是切片（slice）。如下是数组写法：
    // arr := [3]int{1, 2, 3}
    s := []int{1, 2, 3} // 这是切片，长度和容量可变
    s = append(s, 4)
    fmt.Println(s[1:3]) // [2 3]
}
```

### 内置函数
```go
package main

import "fmt"

func main() {
    s := []int{1, 2, 3}
    fmt.Println(len(s), cap(s)) // 长度 容量
}
```

## 🗂️ 映射（Map）

```go
package main

import "fmt"

func main() {
    m := map[string]int{"apple": 5, "pear": 3}
    m["banana"] = 7
    delete(m, "apple")

    for k, v := range m {
        fmt.Println(k, v)
    }
}
```

## ⚙️ 函数

```go
package main

import "fmt"

func add(a int, b int) int {
    return a + b
}

func main() {
    fmt.Println(add(1, 2))
}
```

### 多返回值
```go
package main

import "fmt"

func divide(a, b int) (int, int) {
    return a / b, a % b
}

func main() {
    q, r := divide(10, 3)
    fmt.Println(q, r) // 3 1
}
```

### 命名返回值
```go
package main

import "fmt"

func calc(x int) (double int) {
    double = x * 2
    return // 自动返回 double
}

func main() {
    fmt.Println(calc(5)) // 10
}
```

## 🧩 结构体与方法

```go
package main

import "fmt"

type Person struct {
    Name string
    Age  int
}

func (p Person) SayHello() {
    fmt.Println("Hello, my name is", p.Name)
}

func main() {
    p := Person{Name: "Alice", Age: 20}
    p.SayHello()
}
```

## 🧵 指针

```go
package main

import "fmt"

func main() {
    x := 10
    p := &x
    fmt.Println(*p) // 10
    *p = 20
    fmt.Println(x)  // 20
}
```

## ⚙️ 接口（interface）

```go
package main

import "fmt"

type Speaker interface {
    Speak()
}

type Dog struct{}

func (d Dog) Speak() {
    fmt.Println("Woof!")
}

func makeSpeak(s Speaker) {
    s.Speak()
}

func main() {
    makeSpeak(Dog{})
}
```

## ⚡ 并发（goroutine & channel）

```go
package main

import (
    "fmt"
    "time"
)

func work(msg string) {
    fmt.Println(msg)
}

func main() {
    go work("goroutine 1")
    go work("goroutine 2")

    ch := make(chan string)
    go func() { ch <- "hello from channel" }()
    fmt.Println(<-ch)

    time.Sleep(time.Millisecond) // 等待 goroutine 输出
}
```

## 📦 模块与包

### 初始化模块
```bash
go mod init myapp
```

### 导入自定义包
```go
package main

import (
    "fmt"
    "myapp/utils"
)

func main() {
    fmt.Println(utils.SomeFunc())
}
```

## 📘 常用命令

| 命令 | 功能 |
|:--|:--|
| `go run main.go` | 运行程序 |
| `go build` | 编译生成可执行文件 |
| `go fmt` | 格式化代码 |
| `go test` | 运行测试 |
| `go mod tidy` | 整理依赖 |

## 🧰 错误处理

```go
package main

import (
    "fmt"
    "os"
)

func main() {
    f, err := os.Open("test.txt")
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    defer f.Close()
    // 使用 f 进行读写...
}
```

## 💡 defer / panic / recover

```go
package main

import "fmt"

func safeRun() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("Recovered:", r)
        }
    }()
    panic("Something went wrong!")
}

func main() {
    safeRun()
    fmt.Println("程序继续执行")
}
```

[src: raw/ingested/2技术/go/Go语言语法快速入门.md]

## Related Pages
- [[Go语言基础]]
- [[Slice与Map]]
- [[Go方法接收者]]
- [[Go接口]]
- [[Channel]]
- [[Go并发安全]]
- [[Goroutine调度]]
- [[Go高频面试问题]]
