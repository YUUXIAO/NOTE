## Vue3.0

1. Vue3.0 都有哪些重要新特性？
2. 相比 2.0 的优势
3. Vue3.0 和  React 16.X 都有哪些区别和相似处？
4. Vue3.0 是如何实现代码逻辑复用的？

## React

1. React 16.X  的 Fiber 原理；
2. setState 原理，什么时候是同步的？
3. React Hooks 相对高阶组件和 Class 组件有什么优势/缺点？
4. React 16.X 的生命周期，以及为何要替换掉以前的？
5. React 跨平台的实现原理；
6. 说一说 redux，以及比 flux 先进的原因；

## Vue2源码的性能优化

1. cache 函数，利用闭包实现缓存；
2. 二次依赖收集时，cleanupDeps 方法清除上次存在但本次渲染不存在的依赖；
3. traverse，处理深度监听数据，解除循环引用；
4. 编译优化阶段，optimize 方法标记静态节点；
5. keep-alive 组件利用 LRU 缓存淘汰算法；
6. 异步组件，分两次渲染；