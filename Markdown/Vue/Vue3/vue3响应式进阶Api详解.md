## 响应式 API

[https://cn.vuejs.org/api/reactivity-advanced.html](https://cn.vuejs.org/api/reactivity-advanced.html) 【官方组件档】

### reactive 和 shallowReactive

#### reactive

reactive 的响应式转换是深层的，会影响到所有嵌套的属性，虽然内部实现是先只处理第一层属性，后续如果取到子属性的值再用reactive包装返回

reactive可以算作是Vue 3的根基。返回对象的响应式副本，响应式转换是“深层”的——它影响所有嵌套property。返回Proxy对象，不等于原始对象。建议只操作Proxy对象，不要操作原始对象。

```javascript
import { mutableHandlers,shallowReactiveHandlers } from './baseHandlers'
export const reactiveMap = new WeakMap()
export const shallowReactiveMap = new WeakMap()
export const reactiveMap = new WeakMap() // 定义一个reactive对象地图

export function reactive(target) {
    return createReactiveObject(target, reactiveMap, mutableHandlers)
}

function createReactiveObject(target, proxyMap, proxyHandlers) {
    if (typeof target !== 'object') {
        console.warn('reactive ${target} 必须是一个对象')
        return target
    }
    //在reactive对象地图中查找是否有target，防止重复注册同一个reactive对象
    const existingProxy = proxyMap.get(target)
    if (existingProxy) {
        return existingProxy
    }

    // 通过Proxy 创建代理，不同的Map存储不同类型的reactive依赖关系
    const proxy = new Proxy(target, proxyHandlers)
    proxyMap.set(target, proxy) // 把从未注册过的reactive对象放入reactive地图中
    return proxy // 返回的是一个一个Proxy实例对象
}
// 浅层的代理
export function shallowReactive(target) {
    return createReactiveObject(
        target,
        shallowReactiveMap,
        shallowReactiveHandlers
    )
}
```

从上面可以知道：

- 通过 reactive 包裹的 obj 对象，返回一个 Proxy 实例对象
- reactive 只处理 object 数据类型，不接收原始数据类型

#### shallowReactive

shallowReactive 就是把数据转为浅层次（第一层）的响应式数据，假设对象里是对象B，对象B就不是响应式的（有点类比浅拷贝）

**场景**：如果一个对象的深层不可能变化，那么就没必要深层响应，这时候用shallowReactive可以节省系统开销

```javascript
<template>
<div>
  <button @click="r.b.c++">count is: {{ r.b.c }}</button>
  <button @click="s.b.c++">count is: {{ s.b.c }}</button>
</div>
</template>

<script>
import { reactive, shallowReactive } from "vue";
export default {
  setup() {
    let r = reactive({a: 1, b: {c: 2}});
    console.log(r);
    let s = shallowReactive({a: 1, b: {c: 2}});
    console.log(s);
    return {
      r,s
    };
  },
};
</script>
```

按下第2个button不会有反应，只有又去按下第1个button之后，视图刷新，第二个button才有反应。

### ref 和 shallowRef

#### ref

ref  函数用来将一项数据包装成一个响应式 ref 对象。它接收任意数据类型的参数，作为这个 ref 对象内部的value 的值

- 生成值类型数据（String，Number，Boolean，Symbol）的响应式对象
- 生成对象和数组类型的响应式对象 **（对象和数组一般不选用ref方式，而选用reactive方式，ref 内部是由 reactive 实现的，等价于 reactive(value: xxx)）**

```javascript
export function ref(val) {
      if (isRef(val)) {
        return val
      }
      return new RefImpl(val)
    }
    export function isRef(val) {
      return !!(val && val.__isRef)
    }

    // ref就是利用面向对象的getter和setters进行track和trigget
    class RefImpl {
      constructor(val) {
        this.__isRef = true
        this._val = convert(val)
      }
      get value() {
        track(this, 'value')
        return this._val
      }

      set value(val) {
        if (val !== this._val) {
          this._val = convert(val)
          trigger(this, 'value')
        }
      }
    }

    // ref也可以支持复杂数据结构
    function convert(val) {
      return isObject(val) ? reactive(val) : val
    }

```

从上面可以发现 ：

- ref 不需要使用 Proxy 代理语法，直接使用对象语法的 getter 和 setter 配置，监听 value 属性即可
- ref 函数只是利用 对象的 getter 和 setter 拦截了 value 属性的读写（也是为什么操作 ref 数据需要.value），ref 包裹复杂的数据结构时，内部是使用的 reactive 实现

`Ref`是这样的一种数据结构：它有个key为`Symbol`的属性做类型标识，有个属性`value`用来存储数据。这个数据可以是任意的类型，**唯独不能是被嵌套了Ref类型的类型**

？？？ **为什么 vue2.0 没有区分基本数据类型和引用数据类型做响应式**

- 因为defineproperty就是Object的静态方法，它只是为对象服务的，甚至无法对数组服务，因此Vue 2弄了一个data根对象来存放基本数据类型，这样无论什么类型，都是根对象的property，所以也就能代理基本数据类型。
- Proxy能对所有引用类型代理，Vue 3也不再用data根对象，而是一个个的变量，所以带来了新问题，如何代理基本数据类型呢？并没有原生办法，只能构建一个{value: Proxy Object}结构的对象，这样Proxy也就能代理了。

#### shallowRef

shallowRef的作用是只对value添加响应式，因此，必须是value被重新赋值才会触发响应式。shallowRef的出现主要是为了节省系统开销。

- 如果传入的是基础数据类型，和ref没有区别
- 如果传入的是对象数据类型，那么 ref 底层还是调用了reactive变成 proxy 对象，成为可响应的；而 shallowRef 传入的是对象数据类型，则不会变成响应式(shallowRef 的内部值将会原样存储和暴露，并不会被深层递归地转为响应式，**只处理基本数据类型的响应式，不进行对象的响应式处理**)

可以理解为只响应式.value返回的值，打印的包裹的对象类型数据是 Object 不是 proxy，针对ref包裹的对象类型数据，结果打印是Proxy，所以是响应式的

**可优化场景**：

- refs 组件引用，项目中典型的 ref.value.validate()（表单、表格引用）
- 如果数据是服务器返回的 LIST 数据，而且只显示、不变更，那么最好是使用 shallowRef 来包装数据，可以节能。如果会有变更，那么应该用 ref
- 长列表数据，常常用于对大型数据结构的性能优化或是与外部的状态管理系统集成

  如果不是页面上需要进行视图更新的，我们可以不用reactive、ref更进行声明，可以使用[`shallowRef()`](https://links.jianshu.com/go?to=https://cn.vuejs.org/api/reactivity-advanced.html#shallowref)和 [`shallowReactive()`](https://links.jianshu.com/go?to=https://cn.vuejs.org/api/reactivity-advanced.html#shallowreactive)浅层式响应进行声明（浅层式顶部是响应的，底部都不是响应数据）


```javascript
const shallowArray = shallowRef([
  {...},
   ...
  /* 巨大的列表，里面包含深层的对象 */
])

shallowArray.value.push(newObject)  // 这不会触发更新...
shallowArray.value = [...shallowArray.value, newObject] // 这才会触发更新


shallowArray.value[0].foo = 1 // 这不会触发更新...
shallowArray.value = [
  {
    ...shallowArray.value[0],
    foo: 1
  },
  ...shallowArray.value.slice(1)
] // 这才会触发更新
```

#### triggerRef

强制触发依赖于一个shallowRef的副作用

```javascript
function triggerRef(ref: ShallowRef): void
```

如果既想使用shallowRef生成ref对象（为了节省开销），又想偶尔修改value指向的内部值的某个property，又希望那个ref对象得到响应，这时候可以用triggerRef。

```javascript

<template>
<div>
  <button @click="onClick">count is: {{ r.c }}</button>
</div>
</template>

<script>
import { shallowRef, triggerRef } from "vue";
export default {
  setup() {
    let r = shallowRef({a:{b:2}, c: 4});
    function onClick() {
      r.value.c = 6;
      triggerRef(r);  // 注释掉点击没有反应
    }
    return {
      r, onClick
    };
  },
};
</script>
```

#### customRef

创建一个自定义的 ref，显式声明对其依赖追踪和更新触发的控制方式

参数说明：

- 入参是一个回调函数；
- 回调函数接受 track 和 trigger 两个函数作为参数，并返回一个带有 get 和 set 方法的对象（必须）

比如我们有一个值会频繁刷新调用，我们想手动控制它的刷新时机，创建一个防抖 ref

```javascript
import { customRef } from 'vue'

export function useDebouncedRef(value, delay = 200) {
  let timeout
  return customRef((track, trigger) => {
    return {
      get() {
        track() // 可以理解为添加响应式追踪(vue代码里 一般起名为track 的都是这个作用)
        return value
      },
      set(newValue) {
        clearTimeout(timeout)
        timeout = setTimeout(() => {
          value = newValue
          trigger() // 可以理解为通知vue去重新解析模板，触发页面渲染更新函数
        }, delay)
      }
    }
  })
}

const text = useDebouncedRef('text')
```

使用场景：

- **时机上说**：可以控制视图更新的时机，可以延迟更新。其他ref兄弟API都做不到
- **内容上说**：可以修改传入的原始数据，让原始数据与返回值不相同。其他兄弟API也做不到。

#### ref 和 shallowRef

```javascript

<template>
  <div>
    <button @click="r.b.c++">count is: {{ r.b.c }}</button>
    <button @click="s.b.c++">count is: {{ s.b.c }}</button>
    <button @click="s = { a: 10, b: { c: 20 } }">count is: {{ s.b.c }}</button>
  </div>
</template>

<script>
import { ref, shallowRef } from 'vue';
export default {
  setup() {
    let r = ref({ a: 1, b: { c: 2 } });
    let s = shallowRef({ a: 1, b: { c: 2 } });
    return {
      r,
      s,
    };
  },
};
</script>
```

- 点击button1会有反应
- 点击button2不会有反应。
- 点击button3，如果给s重新赋值，其实相当于给s.value重新赋值，由于value是响应式的，这时候button2和button3都会有变化。

|              | ref                           | shallowRef                              |
| ------------ | ----------------------------- | --------------------------------------- |
| 本质         | reactive({value: 原始数据})   | shallowReactive({value: 原始数据})      |
| 区别点       | {value: 原始数据}被深层响应式 | 只有value被响应式，原始数据没有响应式   |
| 传入基本类型 | 两个API无差别                 | 两个API无差别，性能考虑尽量用shallowRef |
| 传入引用类型 | value指向Proxy                | value指向原始数据                       |

#### ref 和 reactive

|                      | ref                        | reactive   |
| -------------------- | -------------------------- | ---------- |
| **返回数据类型**     | RefImpl对象（也叫ref对象） | Proxy对象  |
| **传入基本类型返回** | {value: 基本类型}          | 禁止这么做 |
| **传入引用类型返回** | {value: Proxy对象}         | Proxy对象  |

**那么对于引用类型，什么时候用ref，什么时候用reactive？**

如果你只打算修改引用类型的一个属性，那么推荐用reactive，如果你打算变量重赋值，那么一定要用ref

```javascript

<template>
  <div>
    {{ jsonData }}  // 打印出什么name：xxx， age： xxx ??
  </div>
</template>

<script>
import { onMounted, reactive } from "vue";
export default {
  setup() {
    let jsonData = reactive([
      {
        name: "牛二",
        age: 13,
      },
    ]);
    jsonData = reactive([
      {
        name: "王五",
        age: 19,
      },
    ]);
    onMounted(() => {
      jsonData = reactive([
        {
          name: "赵六",
          age: 100,
        },
      ]);
    });
    return {
      jsonData,
    };
  },
};
</script>
```

**重赋值对象自身**跟**重赋值对象的属性** ？？

reactive ==> 是响应式的是它的属性，而不是它自身，重赋值它自身跟重赋值它的属性是两码事。

ref ==> 数据具有响应式

#### shallowRef 和 shallowReactive使用场景：

- shallowReactive：只处理**最外层**属性的响应式（浅响应式），适用于对象结构较深，但是只会是最外层发生改变的数据
- shallowRef：只处理基本数据类型的响应式，不进行对象的响应式处理，适用于节点引用或生成新的对象来替换的

### toRef 和 toRefs

#### toRef

基于**响应式对象**上的一个属性，创建一个对应的 ref，创建的 ref 与其源属性保持同步，即改变源属性的值也将更新ref 的值



```javascript
<template>
<div>
  <button @click="r.a++">count is: {{ r.a }}</button>
  <button>count is: {{ s }}</button>
</div>
</template>

<script>
import { reactive, toRef } from "vue";
export default {
  setup() {
    let r = reactive({a:1});
    console.log(r);
    let s = r.a;
    console.log(s);
    return {
      r,s
    };
  },
};
</script>
```

当我点击button1的时候，你说button2会变吗？并不会。变量s就是个基本数据

**使用场景：**

当一个变量指向一个对象的某个property，且这个property是基本数据类型时，必须用toRef才能变量与对象的响应式连接。如果这个property是引用数据类型，就不需要动用toRef。

toRef的用途之一是用于传参，可传递一个响应式的基本数据类型

#### toRefs

将一个响应式对象转换为一个普通对象（破坏响应式对象），并把这个对象的每个属性都是指向源对象相应属性的ref（可以理解为toRefs可以看做批量版本的toRef）

toRefs的一大用途是变相解构Proxy：

- 比如之前没有用 setup 时我们一般return {...toRefs(Proxy)}

```javascript
<template>
<div>
  <button @click="r.c = 3">count is: {{ r.c }}</button>
  <button>count is: {{ s }}</button>
  <button>count is: {{ t.c.value }}</button>
</div>
</template>

<script>
import { reactive, toRef, toRefs } from "vue";
export default {
  setup() {
    let r = reactive({a:{b:2}, c: 4});
    console.log(r);
    let s = toRef(r, 'c');
    console.log(s);
    let t = toRefs(r);
    console.log(t.c)
    return {
      r,s,t
    };
  },
};
</script>
```



#### toRef 和 toRefs

|          | toRef                           | toRefs                                      |
| -------- | ------------------------------- | ------------------------------------------- |
| 用法     | toRef(Proxy, 'xxx')             | toRefs(Proxy)                               |
| 返回     | ObjectRefImpl对象               | ObjectRefImpl对象                           |
| 作用     | 创建变量到Proxy属性的响应式连接 | 创建变量每个属性到Proxy每个属性的响应式连接 |
| 连接关系 | 一对一                          | 多对多                                      |

```javascript
<template>
<div>
  <button @click="r.c = 3">count is: {{ r.c }}</button>
  <button>count is: {{ s }}</button>
  TODO <button>count is: {{ t.c.value }}</button>
</div>
</template>

<script>
import { reactive, toRef, toRefs } from "vue";
export default {
  setup() {
    let r = reactive({a:{b:2}, c: 4});
    console.log(r);
    let s = toRef(r, 'c');
    console.log(s);
    let t = toRefs(r);
    console.log(t.c)
    return {
      r,s,t
    };
  },
};
</script>
```

### readonly 和 shallowReadonly

readonly：让一个响应式数据变为只读的（深只读，所有嵌套结构），不允许被修改，但已经被代理

- 场景： 一个是保护数据不被修改，另一个是提升性能

```javascript

<template>
<div>
  <button @click="r.b.c++">count is: {{ r.b.c }}</button>
  <button @click="s.b.c++">count is: {{ s.b.c }}</button>
</div>
</template>

<script>
import { reactive, readonly } from "vue";
export default {
  setup() {
    let r = reactive({a: 1, b: {c: 2}});
    console.log(r);
    let s = readonly({a: 1, b: {c: 2}});
    console.log(s);
    return {
      r,s
    };
  },
};
</script>
```

第2个button点击永远不会有反应

- shallowReadonly：让一个响应式数据外层属性变为只读（浅只读，深层次嵌套可以修改）

```javascript

<template>
<div>
  <button @click="r.b.c++">count is: {{ r.b.c }}</button>
  <button @click="s.b.c++">count is: {{ s.b.c }}</button>
  <button @click="s.a++">count is: {{ s.a }}</button>
</div>
</template>

<script>
import { readonly, shallowReadonly } from "vue";
export default {
  setup() {
    let r = readonly({a: 1, b: {c: 2}});
    console.log(r);
    let s = shallowReadonly({a: 1, b: {c: 2}});
    console.log(s);
    return {
      r,s
    };
  },
};
</script>
```

- 按下button1会有报错提示：Set operation on key "c" failed: target is readonly.，因为r是深层只读的。
- 按下button2没有任何反应，因为shallowReadonly的深层是指向原始值的，修改原始对象不会反映到视图上。
- 按下button3也会有报错提示：Set operation on key "a" failed: target is readonly.，因为shallowReadonly是浅层只读的，a恰好是浅层property。

？？？ 那为什么定义一个数据为响应式但是为什么又要包装为只读的

**比如一个场景**：数据 A 是从别的组件传入的，但是这个数据只希望你在这个组件去引用别修改的情况，那就可以用 readonly 去处理

### toRaw 与 markRaw

#### toRaw()

toRaw的参数只能是一个响应式的对象（可以理解为是**被 reactive 处理过的数据，针对ref 缔造的响应式数据无效**）

返回是proxy的原始对象，相当于 reactive 的逆运算，把响应式数据还原成普通对象，对这个转换后的对象所有操作，不会引起页面更新

2个作用：可用于临时读取而不会引起proxy访问/跟踪开销，也可用于写入而不会触发视图更新

官方又说，不建议保留对原始对象的持久引用。请谨慎使用。

1. 尽量不要把原始对象赋值给变量，尽量减少中间变量；
2. 将原始对象转换为Proxy之后，如果你临时打算操作一下原始对象，那么也不要因为这个目的就早早的把原始对象赋值给变量，而是应该用toRaw(proxy)，以获取原始对象，比如得到一个变量R，然后你可以操作R，操作完成之后就不要再碰R，而应继续操作Proxy。

```javascript

<template>
<div>
  <button @click="r.b.c++">count is: {{ r.b.c }}</button>
  <button @click="s.b.c++">count is: {{ s.b.c }}</button>
</div>
</template>

<script>
import { reactive, toRaw } from "vue";
export default {
  setup() {
    let r = reactive({a: 1, b: {c: 2}});
    console.log(r);
    let s = toRaw(r);
    console.log(s);
    return {
      r,s
    };
  },
};
</script>
```

- 按下button1，会发现button2也跟着变，这表明Proxy的基本原理：操作Proxy会反映到原始对象身上。
- 按下button2，没有任何反应，操作原始对象不会反映到视图上。这时候重新按下button1，会发现数字跳跃了几个数，这表明直接修改原始对象之后，Proxy对原始对象继续代理，并不需要重新reactive。

#### markRaw

标记一个对象，使其永远不会再成为响应式对象（允许被修改，但不允许被代理）

markRaw是操作原始对象的，它的是将原始对象或者原始对象的某个浅层或深层property标记为“永远不允许被代理”。

Vue3会给对象的第一层或某深层加一个标记__v_skip: true，这样即便原始对象被reactive之后，得到的该层和更深层就不会被代理。

**markRaw的用途：**

先说结论：直接给某个对象全盘markRaw是没有意义的，markRaw的用途应该是允许对象被reactive，但是阻止对象的部分内容被reactive

```javascript

<template>
<div>
  <button @click="s.b.c++">count is: {{ s.b.c }}</button>
</div>
</template>

<script>
import { markRaw, reactive } from "vue";
export default {
  setup() {
    let x = {a:1, b: {c: 2}};
    x.b = markRaw(x.b);
    let s = reactive(x);
    console.log(isReactive(s)); // true
    console.log(isReactive(s.b)); // false
    return {
      s
    };
  },
};
</script>
```

**使用场景**

- 有些值不应该设置为响应式的，比如复杂的第三方类库或 Vue 组件对象等
- 当渲染具有不可变数据源的大列表时，跳过响应式转换可以提高性能

#### markRow 和 shallowReactive 的区别

|              | markRow    | shallowReactive                          |
| ------------ | ---------- | ---------------------------------------- |
| 作用         | 阻止响应式 | 让浅层property响应式，不操作深层property |
| 浅层property | 阻止响应式 | 执行响应式                               |
| 深层property | 阻止响应式 | 不执行响应式，也不阻止                   |

#### markRaw与readonly的区别

- markRaw允许被修改，但不允许被代理。这里尽管说允许修改，但是修改的意义不大，毕竟Vue的核心思想是响应式，在添加响应式之前修改意义不大。
- readonly不允许被修改，但已经被代理

### watch 和 watchEffect()

#### watch()

watch 一般用于侦听一个或多个响应式数据源，并在数据源变化时调用所给的回调函数（与vue2作用是一样的），watch 默认是懒侦听的，即仅在侦听源发生变化时才执行回调函数。

- 当侦听 reactive 定义的响应式数据（因为reactive 只能定义数组或对象类型的响应式）时，oldValue 无法正确获取**，会强制开启深度监视，此时 deep 配置无效**
- 如果是使用的 getter 函数返回响应式对象的形式，那么响应式对象的属性值发生变化，是不会触发 watch 的回调函数的

##### **watch的监听类型**

先看watch 属性的Ts类型定义

```typescript
interface ComponentOptions {
  watch?: {
    [key: string]: WatchOptionItem | WatchOptionItem[]
  }
}

type WatchOptionItem = string | WatchCallback | ObjectWatchOptionItem

type WatchCallback<T> = (
  value: T,
  oldValue: T,
  onCleanup: (cleanupFn: () => void) => void  // 清理副作用
) => void

type ObjectWatchOptionItem = {
  handler: WatchCallback | string
  immediate?: boolean // default: false
  deep?: boolean // default: false
  flush?: 'pre' | 'post' | 'sync' // default: 'pre'
  onTrack?: (event: DebuggerEvent) => void
  onTrigger?: (event: DebuggerEvent) => void
}
```

**参数信息：**

- 第一个参数是侦听器的**源**。这个来源可以是以下几种：

  - 一个返回任意值的函数（**getter 函数**或者**响应式对象的某个属性**）:（）=> xxx；
  - 一个**响应式对象** （ref、computed、reactive）；
  - ...一个包含上述两种数据源的**数组**；

- 第二个参数是在发生变化时要调用的回调函数。

  - 回调函数接受三个参数：**新值、旧值，以及一个用于注册副作用清理的回调函数**。
  - 回调函数会在副作用下一次重新执行前调用，可以用来清除无效的副作用（同watchEffect）；
  - 当侦听多个来源时，回调函数接受两个数组，分别对应来源数组中的新值和旧值。

- 第三个可选的参数是一个对象，支持以下这些选项：

  - **immediate**：在侦听器创建时立即触发回调。第一次调用时旧值是 `undefined`（watchEffect 自带，无须配置）
  - **deep**：如果源是对象，强制深度遍历，以便在深层级变更时触发回调
  - **flush**：调整回调函数的刷新时机（同 watchEffect ）
  - **onTrack / onTrigger**：调试侦听器的依赖（同 watchEffect ）


它本质上充当组件响应式数据的事件监听器，特别是当与异步 API 调用配合使用，在项目中使用场景一般是:

1. 当 ID 改变时，从数据库中获取一个对象【数据初始化】
2. 当 `prop` 更改时重新运行动画 【数据初始化】
3. 监听路由变化
4. `input` 输入框值的特殊处理等

##### **使用场景**

- 你需要控制哪些依赖项会触发该方法（目的明确，有点类似于需要触发某些响应式数据的时机）
- 你需要访问之前的值

#### watchEffect()

它立即执行传入的一个函数，同时响应式**追踪其依赖**，并在其依赖变更时重新运行该函数

比watch不同的是，这个方法是默认初始化会之执行一次副作用函数，不需要添加属性 immediate: true

##### Ts 类型定义

```typescript
function watchEffect(
  effectFn: (onCleanup: OnCleanup) => void,
  options?: WatchEffectOptions
): StopHandle

type OnCleanup = (cleanupFn: () => void) => void

interface WatchEffectOptions {
  flush?: 'pre' | 'post' | 'sync' // 默认：'pre'
  onTrack?: (event: DebuggerEvent) => void  // 在响应式 property 或 ref 作为依赖项被追踪时被调用
  onTrigger?: (event: DebuggerEvent) => void // 在依赖项变更导致副作用被触发时被调用
}

type StopHandle = () => void
```

**参数信息：**

- **effectFn**：要运行的副作用函数，用来注册清理回调。

  清理回函数会在该副作用下一次执行前被调用，可以用来清理无效的副作用，例如等待中的异步请求

  ```javascript
  const CancelToken = axios.CancelToken;
  let cancel;
  
  const getData = ()=>{
    axios.get('xxx', {
      cancelToken: new CancelToken(function executor(c) {
        cancel = c;
      },params:{})
    });
  }
  
  watchEffect((onCleanup) => {
    onCleanup(cancel)  // 先取消上一次的请求
    if(props.id) getData(id) 
  })
  ```

- **options**：一个可选的选项，一般用来调整副作用的刷新时机或调试副作用的依赖

  - 默认情况下，watch的回调函数会在组件重新渲染之前执行，flush 属性可以设置回调函数的触发时机（post：在组件渲染之后再执行；sync：在响应式依赖发生改变时立即触发侦听器），使用时可能会导致页面性能和数据不一致的情况
  - ​flush这个在源码内部其实是通过promise.then方法构造微任务来处理的

- **返回值**：返回一个用来停止该副作用的函数（停止监听）

  ```javascript
  const stop = watchEffect((onCleanup) => {
    if (props.id) {
      getData() // 获取接口信息
      stop() // 达到条件，停止监听
    }
  })
  ```


##### 清除副作用(共享)

**无效副作用**：有时副作用函数会执行一些异步的副作用，这些响应需要在其失效时清除  (即完成之前状态已改变了) 。

侦听副作用传入的函数可以接收一个 onCleanup函数作入参，用来注册清理失效时的回调（每当该方法要再次运行或监视程序停止时，该方法就会运行）

当以下情况发生时，这个失效回调会被触发：

- 副作用即将重新执行时
- 侦听器被停止 (如果在 `setup()` 或生命周期钩子函数中使用了watchEffect，则在组件卸载时)

```javascript
export default {
  setup() {
    watchEffect(onCleanup => {
      const apiCall = someAsyncMethod(props.songID) // 异步 API
      onCleanup(() => {
        apiCall.cancel() // 取消 API 调用
      })
    })
  }
}
```

##### watchEffect 和 watch

既然已经有了 watch方法，为什么这个新的 watchEffect 还会存在呢？

- watchEffect 将在方法的**任何**依赖项发生更改时运行，`watch` 跟踪一个或多个**特定**的响应性属性，并且仅在该属性发生更改时运行。
- 默认情况下，`watch` 是惰性的，因此仅当依赖项更改时才会触发。`watchEffect` 在创建组件后立即运行，然后跟踪依赖关系。

##### watchEffect 和 computed

warchEffect 和 watch 的功能和写法都挺好理解的，反而觉得它和 computed 有点像，对比一下区别

- computed 主要注重的是计算出来的值（回调函数返回值），且必须要写返回值
- watchEffect 始终注重的是过程（函数体），不用写返回值
- computed 如果没有值被使用时是不会调用的，但是watchEffect 一定会至少调用一次（初始化那次，目的是自动获取依赖，这里又区分了watch 和 computed 本质区别）

##### watchEffect 和 useEffect（React）

其实详细了解，我感觉它和React中的useEffect很像（写法的问题），只是不需要传入第二个参数依赖项数组

- 初始化立即执行一次逻辑处理函数
- 依赖的改变触发函数的执行
- 第一个参数返回一个回调函数，可以在该函数中将组件被摧毁之前和再一次触发更新时，将之前的副作用清除掉（不断循环的订阅（计时器，或者递归循环），和watchEffect写法不一样

##### **使用场景**

1. 比 watch 方便，写法、immediate 方向的使用
2. 不需要持续监听【比较适合初始化根据某部分数据执行方法，后面数据不需要再更改】
3. 不需要关心监听的数据具体变化的值，只关注结果

#### watch 与 watchEffect 共享的行为

##### 停止侦听

如果我们想观察一个依赖项直到达到某些目的，然后停止它，这可能很有用。如果我们在它达到目标值后继续观察，等于是在**浪费资源**。

在异步场景下，清理失效的回调，保证当前副作用有效，不会被覆盖；

通常来说，在组件被销毁或者卸载后，监听器也会跟着被销毁。但是总是有一些特殊情况（不断循环的订阅（计时器，或者递归循环），即使组件卸载了，但是监听器依然存在，这个时候其实式需要我们手动关闭它的，否则容易造成内存泄漏

```javascript
<script setup>
import { watchEffect } from 'vue'

watchEffect(() => {}) // 它会自动停止
// ...这个则不会！
setTimeout(() => {
  watchEffect(() => {})
}, 100)
</script>
```

上段代码中我们采用异步的方式创建了一个监听器，这个时候监听器没有与当前组件绑定，所以即使组件销毁了，监听器依然存在。

我们需要用一个变量接收监听器函数的返回值（函数），我们调用该函数，即可关闭当前监听器。

```javascript
const unwatch = watchEffect(() => {})

unwatch() // ...当该侦听器不再需要时
```

##### 副作用刷新时机

如果我们在监听器的回调函数中或取 DOM，这个时候的 DOM 是更新前的，可以通过给监听器多传递一个参数选项：flush: 'post' 来调整获取对应状态的 DOM

Vue 的响应性系统会缓存副作用函数，并异步地刷新它们，这样可以避免同一个“tick” 中多个状态改变导致的不必要的重复调用。

在核心的具体实现中，组件的 `update` 函数也是一个被侦听的副作用。当一个用户定义的副作用函数进入队列时，默认情况下，会在所有的组件 `update` **前**执行：

```javascript
<template>
  <p ref="msgRef">{{ message }}</p>
  <button @click="changeMsg">更改 message</button>
</template>
<script setup lang="ts">
import { computed, reactive, ref, watch, watchEffect } from "vue";


const message = ref("0");
const msgRef = ref<any>(null);
const changeMsg = () => {
  message.value = "1"
};
watch(message, (newValue, oldValue) => {
  console.log("DOM 节点", msgRef.value.innerHTML);
  console.log("新的值:", newValue);
  console.log("旧的值:", oldValue);
});
</script>
```

- `count` 会在初始运行时同步打印出来
- 更改 `count` 时，将在组件**更新前**执行副作用。

如果需要在组件更新**后**重新运行侦听器副作用，我们可以传递带有 flush 选项的参数 (默认为 `'pre'`)：

```javascript
// 在组件更新后触发，这样你就可以访问更新的 DOM。
// 注意：这也将推迟副作用的初始运行，直到组件的首次渲染完成。
watchEffect(
  () => {
    /* ... */
  },
  {
    flush: 'post'
  }
)
```

### v3.2升级的那些好用的api

#### < style > v-bind

该指令适用于`<script setup>`, 并支持 JavaScript 表达式（必须用引号括起来）。

原理就是自定义属性将通过内联样式应用于组件的根元素，并在数值更改时进行响应更新。

```javascript
<template>
    <h2>{{ color }}</h2>
	<button @click="color = 'red'">color red</button>
    <button @click="color = 'yellow'">color yellow</button>
	<button @click="color = 'blue'">color blue</button>
    <button @click="fontSize = '40px'">fontSize 40px</button>
</template>

<script setup>
import { ref } from "vue";
const color = ref("pink");
color.value = "green";
const fontSize = ref("18px");
</script>

<style scoped>
h1 {color: v-bind(color);
}
h2 {font-size: v-bind(fontSize);
}
</style>
```

#### v-memo 指令

提供了记忆一部分模板树的能力。 v-memo 指令使得这部分模板可以跳过虚拟 DOM 的 diff 比较，同时还完全跳过新 VNode 的创建。 虽然很少需要，但它提供了一种在某些情况下想要得到最大性能的方案，例如大型 v-for 列表

可以用来有条件地跳过某些大型子树或者 `v-for` 列表的更新

不仅允许 Vue 跳过虚拟 DOM 差异、甚至可以完全跳过新 VNode 的创建步骤

```javascript
<div v-for="user of users" :key="user.id" v-memo="[user.name]">{{ user.name }}
</div>
```

使用`v-memo`，不会重新创建虚拟元素，并且会重新使用前一个元素，除非`v-memo`（user.name）的条件发生变化。

这个用来渲染大量元素，在性能上来说应该是比前面响应式API再大的改进。

#### effectScope（）

用于创建一个 effect 作用域，可以捕获其中所创建的响应式副作用（computed 和  watchers），这样捕获到的副作用就可以 一起处理（直接销毁作用域）

**类型：**

```javascript
function effectScope(detached?: boolean): EffectScope

interface EffectScope {
  run<T>(fn: () => T): T | undefined // 如果作用域不活跃就为 undefined
  stop(): void
}
```

在v3.2之前，如果想要在组件中停止 computed & watch，可以把我们定义的每一个 computed 和 watch 返回值维护到统一的一个数组内，然后调用数组中的每个 stopHanlde，就可以停止所有响应【可以实现，但是麻烦】

```javascript
const disposables = []

const counter = ref(0)
const doubled = computed(() => counter.value * 2)
disposables.push(() => stop(doubled.effect))

const stopWatch1 = watchEffect(() => {
  console.log(`counter: ${counter.value}`)
})
disposables.push(stopWatch1)

const stopWatch2 = watch(doubled, () => {
  console.log(doubled.value)
})

disposables.push(stopWatch2)

disposables.forEach((f) => f())
disposables = []
```

vue 3.2之后，Effect Scope API 出现了（是 vue 的一个高阶API，主要服务于库作者），专门用来解决上面的问题：

effectScope 是一个函数，返回一个对象：其中包含了 run 函数和 stop 函数：

```javascript
const scope = effectScope()

scope.run(() => {
  const doubled = computed(() => counter.value * 2)

  watch(doubled, () => console.log(doubled.value))

  watchEffect(() => console.log('Count: ', doubled.value))
})

scope.stop() // 处理掉当前作用域内的所有 effect
```

- 一般我们的 computed 和 watch 这些 api 都是在组件中调用，在这期间，代码产生的 effect 是会自动收集且绑定到当前的组件实例上，在组件卸载的时候，它们也会随之 stop，无需开发手动管理清除
- 在vue3 的项目设计时，`@vue/reactivity`这个包是可以独立引入使用的，所以只要是可以跑 js 的地方就可以调用这些 effect 函数，不必去依赖组件（所以就失去了 vue 自动管理 effect 的能力），开发就需要手动收集 effect，再在合适的时机手动 stop

### extends

要继承的“基类”组件，使一个组件可以继承另一个组件的组件选项。

**类型：**

```javascript
interface ComponentOptions {
  extends?: ComponentOptions
}
```

从实现角度来看，`extends` 几乎和 `mixins` 相同。通过 `extends` 指定的组件将会当作第一个 mixin 来处理。

然而，`extends` 和 `mixins` 表达的是不同的目标。`mixins` 选项基本用于组合功能，而 `extends` 则一般更关注继承关系。

同 `mixins` 一样，所有选项都将使用相关的策略进行合并。

```javascript
const CompA = { ... }

const CompB = {
  extends: CompA,
  ...
}
```

## dev 调试钩子

对应 上面 watch的 onTrack、onTrigger方法

页面 renderTracked和renderTriggered

```javascript
interface ComponentOptions {
  renderTracked?(this: ComponentPublicInstance, e: DebuggerEvent): void
}

type DebuggerEvent = {
  effect: ReactiveEffect
  target: object
  type: TrackOpTypes /* 'get' | 'has' | 'iterate' */
  key: any
}
```

```javascript
interface ComponentOptions {
  renderTriggered?(this: ComponentPublicInstance, e: DebuggerEvent): void
}

type DebuggerEvent = {
  effect: ReactiveEffect
  target: object
  type: TriggerOpTypes /* 'set' | 'add' | 'delete' | 'clear' */
  key: any
  newValue?: any
  oldValue?: any
  oldTarget?: Map<any, any> | Set<any>
}
```



### onRenderTracked 组件状态跟踪

当组件渲染过程中**追踪到**响应式依赖时调用  
当组件渲染过程中追踪到响应式依赖时调用，用来调试查看哪些依赖正在被使用（这是一个**生命周期钩子**）

onRenderTracked 会跟踪页面上所有响应式变量和方法的状态（可以理解为用 return 返回的值都会跟踪），只要页面有 update 的情况，就会跟踪，生成一个 event 对象

### onRenderTriggered 组件状态触发

当响应式依赖的变更触发了组件渲染时调用，用来确定哪个依赖正在触发更新

onRenderTriggered 不会跟踪每一个值，而是给你变化值的信息，并且新值和旧值都会给你明确的展示出来。

可以理解为就是跟踪渲染页面的变量，js 变量不会跟踪，更精准，会返回：

- 变化变量的 key
- newValue 更新后变量的值
- oldValue 更新前变量的值
- target 目前页面中的响应变量和函数

感觉和watch 很像，而且是 immidate= true 的时候