# Go 依赖管理最佳实践

## 6.1 版本管理策略

1. **使用语义化版本**
   ```bash
   go get github.com/gin-gonic/gin@v1.9.1  # 指定版本
   go get github.com/gin-gonic/gin@latest  # 最新版本
   ```

2. **定期更新依赖**
   ```bash
   go get -u ./...  # 更新所有依赖
   go mod tidy      # 清理未用依赖，让 go.mod 和 go.sum 清爽整洁。

   // go.mod 是 Go 项目的模块描述文件——
   //    - 记录当前项目需要依赖哪些外部包（包括版本号）
   //    - 声明模块名（如 module myapp）
   //    - 只包含“直接依赖”和 go 相关指令

   // go.sum 是依赖包的哈希校验和清单——
   //    - 每个依赖模块的版本及其hash值
   //    - 用于保障依赖安全性、可重复构建
   //    - 防止篡改，拉包时自动校验

   // 总结：go.mod 决定“要哪些依赖和版本”，go.sum 保证“拿到的内容一定可靠”；两者配合实现严格可控、可溯源的依赖管理。
   ```

3. **锁定依赖版本**
   - `go.sum` 文件自动锁定依赖的校验和
   - 提交 `go.sum` 到版本控制

## 6.2 依赖选择原则

1. **优先使用标准库**
   - 性能好、稳定、无需额外依赖

2. **选择活跃维护的库**
   - 查看 GitHub stars、issues、最近提交时间

3. **避免过度依赖**
   - 评估库的必要性
   - 考虑自己实现的成本

4. **注意依赖大小**
   - 使用 `go mod why` 查看依赖原因
   - 使用 `go mod graph` 查看依赖图

## 6.3 私有仓库配置

**设置 GOPRIVATE 环境变量：**
```bash
# 设置私有仓库（不走代理）
# GOPRIVATE 用于设置哪些仓库属于“私有仓库”，拉取这些 repo 时 Go 不会通过代理，且 go get/publish 时不会将源码和元数据上传到 proxy，也不会走 checksum 校验。
# github.com/mycompany 通常是你们公司或组织在 GitHub 上的私有仓库，用于托管内部 Go 包、组件或业务代码。将其加入 GOPRIVATE 后，拉取这些仓库时 Go 会跳过代理和校验等限制，适合企业内部组件开发与分发。
export GOPRIVATE=gitlab.com/mycompany,github.com/mycompany

// 私有仓库依赖使用举例

// 1. go.mod 示例
module demoapp

go 1.20

require (
    github.com/yourcompany/internal-lib v1.2.3
    gitlab.com/yourteam/secret-util v0.4.1
)

// 2. 拉取和使用私有库
// GOPRIVATE 配置（让 Go 拉取私有库时跳过代理、校验等步骤）
// 举例：你的私有公司仓库是 github.com/yourcompany 和 gitlab.com/yourteam
//
// $ export GOPRIVATE=github.com/yourcompany,gitlab.com/yourteam
//
// 拉取依赖：
// $ go get github.com/yourcompany/internal-lib@v1.2.3
// $ go get gitlab.com/yourteam/secret-util@latest

// 3. 在项目代码中直接 import 并使用：
package main

import (
    "github.com/yourcompany/internal-lib/logger"
    "gitlab.com/yourteam/secret-util/crypto"
    "fmt"
)

func main() {
    logger.Info("hello from private lib")
    encrypted := crypto.Encrypt("secret")
    fmt.Println("加密结果:", encrypted)
}


# 配置 Git 凭证
# 作用：将所有 https://gitlab.com/ 形式的 Git 仓库地址自动替换为使用 SSH 协议（git@gitlab.com:），避免每次从私有仓库拉代码都要输账号密码（需要配置好 SSH key）。
git config --global url."git@gitlab.com:".insteadOf "https://gitlab.com/"
```

**go.mod 中使用私有仓库：**
```go
module myapp

require (
    gitlab.com/mycompany/private-package v1.0.0
)
```

## 6.4 代理配置

**使用 Go Proxy 加速下载：**
```bash
# 设置代理
export GOPROXY=https://goproxy.cn,direct

# 或使用官方代理
export GOPROXY=https://proxy.golang.org,direct

# 禁用代理（仅直连）
export GOPROXY=direct
```

**常用国内代理：**
- `https://goproxy.cn`
- `https://goproxy.io`
- `https://mirrors.aliyun.com/goproxy/`

[src: raw/ingested/2技术/go/第三方库-go库使用-六、依赖管理最佳实践.md]

## Related Pages
- [[Go语言语法快速入门]]
- [[Go第三方库手册]]
- [[Go框架与工具]]
- [[Go构建和编译]]