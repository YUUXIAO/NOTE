js 主线程是负责渲染UI，执行 js代码和处理用户交互的单个执行上下文，因为js是单线程的，所以任何耗时的任务都会阻塞线程

Web Worker（工作线程） 是 HTML5 提供的一个js多线程解决方案，可以将一些耗时的数据处理操作从主线程中剥离再将结果返回，使主线程更加专注于页面渲染和交互；

但是子线程完全受主线程控制，且不得操作DOM，所以这个新标准并没有改变js单线程的本质；

## 用途

我们可以将一些大计算量的代码交由 web Worker 运行而不冻结用户界面；

- 懒加载
- 文本分析
- 流媒体数据处理
- canvas 图形绘制
- 图像处理

## 线程创建

因为 Web Worker 有同源限制，所以在本地调试的时候也需要通过启动本地服务器的方式访问，使用  file:// 协议直接打开的话将会抛出异常；

### 专用线程

专用线程由 Worker( ） 方法创建，接收两个参数，第一个参数是必填的脚本的位置，第二个参数是可选的配置对象，可以指定 type、credentials 和 name 三个属性；

```javascript
var worker = new Worker('worker.js')

var worker2 = new Worker('worker.js', { name: 'dedicatedWorker'})
```

### 共享线程

共享线程使用 Shared Worker( ) 方法创建，接收参数与 Worker( ) 一致；

```javascript
var sharedWorker = new SharedWorker('shared-worker.js')
```

## 数据传递

在 Worker 线程中，self 和 this 都代表子线程的全局对象；

主线程通过 MessagePort 访问专用线程和共享线程；

- 专用线程的 port 会在线程创建时自动设置，并且不会暴露出来；
- 共享线程在传递消息之前，端口必须处于打开状态；

```javascript
var sharedWorker = new SharedWorker('shared-worker.js')
sharedWorker.port.addEventListener('message', function(e) {
    // 业务逻辑
}, false)
sharedWorker.port.start() // 需要显式打开
```

在传递消息时，postMessage()  方法和  onmessage 事件必须通过端口对象调用；

在 Worker 线程中，需要使用 onconnect 事件监听端口的变化，并使用端口的消息处理函数进行响应；

```javascript
// 主线程
sharedWorker.port.postMessage([10, 24])
sharedWorker.port.onmessage = function (e) {
    console.log(e.data)
}

// Worker 线程
onconnect = function (e) {
    let port = e.ports[0]

    port.onmessage = function (e) {
        if (e.data.length > 1) {
            port.postMessage(e.data[1] - e.data[0])
        }
    }
}
```

### 相关API

- Worker.prototype.onmessage: 用于接收另一个线程的回调函数；

- Worker.prototype.postMessage: 向另一个线程发送消息；

- 一次只能发送一个对象，如果需要发送多个参数可以将参数包装为数组或对象再进行传递；

```javascript
// 主线程
var worker = new Worker('worker.js')
worker.postMessage([10, 24])
worker.onmessage = function(e) {
   console.log(e.data)
}

// Worker 线程
onmessage = function (e) {
  if (e.data.length > 1) {
    postMessage(e.data[1] - e.data[0])
  }
}
```

## 关闭Worker

> 可以在主线程中使用 terminate() 方法或在 Worker 线程中使用 close() 方法关闭 worker；

Worker 线程一旦关闭 Worker 后 Worker 将不再响应；

```javascript
// 主线程
worker.terminate()

// Dedicated Worker 线程中
self.close()

// Shared Worker 线程中
self.port.close()
```

## 错误处理

> 可以通过在主线程或 Worker 线程中设置 onerror 和 onmessageerror 的回调函数对错误进行处理；

- onerror 在 Worker 的 error 事件触发并冒泡时执行；
- onmessageerror 在 Worker 收到的消息不能进行反序列化时触发；

```javascript
// 主线程
worker.onerror = function () {
    // ...
}

// 主线程使用专用线程
worker.onmessageerror = function () {
    // ...
}

// 主线程使用共享线程
worker.port.onmessageerror = function () {
    // ...
}

// worker 线程
onerror = function () {
	// ...
}

```

## 加载外部脚本

> Web Worker 提供了 importScripts() 方法，能够将外部脚本文件加载到 Worker 中；

```javascript
importScripts('script1.js')
importScripts('script2.js')

// 以上写法等价于
importScripts('script1.js', 'script2.js')
```

## 子线程

> Worker 可以生成子 Worker；

- 子 Worker 必须与父网页同源；
- 子 Worker 中的 URI 相对于父 Worker 所在的位置进行解析；

## 嵌入式 Worker

> 可以通过 Blob() 将页面中的 Worker 代码进行解析；

```javascript
<script id="worker" type="javascript/worker">
// 这段代码不会被 JS 引擎直接解析，因为类型是 'javascript/worker'

// 在这里写 Worker 线程的逻辑
</script>
<script>
    var workerScript = document.querySelector('#worker').textContent
    var blob = new Blob(workerScript, {type: "text/javascript"})
    var worker = new Worker(window.URL.createObjectURL(blob))
</script>

```

## postMessage

> Web Worker 中，Worker 线程和主线程之间使用结构化克隆算法进行数据通信；

结构化克隆算法是一种通过递归输入对象构建克隆的算法，算法通过保存之前访问过的引用的映射，避免无限遍历循环；

这一过程可以理解为，在发送方使用类似 JSON.stringfy() 的方法将参数序列化，在接收方采用类似 JSON.parse() 的方法反序列化；

一次数据传输就需要同时经过序列化和反序列化，如果数据量大的话，这个过程本身也可能造成性能问题。因此 Worker 中提出了  Transferable Objects 的概念，当数据量较大时，可以选择在将主线程中的数据直接移交给 Worker 线程，这种转移是彻底的，一旦数据成功转移，主线程将不能访问该数据。这个移交的过程仍然通过 postMessage 进行传递；

```javascript
postMessage(message, transferList)

// 示例
let aBuffer = new ArrayBuffer(1)
worker.postMessage({ data: aBuffer }, [aBuffer])
```

### 上下文

Worker 工作在一个 WorkerGlobalDataScope 的上下文中；

每一个 WorkerGlobalDataScope 对象都有不同的 event loop，这个 event loop 没有关联浏览器上下文，它的任务队列也只有事件、回调和联网的活动；

每一个 WorkerGlobalDataScope 都有一个 closing 标志，当这个标志设为 true 时，任务队列将丢弃之后试图加入任务队列的任务，队列中已经存在的任务不受影响（除非另有指定），同时，定时器将停止工作，所有挂起的后台任务将会被删除；

## Worker 中可以使用的函数和类

由于 Worker 工作的上下文不同于普通的浏览器上下文，因此不能访问 window 以及 window 相关的 API，也不能直接操作 DOM；

Worker 中提供了 WorkerNavigator 和 WorkerLocation 接口，它们分别是 window 中 Navigator 和 Location 的子集；

### 时间相关

- clearInterval()
- clearTimeout()
- setInterval()
- setTimeout

### Worker 相关

- importScripts()
- close()
- postMessage()

### 存储相关

- Cache
- IndexedDB

### 网络相关

- Fetch
- WebSocket
- XMLHttpRequest

## 缺点

- **有同源策略限制：**共享线程可以被多个浏览上下文调用，所有这些浏览上下文必须同源；
- 无法访问 DOM 节点；
- 运行在另一个上下文中，无法使用 Window 对象；
- 不是每个浏览器都支持这个新特性；
- Web Worker 的运行不会影响主线程，但与主线程交互时仍受到主线程单线程的瓶颈制约，如果 Worker 线程频繁与主线程进行交互，主线程由于需要处理交互，仍有可能使页面发生阻塞；
- **通信开销：**Web Worker使用postMessage 方法与主线程进行通信，所以是有通信开销的，为了最大限度的减少这种开销，应该只发送必要数据，避免频繁发数据

## 图解

![](../../images/%E4%BA%8B%E4%BB%B6%E6%9C%BA%E5%88%B6/web_workers.jpg)

## 应用场景

### 处理网络请求

当项目中需要发起大量的网络请求，在主线程中执行这些请求可能会导致界面卡顿

上面说到在 worker 可以使用网络请求相关api，所以如果项目中有不限制调用时机和返回的接口，可以从这里处理：

```javascript
// 主线程代码
const worker = new Worker('worker.js');
worker.onmessage = function(event) {
  const response = event.data;
  console.log(response);
};
const ajaxurls = [...]
worker.postMessage({ urls: ajaxurls });


// webworker.js
function request(url) {
  return fetch(url).then(response => response.json());
}

onmessage = async function(event) {
  const urls = event.data.urls;
  const results = await Promise.all(urls.map(request));
  postMessage(results);
};

```

### 并行处理

这个一般是指如果有些业务场景需要大量的独立计算或处理数据量过大，如果在主线程计算过大，用户体验受到影响，这时候可以考虑Web Worker计算