在 Vue 中，只有 data 中已经存在的属性才会被  Observe 为响应式数据，如果是新增的属性是不会成为响应式属性，Vue 提供了 api （vm.$set）来解决这个问题；

流程：

1. 目标对象必须为非空的对象，可以是数组，否则抛出异常；
2. 目标对象不能是 Vue 实例或 Vue 实例的根数据对象，否则报错；
3. 如果目标对象是数组时，调用数组的 splice 方法，遇到数组新增元素的场景，会调用  ob.observeArray（inserted）对数组新增的元素收集依赖；
4. 目标对象是非响应式数据，按照普通对象添加属性的方式来处理；
5. 新增的属性值在原对象中已经存在，则手动访问新的属性值，这一过程会触发依赖收集；
6. 目标对象是响应数据且 key 为新增属性：设置 key 为响应式并手动触发其属性值的更新手动定义新属性的 getter，setter 方法，并通过 notify 触发依赖更新；

```typescript
export function set(target: Array<any> | Object, key: any, val: any): any {
  // 1.类型判断
  // 如果 set 函数的第一个参数是 undefined 或 null 或者是原始类型值，那么在非生产环境下会打印警告信息
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
    // 利用数组的splice变异方法触发响应式
    target.splice(key, 1, val);
    return val;
  }
  //3.对象，且key不是原型上的属性处理,新增对象的属性存在时，直接返回新属性，触发依赖收集
  // target为对象, key在target或者target.prototype上。
  // 同时必须不能在 Object.prototype 上
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
  //5.target是非响应式数据时，直接赋值
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
