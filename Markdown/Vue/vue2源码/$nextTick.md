> 在 nextTick 函数要利用异步方法（宏任务、微任务）把通过参数 cb 传入的函数处理成异步任务；
>

## update 

Vue 在依赖收集的响应式化方法 defineReactive 中的 setter 访问器中进行派发更新 dep.notify 方法，这个方法会通知在 dep 的 subs 中收集的订阅自己的 watchers 执行 update；

```javascript
/* Subscriber接口，当依赖发生改变的时候进行回调 */
update() {
  if (this.computed) {
    // 一个computed watcher有两种模式：activated lazy(默认)
    // 只有当它被至少一个订阅者依赖时才置activated，这通常是另一个计算属性或组件的render function
    // 如果没人订阅这个计算属性的变化
    if (this.dep.subs.length === 0) {       
      // lazy时，只在必要时执行计算，只是简单地将观察者标记为dirty
      // 当计算属性被访问时，实际的计算在this.evaluate()中执行
      this.dirty = true
    } else {
      // activated时，主动执行计算，当值确实发生变化时才通知订阅者
      this.getAndInvoke(() => {
        // 通知渲染watcher重新渲染，通知依赖所有watcher执行update
        this.dep.notify()     
      })
    }
  } else if (this.sync) {	  
    // 同步
    this.run()
  } else {
    // 异步推送到调度者观察者队列中，下一个tick时调用
    queueWatcher(this)        
  }
}
```

## queueWatcher

> queueWatcher 方法会将数据收集的依赖依次推到 queue 数组中，数组会在下一个事件循环 tick 中根据缓冲结果进行视图更新；

如果不是 computed watcher 也非 sync 会把调用 update 的当前 watcher 推送到调度者队列中，下一个 tick 时调用；

```javascript
// 将一个观察者对象push进观察者队列，在队列中已经存在相同的id则该watcher将被跳过，除非它是在队列正被flush时推送
export function queueWatcher (watcher: Watcher) {
  const id = watcher.id
   // 检验id是否存在，已经存在则直接跳过，不存在则标记哈希表hash，用于下次检验
  if (has[id] == null) {    
    has[id] = true
    // 如果没有正在flush，直接push到队列中
    queue.push(watcher)      
    // 标记是否已传给nextTick
    if (!waiting) {         
      waiting = true
      nextTick(flushSchedulerQueue)
    }
  }
}

/* 重置调度者状态 */
function resetSchedulerState () {
  queue.length = 0
  has = {}
  waiting = false
}
```

初始设定 this.deep = this.user = this.lazy = this.sync = false；就是当触发 update 更新的时候，会去执行 queueWatcher 方法；

- waiting 变量是用来标记 flushSchedulerQueue  是否已经传递给 nextick 的标记位，如果已经传递则只 push 到队列中不传递 flushSchedulerQueue  给 nextTick，等到 resetSchedulerState 重置调度者状态的时候 waiting 会置回 false 允许  flushSchedulerQueue 被传递给下一个 tick 的回调，保证了 flushSchedulerQueue 回调只允许被转入 callbacks 一次；

```javascript
/**
 * 将观察者推入观察者队列
 * 具有重复ID的watcher将被跳过
 * 在刷新队列时推送
 */
function queueWatcher (watcher) {
  var id = watcher.id;
  // 保证同一个watcher只执行一次
  if (has[id] == null) {
    has[id] = true;
    // 如果还没有开始清空队列，将任务watcher添加到队列中
    if (!flushing) {
      queue.push(watcher);
    } else {
      var i = queue.length - 1;
      while (i > index && queue[i].id > watcher.id) {
        i--;
      }
      queue.splice(i + 1, 0, watcher);
    }
    ···
    nextTick(flushSchedulerQueue);
  }
}
```

## flushSchedulerQueue 

> flushSchedulerQueue 函数其实就是 watcher 的视图更新；

1. 对 queue 中的 watcher 进行排序；
   - sort 方法把队列中的 watcher 按 id 从小到大排序；
   - 组件更新的顺序是从父组件到子组件的顺序，需要保证父的渲染 watcher 优先于子的渲染 watcher 更新；
   - 一个组件的 user watchers 比 render watcher 先运行，因为 user watchers 比 render watcher 更早创建；
   - 如果一个组件在父组件 watcher 运行时间被销毁，它的 watcher 执行将被跳过；
2. 遍历 watcher ，如果当前 watcher 有 before 配置，则执行 before 方法，对应前而的渲染 watcher ；
   - 在渲染 watcher 实例化时，传递了 before 函数，即在下个 tick 更新视图前，会调用 beforeUpdate 生命周期钩子；
