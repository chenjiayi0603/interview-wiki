# CI/CD 最佳实践

## 6.1 GitHub Actions（小团队首选）

```yaml
# .github/workflows/deploy.yml
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
          docker build -t registry.cn-hangzhou.aliyuncs.com/myteam/api:${{ github.sha }} ./services/api
          docker push registry.cn-hangzhou.aliyuncs.com/myteam/api:${{ github.sha }}

      - name: 部署到服务器
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_KEY }}
          script: |
            cd /opt/myproject
            export TAG=${{ github.sha }}
            docker compose pull api
            docker compose up -d --no-deps api
            docker image prune -f
```

## 6.2 Gitea + Drone CI（私有化方案）

适合对代码安全有要求、不想用 GitHub 的团队：

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
      host:
        from_secret: server_host
      username:
        from_secret: server_user
      key:
        from_secret: ssh_key
      script:
        - cd /opt/myproject
        - TAG=${DRONE_COMMIT_SHA} docker compose up -d --no-deps api
```

[src: raw/ingested/2技术/虚拟化/docker最佳实践-六、CI-CD-最佳实践.md]