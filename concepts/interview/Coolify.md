# Coolify

[Coolify](https://coolify.io/) 是一个开源的 PaaS 平台，可以理解为 **自建版 Vercel/Heroku**。

**核心特点**：
- 支持 Git Push 自动部署
- 内置 SSL 证书管理（Let's Encrypt）
- 支持 Docker Compose、Dockerfile、Nixpacks 多种部署方式
- 内置数据库一键部署（PostgreSQL、MySQL、Redis、MongoDB）
- 有漂亮的 Web UI
- 支持多服务器管理

```bash
# 一行命令安装 Coolify
curl -fsSL https://cdn.coollabs.io/coolify/install.sh | bash
```

> **案例：大量独立开发者和小团队**  
> Coolify 在 GitHub 上有 30k+ Star，被大量小团队用于替代 Heroku/Vercel 的自托管方案。知名案例包括 plausible.io（开源分析工具）的用户推荐部署方式。

[src: raw/ingested/2技术/虚拟化/docker最佳实践-五、阶段三：轻量编排平台（15-30人团队）.md]