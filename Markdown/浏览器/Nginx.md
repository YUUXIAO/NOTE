> Nginx 是一个高性能的 HTTP 和 反向代理 web 服务器；

https://mp.weixin.qq.com/s?__biz=Mzg3NjgzOTg5NA==&mid=2247484689&idx=1&sn=d0b9a22452cc581526d1c91c2e0837a6&chksm=cf2d53a8f85adabe0044c6d376083594e3d5dd50c529b7ffe6f339d9cbfe12a0fbd1b96edc04&scene=132#wechat_redirect

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

正向代理模式屏蔽或者隐藏了真实客户端信息；

## 应用场景

1. http 服务器：Nginx  是一个 http 服务可以独立提供 http 服务，可以做网页静态服务器；
2. 虚拟主机：可以在一台服务器虚拟出多个网站；
   - 基于端口的，不同的端口；
   - 基于域名的，不同域名；
3. 反向代理，负载均衡：当网站的访问量达到一定程度，单台服务器不能满足用户的请求时，需要用多台服务器集群可以使用 Nginx   做反向代理，并且多台服务器可以平均分担负载，不会因为某台服务器负载高宕机而某台服务器闲置的情况；

## 负载均衡

> 负载均衡是用来在多个计算机（计算机集群）、网络连接、CPU、磁盘驱动或其它资源中分配负载，以达到最优化资源使用、最大化吞吐率、最小化响应时间，同时避免过载的目的；

Nginx 实现负载均衡有几种方案：

### 轮询

> 轮询即 Round Robin，根据 Nginx 配置文件中的顺序，依次把客户端的  Web 请求分发到不同的后端服务器；

```javascript
upstream backserver {
  server 192.168.0.14;
  server 192.168.0.15;
}
```

### weight

> 基于权重的负载均衡即 Weight Load Balancing，在这种方式下，可以配置 Nginx 把请求更多的分发到高配置的后端服务器上，把相对较少的请求分发到低配服务器；

```javascript
upstream backserver {
  server 192.168.0.14 weight=3;
  server 192.168.0.15 weight=7;
}
```

### ip_hash

轮询和权重的负载均衡方案中，同一客户端连续的 web 请求可能会被分发到不同的后端服务器进行处理，如果涉及到会话 session 会话会比较复杂；

基于 IP 地址哈希的负载均衡方案，这样同一客户端连续的 web 请求都会被分发到同一服务器进行处理；

```javascript
upstream backserver {
    ip_hash;
    server 192.168.0.14:88;
    server 192.168.0.15:80;
}
```

### fair

> 按后端服务器的响应时间来分配请求，响应时间短的优先分配；

```javascript
upstream backserver {
    server server1;
    server server2;
    fair;
}
```

### url_hash

> 按访问 url 的 hash 结果来分配请求，使每个 url 定向到同一个（对应）的后端服务器，后端服务器为缓存时比较有效；

```javascript
upstream backserver {
    server squid1:3128;
    server squid2:3128;
    hash $request_uri;
    hash_method crc32;
}
```

在需要使用负载均衡的server中增加

```javascript
proxy_pass http://backserver/; 
upstream backserver{ 
  ip_hash; 
  server 127.0.0.1:9090 down; (down 表示单前的server暂时不参与负载) 
  server 127.0.0.1:8080 weight=2; (weight 默认为1.weight越大，负载的权重就越大) 
  server 127.0.0.1:6060; 
  server 127.0.0.1:7070 backup; (其它所有的非backup机器down或者忙的时候，请求backup机器) 
} 
```

- max_fails ：允许请求失败的次数，默认为1，当超过最大次数时，返回proxy_next_upstream 模块定义的错误；
- fail_timeout：max_fails 次失败后，暂停的时间；

```
#user  nobody;

worker_processes  4;
events {
# 最大并发数
worker_connections  1024;
}
http{
    # 待选服务器列表
    upstream myproject{
        # ip_hash指令，将同一用户引入同一服务器。
        ip_hash;
        server 125.219.42.4 fail_timeout=60s;
        server 172.31.2.183;
    }

    server{
        # 监听端口
        listen 80;
        # 根目录下
        location / {
        # 选择哪个服务器列表
            proxy_pass http://myproject;
        }

    }
}
```

