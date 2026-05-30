# Docker Compose vs 纯 Shell 脚本管理容器 —— 全面对比

## 为什么不直接写 Shell 脚本？

很多人的第一反应是：我直接写 `docker run` 脚本不就行了？来看一个真实对比。

## 场景：部署一个典型 Web 应用（Nginx + API + Redis + MySQL）

### 纯 Shell 脚本方式

```bash
#!/bin/bash

# 创建网络
docker network create myapp-net 2>/dev/null || true

# 启动 MySQL
docker run -d \
  --name myapp-mysql \
  --network myapp-net \
  --restart always \
  -e MYSQL_ROOT_PASSWORD=secret123 \
  -e MYSQL_DATABASE=mydb \
  -v /data/mysql:/var/lib/mysql \
  --log-driver json-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  --memory 1g \
  --cpus 1.0 \
  --health-cmd "mysqladmin ping -h localhost" \
  --health-interval 30s \
  --health-timeout 5s \
  --health-retries 3 \
  mysql:8.0

# 等 MySQL 就绪
echo "等待 MySQL 启动..."
until docker exec myapp-mysql mysqladmin ping -h localhost --silent; do
  sleep 2
done

# 启动 Redis
docker run -d \
  --name myapp-redis \
  --network myapp-net \
  --restart always \
  -v /data/redis:/data \
  --log-driver json-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  --memory 512m \
  redis:7-alpine redis-server --requirepass redispwd --maxmemory 256mb

# 启动 API（依赖 MySQL 和 Redis）
docker run -d \
  --name myapp-api \
  --network myapp-net \
  --restart always \
  -e DB_HOST=myapp-mysql \
  -e REDIS_HOST=myapp-redis \
  -e DB_PASSWORD=secret123 \
  -p 8080:8080 \
  --log-driver json-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  --memory 512m \
  --health-cmd "curl -f http://localhost:8080/health || exit 1" \
  --health-interval 15s \
  myregistry/api:latest

# 启动 Nginx
docker run -d \
  --name myapp-nginx \
  --network myapp-net \
  --restart always \
  -p 80:80 \
  -p 443:443 \
  -v /opt/nginx/conf.d:/etc/nginx/conf.d:ro \
  --log-driver json-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  nginx:1.25-alpine
```

还需要额外的脚本：

```bash
# stop.sh —— 停止所有服务
docker stop myapp-nginx myapp-api myapp-redis myapp-mysql
docker rm myapp-nginx myapp-api myapp-redis myapp-mysql

# restart.sh —— 重启某个服务
docker stop myapp-api && docker rm myapp-api
# 然后重新执行上面那一大段 docker run...（复制粘贴？还是抽成函数？）

# update.sh —— 更新某个服务
docker pull myregistry/api:latest
docker stop myapp-api && docker rm myapp-api
# 又要重新 docker run... 参数一个都不能少

# status.sh —— 查看状态
docker ps --filter name=myapp-

# logs.sh —— 查看日志
docker logs -f myapp-api
```

### Docker Compose 方式

```yaml
# docker-compose.yml —— 一个文件搞定一切
services:
  nginx:
    image: nginx:1.25-alpine
    ports: ["80:80", "443:443"]
    volumes: ["./nginx/conf.d:/etc/nginx/conf.d:ro"]
    depends_on: [api]
    restart: always

  api:
    image: ${REGISTRY}/api:${TAG:-latest}
    ports: ["8080:8080"]
    environment:
      DB_HOST: mysql
      REDIS_HOST: redis
      DB_PASSWORD: ${DB_PASSWORD}
    depends_on:
      mysql: { condition: service_healthy }
      redis: { condition: service_started }
    deploy:
      resources:
        limits: { memory: 512M }
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 15s
    restart: always

  redis:
    image: redis:7-alpine
    command: redis-server --requirepass ${REDIS_PASSWORD} --maxmemory 256mb
    volumes: ["redis-data:/data"]
    deploy:
      resources:
        limits: { memory: 512M }
    restart: always

  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_PASSWORD}
      MYSQL_DATABASE: mydb
    volumes: ["mysql-data:/var/lib/mysql"]
    deploy:
      resources:
        limits: { memory: 1G, cpus: "1.0" }
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 30s
    restart: always

volumes:
  redis-data:
  mysql-data:
```

日常操作只需要：

```bash
docker compose up -d          # 启动全部
docker compose down           # 停止全部
docker compose restart api    # 重启某个服务
docker compose pull && docker compose up -d  # 更新
docker compose ps             # 查看状态
docker compose logs -f api    # 查看日志
docker compose up -d --scale api=3  # 扩容
```

---

### 逐项对比

