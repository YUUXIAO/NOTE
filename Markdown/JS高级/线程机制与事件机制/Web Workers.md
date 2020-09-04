---
typora-root-url: ..\..\..
---

1. Web Workers 是 HTML5 提供的一个javascript多线程解决方案；
2. 我们可以将一些大计算量的代码交由web Worker运行而不冻结用户界面；
3. 但是子线程完全受主线程控制，且不得操作DOM，所以，这个新标准并没有改变JavaScript单线程的本质；

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

## 缺点

1. 速度慢；
2. 不能跨域加载 JS ；
3. worker 内代码不能访问 DOM （更新UI）;
4. 不是每个浏览器都支持这个新特性；