## 强缓存（200 OK from cache）

强缓存**返回的状态码是200**，直接从缓存内读取，浏览器不用询问服务器

**强缓存可以理解为是浏览器层面的**，它的依据来自于缓存是否过期，而不关心服务端文件是否已经更新，这可能会导致加载的文件不是服务端最新的内容，所以一般用来缓存一些对准确性要求不高或者长时间不修改的数据

![](../images/%E6%B5%8F%E8%A7%88%E5%99%A8%E7%BC%93%E5%AD%98_%E5%BC%BA%E7%BC%93%E5%AD%98.png)



### Expires（HTTP1.0）

`Expires` 指的是过期时间（缓存的截止时间），时间是相对于服务器的时间而言的，存在于服务端返回的响应头中，允许客户端在过期时间之前可以直接从缓存里面获取数据，无需再次请求；

```javascript
// 该资源在2023年7月29日11:10:23过期，过期就会重新向服务器发起请求
Expires:Mon, 29 Jun 2023 11:10:23 GMT
```

这个方式有一个问题：返回的到期时间是服务器端的时间，服务器的时间和浏览器的时间可能并不一致，所以HTTP1.1 提出新的字段代替它；

### Cache-Control（HTTP1.1）

`Cache-Control` 是过期时长

当 `Expires` 和 `Cache-Control` 同时存在时，优先考虑 `Cache-Control`；

```javascript
// 资源返回后6000秒内，可以直接使用缓存
Cache-Control:max-age=6000
```

常见的请求头中的 Cache-Control 指令：

```javascript
Cache-Control: max-age=<seconds>  // 缓存存储的最大周期，单位为秒
Cache-Control: s-maxage=<seconds>  // 和max-age是一样的，不过它只针对代理服务器缓存而言；
Cache-Control: max-stale[=<seconds>] // 客户端愿意接收一个已经过期的资源
Cache-control: no-cache // 重新验证是否缓存(协商缓存验证)，强制客户端直接向服务器发送请求，不会走强缓存的逻辑，服务器接收到请求后判断资源是返回新内容还是304
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

**协商缓存最终由服务器来决定** 是否使用缓存，即客户端与服务器之间存在一次通信，如果命中走缓存的话，**返回的状态码是 304 (not modified)**

在浏览器向服务端请求资源时，响应头会有一个`Last-Modified`的属性标记此文件在服务期端最后被修改的时间，另外会也有个`Etag`：

```javascript
Last-Modified:Fri, 15 Feb 2022 03:06:18 GMT
ETag:"be15b26c29bce1:0" // 可选，这里为了准确确认资源是否变化
```

客户端后续请求同个资源时，请求头上会携带 `If-Modified-Since`和`If-None-Match`(Etag的值)，询问该时间之后文件是否有被修改过：

```javascript
If-Modified-Since:Sat, 16 Feb 2022 07:30:07 GMT
If-None-Match:"be15b26c29bce1:0" // 可选，这里为了准确确认资源是否变化
```

如果服务器端的资源没有变化，则自动返回 HTTP 304 （Not Changed）状态码内容为空，否则重新发起请求，请求下载资源这样就节省了传输数据量。

![](../images/%E6%B5%8F%E8%A7%88%E5%99%A8%E7%BC%93%E5%AD%98_%E5%8D%8F%E5%95%86%E7%BC%93%E5%AD%98.png)

### Last-Modified

`Last-Modified` 是响应头字段用来标识资源的有效性，表示的是最后修改时间  
对应浏览器再次请求时请求头带上的 `If-Modified-Since`

```javascript
Last-Modified:Fri, 15 Feb 2013 03:06:18 GMT
```

### Etag

`Etag` 是响应头字段表示资源的版本，对应浏览器再次请求时请求头带上的 `If-None-Match`

Etag 是服务器根据当前文件的内容，对文件生成唯一的标识，比如MD5算法，只要里面的内容有改动，这个值就会修改；  
因为`Last-Modified` 只做到了秒级的验证，无法识别毫秒、微秒的校验，缺少精确度

- 服务器接收到 If-None-Match 后，会跟服务器上该资源的 ETag 进行比对：

  - 如果两者一样的话，直接返回304，告诉浏览器直接使用缓存；
  - 如果不一样的话，说明内容更新了，返回新的资源；


### 两者对比

1. **性能上Last-Modified 优于 ETag**：Last-Modified 记录的是时间点，而Etag需要根据文件的 MD5 算法生成对应的 hash 值；
2. **精度上ETag 优于 Last-Modified：**ETag 按照内容给资源带上标识能准确感知资源变化，Last-Modified 某些场景并不能准确感知变化：

   - 编辑了资源文件，但是文件内容并没有更改，也会造成缓存失效；
   - Last-Modified 能够感知的单位时间是秒，如果文件在 1 秒内改变了多次，那么这时候的 Last-Modified 并没有体现出修改了；
   - 如果两种方式都支持的话，服务器会优先考虑 ETag；


## 缓存位置

浏览器默认的缓存是放在内存中的，但是内存里的缓存会因为进程的结束或浏览器的关闭而被清除，存在硬盘里的缓存才能被长期保留下去；

浏览器缓存的位置的话，优先级从高到低排列分别：

### Service Worker

Service worker是一个注册在**指定源和路径下的事件驱动worker**。它采用JS控制关联的页面或者网站，拦截并修改访问和资源请求，细粒度地缓存资源。

它能完成的功能比如：离线缓存、消息推送和网络代理，其中离线缓存就是 Service Worker Cache；

Service Worker 的特点：

- 独立于主 Javascript 线程，也就是说运行时不会影响主进程的加载性能
- 设计完全异步，大量使用 Promise
- 不能访问 DOM，不能使用 xhr 和 localstorage（感觉有点像 web worker，脱离了浏览器的窗体）
- 有别与其他缓存机制，我们可以自由控制缓存、文件匹配、读取缓存，并且是持续性的
- 因为 Service Worker 中涉及到请求拦截，所以必须使用 HTTPS 传输协议来保障安全

Service Worker 实现缓存功能的三个步骤：

1. 首先需要**注册 Service Worker（ServiceWorkerContainer.register()）**

```javascript
if ('serviceWorker' in navigator) {
    navigator.serviceWorker.register(‘/xxx-sw.js', {scope: '/'}).then(function(reg) {
	// reg可以查看当前sw的状态和作用域等 
    }).catch(function(error) {
        // registration failed
        console.log('Registration failed with ' + error);
    });
}
```

2. 然后监听 install 事件后就可以缓存需要的文件
3. 最后在下次用户访问的时候，通过拦截请求的方式查询是否存在缓存，如果存在就直接读取缓存文件否则就去请求数据

附上一段代码

```javascript

