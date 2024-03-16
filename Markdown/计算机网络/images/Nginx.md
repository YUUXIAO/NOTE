> Nginx 是一个高性能的 HTTP 和 反向代理 web 服务器；



## 什么是Nginx

1. 在性能上，Nginx 占用很少的系统资源，能支持更多的并发连接，达到更高效的访问效率；
2. 在功能上，Nginx  是优秀的代理服务器和负载均衡服务器；
3. 在安装配置上，Nginx 安装简单、配置灵活；

Nginx  正在被越来越多的项目采用作为网关来使用，配合 Lua 做限流、熔断等控制；

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/03df0349279b4249ba6197815dcae4ad~tplv-k3u1fbpfcp-zoom-1.image?imageslim)

### 反向代理

> 代理模式：给某个对象提供一个代理对象，并由代理对象控制原对象；
> 
> 反向代理：客户端 一> 代理 <一> 服务端；

Nginx  根据接收到的请求的端口、域名和 url，将请求转发给不同的机器，不同的端口或直接返回结果，然后将返回的数据返回给客户端；

1. Nginx  没有自己的地址，它的地址就是服务器的地址，对外部来说，它就是数据的生产者；
2. Nginx  明确的知道应该去哪个服务器获取数据（在未接收到请求之前，已经确定应该连接哪台服务器）；

### 正向代理

> 正向代理就是顺着请求的方向进行代理，即代理服务器它是由你配置为你服务，去请求目标服务器地址；
> 
> 正向代理：客户端 <一> 代理 一>服务端；

正向代理最大的特点就是客户端非常明确要访问的服务器地址，服务器只清楚请求来自哪个代理服务器，而不清楚来自哪个具体的客户端；

正向代理模式屏蔽或者隐藏了真实客户端信息（比如vpn工具就是正向代理）；

### 应用场景

1. **http 服务器（正向代理）**：Nginx  是一个 http 服务可以独立提供 http 服务，可以做网页静态服务器；
2. **虚拟主机（反向代理）**：可以在一台服务器虚拟出多个网站；

   - 基于端口的：用一个端口（比如80、443端口）实现不同服务的不同端口访问（8080、9000等）；

3. **反向代理，负载均衡：**当网站的访问量达到一定程度，单台服务器不能满足用户的请求时，需要用多台服务器集群（分布式）可以使用 Nginx  做反向代理，并且多台服务器可以平均分担负载，不会因为某台服务器负载高宕机而某台服务器闲置的情况；

## 负载均衡

> 负载均衡是用来在多个计算机（计算机集群）、网络连接、CPU、磁盘驱动或其它资源中分配负载，以达到最优化资源使用、最大化吞吐率、最小化响应时间，同时避免过载的目的；

Nginx 实现负载均衡有几种方案：

### 轮询（Round Robin）

> 轮询就是根据 Nginx 配置文件中的顺序，按顺序把客户端的请求分发到不同的后端服务器；

```javascript
upstream myproject {
  server 192.168.0.14;
  server 192.168.0.15;
}
```

### 权重（weight）

> 基于权重的负载均衡即 Weight Load Balancing，在这种方式下，可以配置 Nginx 把请求更多的分发到高配置的后端服务器上，把相对较少的请求分发到低配服务器；

如果配置服务器权重一样的话，那就是轮询分配

```javascript
upstream myproject {
  server 192.168.0.14 weight=3;
  server 192.168.0.15 weight=7;
}
```

### ip_hash

轮询和权重的负载均衡方案中，同一客户端连续的 web 请求可能会被分发到不同的后端服务器进行处理；

基于 IP 地址哈希的负载均衡方案，这样同一客户端连续的 web 请求都会被分发到同一服务器进行处理；

```javascript
upstream myproject {
    ip_hash;
    server 192.168.0.14:88;
    server 192.168.0.15:80;
}
```

### fair

> 按后端服务器的响应时间来分配请求，响应时间短的优先分配；

```javascript
upstream myproject {
    server server1;
    server server2;
    fair;
}
```

### url_hash

> 按访问 url 的 hash 结果来分配请求，使每个 url 定向到同一个（对应）的后端服务器，后端服务器为缓存时比较有效；

```javascript
upstream myproject {
    server squid1:3128;
    server squid2:3128;
    hash $request_uri;
    hash_method crc32;
}
```

在需要使用负载均衡的server中增加，比如下面这个配置就是代理四个端口的服务

