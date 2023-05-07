* 计算生产环境大致的网络流量，看看在当前准备作为负载的机器的物理链路能否承受的住？主要有：交换机、路由器、网线、网卡；
* 总的流量均分给负载的后端节点时候每个节点到负载服务器之间的链路是否存在瓶颈？闲时？忙时？是否留了足够的冗余；
* 进程数设置为cpu核心数
* 当监听80端口时需要增加default_server申明 `listen 80 default_server`

```
#反向代理的多级转发配置示例
server {
    listen 8000;
    proxy_intercept_errors on;
    recursive_error_pages on;
    location / {
        proxy_pass http://newip:8001;
        error_page 301 302 307 = @myredirect;
    }
    location @myredirect {
        set $saved_redirect_location '$upstream_http_location';
        proxy_pass $saved_redirect_location;
    }
}
```

## 带宽限制

```

http {
   limit_rate 25k;                              #每个连接的速度限制
   
}
```

## 允许跨域

```
#允许跨域请求的域，*代表所有
add_header 'Access-Control-Allow-Origin' *;
#允许带上cookie请求
add_header 'Access-Control-Allow-Credentials' 'true';
#允许请求的方法，比如 GET/POST/PUT/DELETE
add_header 'Access-Control-Allow-Methods' *;
#允许请求的header
add_header 'Access-Control-Allow-Headers' *;
```

## 参数转请求

```
#用于内外网络隔离的情况使用
location /getimg {
    if ($request_method = 'OPTIONS') {
        add_header 'Access-Control-Allow-Origin' "*" always; 
        add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE, OPTIONS' always; 
        add_header 'Access-Control-Allow-Headers' 'DNT,X-CustomHeader,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type'; 
        add_header 'Access-Control-Allow-Credentials' true;
        add_header 'Access-Control-Max-Age' 86400;
        return 200; 
    }
    proxy_pass $arg_url;
        #允许跨域请求的域，*代表所有
    add_header 'Access-Control-Allow-Origin' "*" always;
    #允许带上cookie请求
    add_header 'Access-Control-Allow-Credentials' 'true' always;
    #允许请求的方法，比如 GET/POST/PUT/DELETE
    add_header 'Access-Control-Allow-Methods' "*" always;
    #允许请求的header
    add_header 'Access-Control-Allow-Headers' "*" always;
}
```
