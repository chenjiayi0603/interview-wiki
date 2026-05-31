# Go 常用数据结构

> 本笔记涵盖 Go 编程中最常用的内置数据结构，适合学习、项目开发与面试速查。

---

## 一、数组（Array）

固定长度，值类型。

```go
var a [3]int = [3]int{1, 2, 3}
fmt.Println(len(a))  // 3
fmt.Println(a[1])    // 2
```

---

## 二、切片（Slice）

动态数组，Go 中最常用的数据结构。

```go
s := []int{1, 2, 3}
s = append(s, 4)
fmt.Println(s[1:3])  // [2 3]
```

> ✅ 自动扩容、共享底层数组、支持 copy/append。

---

## 三、映射（Map）

键值对结构（哈希表）。

```go
m := map[string]int{"apple": 5, "pear": 3}
m["banana"] = 7
for k, v := range m {
    fmt.Println(k, v)
}
```

> ✅ O(1) 查找，线程不安全（并发场景用 sync.Map）。

---

## 四、结构体（Struct）

复合数据类型，用于对象建模。

```go
type User struct {
    Name string
    Age  int
}

u := User{"Alice", 18}
fmt.Println(u.Name)
```

---

## 五、指针（Pointer）

```go
x := 10
p := &x
fmt.Println(*p) // 10
```

> ✅ 无指针运算，自动垃圾回收。

---

## 六、接口（Interface）

定义行为（多态机制）。

```go
type Speaker interface {
    Speak()
}

type Dog struct{}
func (Dog) Speak() { fmt.Println("Woof!") }

func makeSpeak(s Speaker) { s.Speak() }
```

---

## 七、通道（Channel）

goroutine 之间通信的核心。

```go
ch := make(chan string)
go func() { ch <- "ping" }()
msg := <-ch
fmt.Println(msg)
```

> ✅ CSP 模型（通信顺序进程），并发安全。

---

## 八、列表（container/list）

标准库提供双向链表。

```go
import "container/list"

l := list.New()
l.PushBack(1)
l.PushFront(2)
for e := l.Front(); e != nil; e = e.Next() {
    fmt.Println(e.Value)
}
```

---

## 九、堆（container/heap）

最小堆实现，可自定义排序规则。

```go
import (
    "container/heap"
    "fmt"
)

type IntHeap []int

func (h IntHeap) Len() int           { return len(h) }
func (h IntHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h IntHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }

func (h *IntHeap) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *IntHeap) Pop() interface{} {
    old := *h
    n := len(old)
    x := old[n-1]
    *h = old[0 : n-1]
    return x
}

func main() {
    h := &IntHeap{2, 1, 5}
    heap.Init(h)
    heap.Push(h, 3)
    fmt.Println(heap.Pop(h)) // 1
}
```

---

## 十、环形队列（container/ring）

循环结构，常用于缓存。

```go
import "container/ring"

r := ring.New(3)
for i := 0; i < 3; i++ {
    r.Value = i
    r = r.Next()
}
r.Do(func(p interface{}) { fmt.Println(p) })
```

[src: raw/ingested/2技术/go/go常用数据结构.md]

## Related Pages
- [[Slice与Map]]
- [[Go接口]]
- [[Channel]]
- [[Go语言基础]]
- [[Go并发安全]]
