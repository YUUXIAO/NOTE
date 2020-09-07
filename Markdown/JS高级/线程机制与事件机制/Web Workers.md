---
typora-root-url: ..\..\..
---

1. Web Workers 是 HTML5 提供的一个javascript多线程解决方案；
2. 我们可以将一些大计算量的代码交由web Worker运行而不冻结用户界面；
3. 但是子线程完全受主线程控制，且不得操作DOM，所以，这个新标准并没有改变JavaScript单线程的本质；

## 理解

> H5规范提供了js分线程的实现, 取名为: Web Workers；

### 相关API

1. Worker: 构造函数, 加载分线程执行的js文件；
2. Worker.prototype.onmessage: 用于接收另一个线程的回调函数；
3. Worker.prototype.postMessage: 向另一个线程发送消息；

### 缺点

1.  worker内代码不能操作DOM(更新UI)；
2.  不能跨域加载JS；
3.  不是每个浏览器都支持这个新特性；

## 使用

1. 创建在分线程执行的js文件；

```javascript
//  不能用函数声明
var onmessage =function (event){ 
  console.log('onMessage()22');
  //  通过event.data获得发送来的数据
  var upper = event.data.toUpperCase();
  //  将获取到的数据发送会主线程
  postMessage( upper );
}
```

在主线程中的js中发消息并设置回调；

```javascript
//创建一个Worker对象并向它传递将在新线程中执行的脚本的URL
var worker = new Worker("worker.js");  
//接收worker传过来的数据函数
worker.onmessage = function (event) {     
    console.log(event.data);             
};
//向worker发送数据
worker.postMessage("hello world");    
```

## 图解

![web_workers](/images/事件机制/web_workers.jpg)

## 应用场景

1. 计算得到fibonacci数列中第n个数的值；
2. 在主线程计算: 当位数较大时, 会阻塞主线程, 导致界面卡死；
3. 在分线程计算: 不会阻塞主线程；

```javascript
// index.html
var input = document.getElementById('number')
document.getElementById('btn').onclick = function () {
  var number = input.value;

  //创建一个Worker对象
  var worker = new Worker('worker.js')
  // 绑定接收消息的监听
  worker.onmessage = function (event) {
    console.log('主线程接收分线程返回的数据: '+event.data)
    alert(event.data)
  }

  // 向分线程发送消息
  worker.postMessage(number)
  console.log('主线程向分线程发送数据: '+number)
}

// worker.js
function fibonacci(n) {
  return n<=2 ? 1 : fibonacci(n-1) + fibonacci(n-2)  //递归调用
}

console.log(this)
this.onmessage = function (event) {
  var number = event.data
  console.log('分线程接收到主线程发送的数据: '+number)
  //计算
  var result = fibonacci(number)
  postMessage(result)
  console.log('分线程向主线程返回数据: '+result)
  // 分线程中的全局对象不再是window, 所以在分线程中不可能更新界面
}
```

