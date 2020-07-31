## Cookie

> HTTP是一个无状态的协议，主要指的是HTTP1.x版本，为了解决 HTTP 无状态导致的问题（HTTP1.x），后来出现了 Cookie。
>
> Cookie 的存在也不是为了解决通讯协议无状态的问题，只是为了解决客户端与服务端会话状态的问题，这个状态是指后端服务的状态而非通讯协议的状态。
>
> Cookie存放在本地的好处就在于即使你关闭了浏览器，Cookie 依然可以生效。
>
> 默认保存在内存中，随浏览器关闭失效（如果设置过期时间，在到过期时间后失效）

### Cookie设置

1. 客户端发送 HTTP 请求到服务器
2. 当服务器收到HTTP请求时，在响应头里面添加一个Set-Cookie字段
3. 浏览器收到响应后保存下Cookie
4. 之后对该服务器每一次请求中都通过Cookie字段将Cookie信息发送给服务器

### Cookie指令

#### Expires/Max-Age

假如 Expires 和 Max-Age 都存在，Max-Age 优先级更高。

> Expires 用于设置 Cookie 的过期时间
>
> ```javascript
> Set-Cookie: id=aad3fWa; Expires=Wed, 21 May 2020 07:28:00 GMT;
> ```

> Max-Age 用于设置在 Cookie 失效之前需要经过的秒数
>
> ```javascript
> Set-Cookie: id=a3fWa; Max-Age=604800;
> ```

- 当Expires属性缺省时，表示是会话性Cookie
- 当 Expires 的值为 Session，表示是会话性 Cookie
- 会话性 Cookie 的时候，值保存在客户端内存中，决不会写入磁盘，并在用户关闭浏览器时失效
- 与会话性 Cookie 相对的是持久性 Cookie，持久性 Cookies 会保存在用户的硬盘中，直至过期或者清除 Cookie。

#### Domain

> Domain 指定了 Cookie 可以送达的主机名。假如没有指定，那么默认值为当前文档访问地址中的主机部分

#### Path

> Path 指定了一个 URL 路径，这个路径必须出现在要请求的资源的路径中才可以发送 Cookie 首部
>
> 比如设置 “ Path=/docs ”，“ /docs/Web/ ” 下的资源会带 Cookie 首部，“ /test ” 则不会携带 Cookie 首部

Domain 和 Path 标识共同定义了 Cookie 的作用域：即 Cookie 应该发送给哪些 URL；

#### Secure

> 标记为 Secure 的 Cookie 只应通过被HTTPS协议加密过的请求发送给服务端
>
> 使用 HTTPS 安全协议，可以保护 Cookie 在浏览器和 Web 服务器间的传输过程中不被窃取和篡改

#### HTTPOnly

> HTTPOnly 属性可以防止客户端脚本通过 document.cookie 等方式访问 Cookie，有助于避免 XSS 攻击

#### SameSite

> SameSite 属性可以让 Cookie 在跨站请求时不会被发送，从而可以阻止跨站请求伪造攻击（CSRF）

SameSite可以设置为三个值，Strict、Lax和None。

- 在Strict模式下，浏览器完全禁止第三方请求携带Cookie。比如请求sanyuan.com网站只能在sanyuan.com域名当中请求才能携带 Cookie，在其他网站请求都不能。
- 在Lax模式，就宽松一点了，但是只能在 get 方法提交表单或者a 标签发送 get 请求的情况下可以携带 Cookie，其他情况均不能。
- 在None模式下，Cookie将在所有上下文中发送，即允许跨域发送。

### Cookie 的作用

1. 会话状态管理：如用户登陆状态，购物车，游戏分数或者其他需要记录的信息
2. 个性化设置：如用户自定义设置、主题等
3. 浏览器行为跟踪：如跟踪分析用户行为等

### Cookie 的缺点

1. 容量缺陷：Cookie 的体积上限只有4KB，只能用来存储少量的信息
2. 降低性能：Cookie紧跟着域名，不管域名下的某个地址是否需要这个Cookie，请求都会带上完整的Cookie，请求数量增加，会造成巨大的浪费
3. 安全缺陷：Cookie是以纯文本的形式在浏览器和服务器中传递，很容易被非法用户获取，当HTTPOnly为false时，Cookie信息还可以直接通过JS脚本读取。

## LocalStorage

- 理论上永久有效的，除非主动清除
- 存储容量有4.98MB（不同浏览器情况不同，safari 2.49M）
- 保存在客户端，不与服务端交互，节省网络流量
- localStorage其实存储的都是字符串，如果是存储对象需要调用JSON的stringify方法，并且用JSON.parse来解析成对象

### 应用场景

localStorage 适合持久化缓存数据，比如页面的默认偏好配置，如官网的logo，存储Base64格式的图片资源等

## sessionStorage

- 仅在当前网页会话下有效，关闭页面或浏览器后会被清除
- 存储容量有4.98MB（部分浏览器没有限制）
- 保存在客户端，不与服务端交互，节省网络流量

### 应用场景

sessionStorage 适合一次性临时数据保存，存储本次浏览信息记录，这样子页面关闭的话，就不需要这些记录了，还有对表单信息进行维护，这样子页面刷新的话，也不会让表单信息丢失。