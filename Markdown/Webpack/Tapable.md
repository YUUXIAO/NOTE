1. Webpack 本质上是一种事件流的机制，它的工作流程就是将各个插件串联起来，实现这一核心就是 Tapable；
2. Webpack 中最核心的负责编译的 Compiler 和 负责创建 bundles 的 Compilation 都是 Tapable 的实例；

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

