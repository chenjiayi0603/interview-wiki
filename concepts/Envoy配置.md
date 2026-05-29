# Envoy配置

## 3. Envoy配置

```yaml
static_resources:
  listeners:
  - name: http
    address:
      socket_address:
        address: 0.0.0.0
        port_value: 8080
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: ingress_http
          route_config:
            virtual_hosts:
            - name: service
              domains: ["*"]
              routes:
              - match: { prefix: "/" }
                route:
                  cluster: my_service
          http_filters:
          - name: envoy.filters.http.router
  
  clusters:
  - name: my_service
    connect_timeout: 0.25s
    type: STRICT_DNS
    lb_policy: ROUND_ROBIN
    load_assignment:
      cluster_name: my_service
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: my-service
                port_value: 8080
```

[src: raw/ingested/2技术/虚拟化/云原生与K8s-二、服务网格（Service-Mesh）.md]