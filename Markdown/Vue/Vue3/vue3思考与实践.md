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

