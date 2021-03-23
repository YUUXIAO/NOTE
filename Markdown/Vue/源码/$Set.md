在 Vue 中，只有 data 中已经存在的属性才会被  Observe 为响应式数据，如果是新增的属性是不会成为响应式属性，Vue 提供了 api （vm.$set）来解决这个问题；

1. 目标对象必须为非空的对象，可以是数组，否则抛出异常；
2. 目标对象不能是 Vue 实例或 Vue 实例的根数据对象，否则报错；
3. 如果目标对象是数组时，调用数组的 splice 方法，遇到数组新增元素的场景，会调用  ob.observeArray（inserted）对数组新增的元素收集依赖；
4. 目标对象是非响应式数据，按照普通对象添加属性的方式来处理；
5. 新增的属性值在原对象中已经存在，则手动访问新的属性值，这一过程会触发依赖收集；
6. 目标对象是响应数据且 key 为新增属性：设置 key 为响应式并手动触发其属性值的更新手动定义新属性的 getter，setter 方法，并通过 notify 触发依赖更新；

```javascript
export function set(target: Array<any> | Object, key: any, val: any): any {
  // 类型判断
  .......
  // 数组处理
  if (Array.isArray(target) && isValidArrayIndex(key)) {
    // 修改数组的长度, 避免索引>数组长度导致splcie()执行有误
    target.length = Math.max(target.length, key);
    // 利用数组的splice变异方法触发响应式
    target.splice(key, 1, val);
    return val;
  }
  // 对象，且key不是原型上的属性处理,新增对象的属性存在时，直接返回新属性，触发依赖收集
  // target为对象, key在target或者target.prototype上
  // 同时必须不能在 Object.prototype 上
  if (key in target && !(key in Object.prototype)) {
    target[key] = val;
    return val;
  }
  // 以上都不成立, 即开始给target创建一个全新的属性
  // 获取Observer实例
  const ob = (target: any).__ob__;
  // Vue 实例对象拥有 _isVue 属性, 即不允许给Vue 实例对象添加属性
  ......

  // target是非响应式数据时，直接赋值
  if (!ob) {
    target[key] = val;
    return val;
  }
  // target对象是响应式数据时,设置 key 为响应式手动触发更新
  defineReactive(ob.value, key, val);
  ob.dep.notify();
  return val;
}
```
