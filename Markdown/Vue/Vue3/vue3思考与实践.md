https://www.cnblogs.com/zdsdididi/p/16396088.html 【vue3特性】

- watch和watchEffect的区别？

  - https://zhuanlan.zhihu.com/p/528715632?utm_id=0
  - https://blog.csdn.net/weixin_52148548/article/details/125073998
  - 它和React中的useEffect很像，只不过watchEffect不需要传入依赖项
  - computed和watch所依赖的数据必须是响应式的。Vue3引入了watchEffect,watchEffect 相当于将 watch 的依赖源和回调函数合并，当任何你有用到的响应式依赖更新时，该回调函数便会重新执行。不同于 watch的是watchEffect的回调函数会被立即执行，即（{ immediate: true }）

- **defineProps** 不能引用外部的 Ts？

- ### attrs和listeners

  - Vue3中使用attrs调用父组件方法时，方法前需要加上on；如parentFun->onParentFun

- 组件通信

  - 在组合式API中，如果想在子组件中用其它变量接收props的值时需要使用toRef将props中的属性转为响应式。

- build.polyfillDynamicImport ？？？ 已经废弃

- inheritAttrs 属性

- `Attribute`强制策略


## 响应式 API

https://cn.vuejs.org/api/reactivity-advanced.html 【官方组件】

## shallowRef()

场景：长列表数据，常常用于对大型数据结构的性能优化或是与外部的状态管理系统集成

## dev 调试 Api

### onRenderTracked 状态跟踪

当组件渲染过程中追踪到响应式依赖时调用

onRenderTracked 会跟踪页面上所有响应式变量和方法的状态（可以理解为用 return 返回的值都会跟踪），只要页面有 update 的情况，就会跟踪，生成一个 event 对象



### onRenderTriggered 状态触发

当响应式依赖的变更触发了组件渲染时调用

onRenderTriggered 不会跟踪每一个值，而是给你变化值的信息，并且新值和旧值都会给你明确的展示出来。

可以理解为就是跟踪渲染页面的变量，js 变量不会跟踪，更精准，会返回：

- 变化变量的 key 
- newValue 更新后变量的值
- oldValue 更新前变量的值
- target 目前页面中的响应变量和函数

感觉和watch 很像，而且是 immidate= true