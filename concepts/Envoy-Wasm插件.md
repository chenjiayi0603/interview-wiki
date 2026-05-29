# Envoy-Wasm插件

## 4. Envoy Wasm插件

```cpp
// EnvoyFilter示例
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: my-wasm-filter
spec:
  workloadSelector:
    labels:
      app: my-app
  configPatches:
  - applyTo: HTTP_FILTER
    match:
      context: SIDECAR_INBOUND
      listener:
        filterChain:
          filter:
            name: envoy.filters.network.http_connection_manager
    patch:
      operation: INSERT_BEFORE
      value:
        name: my-wasm-filter
        config_discovery:
          config_source:
            ads: {}
          type_url: type.googleapis.com/envoy.extensions.filters.http.wasm.v3.Wasm
```

[src: raw/ingested/2技术/虚拟化/云原生与K8s-二、服务网格（Service-Mesh）.md]