# Nginx 配置 - WebSocket 反向代理

## 4.1 主配置文件

**nginx.conf（主配置文件）**

```nginx
http {
    ...
    include /etc/nginx/conf/conf.d/*.conf;    # 引入所有子配置
}
```

## 4.2 WebSocket 反向代理配置

**websocket.conf（WebSocket 反向代理配置）**

```nginx
upstream wx_443_pools {
    ip_hash;     # 根据IP哈希做会话保持，避免一个用户频繁切换后端
    server 10.3.0.241:27006 max_fails=3 fail_timeout=30s;   # 业务Access节点地址和健康检查
}

server {
    listen       443 ssl;                                  # 监听443端口，并启用SSL
    server_name  websocket.raymannet.com;                   # 绑定域名
    ssl_certificate /etc/nginx/conf/ssl/raymannet.com.pem;      # 公钥证书（提供给客户端，证明服务器身份，并用于协商对称加密密钥）
    ssl_certificate_key /etc/nginx/conf/ssl/private/raymannet.com.key; # 私钥证书（仅服务器自己持有，用于配合公钥完成加密通信和身份验证）
    #ssl_session_cache shared:SSL:1m;                       # （可选）SSL Session缓存
    ssl_session_timeout  10m;
    ssl_ciphers HIGH:!aNULL:!MD5;                # "HIGH" 会包括 ECDHE（提供前向安全性），只允许使用高强度加密套件（如 AES），提高安全性，禁止弱加密和无加密（aNULL）、MD5 算法
    #ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:!aNULL:!MD5';  # 同时支持 ECDHE 和 RSA 套件，首选 ECDHE（前向安全），如客户端不支持则回落到 RSA


    # 这里建议优先用 ECDHE（椭圆曲线 Diffie-Hellman 临时密钥交换）方式，可实现"前向安全性"（Perfect Forward Secrecy），安全性远高于只用 RSA 握手
    # 推荐配置只允许 ECDHE 开头的加密套件（如 TLS_ECDHE_RSA_*、TLS_ECDHE_ECDSA_*），如确有兼容需求可辅以 RSA
    #ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:!aNULL:!MD5';  # 只启用支持 ECDHE 的高强度套件
    ssl_prefer_server_ciphers on;                # 优先使用服务器端的加密套件设置
        
    access_log logs/websocket-443.access.log main;           # 访问日志
    error_log  logs/websocket-443.error.log;                 # 错误日志

    location / {
        proxy_http_version  1.1;                             # 必须设置1.1以支持WebSocket
        proxy_set_header    Upgrade    "websocket";          # 协议升级头，WebSocket必需
        # WebSocket相关超时与连接设置
        proxy_set_header    Connection "Upgrade";      # 保证WebSocket协议升级
        proxy_connect_timeout 100s;                    # 与后端建立连接超时时间
        proxy_read_timeout 300s;                       # 后端数据读取最大超时
        proxy_send_timeout 300s;                       # 发送到后端最大超时
        send_timeout 300s;                             # 向客户端发送响应超时
        
        proxy_pass_header Server;
        proxy_pass       http://wx_443_pools;                # 反向代理到upstream
        
        proxy_set_header Host $http_host;                    # 真实请求Host
        proxy_set_header X-Real-IP $remote_addr;             # 客户端真实IP
        proxy_set_header X-Scheme $scheme;                   # 请求协议
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;  # 转发链路真实IP
        proxy_redirect off;
    }
}
```

**关键配置说明：**

- `listen 443 ssl`：Nginx 监听加密端口，启用 SSL
- `ssl_certificate`/`ssl_certificate_key`：配置证书，实现 wss
- `proxy_set_header Upgrade ...`：保证 WebSocket 协议升级
- `proxy_pass ...`：流量转发至 Access 节点（ws）

[src: raw/ingested/3项目/分布式IM-雷漫/登录-websocket网关总结-四、Nginx-配置.md]