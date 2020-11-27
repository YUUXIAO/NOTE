## 目的

在 Vue 中，只有 data 中已经存在的属性才会被  Observe 为响应式数据，如果是新增的属性是不会成为响应式属性，Vue 提供了 api （vm.$set）来解决这个问题；

## 原理

> vm.$set()在 new Vue()时候就被注入到 Vue 的原型上；

```javascript
// vue/src/core/instance/index.js

import { initMixin } from "./init";
import { stateMixin } from "./state";
import { renderMixin } from "./render";
import { eventsMixin } from "./events";
import { lifecycleMixin } from "./lifecycle";
import { warn } from "../util/index";

function Vue(options) {
  if (process.env.NODE_ENV !== "production" && !(this instanceof Vue)) {
    warn("Vue is a constructor and should be called with the `new` keyword");
  }
  this._init(options);
}

initMixin(Vue);
// 给原型绑定代理属性$props, $data
// 给Vue原型绑定三个实例方法: vm.$watch，vm.$set，vm.$delete
stateMixin(Vue);
// 给Vue原型绑定事件相关的实例方法: vm.$on, vm.$once ,vm.$off , vm.$emit
eventsMixin(Vue);
// 给Vue原型绑定生命周期相关的实例方法: vm.$forceUpdate, vm.destroy, 以及私有方法_update
lifecycleMixin(Vue);
// 给Vue原型绑定生命周期相关的实例方法: vm.$nextTick, 以及私有方法_render, 以及一堆工具方法
renderMixin(Vue);

export default Vue;
```

$set（）源码位置：vue/src/core/observer/index.js

```javascript
export function set(target: Array<any> | Object, key: any, val: any): any {
  // 1.类型判断
  // 如果 set 函数的第一个参数是 undefined 或 null 或者是原始类型值，那么在非生产环境下会打印警告信息
  // 这个api本来就是给对象与数组使用的
  if (
    process.env.NODE_ENV !== "production" &&
    (isUndef(target) || isPrimitive(target))
  ) {
    warn(
      `Cannot set reactive property on undefined, null, or primitive value: ${(target: any)}`
    );
  }
  // 2.数组处理
  if (Array.isArray(target) && isValidArrayIndex(key)) {
    // 类似$vm.set(vm.$data.arr, 0, 3)
    // 修改数组的长度, 避免索引>数组长度导致splcie()执行有误
    //如果不设置length，splice时，超过原本数量的index则不会添加空白项
    target.length = Math.max(target.length, key);
    // 利用数组的splice变异方法触发响应式, 这个前面讲过
    target.splice(key, 1, val);
    return val;
  }
  //3.对象，且key不是原型上的属性处理
  // target为对象, key在target或者target.prototype上。
  // 同时必须不能在 Object.prototype 上
  // 直接修改即可, 有兴趣可以看issue: https://github.com/vuejs/vue/issues/6845
  if (key in target && !(key in Object.prototype)) {
    target[key] = val;
    return val;
  }
  // 以上都不成立, 即开始给target创建一个全新的属性
  // 获取Observer实例
  const ob = (target: any).__ob__;
  // Vue 实例对象拥有 _isVue 属性, 即不允许给Vue 实例对象添加属性
  // 也不允许Vue.set/$set 函数为根数据对象(vm.$data)添加属性
  if (target._isVue || (ob && ob.vmCount)) {
    process.env.NODE_ENV !== "production" &&
      warn(
        "Avoid adding reactive properties to a Vue instance or its root $data " +
          "at runtime - declare it upfront in the data option."
      );
    return val;
  }
  //5.target是非响应式数据时
  // target本身就不是响应式数据, 直接赋值
  if (!ob) {
    target[key] = val;
    return val;
  }
  //6.target对象是响应式数据时
  //定义响应式对象
  defineReactive(ob.value, key, val);
  //watcher执行
  ob.dep.notify();
  return val;
}
```

- defineReactive 方法就是 Vue 在初始化对象时，给对象属性采用 Object.defineProperty 动态添加 getter 和 setter 的功能所调用的方法；

## 主要逻辑

1. 类型判断；
2. target 为数组：调用 splice 方法；
3. target 为对象且 key 不是原型上的属性：直接修改；
4. target 不能是 Vue 实例或 Vue 实例的根数据对象，否则报错；
5. target 是非响应式数据，按照普通对象添加属性的方式来处理；
6. target 是响应数据且 key 为新增属性：设置 key 为响应式并手动触发其属性值的更新；

