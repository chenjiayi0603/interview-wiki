# 轻量编排与 CI/CD

> Docker Swarm、Coolify、Nomad、CI/CD 流水线实践。

---

## 一、Docker Swarm

Docker 原生集群方案，**零学习成本**（会 Compose 就会 Swarm）：

```bash
# 初始化 Swarm 集群
docker swarm init --advertise-addr 192.168.1.100

# 工作节点加入
docker swarm join --token SWMTKN-xxx 192.168.1.100:2377

# 部署服务（直接用 Compose 文件）
docker stack deploy -c docker-compose.yml myapp

# 扩缩容
docker service scale myapp_api=3

# 滚动更新
docker service update --image myregistry/api:v2 myapp_api
```

### Swarm Compose 示例

```yaml
version: "3.8"
services:
  api:
    image: ${REGISTRY}/api:${TAG}
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
      rollback_config:
        parallelism: 1
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
        reservations:
          cpus: "0.5"
          memory: 256M
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 15s
      timeout: 5s
      retries: 3
```

### Swarm 优势

- ✅ 内置负载均衡（Routing Mesh）
- ✅ 内置服务发现（DNS）
- ✅ 内置滚动更新/回滚
- ✅ 与 Docker Compose 文件兼容
- ✅ 学习曲线极低

---

## 二、Coolify

开源 PaaS 平台，自建版 Vercel/Heroku：

```bash
# 一行命令安装
curl -fsSL https://cdn.coollabs.io/coolify/install.sh | bash
```

**核心特点**：
- Git Push 自动部署
- 内置 SSL 证书管理（Let's Encrypt）
- 支持 Docker Compose / Dockerfile / Nixpacks
- 内置数据库一键部署（PostgreSQL、MySQL、Redis）
- 多服务器管理
- GitHub 30k+ Star

---

## 三、HashCorp Nomad

K8s 的轻量替代，适合 15-30 人团队：

```hcl
# nomad-job.hcl
job "api" {
  datacenters = ["dc1"]
  group "api" {
    count = 3
    network {
      port "http" { to = 8080 }
    }
    task "api" {
      driver = "docker"
      config {
        image = "myregistry/api:latest"
        ports = ["http"]
      }
      resources {
        cpu    = 500
        memory = 256
      }
      service {
        name = "api"
        port = "http"
        check {
          type     = "http"
          path     = "/health"
          interval = "10s"
          timeout  = "2s"
        }
      }
    }
  }
}
```

**适用**：Cloudflare、Roblox、CircleCI 等使用，国内 PingCAP 部分内部工具使用。

---

## 四、CI/CD 流水线

### GitHub Actions（小团队首选）

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
          echo "${{ secrets.REGISTRY_PASSWORD }}" | docker login registry.cn-hangzhou.aliyuncs.com -u ${{ secrets.REGISTRY_USER }} --password-stdin
          docker build -t registry.cn-hangzhou.aliyuncs.com/team/api:${{ github.sha }} ./services/api
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
            docker image prune -f
```

### Gitea + Drone CI（私有化方案）

```yaml
# .drone.yml
kind: pipeline
type: docker
name: deploy
steps:
  - name: build
    image: docker:latest
    volumes:
      - name: docker_sock
        path: /var/run/docker.sock
    commands:
      - docker build -t myregistry/api:${DRONE_COMMIT_SHA} .
      - docker push myregistry/api:${DRONE_COMMIT_SHA}

  - name: deploy
    image: appleboy/drone-ssh
    settings:
      host: { from_secret: server_host }
      username: { from_secret: server_user }
      key: { from_secret: ssh_key }
      script: |
        cd /opt/project
        TAG=${DRONE_COMMIT_SHA} docker compose up -d --no-deps api
```

---

## 五、团队规模与技术选型

| 团队规模 | 方案 | 典型场景 |
|---------|------|---------|
| 1-5人 | Docker Compose + Portainer | 创业初期，日活 10 万级 |
| 5-15人 | Compose + Ansible + GitHub Actions | 成长期，日活 50 万级 |
| 15-30人 | Swarm / Coolify / Nomad | 扩张期，日活百万级 |
| 30人+ | 托管 K8s（ACK/TKE/CCE） | 规模化，**不自建 K8s** |

**核心建议**：把精力放在业务上，等真正遇到瓶颈时再演进。