3. 依次执行 queue 中的 watcher 的 run 方法；

```javascript
// nextTick的回调函数，在下一个tick时flush掉两个队列同时运行watchers
function flushSchedulerQueue () {
  flushing = true
  let watcher, id
  // 对queue的watcher进行排序
  queue.sort((a, b) => a.id - b.id)					
  // 循环执行queue.length，为了确保由于渲染时添加新的依赖导致queue的长度不断改变
  for (index = 0; index < queue.length; index++) {	 
    watcher = queue[index]
    // 如果watcher有before则执行
    if (watcher.before) {        
      watcher.before()
    }
    id = watcher.id
    // 将has的标记删除
    has[id] = null          
    // 执行watcher
    watcher.run()         
    
    // 在dev环境下检查是否进入死循环【警告】
    .....
  }
  
  // 重置调度者状态
  resetSchedulerState()           
  // 使子组件状态都置成active同时调用activated钩子
  callActivatedHooks()           
  // 调用updated钩子
  callUpdatedHooks()              
}

```

## nextTick原理

### 数据频繁变化只更新一次

1. 检测到数据变化；
2. 开启一个队列；
3. 在同一事件循环中缓冲所有的数据改变；
4. 如果同一个 watcher（watcher Id 相同）被多次触发，只会被推入队列一次；

执行顺序 update  -> queueWatcher -> 维护观察者队列（重复id的Watcher处理） -> waiting标志位处理 -> 处理$nextTick（在微任务或者宏任务中异步更新DOM）；

### nextTick 函数

1. 把传入的 cb 回调函数用 try-catch 包裹后放在一个匿名函数中推入 callbacks 数组中，每个 cb 都包裹是防止这些回调函数如果执行错误不会相互影响；
2. 通过 callbacks 数组来模拟事件队列，事件队列里的事件，通过 nextTickHandler 方法来执行调用，何时执行是 timerFunc 来决定；
3. 检查 pending 状态，它是一个标记位，初始值是 false，在执行 timerFunc 前置为 true，因此下次调用 nextTick 不会进入 timerFunc 方法，这个方法会在下一个 macro/micro tick 时  flushCallbacks 异步去执行 callbacks 队列中的任务，flushCallbacks 方法一开始会把 pending 置为 false，所以下一次调用 nextTick 时又能开启新一轮的 timerFunc ,这样就形成了 vue 中的 event loop；
4. 最后执行 if (!cb && typeof Promise !== 'undefined') ,判断参数 cb 不存在且浏览器支持 Promise，则返回一个 Promise 类实例化对象，例如  nextTick().then(() => {})，当 _resolve 函数执行，就会执行 then 的逻辑中；
5. 在外层定义的 callbacks 变量用来存储所有需要执行的回调函数，在 nextTick 的外层定义就形成了一个闭包，每次调用 $nextTick 的过程就是在向 callbacks 新增回调函数的过程；
6. 由于宏任务耗费时间是大于微任务的，所以在浏览器支持的情况下，优先使用微任务；

```javascript
export const nextTick = (function () {
  const callbacks = []
  let pending = false； // 用来标志是否正在执行回调函数
  let timerFunc

  function nextTickHandler () {
    pending = false
    const copies = callbacks.slice(0)
    callbacks.length = 0
    for (let i = 0; i < copies.length; i++) {
      copies[i]()
    }
  }

  if (typeof setImmediate !== 'undefined' && isNative(setImmediate)) {
    timerFunc = () => {
      setImmediate(nextTickHandler)
    }
  } else if (typeof MessageChannel !== 'undefined' && (
    isNative(MessageChannel) ||
    // PhantomJS
    MessageChannel.toString() === '[object MessageChannelConstructor]'
  )) {
    const channel = new MessageChannel()
    const port = channel.port2
    channel.port1.onmessage = nextTickHandler
    timerFunc = () => {
      port.postMessage(1)
    }
  } else
  /* istanbul ignore next */
  if (typeof Promise !== 'undefined' && isNative(Promise)) {
    const p = Promise.resolve()
    timerFunc = () => {
      p.then(nextTickHandler)
    }
  } else {
    // fallback to setTimeout
    timerFunc = () => {
      setTimeout(nextTickHandler, 0)
    }
  }

  return function queueNextTick (cb?: Function, ctx?: Object) {
    let _resolve
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
      timerFunc()
    }
    if (!cb && typeof Promise !== 'undefined') {
      return new Promise((resolve, reject) => {
        _resolve = resolve
      })
    }
  }
})()

```

