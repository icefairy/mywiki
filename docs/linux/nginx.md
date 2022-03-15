# nginx负载均衡
+ 计算生产环境大致的网络流量，看看在当前准备作为负载的机器的物理链路能否承受的住？主要有：交换机、路由器、网线、网卡；
+ 总的流量均分给负载的后端节点时候每个节点到负载服务器之间的链路是否存在瓶颈？闲时？忙时？是否留了足够的冗余；
+ 进程数设置为cpu核心数
+ 当监听80端口时需要增加default_server申明`listen 80 default_server`

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