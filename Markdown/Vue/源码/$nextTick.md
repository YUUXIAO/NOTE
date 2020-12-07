## 异步更新

Vue官网对数据操作是这么描述的：

> Vue 在更新 DOM 时是异步执行的。只要侦听到数据变化，Vue 将开启一个队列，并缓冲在同一事件循环中发生的所有数据变更。如果同一个 watcher 被多次触发，只会被推入到队列中一次。这种在缓冲时去除重复数据对于避免不必要的计算和 DOM 操作是非常重要的。然后，在下一个的事件循环“tick”中，Vue 刷新队列并执行实际 (已去重的) 工作；

Vue 中的 dom 不是实时的，当数据改变后，vue 会把 渲染 watcher 添加到异步队列，异步执行，同步代码执行完成后再统一修改 dom；

- 把传入的 cb 回调函数用 try-catch 包裹后放在一个匿名函数中推入 callbacks 数组中，这么做是防止单个 cb 如果执行错误不至于让整个JS 线程挂掉，每个 cb 都包裹是防止这些回调函数如果执行错误不会相互影响；

## 前置知识

### JS 运行机制

JS 运行机制简单来说可以按以下几个步骤：

1. 所有的同步任务都会在主线程上执行，形成一个执行栈；
2. 主线程之外还存在一个任务队列，只要异步任务有了运行结果，会把其回调函数作为一个任务添加到任务队列中；
3. 一旦执行栈中的所有同步任务执行完毕，就会读取任务队列，看看里面有哪些任务，将其添加到执行栈，开始执行；
4. 主线程不断重复上面第三步，就也是常说的事件循环（Event Loop）;

### 异步任务的类型

主线程的执行过程就是一个 tick，而所有的异步任务都是通过任务队列来一一执行。任务队列中存放的是一个个的任务（task），规定 task 分为两大类，分别是宏任务（macro task）和微任务 （micro task），并且每个 macro task 结束后，都要清空所有的 micro task；

```javascript
for (macroTask of macroTaskQueue) {
	handleMacroTask();
	
	for (microTask of microTaskQueue) {
		handleMicroTask(microTask);
	}
}
```

 常见的创建 macro task 的方法有:

- setTimeout、setInterval、postMessage、MessageChannel(队列优先于setTimeiout执行)
- 网络请求IO
- 页面交互：DOM、鼠标、键盘、滚动事件
- 页面渲染

常见的创建 micro task 的方法:

- Promise.then
- MutationObserve
- process.nexttick

在 nextTick 函数要利用这些方法把通过参数 cb 传入的函数处理成异步任务；

## nextTick 源码分析

- 在外层定义的 callbacks 变量用来存储所有需要执行的回调函数，在 nextTick 的外层定义就形成了一个闭包，所以每次调用 $nextTick 的过程就是在向 callbacks 新增回调函数的过程；

### nextTick 函数

1. 在 nextTick 函数中把通过 cb 传入的函数，做一下包装然后 push 到 callbacks 数组中；
2. ​
3. 用变量 pending 来保证执行一个事件循环中只执行一次 timerFunc();
4. 最后执行 if (!cb && typeof Promise !== 'undefined') ,判断参数 cb 不存在且浏览器支持 Promise，则返回一个 Promise 类实例化对象，例如  nextTick().then(() => {})，当 _resolve 函数执行，就会执行 then 的逻辑中；

```javascript
var callbacks = [];
var pending = false;
function nextTick(cb, ctx) {
	var _resolve;
	callbacks.push(function() {
		if (cb) {
			try {
				cb.call(ctx);
			} catch (e) {
				handleError(e, ctx, 'nextTick');
			}
		} else if (_resolve) {
			_resolve(ctx);
		}
	});
	if (!pending) {
		pending = true;
		timerFunc();
	}
	if (!cb && typeof Promise !== 'undefined') {
		return new Promise(function(resolve) {
			_resolve = resolve;
		})
	}
}
```

### timerFunc 函数

timerFunc 函数就是用各种异步执行的方法调用 flushCallbacks 函数；

#### Promise 创建异步执行函数

```javascript
var timerFunc;
if (typeof Promise !== 'undefined' && isNative(Promise)) {
	var p = Promise.resolve();
	timerFunc = function() {
		p.then(flushCallbacks);
		if (isIOS) {
			setTimeout(noop);
		}
	};
}
```

#### MutationObserver 创建异步执行函数

> MutationObserver() 创建并返回一个新的 MutationObserver，它会在指定的 DOM 发生变化时被调用；

1. MutationObserver（）在 IE11浏览器才兼容，所以执行 !isIE排除 IE浏览器；
2. MutationObserver.toString() === '[object MutationObserverConstructor]' 是对 PhantomJS 浏览器 和 iOS 7.x版本浏览器的支持情况进行判断；
3. 创建一个新的 MutationObserver 赋值给常量 observer， 并把 flushCallbacks作为回到函数传入，当 observer 指定的 DOM 要监听的属性发生变化时会调用 flushCallbacks 函数；
4. MutationObserver 是个微任务 （micro task）类型；



```javascript
if (!isIE && typeof MutationObserver !== 'undefined' &&
    (isNative(MutationObserver) ||
    MutationObserver.toString() === '[object MutationObserverConstructor]')
) {
    var counter = 1;
    var observer = new MutationObserver(flushCallbacks);
  	// 创建一个文本节点   
  	var textNode = document.createTextNode(String(counter));
  	// 调用 MutationObserver 的实例方法 observe 监听 textNode 文本节点的内容
    observer.observe(textNode, {
        characterData: true
    });
    timerFunc = function() {
        counter = (counter + 1) % 2;
      	// 文本节点内容变化，调用flushCallbacks函数
        textNode.data = String(counter);
    };
    isUsingMicroTask = true;
}
```

#### setImmediate 创建异步执行函数

> setImmediate  只兼容 IE10 以上浏览器，其他浏览器均不兼容;
>
> 是个宏任务 (macro task)，消耗的资源比较小；

```javascript
if (typeof setImmediate !== 'undefined' && isNative(setImmediate)) {
    timerFunc = function() {
        setImmediate(flushCallbacks);
    };
} 
```

#### setTimeout 创建异步执行函数

> setTimeout 兼容 IE10 以下的浏览器，创建异步任务；
>
> 是个宏任务 (macro task)，消耗资源较大;

```javascript
timerFunc = function() {
    setTimeout(flushCallbacks, 0);
}
```

#### 创建异步执行函数的顺序

- 第一版的中实现 timerFunc 的顺序为 Promise，MutationObserver，setTimeout；
- 在2.5.0版本中实现 timerFunc 的顺序改为 setImmediate，MessageChannel，setTimeout；在这个版本把创建微任务的方法都移除了，因为微任务优先级太高了；
- 后面实现 timerFunc 的顺序又改为 Promise，MutationObserver，setImmediate，setTimeout；

### flushCallbacks 函数

```javascript
var callbacks = [];
var pending = false;
function flushCallbacks() {
  	// 使下个事件循环中能 nextTick 函数中调用 timerFunc 函数
	pending = false;
	var copies = callbacks.slice(0);
	callbacks.length = 0;
	for (var i = 0; i < copies.length; i++) {
		copies[i]();
	}
}
```

## 流程分析

1. 把回调函数放到 callbacks 等待执行；
2. 将执行任务放到微任务或者宏任务中；
3. 事件循环到了微任务或者宏任务，执行函数依次执行 callbacks 中的回调；