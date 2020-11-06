

## 异步更新

Vue官网对数据操作是这么描述的：

> Vue 在更新 DOM 时是异步执行的。只要侦听到数据变化，Vue 将开启一个队列，并缓冲在同一事件循环中发生的所有数据变更。如果同一个 watcher 被多次触发，只会被推入到队列中一次。这种在缓冲时去除重复数据对于避免不必要的计算和 DOM 操作是非常重要的。然后，在下一个的事件循环“tick”中，Vue 刷新队列并执行实际 (已去重的) 工作；

Vue 中的 dom 不是实时的，当数据改变后，vue 会把 渲染 watcher 添加到异步队列，异步执行，同步代码执行完成后再统一修改 dom；

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

```javascript
import { noop } from 'shared/util'
import { handleError } from './error'
import { isIOS, isNative } from './env'

// 存储所有需要执行的回调函数
const callbacks = []
// 标志是否正在执行回调函数
let pending = false

// 对 callbacks 遍历，然后执行相应的回调函数
function flushCallbacks () {
  pending = false
  const copies = callbacks.slice(0)
  callbacks.length = 0
  for (let i = 0; i < copies.length; i++) {
    copies[i]()
  }
}

let microTimerFunc
let macroTimerFunc
let useMacroTask = false

/* istanbul ignore if */
if (typeof setImmediate !== 'undefined' && isNative(setImmediate)) {
  macroTimerFunc = () => {
    setImmediate(flushCallbacks)
  }
} else if (typeof MessageChannel !== 'undefined' && (
  isNative(MessageChannel) ||
  // PhantomJS
  MessageChannel.toString() === '[object MessageChannelConstructor]'
)) {
  const channel = new MessageChannel()
  const port = channel.port2
  channel.port1.onmessage = flushCallbacks
  macroTimerFunc = () => {
    port.postMessage(1)
  }
} else {
  /* istanbul ignore next */
  macroTimerFunc = () => {
    setTimeout(flushCallbacks, 0)
  }
}

/* istanbul ignore next, $flow-disable-line */
if (typeof Promise !== 'undefined' && isNative(Promise)) {
  const p = Promise.resolve()
  microTimerFunc = () => {
    p.then(flushCallbacks)
    if (isIOS) setTimeout(noop)
  }
} else {
  // fallback to macro
  microTimerFunc = macroTimerFunc
}

// 对函数做一层包装,确保函数执行过程中对数据的任意修改，触发变化执行 nextTick 的时候强制走 macroTimerFunc;
// 比如对于一些 DOM 交互事件，如 v-on 绑定的事件回调函数的处理，会强制走 macro task
export function withMacroTask (fn: Function): Function {
  return fn._withTask || (fn._withTask = function () {
    useMacroTask = true
    const res = fn.apply(null, arguments)
    useMacroTask = false
    return res
  })
}

export function nextTick (cb?: Function, ctx?: Object) {
  let _resolve
  // 保证在同一个tick 内多次执行 nextTick，不会开启多个异步任务，而是把这些异步任务都压成一个同步任务，在下一个tick执行完毕 
  callbacks.push(() => {
    if (cb) {
      try {
        cb.call(ctx)
      } catch (e) {
        handleError(e, ctx, 'nextTick')
      }
    } else if (_resolve) {
      _resolve(ctx)
    }
  })
  if (!pending) {
    pending = true
    if (useMacroTask) {
      macroTimerFunc()
    } else {
      microTimerFunc()
    }
  }
  // 当 nextTick 不传入 cb 参数的时候,提供一个 Promise 化的调用；
  if (!cb && typeof Promise !== 'undefined') {
    return new Promise(resolve => {
      _resolve = resolve
    })
  }
}
```

- 在外层定义的 callbacks 变量用来存储所有需要执行的回调函数，在 nextTick 的外层定义就形成了一个闭包，所以每次调用 $nextTick 的过程就是在向 callbacks 新增回调函数的过程；
- 在 nextTick 是对 setTimeout 进行了多种兼容性的处理，不过 nextTick 优先放入 微任务执行，而 setTimeout 是宏任务，因此 nextTick 一般情况下问题优先于 setTimeout执行；

## 流程分析

1. 把回调函数放到 callbacks 等待执行；
2. 将执行任务放到微任务或者宏任务中；
3. 事件循环到了微任务或者宏任务，执行函数依次执行 callbacks 中的回调；