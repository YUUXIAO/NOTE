> webpack 的插件架构主要是基于 Tapable 实现的，Tapable 是 webpack 项目组的一个内部库，主要是抽象了一套插件机制，专注于自定义事件的触发和操作；

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8a1d41bd086d4cf68afd7c9486979a1b~tplv-k3u1fbpfcp-zoom-1.image)

webpack3及其以前使用的Tapable，提供了包括：

1. plugin(name:string, handler:function)：注册插件到 Tapable 对象中；
2. apply(…pluginInstances: (AnyPlugin|function)[]) ：调用插件的定义，将事件监听器注册到 Tapable 实例注册表中；
3. applyPlugins*(name:string, …) ：多种策略细致地控制事件的触发，包括 applyPluginsAsync、applyPluginsParallel 等方法实现对事件触发的控制，实现：
   - 多个事件连续执行；
   - 并行执行；
   - 异步执行；
   - 一个接一个地执行插件，前面的输出是后一个插件的输入的瀑布流执行顺序；
   - 在允许时停止执行插件，即某个插件返回了一个 undefined 的值，即退出执行；

## Tapable 1.0

> 暴露出很多的钩子，可以使用它们为插件创建钩子函数；

```javascript
const {
	SyncHook,
	SyncBailHook,
	SyncWaterfallHook,
	SyncLoopHook,
	AsyncParallelHook,
	AsyncParallelBailHook,
	AsyncSeriesHook,
	AsyncSeriesBailHook,
	AsyncSeriesWaterfallHook
 } = require("tapable");
```

