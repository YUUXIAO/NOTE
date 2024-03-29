Web 端即时通讯技术是为了实现服务器端可以即时地将数据的更新或者变化反应到客户端，例如消息推送；

## 短轮询

### 基本思路

浏览器每隔一段时间向服务器发送 http 请求，服务器在收到请求后，不论是否有数据更新，都直接进行回应；

这种方式实现的即时通信，本质上是浏览器发送请求，服务器接受请求的一个过程，通过让客户端不断的请求，使得客户端能够模拟实时地收到服务器端的数据变化；

### 优缺点

1. 优点是比较简单，易于理解；
2. 缺点是这种方式需要不断的建立 http 连接，严重浪费了服务器和客户端的资源，人数越多，服务器端压力越大；

```javascript
var xhr = new XMLHttpRequest();
setInterval(function(){
    xhr.open('GET','/user');
    xhr.onreadystatechange = function(){

    };
    xhr.send();
},1000)
```

## 长轮询

> 发送请求给服务端，直到服务端觉得可以返回数据了再返回响应，否则这个请求一直挂起；

### 基本思路

1. 首先由客户端向服务器发起请求，当服务器收到客户端发来的请求后，服务端不会直接进行响应，而是先将这个请求挂起，然后判断服务器端数据是否有更新；
2. 如果有更新则进行响应，如果一直没有数据，则达到一定的时间限制才返回；
3. 客户端在响应处理函数会在处理完服务器返回的信息后，再次发出请求重新建立连接；

### 优缺点

1. 长轮询和短轮询比起来，明显减少了很多不必要的 http 请求次数，节约了资源；
2. 长轮询的缺点在于，连接挂起也会导致资源的浪费；

```javascript
function ajax(){
    var xhr = new XMLHttpRequest();
    xhr.open('GET','/user');
    xhr.onreadystatechange = function(){
          ajax();
    };
    xhr.send();
}
```

## 轮询的缺陷

短轮询和长轮询都是基于 HTTP 的，两者本身都存在一个缺陷：

1. 短轮询需要更快的处理速度；
2. 长轮询则更要求处理并发的能力；两者都是“被动型服务器”，服务器不会主动发送信息，而是在客户端发送 ajax 请求后进行返回的响应；

## 长连接（SSE）

> 长连接（Server-Sent Events）是 html5 新增的功能，它可以允许服务器推送数据到客户端；

SSE 本质上与轮询不同，虽然都是基于 http 协议的，但是轮询需要客户端先发送请求，而 SSE 不需要客户端发送请求，可以实现只要服务器端数据有更新，就可以马上发送到客户端；

SSE 的优势很明显，它不需要建立或者保持大量的客户端发往服务端的请求，节约了很多的资源，提升应用性能；

### WebSocket

> Websocket 是 html5 定义的一个新协议，与传统的 http 协议不同是该协议允许由服务器主动向客户端推送信息；

1. webSockets 的目标是在一个单独的持久连接上提供全双工、双向通信；
2. 在 JS 创建了Web Socket之后，会有一个HTTP请求发送到浏览器以发起连接；
3. 在取得服务器响应后，建立的连接会将 HTTP 升级从 HTTP 协议交换为WebSocket 协议，后面的数据交互都复用这个TCP通道；

```javascript
const ws = new WebSocket('ws://localhost:8080');
// 连接建立事件
ws.onopen = function () {
    ws.send('123')
    console.log('open')
}
// 接收数据事件
ws.onmessage = function () {
    console.log('onmessage')
}
// 连接错误事件
ws.onerror = function () {
    console.log('onerror')
}
// 连接关闭事件
ws.onclose = function () {
    console.log('onclose')
}
```