### timerFunc 

> timerFunc 是真正将任务队列推到微任务队列中的函数，实质上就是用各种异步执行的方法调用 flushCallbacks 函数；

#### Promise 

如果浏览器执行 Promise，默认以 Promise 将执行过程推到任务队列中；

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
   // 使用微任务队列的标志
   isUsingMicroTask = true;
}
```

#### MutationObserver 

> MutationObserver() 是一个用于监视DOM变动的接口，可以监听一个 DOM 对象上发生的子节点删除、属性修改和文本内容修改等，创建并返回一个新的 MutationObserver；

1. MutationObserver 是个微任务 （micro task）类型；
2. MutationObserver（）在 IE11浏览器才兼容，所以排除 IE浏览器；
3. MutationObserver.toString() === '[object MutationObserverConstructor]' 是对 PhantomJS 浏览器 和 iOS 7.x版本浏览器的支持情况进行判断；
4. 创建一个新的 MutationObserver 赋值给常量 observer， 并把 flushCallbacks作为回到函数传入，当 observer 指定的 DOM 要监听的属性发生变化时会调用 flushCallbacks 函数；



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

#### setImmediate

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

#### setTimeout 

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
- 在 2.5.0 版本中实现 timerFunc 的顺序改为 setImmediate，MessageChannel，setTimeout，在这个版本把创建微任务的方法都移除了，因为微任务优先级太高了；
- 后面实现 timerFunc 的顺序又改为 Promise，MutationObserver，setImmediate，setTimeout；

### flushCallbacks

> flushCallbacks 是异步更新的函数，对callbacks进行遍历，然后执行相应的回调函数；

拷贝 callbacks 数组的原因：有的 cb 执行过程中又会往callbacks中加入内容，比如 nextTick 的回调函数里还有 $nextTick，后者的应该放到下一轮的nextTick 中执行，所以拷贝一份当前的，遍历执行完当前的即可，避免无休止的执行下去；

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


## 举例

```javascript
<div id="app">
  <span id='name' ref='name'>{{ name }}</span>
  <button @click='change'>change name</button>
</div>

<script>
methods: {
  change() {
    const $name = this.$refs.name
    this.$nextTick(() => console.log('setter前：' + $name.innerHTML))
    this.name = ' name改喽 '
    console.log('同步方式：' + this.$refs.name.innerHTML)
    setTimeout(() => this.console("setTimeout方式：" + this.$refs.name.innerHTML))
    this.$nextTick(() => console.log('setter后：' + $name.innerHTML))
    this.$nextTick().then(() => console.log('Promise方式：' + $name.innerHTML))
  }
}
</script>

// 同步方式：SHERlocked93 
// setter前：SHERlocked93 
// setter后：name改喽 
// Promise方式：name改喽 
// setTimeout方式：name改喽
```

1. 同步方式：当把 data 中的 name 修改后，会触发 name 的 setter 中的 dep.notify 通知依赖本 data 的 render watcher 去 update，update 会把 flushSchedulerQueue 函数传递给 nextTick，render watcher 在 flushSchedulerQueue 函数中运行时 watcher.run 再走 diff --> patch 的过程重新渲染视图，这个过程会重新依赖收集，这个过程是异步的；当修改了 name 之后打印，这时异步的改动还没有 patch 到视图上；
2. setter 前：nextTick 在被调用时把回调挨个 push 进 callbacks 数组，之后执行也是 for 循环挨个执行，在修改 name 后，触发把 render watcher 填入 schedulerQueue 队列并把他的执行函数 flushSchedulerQueue 传递给 nextTick，此时 callbacks 队列中已经有了 setter前函数，按照先入先出的执行 callbacks 回调时先执行 setter前函数，这时还没有执行 render watcher 的 watcher.run 方法，所以打印的还是原来的内容；
3. setter 后：setter 后时已经执行完 watcher.run 方法，这时 render watcher 已经把改动 patch 到视图上；
4. Promise 方式：相当于 Promise.then 方式执行此函数，此时 Dom 已经修改；
5. setTimeout 方式：最后执行 macro task 任务，此时 Dom 已经修改；
6. 在执行 setter 前函数时，同步代码已经执行完毕，异步的任务还未执行，所有 $nextTick 函数也执行完毕，所有回调都被 push 进了 callbacks 队列等待执行，此时 callbacks 队列是：[ setter前函数，flushSchedulerQueue，setter后函数，Promise方式函数 ]，它是一个 micro task 队列，执行完毕后执行 macro task 任务 setTimeout；