| 维度 | 纯 Shell 脚本 | Docker Compose |
|------|--------------|----------------|
| **代码量** | 4 个服务 ≈ 80+ 行脚本，还需额外 stop/restart/update 脚本 | 1 个 YAML 文件 ≈ 45 行，零额外脚本 |
| **启动顺序** | 手动写 `sleep`/`until` 循环等待依赖就绪 | `depends_on` + `condition: service_healthy` 自动编排 |
| **网络管理** | 手动 `docker network create`，手动指定 `--network` | 自动创建项目专属网络，服务名即主机名 |
| **服务发现** | 用容器名 `myapp-mysql` 硬编码，改名要改所有脚本 | 用服务名 `mysql` 直接访问，自动 DNS 解析 |
| **环境变量** | 散落在脚本各处，或需要额外 source 一个 env 文件 | 统一 `.env` 文件 + `env_file` 指令 |
| **更新服务** | 必须 stop → rm → run，参数不能漏 | `docker compose up -d --no-deps api` 一条命令 |
| **回滚** | 需要自己记住上一个版本号和所有参数 | `docker compose up -d` 配合 `.env` 中的 TAG 即可 |
| **扩缩容** | 手动起多个容器，名字不能重复，还要自己管端口 | `--scale api=3` 一个参数搞定 |
| **查看状态** | `docker ps --filter` 拼筛选条件 | `docker compose ps` 只显示本项目的容器 |
| **查看日志** | 每个容器单独 `docker logs` | `docker compose logs` 聚合所有服务日志，支持按服务筛选 |
| **资源隔离** | 项目间容器混在一起，`docker ps` 一屏看不完 | 每个项目有独立的命名空间（项目名前缀） |
| **配置即文档** | 脚本就是"配置"，但可读性差，新人看不懂 | YAML 声明式，结构清晰，新人 10 分钟能看懂 |
| **多环境** | 写多套脚本或大量 if-else | `docker-compose.override.yml` 覆盖机制 |
| **Volume 管理** | 手动 `-v` 绑定，清理时容易遗漏 | 声明式 volumes，`docker compose down -v` 一键清理 |
| **幂等性** | 二次执行会报"容器名已存在"，需要额外判断 | 天然幂等，重复执行只更新变化的部分 |
| **团队协作** | "你看看那个脚本第 47 行的参数对不对" | "你看 compose 文件里 api 服务的配置" |

---

### 深入分析几个关键优势

#### 1. 声明式 vs 命令式

Shell 脚本是 **命令式**（告诉系统"怎么做"）：
```bash
docker network create myapp-net       # 第一步
docker run -d --name mysql ...         # 第二步
sleep 10                                # 第三步
docker run -d --name api ...           # 第四步
```
如果中间某步失败，后续步骤可能在错误状态下继续执行。

Docker Compose 是 **声明式**（告诉系统"我要什么"）：
```yaml
services:
  api:
    depends_on:
      mysql: { condition: service_healthy }
```
Compose 引擎负责编排执行顺序、处理依赖、确保最终状态正确。

> 类比：Shell 脚本像手动挡开车，Docker Compose 像自动挡。日常通勤没人想开手动挡。

#### 2. 幂等性（最被低估的优势）

```bash
# Shell 脚本执行两次会报错
$ bash start.sh
Error: container name "myapp-mysql" already in use

# Docker Compose 执行两次完全没问题
$ docker compose up -d
myapp-mysql-1  Running    # 已在运行，跳过
myapp-api-1    Recreated  # 检测到配置变更，自动重建
myapp-redis-1  Running    # 已在运行，跳过
```

Compose 会智能比较当前状态与目标状态的差异，**只更新变化的部分**。Shell 脚本要实现同样的效果，需要大量的条件判断代码。

#### 3. 依赖管理与健康检查联动

```bash
# Shell 脚本：手写轮询等待，脆弱且丑陋
until docker exec myapp-mysql mysqladmin ping -h localhost --silent 2>/dev/null; do
  echo "等待 MySQL..."
  sleep 2
  COUNTER=$((COUNTER+1))
  if [ $COUNTER -gt 30 ]; then
    echo "MySQL 启动超时！"
    exit 1
  fi
done
```

```yaml
# Docker Compose：优雅且可靠
api:
  depends_on:
    mysql:
      condition: service_healthy
    redis:
      condition: service_started
```

#### 4. 配置变更管理

修改 API 的内存限制：

```bash
# Shell 脚本：找到脚本 → 找到那行 → 改参数 → stop → rm → run
# 如果参数改错了，整个容器挂掉，还得回忆之前的参数是什么
```

```yaml
# Docker Compose：改 YAML 中一个值 → docker compose up -d
# 只有改变的服务会被重建，其他服务不受影响
deploy:
  resources:
    limits:
      memory: 1G   # 512M → 1G，保存后 up -d 即可
```

---

### 那 Shell 脚本完全没用了吗？

不是。Shell 脚本适合做 **Compose 之上的编排**，两者是互补关系：

```
┌─────────────────────────────────┐
│  deploy.sh（Shell 脚本）         │ ← 编排层：备份、拉代码、通知、回滚
│  ┌───────────────────────────┐  │
│  │  docker-compose.yml       │  │ ← 容器管理层：定义和运行容器
│  │  (Docker Compose)         │  │
│  └───────────────────────────┘  │
└─────────────────────────────────┘
```

最佳实践是：
- **Docker Compose** 负责声明容器的配置和编排
- **Shell 脚本** 负责 Compose 之外的流程（备份数据库、拉取代码、发通知、灰度逻辑等）

```bash
#!/bin/bash
# deploy.sh —— Shell 脚本调用 Compose，各司其职
git pull origin main                              # Shell 擅长的
./scripts/backup-db.sh                            # Shell 擅长的
docker compose pull                               # 交给 Compose
docker compose up -d --no-deps --build api        # 交给 Compose
curl -X POST "$WEBHOOK" -d '{"text":"部署完成"}'   # Shell 擅长的
```

### 一句话总结

> **Docker Compose 管容器，Shell 脚本管流程。用 Compose 替代 `docker run`，用 Shell 编排 Compose，这才是中小团队的最优解。**

[src: raw/ingested/2技术/虚拟化/docker最佳实践-附录：Docker-Compose-vs-纯-Shell-脚本管理容器-——-全面对比.md]