```nginx
proxy_pass http://yabby.com/; 
upstream backserver{ 
  ip_hash; 
  server 127.0.0.1:9090 down; # (down 表示单前的server暂时不参与负载) 
  server 127.0.0.1:8080 weight=2; #(weight 默认为1.weight越大，负载的权重就越大) 
  server 127.0.0.1:6060; 
  server 127.0.0.1:7070 backup; #(其它所有的非backup机器down或者忙的时候，请求backup机器) 
} 
```

- max_fails ：允许请求失败的次数，默认为1，当超过最大次数时，返回proxy_next_upstream 模块定义的错误；
- fail_timeout：max_fails 次失败后，暂停的时间；

下面都用这个例子为说明

```nginx
worker_processes  4;
events {
 worker_connections  1024
}


http{
    # 待选服务器列表,这里可以配置负载均衡
    # ip_hash指令，将同一用户引入同一服务器。
    upstream myproject{
        ip_hash;
        server 125.219.42.4 fail_timeout=60s;
    }
    upstream myprojectb{
        # weight指令
        server 125.219.42.5 weight=1 fail_timeout=60s;
        server 125.219.42.6 weight=2 fail_timeout=60s;
    }


    server{
        server_name yabby.com; # 这个server 的配置只对这个域名起效果
        listen 80; # 监听端口
        # location 可以理解为是根据请求地址URI匹配出选择哪个服务器
        location / {
            proxy_pass http://myproject; # 选择哪个服务器列表
        }
        location /test {
            proxy_pass http://myprojectb; # 选择哪个服务器列表
        }




    }
}

```

## Nginx 请求分发原理

以下就是nginx转发的基本原理：

1. **客户端发送请求：**客户端向 Nginx服务器发送 HTTP 请求（http:yabby.com/test)
2. **Nginx接收请求**：Nginx服务器接收到客户端的请求
3. **在Nginx配置中，通过配置文件指定需要转发的目标服务器的地址和端口：**比如上面配置中根据域名匹配到 server，同时根据location(URI)：/test 匹配到了 proxy_pass（upstream为myprojectb，拿到了目标服务器地址125.219.42.6（权重比较大）
4. **建立连接：**Nginx与上游服务器建立连接
5. **转发请求：**Nginx将接收到的请求转发给上游服务器
6. **上游服务器处理请求：**上游服务器接收到请求后进行处理，并生成响应
7. **响应返回给Nginx：**上游服务器将生成的响应发送回Nginx服务器
8. **Nginx接收响应：**Nginx服务器接收到来自上游服务器的响应
9. **响应返回给客户端：**Nginx将接收到的响应返回给发起请求的客户端

## Nginx 跨域配置

```nginx
 location /pub/(.+) {
    if ($http_origin ~ <允许的域（正则匹配）>) {
           add_header 'Access-Control-Allow-Origin' "$http_origin";
           add_header 'Access-Control-Allow-Credentials' "true";
           # options预检请求直接返回204
       if ($request_method = "OPTIONS") {
               add_header 'Access-Control-Max-Age' 86400;
             add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS, DELETE';
              add_header 'Access-Control-Allow-Headers' 'reqid, nid, host, x-real-ip, x-forwarded-ip, event-type, event-id, accept, content-type';
              add_header 'Content-Length' 0;
             add_header 'Content-Type' 'text/plain, charset=utf-8';
             return 204;
         }
     }
    # 正常nginx配置
    ......
}
```



## Nginx 缓存机制

默认的情况下，Nginx 不会缓存响应头 `Cache-Control` 设置为 Private，No-cache 或 Set-Cookie在响应头；Nginx 只缓存 get 和 head 客户端请求

```nginx
localtion  ~ .*\.(?:jpg|jpeg|gif|png|ico|cur|gz|svg|svgz|mp4|ogg|ogv|webm)$ {
    proxy_cache yabby_cache; # 开启对后端响应的缓存
    proxy_ignore_headers Cache-Control;
    proxy_cache_valid 200 30m; # 状态码为200时，缓存有效期为30分钟
    proxy_cache_min_uses 3; #被请求3次以上时才缓存
    proxy_cache_methods GET HEAD POST; #如果你想要允许多种请求类型缓存
    proxy_cache_bypass $cookie_nocache $arg_nocache$arg_comment; #请求中有下面参数值时不走缓存
    ...
}
```

