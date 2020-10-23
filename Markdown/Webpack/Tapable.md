> webpack 的插件架构主要是基于 Tapable 实现的，Tapable 是 webpack 项目组的一个内部库，主要是抽象了一套插件机制，专注于自定义事件的触发和操作；

# hooks概览



![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8a1d41bd086d4cf68afd7c9486979a1b~tplv-k3u1fbpfcp-zoom-1.image)





> 常用的钩子主要包含以下几种，分为同步和异步，异步又分为并发执行和串行执行；

```javascript
const {
  	// 同步串行: 不关心监听函数的返回值
	SyncHook,
  	// 同步串行: 只要监听函数中有一个函数的返回值不为 null，则跳过剩下所有的逻辑
	SyncBailHook,
  	// 同步串行: 上一个监听函数的返回值可以传给下一个监听函数
	SyncWaterfallHook,
  	// 同步循环: 当监听函数被触发的时候，如果该监听函数返回true时则这个监听函数会反复执行，如果返回 undefined 则表示退出循环
	SyncLoopHook,
    // 异步并发: 不关心监听函数的返回值
	AsyncParallelHook,
  	// 异步并发: 只要监听函数的返回值不为 null，就会忽略后面的监听函数执行，直接跳跃到callAsync等触发函数绑定的回调函数，然后执行这个被绑定的回调函数
	AsyncParallelBailHook,
  	// 异步串行: 不关系callback()的参数
	AsyncSeriesHook,
  	// 异步串行: callback()的参数不为null，就会直接执行callAsync等触发函数绑定的回调函数
	AsyncSeriesBailHook,
  	// 异步串行: 上一个监听函数的中的callback(err, data)的第二个参数,可以作为下一个监听函数的参数
	AsyncSeriesWaterfallHook
 } = require("tapable");
```

