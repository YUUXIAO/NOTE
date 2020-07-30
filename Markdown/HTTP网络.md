## 跨域

> 跨域是指浏览器不能执行其他网站的脚本；
>
> 它是由浏览器的同源策略造成的，是浏览器对Javascript实施的安全限制；

### 同源策略

> 同源策略是一个安全策略。所谓的同源,指的是协议,域名,端口相同。
>
> 浏览器处于安全方面的考虑,只允许本域名下的接口交互,不同源的客户端脚本,在没有明确授权的情况下,不能读写对方的资源。

限制了一下行为：

- Cookie、LocalStorage和IndexDB无法获取
- DOM的JS对象无法获取
- Ajax请求发送不出去

### 解决方案

#### jsonp跨域

> 利用script标签没有跨域限制的漏洞，网页可以拿到从其他来源产生动态JSON数据，当然了JSONP请求一定要对方的服务器做支持才可以 

**与AJAX对比：**

JSONP和AJAX相同，都是客户端向服务器发送请求，从服务器获取数据的方式。但是AJAX属于同源策略，JSONP属于非同源策略(跨域请求)

**思路：**

1. 创建script标签
2. 设置script标签的src属性，以问号传递参数，设置好回调函数callback名称
3. 插入html文本中
4. 调用回调函数，res参数就是获取的数据

**优点：**

- 不受同源策略的限制
- 兼容性更好，在更加古老的浏览器中都可以运行，不需要XMLHttpRequest或ActiveX的支持
- 在请求完毕后可以通过调用callback的方式回传结果

**缺点：**

- 只支持GET请求而不支持POST等其它类型的HTTP请求
- 不安全，可能会受到XSS攻击
- 只支持跨域HTTP请求这种情况，不能解决不同域的两个页面之间如何进行JavaScript调用的问题

#### 跨域资源共享 CORS

> CORS（Cross-Origin Resource Sharing）跨域资源共享，定义了必须在访问跨域资源时，浏览器与服务器应该如何沟通。
>
> CORS的基本思想就是使用自定义的HTTP头部让浏览器与服务器进行沟通，从而决定请求或响应是应该成功还是失败。
>
> 目前，所有浏览器都支持该功能，IE浏览器不能低于IE10。
>
> 整个CORS通信过程，都是浏览器自动完成，不需要用户参与。对于开发者来说，CORS通信与同源的AJAX通信没有差别，代码完全一样。浏览器一旦发现AJAX请求跨源，就会自动添加一些附加的头信息，有时还会多出一次附加的请求，但用户不会有感觉。

- CORS 需要浏览器和后端同时支持。IE 8 和 9 需要通过 XDomainRequest 来实现
- 浏览器会自动进行 CORS 通信，实现 CORS 通信的关键是后端。只要后端实现了 CORS，就实现了跨域
- 服务端设置 Access-Control-Allow-Origin 就可以开启 CORS。 该属性表示哪些域名可以访问资源，如果设置通配符则表示所有网站都可以访问资源
-  复杂请求的CORS请求，会在正式通信之前，增加一次HTTP查询请求，称为"预检"请求,该请求是 option 方法的，通过该请求来知道服务端是否允许跨域请求。

满足下面两个条件，就属于简单请求，否则为复杂请求

1. 使用方法为：GET、POST、HEAD


1. 使用方法为：GET、POST、HEAD


2. Content-Type 的值仅限：text/plain、multipart/form-data、application/x-www-form-urlencoded

#### WebSocket协议跨域

> Websocket是HTML5的一个持久化的协议，它实现了浏览器与服务器的全双工通信，同时也是跨域的一种解决方案。
>
> WebSocket和HTTP都是应用层协议，都基于 TCP 协议。但 WebSocket 是一种双向通信协议，在建立连接之后，WebSocket 的 server 与 client 都能主动向对方发送或接收数据。
>
> WebSocket 在建立连接时需要借助 HTTP 协议，连接建立好了之后 client 与 server 之间的双向通信就与 HTTP 无关了。

示例：

```javascript
// socket.html
<script>
    let socket = new WebSocket('ws://localhost:3000');
    socket.onopen = function () {
      socket.send('我爱你');//向服务器发送数据
    }
    socket.onmessage = function (e) {
      console.log(e.data);//接收服务器返回的数据
    }
</script>
```

#### nginx反向代理

## XSS攻击

> XSS的全称是 Cross Site Scriptng ,为了与CSS区分开，所以简称XSS
>
> XSS是指黑客往HTML文件中或者DOM中注入恶意脚本，从而在用户浏览页面时利用注入的恶意脚本对用户实施攻击的一种手段

注入恶意脚本可以完成这些事情：

1. 窃取Cookie
2. 监听用户行为，比如输入账号密码后发给黑客服务器
3. 在网页中生成浮窗广告
4. 修改DOM伪造登入表单

XSS攻击有三种实现方式：

- 存储型XSS攻击

  1. 首先黑客利用站点漏洞将一段恶意 JavaScript 代码提交到网站的数据库中；
  2. 然后用户向网站请求包含了恶意 JavaScript 脚本的页面；
  3. 当用户浏览该页面的时候，恶意脚本就会将用户的 Cookie 信息等数据上传到服务器。

  场景：

  在评论区提交一份脚本代码，假设前后端没有做好转义工作，那内容上传到服务器，在页面渲染的时候就会直接执行，相当于执行一段未知的JS代码。这就是存储型 XSS 攻击。

- 反射型XSS攻击

  > 反射型 XSS 攻击指的就是恶意脚本作为网络请求的一部分，随后网站又把恶意的JavaScript脚本返回给用户，当恶意 JavaScript 脚本在用户页面中被执行时，黑客就可以利用该脚本做一些恶意操作。

  场景：

  ```javascript
  http://TianTianUp.com?query=<script>alert("你受到了XSS攻击")</script>
  ```

  如上，服务器拿到后解析参数query，最后将内容返回给浏览器，浏览器将这些内容作为HTML的一部分解析，发现是Javascript脚本，直接执行，这样子被XSS攻击了。

  这也就是反射型名字的由来，将恶意脚本作为参数，通过网络请求，最后经过服务器，在反射到HTML文档中，执行解析。

- 基于DOM的XSS攻击

  > 基于 DOM 的 XSS 攻击是不牵涉到页面 Web 服务器的。具体来讲，黑客通过各种手段将恶意脚本注入用户的页面中，在数据传输的时候劫持网络数据包

  常见的劫持手段有：

  - WIFI路由器劫持
  - 本地恶意软件

#### 阻止 XSS 攻击的策略

1. 对脚本进行过滤或转码
2. ​

<https://juejin.im/post/5f184aade51d4534aa4ad7c0#heading-72>