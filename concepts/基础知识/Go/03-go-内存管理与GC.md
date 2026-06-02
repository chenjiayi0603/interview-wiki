# 内存管理与 GC

> 内存逃逸分析、GC 三色标记法、混合写屏障、性能调优（pprof/火焰图）。

---

## 一、内存逃逸分析

### 1.1 什么是逃逸

**解决的问题**：编译器决定变量分配在栈上还是堆上。栈分配更快（无需 GC），堆分配有 GC 开销。

```go
// 栈分配 —— 函数返回后变量消亡
func stackAlloc() int {
    x := 42      // x 在栈上分配
    return x
}

// 堆分配（逃逸）—— 函数返回后变量仍被引用
func heapAlloc() *int {
    x := 42      // x 逃逸到堆上
    return &x
}
```

**查看逃逸分析结果**：

```bash
go build -gcflags="-m" main.go       # 查看哪些变量逃逸
go build -gcflags="-m -m" main.go    # 更详细
```

### 1.2 六种逃逸场景

| 场景 | 示例 | 说明 |
|:-----|:-----|:------|
| **1. 返回局部变量指针** | `return &x` | 指针被函数返回后仍可访问 |
| **2. 存储到 interface** | `fmt.Println(x)` | interface 类型不确定，堆分配 |
| **3. 闭包捕获变量** | `func() { return x }` | 闭包内引用外部变量 |
| **4. 发送到 channel** | `ch <- &x` | 指针被其他 goroutine 使用 |
| **5. 大对象** | `make([]int, 10000)` | 超过栈帧大小限制 |
| **6. 切片扩容** | `append(s, ...)` | 动态扩容时可能重新分配 |

```go
// 场景 1：返回指针
func escape1() *int {
    x := 42
    return &x  // x 逃逸到堆
}

// 场景 2：interface 调用
func escape2() {
    x := 42
    fmt.Println(x)  // x 逃逸（fmt 参数为 interface{}）
}

// 场景 3：闭包
func escape3() func() int {
    x := 42
    return func() int {
        return x  // x 被闭包捕获，逃逸
    }
}

// 场景 4：channel 发送指针
func escape4() {
    ch := make(chan *int)
    x := 42
    ch <- &x  // x 逃逸
}

// 场景 5：大对象（> 64KB）
func escape5() {
    data := make([]byte, 100000)  // 逃逸到堆
    _ = data
}
```

### 1.3 性能影响

| 分配位置 | 速度 | 是否需要 GC |
|:---------|:-----|:------------|
| 栈 | ~1 ns | 否（函数返回自动释放） |
| 堆 | ~30 ns | 是（增加 GC 压力） |

**优化建议**：
- 避免返回指针，尽量用值传递
- 小对象优先用值传递
- 预分配 slice 容量，避免扩容逃逸
- 使用 `sync.Pool` 复用频繁分配的对象

---

## 二、GC 三色标记法

### 2.1 三色标记原理

**解决的问题**：Go 的 GC 使用并发三色标记清除算法，在 STW（Stop The World）时间极短的前提下回收不再使用的内存。

**三种颜色**：

| 颜色 | 含义 |
|:-----|:------|
| **白** | 未被 GC 访问，对象不可达（最终被回收） |
| **灰** | 已被 GC 访问，但子对象未扫描完 |
| **黑** | 已被 GC 访问，且所有子对象已扫描完 |

**三色标记流程**：

```text
1. 初始：所有对象为白色
2. GC 开始时：从 ROOT（全局变量、goroutine 栈）开始遍历，标记为灰色
3. 扫描灰色对象：将灰色对象引用的白色对象标记为灰色，自身标记为黑色
4. 重复步骤 3，直到无灰色对象
5. 剩余的白色对象为不可达对象，被回收
```

```text
初始状态：         扫描中：             完成：
[W] [W] [W]       [G] -> [W] [W]       [B] [B]
                  |                    |
                  +-> [W]             +-> [B]
```

### 2.2 GC 演进历史

| Go 版本 | GC 算法 | STW 时间 | 关键改进 |
|:--------|:--------|:---------|:---------|
| Go 1.0 | 标记-清除（串行） | ~100ms-1s | 基础 GC |
| Go 1.3 | 标记-清除（并行） | ~100ms | 并行标记 |
| Go 1.5 | **三色并发标记** | ~10ms | 并发标记，写屏障 |
| Go 1.8 | **混合写屏障** | < 1ms | 插入+删除写屏障 |
| Go 1.19 | 限制 GC CPU 使用 | < 1ms | 软硬结合限制 |

