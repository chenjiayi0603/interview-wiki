# Go 依赖管理详解

## 三、依赖管理详解

### 3.1 go.mod 文件结构

```go
module github.com/user/project

go 1.21

require (
    github.com/gin-gonic/gin v1.9.1
    github.com/go-redis/redis/v8 v8.11.5
)

require (
    github.com/bytedance/sonic v1.8.0 // indirect
    github.com/cespare/xxhash/v2 v2.2.0 // indirect
)

exclude (
    github.com/old/package v1.0.0  // 排除特定版本
)

replace (
    github.com/old/package => github.com/new/package v1.0.0  // 替换依赖
    github.com/local/package => ./local/package  // 使用本地路径
)
```

### 3.2 版本选择规则

**Go Modules 使用最小版本选择（MVS）算法：**

1. **直接依赖**：使用 `go.mod` 中指定的版本
2. **间接依赖**：选择满足所有直接依赖要求的最小版本
3. **版本升级**：使用 `go get -u` 升级到最新兼容版本

**版本标识符：**
```bash
v1.2.3          # 语义化版本
v1.2.3-beta.1   # 预发布版本
v1.2.3+incompatible  # 不兼容版本
latest          # 最新版本
master          # 主分支（不推荐）
```

### 3.3 依赖替换（replace）

**用于本地开发、fork 修复、私有仓库等场景：**

```go
// go.mod
module myapp

replace github.com/gin-gonic/gin => ../gin  // 使用本地路径
replace github.com/old/package => github.com/new/package v1.0.0  // 替换为其他包
replace github.com/private/repo => gitlab.com/private/repo v1.0.0  // 私有仓库
```

**命令行方式：**
```bash
go mod edit -replace=github.com/gin-gonic/gin=../gin
go mod edit -dropreplace=github.com/gin-gonic/gin  # 移除 replace
```

### 3.4 依赖排除（exclude）

**排除特定版本，通常用于安全漏洞修复：**

```go
// go.mod
exclude (
    github.com/vulnerable/package v1.0.0
    github.com/vulnerable/package v1.0.1
)
```

**命令行方式：**
```bash
go mod edit -exclude=github.com/vulnerable/package@v1.0.0
```

[src: raw/ingested/2技术/go/第三方库-go库使用-三、依赖管理详解.md]