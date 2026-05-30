# HashiCorp Nomad

HashiCorp Nomad 是 K8s 的轻量替代。

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

> **案例：Cloudflare、Roblox、CircleCI**  
> 虽然这些是大厂，但 Nomad 的轻量特性让它特别适合中小团队。国内如 PingCAP 的部分内部工具也使用 Nomad。

[src: raw/ingested/HashiCorp Nomad.md]