### 2.3 GC 触发条件

| 触发方式 | 说明 |
|:---------|:------|
| **内存阈值** | 堆内存增长到上次 GC 后的 2x 时触发（可通过 GOGC 调整） |
| **定时触发** | 如果 2 分钟内未触发 GC，强制启动 |
| **手动触发** | `runtime.GC()` 手动触发 |

**GC 调优**：

```bash
GOGC=100  # 默认值，堆大小翻倍时触发 GC
GOGC=200  # 减少 GC 频率，但内存使用增加
GOGC=off  # 禁用 GC（不推荐生产使用）
```

---

## 三、写屏障

### 3.1 为什么需要写屏障

**解决的问题**：并发标记时，防止黑色对象引用白色对象导致误回收。

**问题场景**：

```text
GC 标记中：黑色对象 B 引用了白色对象 W
如果 B->W 的引用被创建，但 W 未被重新标记
-> W 会被 GC 误回收！
```

### 3.2 混合写屏障（Go 1.8+）

**解决的问题**：结合插入写屏障和删除写屏障的优点，STW 时间降至 < 1ms。

```text
混合写屏障规则：
1. 被添加引用的对象（slot 指向的对象）标记为灰色
2. 被删除引用的对象（slot 原来指向的对象）标记为灰色
3. 满足 1 或 2 任意一条即触发
```

**写屏障演进**：

| 版本 | 写屏障类型 | STW 时间 |
|:-----|:-----------|:---------|
| Go 1.5 | 插入写屏障（Dijkstra） | ~10ms |
| Go 1.7 | 插入+合作式抢占 | ~2ms |
| **Go 1.8** | **混合写屏障** | **< 1ms** |

---

## 四、性能调优

### 4.1 pprof

**解决的问题**：Go 程序性能分析工具，采集 CPU、内存、goroutine 等数据。

```go
import "net/http/pprof"

func main() {
    go func() {
        log.Println(http.ListenAndServe("localhost:6060", nil))
    }()
    // ...
}
```

```bash
# 采集 CPU 分析
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# 采集内存分析
go tool pprof http://localhost:6060/debug/pprof/heap

# 采集 goroutine 分析
go tool pprof http://localhost:6060/debug/pprof/goroutine

# 交互式分析
top10       # 显示前 10 热点
list funcX  # 查看具体函数源码
web         # 生成 SVG 调用图
```

### 4.2 火焰图

**解决的问题**：可视化性能热点，快速定位 CPU 消耗和阻塞点。

```bash
# 生成火焰图
go tool pprof -http=:8080 cpu.prof

# 或使用 fgprof（采样更精确）
go get github.com/felixge/fgprof
```

### 4.3 内存分析关键指标

| 指标 | 说明 | 关注点 |
|:-----|:-----|:-------|
| `alloc_objects` | 累计分配对象数 | 是否频繁分配小对象 |
| `alloc_space` | 累计分配空间 | 内存使用量 |
| `inuse_objects` | 当前存活对象数 | 是否存在泄漏 |
| `inuse_space` | 当前存活空间 | 实际内存占用量 |

```bash
# 按 alloc_space 排序
go tool pprof -alloc_space http://localhost:6060/debug/pprof/heap

# 对比两个时间点的内存差异
go tool pprof -base base.prof current.prof
```

### 4.4 GC 调优建议

| 场景 | 建议 |
|:-----|:------|
| 延迟敏感 | 降低 GOGC（如 50），但增加 GC 频率 |
| 吞吐优先 | 提高 GOGC（如 200），减少 GC 频率 |
| 内存充裕 | 提高 GOGC，以内存换性能 |
| 内存紧张 | 降低 GOGC，减少堆内存峰值 |

```go
// 程序化设置 GOGC
debug.SetGCPercent(200)  // 堆大小翻 2 倍时触发 GC
```

---

## 五、常见陷阱

| 陷阱 | 说明 | 解决 |
|:-----|:-----|:------|
| 大对象逃逸 | 超过栈大小的对象强制堆分配 | 用指针/切片引用大对象 |
| interface 逃逸 | fmt.Println/encoding/json 等导致逃逸 | 热路径避免 interface{} |
| 闭包误用 | 循环内创建闭包导致变量逃逸 | 用参数传递而非捕获 |
| GC 频率过高 | 频繁分配小对象导致 GC 压力 | sync.Pool 复用 |
| goroutine 泄漏 | 不退出导致根集合增长 | Context 管理生命周期 |
