# Docker Compose 实践

> 项目结构、编排配置、多机部署、Compose vs Shell 全面对比。

---

## 一、项目结构标准化

```
project/
├── docker-compose.yml          # 主编排文件
├── docker-compose.prod.yml     # 生产环境覆盖
├── docker-compose.dev.yml      # 开发环境覆盖
├── .env                        # 环境变量（不提交 Git）
├── .env.example                # 环境变量模板
├── nginx/
│   └── conf.d/
│       └── default.conf
├── services/
│   ├── api/
│   │   └── Dockerfile
│   └── web/
│       └── Dockerfile
└── scripts/
    ├── deploy.sh               # 一键部署脚本
    ├── backup.sh               # 数据库备份
    └── rollback.sh             # 回滚脚本
```

---

## 二、典型 docker-compose.yml

```yaml
version: "3.8"

services:
  nginx:
    image: nginx:1.25-alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
    depends_on:
      - api
      - web
    restart: always
    logging: &default-logging
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

  api:
    build:
      context: ./services/api
    image: ${REGISTRY}/api:${TAG:-latest}
    env_file: .env
    volumes:
      - upload-data:/app/uploads
    restart: always
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 5s
      retries: 3
    logging: *default-logging

  redis:
    image: redis:7-alpine
    command: redis-server --requirepass ${REDIS_PASSWORD} --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redis-data:/data
    restart: always

  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
    volumes:
      - mysql-data:/var/lib/mysql
      - ./mysql/init:/docker-entrypoint-initdb.d
    restart: always

volumes:
  upload-data:
  redis-data:
  mysql-data:
```

---

## 三、多机部署架构

```
负载均衡层 (Nginx/Traefik/SLB)
     │              │
┌────▼──────┐ ┌─────▼───────┐
│  Node-1   │ │   Node-2    │
│ API + Web │ │ API + Web   │
│ (Compose) │ │ (Compose)   │
└───────────┘ └─────────────┘
     │              │
┌────▼──────────────▼───────┐
│     数据层 (Node-3)       │
│  MySQL + Redis + MinIO    │
│      (Compose)            │
└───────────────────────────┘
```

### Ansible 批量部署

```yaml
# deploy.yml
- hosts: app_servers
  become: yes
  tasks:
    - name: 拉取最新代码
      git:
        repo: "git@github.com:team/project.git"
        dest: /opt/project
        version: main

    - name: 滚动更新服务
      command: docker compose up -d --no-deps {{ item }}
      args:
        chdir: /opt/project
      loop:
        - api
        - web
      loop_control:
        pause: 10
```

---

## 四、CI/CD 一键部署

```bash
#!/bin/bash
# deploy.sh
set -e
DATE=$(date +%Y%m%d_%H%M%S)

# 1. 拉取最新代码
git pull origin main

# 2. 备份数据库
docker compose exec -T mysql mysqldump -u root -p"$MYSQL_ROOT_PASSWORD" --all-databases \
  | gzip > "/data/backups/db_${DATE}.sql.gz"

# 3. 构建并更新服务
docker compose build --parallel
docker compose up -d --no-deps --build api
sleep 5  # 等待健康检查
docker compose up -d --no-deps --build web

# 4. 清理旧镜像
docker image prune -f
```

### GitHub Actions

```yaml
name: Deploy
on:
  push:
    branches: [main]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: 构建并推送镜像
        run: |
          docker build -t registry.cn-hangzhou.aliyuncs.com/team/api:${{ github.sha }} ./api
          docker push registry.cn-hangzhou.aliyuncs.com/team/api:${{ github.sha }}
      - name: 部署到服务器
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_KEY }}
          script: |
            cd /opt/project
            export TAG=${{ github.sha }}
            docker compose pull api
            docker compose up -d --no-deps api
```

---

## 五、Docker Compose vs Shell 脚本

### 5.1 对比

| 维度 | 纯 Shell 脚本 | Docker Compose |
|------|--------------|----------------|
| 代码量 | 80+ 行 + 额外脚本 | 1 个 YAML ≈ 45 行 |
| 启动顺序 | 手写 sleep/until 轮询 | `depends_on` + `condition: service_healthy` |
| 网络管理 | 手动 `docker network create` | 自动创建，服务名即主机名 |
| 幂等性 | 二次执行报"容器名已存在" | 天然幂等，只更新变化 |
| 服务发现 | 容器名硬编码 | 自动 DNS 解析 |
| 缩扩容 | 手动起/停，管理端口 | `--scale api=3` 一个参数 |
| 配置管理 | 散落在脚本各处 | YAML 声明式，结构清晰 |

### 5.2 最佳分工

```
Shell 脚本（编排层）： 备份数据库、拉取代码、通知、回滚逻辑
Docker Compose（容器层）： 定义和运行容器
```

```bash
#!/bin/bash
# deploy.sh —— Shell 调 Compose，各司其职
git pull origin main           # Shell 擅长的
./scripts/backup-db.sh         # Shell 擅长的
docker compose pull            # 交给 Compose
docker compose up -d --no-deps --build api  # 交给 Compose
curl -X POST "$WEBHOOK" -d '{"text":"部署完成"}'  # Shell 擅长的
```

---

## 六、Docker vs 直接运行二进制

| 维度 | 直接运行二进制 | Docker |
|------|--------------|--------|
| 安装依赖 | 每个软件不同方式，经常冲突 | `docker pull` 自带依赖 |
| 环境一致性 | "在我机器上能跑" | 镜像即环境，100% 一致 |
| 多版本共存 | Node 16 和 20 共存？痛苦 | 不同容器跑不同版本 |
| 资源隔离 | 共享宿主机，OOM 拖垮整机 | cgroup 隔离 |
| 服务迁移 | 重新装一遍环境（一整天） | `docker compose up -d`（5 分钟） |
| 回滚 | 希望做了备份 | `docker run image:v1.0` 秒回 |
| 安全 | 入侵后可访问整个系统 | namespace 隔离 |

**总结**：Docker 的代价几乎可忽略（性能损耗 < 2%），而收益是系统性的。