// sw.js
const SW_VERSION = 'V1';
const CACHE_FILE_TYPE = [ 'js','css', 'html','jpg','json','png''mp3','wav','mp4','ttf'];
//需要确认缓存的文件
const CACHE_FILE_LIST = [];
// 需要忽悠的文件列表
const IGNORE_FILE_LIST = [
  '/test/index.js',
];
/**
 * 是否是对应的文件类型
 * @param {*} url 
 */
function isAcceptFile(url) {
  var r = new RegExp("\\.(" + CACHE_FILE_TYPE.join('|') + ")$");
  return r.test(url);
}
/**
 * 检查文件名
 */
function checkIgnoreFileName(url) {
  var r = new RegExp("(" + IGNORE_FILE_LIST.join('|') + ")$");
  return r.test(url);
}
self.addEventListener('install', function(event) {
    event.waitUntil(
      caches.open(SW_VERSION).then(function(cache) {
        return cache.addAll(CACHE_FILE_LIST);
      })
    );
  });
self.addEventListener('activate', function(event) {
    var cacheWhitelist = [SW_VERSION];
    event.waitUntil(
      caches.keys().then(function(keyList) {
        return Promise.all(keyList.map(function(key) {
          if (cacheWhitelist.indexOf(key) === -1) {
            return caches.delete(key);
          }
        }));
      })
    );
  });
// 监听浏览器的所有fetch请求，对已缓存的资源使用本地缓存回复  
self.addEventListener('fetch', function(event) {
    const {method, url} = event.request;
    event.respondWith(
      caches.match(event.request).then(function(response) {
          if (response !== undefined) {
              return response;  // 找到缓存了，直接返回
          } else {
              return fetch(event.request).then(function (response) {
                  let responseClone = response.clone();
                  if (method === 'POST') {
                    return response
                  }
                  if (!isAcceptFile(url)) {
                    return response
                  }
                  if (checkIgnoreFileName(url)) {
                    return response
                  }
                  caches.open(SW_VERSION).then(function (cache) {
                    cache.put(event.request, responseClone); // 写入缓存
                  });
                  return response;
              }).catch(function (error) {
                 return Promise.reject(error);
              });
          }
      })
    );
  });

```

### Memory Cache（内存缓存）

内存缓存从效率上来讲是最快的，但是也是存活时间最短的，当浏览器或者tab页面关闭，内存就会被释放了

- 这是**浏览器**为了加快读取缓存速度而进行的**自身的优化行为**，不受开发者控制，也不受 HTTP 协议头的约束
- 在缓存资源时并不关心返回资源的请求头 Cache-Control 的值是什么，同时资源的匹配也并非仅仅是对 URL 做判断，还可能会对 Content-Type、Cors 等其他特征做校验
- 普通刷新 F5 操作，优先使用 Memory Cache

### Disk Cache

Disk Cache 将资源存储在硬盘中，读取速度次于Memory Cache。可长期存储，可定义时效时间、容量大但是读取速度慢。

在浏览器接收到服务器响应后，会检测响应头部，如果有 Etag 字段，那么浏览器就会将本次缓存写入硬盘；

- 根据http header 请求头触发的缓存策略来做缓存，包含强缓存和协商缓存
- 它根据 HTTP Header 中的字段判断哪些资源需要被缓存、是否直接使用、是否过期需要重新请求；即使在跨站点的情况下相同地址的资源一旦被硬盘缓存下来就不会再次去请求数据
- 对于大文件来说大概率在硬盘中缓存，反之内存缓存优先；当前系统内存使用率高的情况下文件优先存储进硬盘

### Push Cache

推送缓存，它是HTTP/2的内容；  
它只在会话（Session）中存在，一旦会话结束就被释放，并且缓存时间也很短暂。这种缓存一般用不到

### Disk Cache & Memory Cache

- 内容使用率高的话，文件优先进入磁盘；
- 比较大的JS，CSS文件会直接放入磁盘，反之放入内存；