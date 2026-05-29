# Docker Compose 单机方案（1-5人团队）

## 3.1 项目结构标准化

```
project/
├── docker-compose.yml          # 主编排文件
├── docker-compose.prod.yml     # 生产环境覆盖
├── docker-compose.dev.yml      # 开发环境覆盖
├── .env                        # 环境变量（不提交到 Git）
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

## 3.2 典型的 docker-compose.yml

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
      - ./certbot/conf:/etc/letsencrypt:ro
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
      dockerfile: Dockerfile
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

  web:
    build:
      context: ./services/web
      dockerfile: Dockerfile
    image: ${REGISTRY}/web:${TAG:-latest}
    restart: always
    logging: *default-logging

  redis:
    image: redis:7-alpine
    command: redis-server --requirepass ${REDIS_PASSWORD} --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redis-data:/data
    restart: always
    logging: *default-logging

  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
    volumes:
      - mysql-data:/var/lib/mysql
      - ./mysql/init:/docker-entrypoint-initdb.d
    restart: always
    logging: *default-logging

volumes:
  upload-data:
  redis-data:
  mysql-data:
```

## 3.3 一键部署脚本 deploy.sh

```bash
#!/bin/bash
set -e

APP_NAME="myproject"
BACKUP_DIR="/data/backups"
COMPOSE_FILE="docker-compose.yml -f docker-compose.prod.yml"
DATE=$(date +%Y%m%d_%H%M%S)

echo "=== [$APP_NAME] 开始部署 $DATE ==="

# 1. 拉取最新代码
git pull origin main

# 2. 备份数据库（MySQL 为例）
echo ">>> 备份数据库..."
docker compose exec -T mysql mysqldump -u root -p"$MYSQL_ROOT_PASSWORD" "$MYSQL_DATABASE" \
  | gzip > "$BACKUP_DIR/db_${DATE}.sql.gz"

# 保留最近 7 天的备份
find "$BACKUP_DIR" -name "db_*.sql.gz" -mtime +7 -delete

# 3. 构建并更新服务（零停机滚动更新）
echo ">>> 构建镜像..."
docker compose -f $COMPOSE_FILE build --parallel

echo ">>> 滚动更新服务..."
docker compose -f $COMPOSE_FILE up -d --no-deps --build api
sleep 5  # 等待健康检查通过
docker compose -f $COMPOSE_FILE up -d --no-deps --build web

# 4. 清理旧镜像
docker image prune -f

echo "=== 部署完成 ==="
docker compose -f $COMPOSE_FILE ps
```

## 3.4 使用 Portainer 做可视化管理

```yaml
# 在 docker-compose.yml 中加入 Portainer
  portainer:
    image: portainer/portainer-ce:latest
    ports:
      - "9443:9443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer-data:/data
    restart: always
```

**Portainer 的价值**：让非运维人员也能查看容器状态、日志、重启服务，降低整个团队的协作成本。

> **案例：有赞早期**  
> 有赞在早期团队规模较小时（2014-2015），就是用 Docker Compose + 简单脚本管理十几个微服务，直到团队规模增长到 50+ 才逐步切换到 K8s。

[src: raw/ingested/2技术/虚拟化/docker最佳实践-三、阶段一：Docker-Compose-单机方案（1-5人团队）.md]