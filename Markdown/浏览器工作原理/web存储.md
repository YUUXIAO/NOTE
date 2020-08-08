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

### Cookie 的作用

1. 会话状态管理：如用户登陆状态，购物车，游戏分数或者其他需要记录的信息
2. 个性化设置：如用户自定义设置、主题等
3. 浏览器行为跟踪：如跟踪分析用户行为等

### Cookie 的缺点

1. 容量缺陷：Cookie 的体积上限只有4KB，只能用来存储少量的信息；同一个域名下存放的Cookie的个数是有限制的，不同的浏览器的个数不一样，一般为20个
2. 降低性能：Cookie紧跟着域名，不管域名下的某个地址是否需要这个Cookie，请求都会带上完整的Cookie，请求数量增加，会造成巨大的浪费
3. 安全缺陷：Cookie是以纯文本的形式在浏览器和服务器中传递，很容易被非法用户获取，当HTTPOnly为false时，Cookie信息还可以直接通过JS脚本读取。

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

> Domain 指定了 Cookie 可以送达的主机名

注意：

1. 在setcookie中省略domain参数，那么domain默认为当前域名；
2. domain参数可以设置父域名以及自身，但不能设置其它域名，包括子域名，否则cookie不起作用。
3. cookie的作用域是domain本身以及domain下的所有子域名；

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

## LocalStorage

> 一种持久化的存储方式，也就是说如果不手动清除，数据就永远不会过期
>
> 采用键值对的方式存储数据，按域名将数据分别保存到对应数据库文件里

### 特点

- 在同源的所有标签页和窗口之间共享数据
- 数据持久存在且不会过期，重启浏览器后仍然存在
- 存储容量有4.98MB（不同浏览器情况不同，safari 2.49M）
- 保存在客户端，不与服务端交互，节省网络流量
- localStorage其实存储的都是字符串，如果是存储对象需要调用JSON的stringify方法，并且用JSON.parse来解析成对象

### 应用场景

localStorage 适合持久化缓存数据，比如页面的默认偏好配置，如官网的logo，存储Base64格式的图片资源等

## sessionStorage

> SessionStorage是一种会话级别的缓存，关闭浏览器时数据会被清除
>
> SessionStorage的作用域是窗口级别的，也就是说不同窗口之间保存的sessionStorage数据不能共享

### 特点

- 数据只存在于当前浏览器的标签页，关闭页面或浏览器后会被清除
- 存储容量有4.98MB（部分浏览器没有限制）
- 保存在客户端，不与服务端交互，节省网络流量
- 与LocalStorage拥有统一的火火火火火火火火火火ooo接口

### 应用场景

sessionStorage 适合一次性临时数据保存，存储本次浏览信息记录，这样子页面关闭的话，就不需要这些记录了，还有对表单信息进行维护，这样子页面刷新的话，也不会让表单信息丢失。

## IndexedDB

> IndexedDB是一种底层的API，用于客户端存储大量结构化数据，包括文件、二进制大型对象。该API使用索引来实现对该数据的高性能搜索。

### 特点

1. 存储空间大：存储之间可以达到几百兆甚至更多
2. 支持二进制存储：不仅可以存储字符串，而且还可以存储进制数据
3. 同源限制：每一个数据库只能在自身的域名下能访问，不能跨域名访问
4. 支持事务型：IndexedDB执行的操作都会免按照事务来分组的，在一个事务中，要么所有的操作都成功，要么所有的操作都失败
5. 键值对存储： IndexedDB内部采用对象仓库存放数据。所有类型的数据都可以直接存入，包括JavaScript 对象。对象仓库中,数据以键值对的形式保存，每一个数据记录都有对应的主键，主键是独一无二的，不能重复，否则会抛出一个错误
6. 数据操作都是异步的：使用IndexedDB执行的操作都是异步执行的，以免阻塞应用程序

## Web SQL

> Web SQL数据库API实际上不是HTML5规范的一部分，而是一个单独的规范，它引入了一组API来使用SQL来操作客户端数据库。
>
> ~~HTML5已经放弃Web SQL数据库~~

### 特点

1. Web SQL能方便的进行对象存储
2. Web SQL支持事务，能方便地进行数据查询和数据处理操作

### 方法

1. openDatabase: 这个方法使用现有数据库或新建数据库来创建数据库对象
2. transaction: 这个方法允许我们根据情况控制事务的提交或回滚
3. executeSql: 这个方法用于执行真实的SQL语句

