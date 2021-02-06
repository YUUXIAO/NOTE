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

## SPA单页面

> SPA 仅在 web 页面初始化时加载相应的 HTML，Javascript 和 css，一旦页面加载完成，SPA 不会因为用户的操作而进行页面的重新加载或跳转，主要是利用路由机制实现 HTML 内容的变换，避免页面的重新加载；

### 优点

1. 用户体验好、快，内容的改变不不需要重新加载整个页面，避免了不必要的跳转和重复渲染；
2. SPA 相对服务器压力小；
3. 前后端职责分离，架构清晰，前端进行交互逻辑，后端负责数据处理；

### 缺点

1. 初次加载耗时多：为实现单页 Web 应用功能及显示效果，需要在加载页面的时候将 JavaScript、CSS 统一加载，部分页面按需加载；
2. 前进后退路由管理：单页应用在一个页面中显示所有的内容，所以不能使用浏览器的前进后退功能，所有页面切换需要自己建立堆栈管理；
3. SEO 难度较大：由于所有的内容都在一个页面中动态替换显示，所以在 SEO 上其有着天然的弱势；

## Vue中key的作用

> key 是 Vue 中 vnode 的唯一标记，通过这个 key，diff 操作可以更准确、迅速；

1. 更准确：因为带 key 就不是就地复用了，在 sameNode 函数 a.key === b.key 对比中可以避免就地复用的情况；
2. 更快速：利用 key 的唯一性生成 map 对象来获取对应节点，比遍历更快；

```javascript
function createKeyToOldIdx (children, beginIdx, endIdx) {
  let i, key
  const map = {}
  for (i = beginIdx; i <= endIdx; ++i) {
    key = children[i].key
    if (isDef(key)) map[key] = i
  }
  return map
}
```

## Vue项目优化

### 代码层面 

1. v-if 和 v-show 区分使用场景；
2. computed 和 watch 区分使用场景；
3. v-for 遍历必须为 item 添加 key，且避免同时使用 v-if；
4. 长列表性能优化；
5. 事件的销毁；
6. 图片资源懒加载；
7. 路由懒加载；
8. 第三方插件的按需引入；
9. 优化无限列表性能；
10. 服务端渲染 SSR or 预渲染；

### webpack层面

1. Webpack 对图片进行压缩；
2. 减少 ES6 转为 ES5 的冗余代码；
3. 提取公共代码；
4. 模板预编译；
5. 提取组件的 CSS；
6. 优化 SourceMap；
7. 构建结果输出分析；
8. Vue 项目的编译优化；

### 基础Web 技术

1. 开启 gzip 压缩；
2. 浏览器缓存；
3. CDN 的使用
4. 使用 Chrome Performance 查找性能瓶颈；

## Vue2源码的性能优化

1. cache 函数，利用闭包实现缓存；
2. 二次依赖收集时，cleanupDeps 方法清除上次存在但本次渲染不存在的依赖；
3. traverse，处理深度监听数据，解除循环引用；
4. 编译优化阶段，optimize 方法标记静态节点；
5. keep-alive 组件利用 LRU 缓存淘汰算法；
6. 异步组件，分两次渲染；