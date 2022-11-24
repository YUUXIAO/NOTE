---
typora-root-url: ..\..
---

https://code.zuifengyun.com/2022/10/3160.html 【service worker】

- 强缓存的依据来自于缓存是否过期，而不关心服务端文件是否已经更新，这可能会导致加载的文件不是服务端最新的内容，此时我们就需要用到协商缓存
- 如果浏览器命中强缓存，则不需要给服务器发请求；而协商缓存最终由服务器来决定是否使用缓存，即客户端与服务器之间存在一次通信。
- 强缓存返回的状态码是200，直接从缓存内读取，浏览器不用向服务器发起Http请求；
- 协商缓存如果命中走缓存的话，请求的状态码是 304 (not modified)



## 强缓存（200 OK from cache）

![浏览器缓存_强缓存](/images/浏览器缓存_强缓存.png)

### Expires（HTTP1.0）

> Expires 即过期时间，时间是相对于服务器的时间而言的，存在于服务端返回的响应头中，在这个过期时间之前可以直接从缓存里面获取数据，无需再次请求；

Expires = 时间，缓存的截止时间，允许客户端在这个时间之前不去检查（发请求）

```javascript
Expires:Mon, 29 Jun 2020 11:10:23 GMT
```

该资源在2020年7月29日11:10:23过期，过期就会重新向服务器发起请求；

这个方式有一个问题：返回的到期时间是服务器端的时间，服务器的时间和浏览器的时间可能并不一致，所以HTTP1.1 提出新的字段代替它；

### Cache-Control（HTTP1.1）

> Cache-Control 是过期时长，对应的是 max-age

当 Expires 和 Cache-Control 同时存在时，优先考虑 Cache-Control；

```javascript
// 资源返回后6000秒内，可以直接使用缓存
Cache-Control:max-age=6000
```

常见的请求头中的 Cache-Control 指令：

```javascript
Cache-Control: max-age=<seconds>  // 缓存存储的最大周期，单位为秒
Cache-Control: s-maxage=<seconds>  // 和max-age是一样的，不过它只针对代理服务器缓存而言；
Cache-Control: max-stale[=<seconds>] // 客户端愿意接收一个已经过期的资源
Cache-control: no-cache // 重新验证是否缓存(协商缓存验证)，强制客户端直接向服务器发送请求，每次请求都必须向服务器发送，服务器接收到请求后判断资源是返回新内容还是304
Cache-control: no-store // 禁止一切缓存                       
```

常见的响应头中的 Cache-Control 指令：

```javascript
Cache-control: must-revalidate // 资源过期后重新请求
Cache-control: no-cache // 请求是验证是否缓存(协商缓存验证)
Cache-control: no-store // 不使用任何缓存
Cache-control: private // 同意本地浏览器缓存，不能被代理服务器缓存
Cache-control: public // 可被任何缓存区缓存
```

## 协商缓存（304）

在浏览器第一次请求某一个URL时，服务器端的返回状态会是200，资源响应头有一个Last-Modified的属性标记此文件在服务期端最后被修改的时间，另外一半也有个Etag，格式类似这样

```javascript
Last-Modified:Fri, 15 Feb 2022 03:06:18 GMT
ETag:"be15b26c29bce1:0" #可选，这里为了准确确认资源是否变化
```

客户端第二次请求此URL时，根据 HTTP 协议的规定，浏览器会向服务器传送 If-Modified-Since和If-None-Match(可选报头，值Etag的值) 报头，询问该时间之后文件是否有被修改过：

```javascript
If-Modified-Since:Sat, 16 Feb 2022 07:30:07 GMT
If-None-Match:"be15b26c29bce1:0" #可选，这里为了准确确认资源是否变化
```

如果服务器端的资源没有变化，则自动返回 HTTP 304 （Not Changed.）状态码，内容为空，否则重新发起请求，请求下载资源这样就节省了传输数据量。

![浏览器缓存_协商缓存](/images/浏览器缓存_协商缓存.png)

### Last-Modified

> Last-Modified 是响应头字段用来标识资源的有效性；

```
Last-Modified:Fri, 15 Feb 2013 03:06:18 GMT
ETag:"be15b26c29bce1:0" #可选，这里为了准确确认资源是否变化
```



1. Last-Modified 表示的是最后修改时间，在浏览器第一次给服务器发送请求后，服务器会在响应头中加上这个字段；
2. 浏览器接收到后，如果再次请求，会在请求头中携带 If-Modified-Since字段，这个字段表示服务器传来的最后修改时间；
3. 服务器拿到请求头中的 If-Modified-Since 的字段后，和这个服务器中该资源的最后修改时间对比：
   - 如果请求头中的这个值小于最后修改时间，说明是时候更新了，返回新的资源；
   - 否则返回304，告诉浏览器直接使用缓存；

### Etag

> Etag 是响应头字段表示资源的版本；

ETag 出现的原因是 Last-Modified 只做到了秒级的验证，无法识别毫秒、微秒的校验，缺少精确度

Etag 是服务器根据当前文件的内容，对文件生成唯一的标识，比如MD5算法，只要里面的内容有改动，这个值就会修改；

