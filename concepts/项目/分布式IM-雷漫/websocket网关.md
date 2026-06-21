
# nginx反向代理支持wss

关键配置
配置文件
/etc/nginx/conf/nginx.conf
http {
...
  include /etc/nginx/conf/conf.d/*.conf;
}
配置文件
/etc/nginx/conf/conf.d/websocket.conf
upstream wx_443_pools {
ip_hash;
server 10.3.0.241:27006 max_fails=3 fail_timeout=30s;
}
server {
	listen       443 ssl ;
	server_name  websocket.raymannet.com;
	ssl_certificate /etc/nginx/conf/ssl/raymannet.com.pem; 
	ssl_certificate_key /etc/nginx/conf/ssl/private/raymannet.com.key; 
	#ssl_session_cache shared:SSL:1m;
	ssl_session_timeout  10m;
	ssl_ciphers HIGH:!aNULL:!MD5;
	ssl_prefer_server_ciphers on;
	access_log logs/websocket-443.access.log main;
	error_log  logs/websocket-443.error.log;
	location  /  {
		proxy_http_version  1.1;
		proxy_set_header    Upgrade    "websocket";
		proxy_set_header    Connection "Upgrade";
		proxy_connect_timeout 100s;
		proxy_read_timeout 300s; 
		proxy_send_timeout 300s; 
		send_timeout 300s;
		proxy_pass_header Server;
		proxy_pass       http://wx_443_pools;
		proxy_set_header Host $http_host;
		proxy_set_header X-Real-IP $remote_addr;
		proxy_set_header X-Scheme $scheme;
		proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_redirect off;
	}
}
10.3.0.241:27006 是真正的服务端地址，nginx所在域名是websocket.raymannet.com，代理的端口号是443，所以前端访问的时候这样配置：
WEBSOCKET_URL: 'wss://websocket.raymannet.com:443/hello/shake'
Nginx接收客户端wss协议消息，转发ws协议消息到Access；
Nginx接收Access的ws协议消息，转发wss协议消息到客户端。

![图](assets/websocket网关_image1.emf)

nginx -v
nginx version: nginx/1.18.0
启动
nginx -c /etc/nginx/conf/nginx.conf
重新加载配置文件：
nginx -s reload
关闭
nginx -s stop

# Access网关

配置文件
/app/thunder/deploy/Access/confweb/Access.json
关键配置：
"node_type":"ACCESS",
    "access_host":"10.3.0.241",
    "access_port":27006,
    "access_codec":10,
    "inner_host":"10.3.0.241",
"inner_port":27007,
"//access_verify_time":"外部连接校验时间（单位秒）",
"access_verify_time":30,
"server_name":"Access_web_im",
 "module":[
    	{"url_path":"/hello/shake","so_path":"plugins/ModuleShake.so","entrance_symbol":"create", "load":true, "version":1}
],  
ModuleShake.so用来处理握手协议
握手成功后，业务协议与app协议一致，在目录public\消息协议pb

# Web客户端

需要保持心跳（心跳间隔3分半）。连接后需在30s内完成登录（消息1001）。