1. 浏览器在发送请求时会带 If-None-Match 字段来询问服务器该版本是否可用；
2. 服务器接收到 If-None-Match 后，会跟服务器上该资源的 ETag 进行比对：
   - 如果两者一样的话，直接返回304，告诉浏览器直接使用缓存；
   - 如果不一样的话，说明内容更新了，返回新的资源；

### 两者对比

1. 性能上，Last-Modified 优于 ETag，Last-Modified 记录的是时间点，而Etag需要根据文件的 MD5 算法生成对应的 hash 值；
2. 精度上，ETag 优于 Last-Modified：ETag 按照内容给资源带上标识，能准确感知资源变化，Last-Modified 某些场景并不能准确感知变化：
  - 编辑了资源文件，但是文件内容并没有更改，也会造成缓存失效；
  - Last-Modified 能够感知的单位时间是秒，如果文件在 1 秒内改变了多次，那么这时候的 Last-Modified 并没有体现出修改了；
3. 如果两种方式都支持的话，服务器会优先考虑 ETag；

## 缓存位置

浏览器默认的缓存是放在内存中的，但是内存里的缓存会因为进程的结束或浏览器的关闭而被清除，存在硬盘里的缓存才能被长期保留下去；

浏览器缓存的位置的话，优先级从高到低排列分别：

### Service Worker

Service worker是一个注册在指定源和路径下的事件驱动worker。它采用JavaScript控制关联的页面或者网站，拦截并修改访问和资源请求，细粒度地缓存资源。Service Worker 可以使你的应用先访问本地缓存资源，包括js、css、png、json等多种静态资源。

当我们启用ServiceWorker时，浏览器就会显示我们的资源来自ServiceWorker。Service Worker的缓存有别与其他缓存机制，我们可以自由控制缓存、文件匹配、读取缓存，并且是持续性的。当ServiceWorker没有命中时才会去重新请求数据。

因为 Service Worker 中涉及到请求拦截，所以必须使用 HTTPS 传输协议来保障安全 它可以让我们自由控制缓存哪些文件、如何匹配缓存、如何读取缓存，并且缓存是持续性的。

Service Worker 实现缓存功能的三个步骤：

1. 首先需要注册 Service Worker
2. 然后监听 install 事件后就可以缓存需要的文件
3. 最后在下次用户访问的时候，通过拦截请求的方式查询是否存在缓存，如果存在就直接读取缓存文件否则就去请求数据

**特点：**

1. 独立于主JavaScript线程（这就意味着它的运行丝毫不会影响我们主进程的加载性能）
2. 设计完全异步,大量使用Promise（因为通常Service Worker通常会等待响应后继续，Promise再合适不过了）
3. 不能访问DOM，不能使用XHR和localStorage
4. Service Worker只能由HTTPS承载(出于安全考虑)



这个应用场景比如 PWA，它借鉴了Web Worker思路，由于它脱离了浏览器的窗体，因此无法直接访问DOM；

它能完成的功能比如：离线缓存、消息推送和网络代理，其中离线缓存就是 Service Worker Cache；

### Memory Cache

Memory Cache为内存中的缓存，读取内存中的资源是非常快的。 是**浏览器**为了加快读取缓存速度而进行的**自身的优化行为**，不受开发者控制，也不受 HTTP 协议头的约束。**内存读取虽然快且高效，但它是短暂的，当浏览器或者tab页面关闭，内存就会被释放了。**而且内存占用过多会导致浏览器占用计算机过大内存。

- 内存中的缓存：读取高效、持续时效短，会随着进程的释放而释放
- 在缓存资源时并不关心返回资源的请求头 Cache-Control 的值是什么，同时资源的匹配也并非仅仅是对 URL 做判断，还可能会对 Content-Type、Cors 等其他特征做校验
- 普通刷新 F5 操作，优先使用 Memory Cache

指的是内存缓存，从效率上讲它是最快的，从存活时间来讲又是最短的，当渲染进程结束后，内存缓存也就不存在了；

### Disk Cache

Disk Cache 将资源存储在硬盘中，读取速度次于Memory Cache。

**优点：**可长期存储，可定义时效时间、容量大；**缺点：**读取速度慢。

根据http header 请求头触发的缓存策略来做缓存，包含强缓存和协商缓存。

- 硬盘中的缓存：对比 Memory Cache 读取速度慢，但是容量大、缓存时效长
- 它根据 HTTP Header 中的字段判断哪些资源需要被缓存、哪些资源可以不请求直接使用、哪些资源已经过期需要重新请求。即使在跨站点的情况下相同地址的资源一旦被硬盘缓存下来就不会再次去请求数据
- 对于大文件来说大概率在硬盘中缓存，反之内存缓存优先；当前系统内存使用率高的情况下文件优先存储进硬盘

存储在磁盘中的缓存，从存取效率上讲是比内存缓存慢的，优势在于存储容量和存储时长；

在浏览器接收到服务器响应后，会检测响应头部，如果有 Etag 字段，那么浏览器就会将本次缓存写入硬盘；

### Push Cache

推送缓存，它是HTTP/2的内容；

### Disk Cache & Memory Cache

- 内容使用率高的话，文件优先进入磁盘；
- 比较大的JS，CSS文件会直接放入磁盘，反之放入内存；

