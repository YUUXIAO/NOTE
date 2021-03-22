## 用法

### ref/reactive/toRef/toRefs

#### ref

> ref 用来包装原始值类型（确切的说是基本数据类型 int 或 string），返回的是一个包含 .value 属性的对象；

- 包装对象的意义在于提供一个让我们能够在函数之间以引用的方式传递任意类型值的容器；

  ```javascript
  setup() {
    // valueA 可能被 useLogicA() 内部的代码修改从而触发更新
    const valueA = useLogicA() 
    const valueB = useLogicB()
    return {
      valueA,
      valueB
    }
  ```


- isRef 就是判断一下是不是ref生成的响应式数据对象；

```javascript
setup(props, context) {
  const count = ref<number>(1);
  count.value = 2;
  console.log('count.value :>> ', count.value);
  return { count };
}
```

在 template 中 ref 包装对象会被自动展开，模板中不再用 .value；

```html
<template>  
  {{count}}
</template>
```

#### reactive

> reactive( ) 函数接受一个普通对象 返回一个响应式数据对象；

它用来返回一个响应式对象，本身就是对象，所以不需要包装，使用它的属性时不需要加 .value 来获取；

```javascript
const obj = reactive({
  count: 0
})
obj.count++
```

#### toRefs

> toRefs 可以将reactive创建出的对象展开为基础类型；

因为 props 是响应式的，不能使用 ES6 解构，它会消除 prop 的响应性；

为了方便对它进行包装，toRefs 可以成批量包装 props 对象；

可以理解是因为要使用解构，toRefs 所采取的解决方案；

```javascript
const { name } = toRefs(props);
const handleClick = () => {
  console.log('name :>> ', name.value);
};
```

#### toRef

> toRef 的用法，就是多了一个参数，允许针对一个 key 进行包装；

```javascript
const name = toRef(props,'name');
console.log('name :>> ', name.value);
```

### watchEffect & watch

#### watchEffect 

> 它立即执行传入的一个函数，同时响应式追踪其依赖，并在其依赖变更时重新运行该函数；

执行失效的回调有两个时机：

1. 副作用即将重新执行时，也就是监听的数据发生改变时；
2. 组件卸载时；

在异步场景下，清理失效的回调，保证当前副作用有效，不会被覆盖；

```javascript
watchEffect(onInvalidate => {
  // 执行副作用
  // ...
  onInvalidate(() => {
    // 执行/清理失效回调
    // ...
  })
})
```

#### watch

> watch（） API 提供了基于观察状态的变化来执行副作用的能力；

watch（）接收的第一个参数被称作 “数据源”，它可以是：

1. 一个返回任意值的函数；
2. 一个包装对象；
3. 一个包含上述两种数据源的数组；

第二个参数是回调函数，回调函数只有当数据源发生变动时才会被触发：

```javascript
watch(
  // getter
  () => count.value + 1,
  // callback
  (value, oldValue) => {
    console.log('count + 1 is: ', value) // -> count + 1 is: 1
  }
)
count.value++;  // -> count + 1 is: 2

```

##### 观察多个数据源

> watch（）可以观察一个包含多个数据源的数组，此时任意一个数据源的变化都会触发回调，回调会接收到包含对应值的数组作为参数；

```javascript
watch(
  [refA, () => refB.value],
  ([a, b], [prevA, prevB]) => {
    console.log(`a is: ${a}`)
    console.log(`b is: ${b}`)
  }
)
```

##### 停止观察

> watch（）返回一个停止观察的函数；

如果 watch（）是在一个组件的 setup（）或是生命周期函数中被调用的，那该 watch（）会在当前组件被销毁时也被自动停止；

```javascript
// 返回一个停止观察的函数
const stop = watch(...)
stop()

// 在 setup()中被调用
export default {
  setup() {
    // 组件销毁时也会被自动停止
    watch(/* ... */)
  }
}
```

##### 清理副作用

> 有时当观察的数据源变化后可能需要对之前所执行的副作用进行清理；

watcher 的回调会接收到第三个参数是一个用来注册清理操作的函数，调用这个函数可以注册一个清理函数；

清理函数会在下属情况下被调用：

1. 在回调被下一次调用前；
2. 在 watcher 被停止前；

```javascript
watch(idValue, (id, oldId, onCleanup) => {
  const token = performAsyncOperation(id)
  onCleanup(() => {
    // id 发生了变化，或是 watcher 即将被停止.
    // 取消还未完成的异步操作。
    token.cancel()
  })
})
```

##### Watcher 回调的调用时机

> 默认情况下，所有的 watcher 回调都会在当前的 renderer flush 之后被调用，确保了在回调中 DOM 永远都已经被更新完毕；

如果想要让回调在 DOM 更新之前或是被同步触发，可以使用 flush 选项；

```javascript
watch(
  () => count.value + 1,
  () => console.log(`count changed`),
  {
    flush: 'post', // default, fire after renderer flush
    flush: 'pre', // fire right before renderer flush
    flush: 'sync' // fire synchronously
  }
)
```

##### 全部的 watch 选项

```typescript
interface WatchOptions {
  lazy?: boolean  // 与 2.x 的 immediate 正好相反
  deep?: boolean  // 与 2.x 行为一致
  flush?: 'pre' | 'post' | 'sync'
  // onTrack 和 onTrigger 是两个用于 debug 的钩子，分别在 watcher 追踪到依赖和依赖发生变化的时候被调用，获得的参数是一个包含了依赖细节的 debugger event
  onTrack?: (e: DebuggerEvent) => void
  onTrigger?: (e: DebuggerEvent) => void
}

interface DebuggerEvent {
  effect: ReactiveEffect
  target: any
  key: string | symbol | undefined
  type: 'set' | 'add' | 'delete' | 'clear' | 'get' | 'has' | 'iterate'
}
```

##### 与2.x比较

和 2.x 的 $watch 不同的是，watch（）的回调会在创建的时候就执行一次，默认情况下 watch（）的回调总是会在当前的 render flush 之后才会被调用，所以 watch（）的回调在触发时，DOM 处于一个被更新过的状态；

### computed

> computed（）返回的是一个只读的包装对象，它可以和普通的包装对象一样在 setup 中被返回，在渲染上下文中被自动展开；

双向计算值可以通过传给 computed 第二个参数作为 setter 来创建；

```javascript
const count = value(0)
const writableComputed = computed(
  // read
  () => count.value + 1,
  // write
  val => {
    count.value = val - 1
  }
)
```

### fragment

> 在 Vue3 允许我们有多个 root ，也就是片段；

当 inheritAttrs=true [默认]时，组件会自动在 root 继承合并 class ；

```html
// 子组件
<template>
  <div class="fragment">
    <div>div1</div>
    <div>div2</div>
  </div>
</template>

// 父组件
<MyFragment class="extend-class" />

// 子组件渲染后
<div class="fragment extend-class">
  <div> div1 </div>
  <div> div2 </div>
</div>
```

如果我们使用了 fragment，就需要显式的去指定绑定 attrs；

```html
<template>
  <div v-bind="$attrs">div1</div>
  <div>div2</div>
</template>
```

### v-model

可以在一个组件上使用 多个 v-model 语法糖；

```javascript
<VModel v-model="show"
        v-model:model1="check"
        v-model:model2.hello="textVal" />

// ...
props: {
  modelValue: { type: Boolean, default: false },
  model1: { type: Boolean, default: false },
  model2: { type: String, default: '' },
  model2Modifiers: {
    type: Object,
    default: () => ({})
  }
},
emits: ['update:modelValue', 'update:model1', 'update:model2'],
// ...
```



## 底层优化

### Proxy代理

> Proxy 代理的是整个对象而不是对象的属性，对于整个对象进行操作；

1. 用 Proxy 对 对象动态添加的属性也会被拦截到；
2. 可以监听数组变化；

```javascript
const targetObj = { id: '1', name: 'zhagnsan' };
const proxyObj = new Proxy(targetObj, {
  get: function (target, propKey, receiver) {
    console.log(`getting key：${propKey}`);
    return Reflect.get(...arguments);
  },
  set: function (target, propKey, value, receiver) {
    console.log(`setting key：${propKey}，value：${value}`);
    return Reflect.set(...arguments);
  }
});
proxyObj.age = 18;
// setting key：age，value：18
```

### 静态提升vdom

> Vue3 优化了 vdom 的更新性能，静态提升包含静态节点和静态属性的提升，把一些静态的不会变的节点用变量缓存起来，提供下次 re-render 直接调用；

```javascript
// template
<div class="div">
  <div>content</div>
  <div>{{message}}</div>
</div>

// 没有静态提升
function render(_ctx, _cache, $props, $setup, $data, $options) {
  return (_openBlock(), _createBlock("div", { class: "div" }, [
    _createVNode("div", null, "content"),
    _createVNode("div", null, _toDisplayString(_ctx.message), 1 /* TEXT */)
  ]))
}

// 有静态提升 
const _hoisted_1 = { class: "div" }
const _hoisted_2 = /*#__PURE__*/_createVNode("div", null, "content", -1 /* HOISTED */)

function render(_ctx, _cache, $props, $setup, $data, $options) {
  return (_openBlock(), _createBlock("div", _hoisted_1, [
    _hoisted_2,
    _createVNode("div", null, _toDisplayString(_ctx.message), 1 /* TEXT */)
  ]))
}
```

## build

### tree-shaking

> tree-sharking 即在构建工具构建后消除程序中无用的代码，来减少包的体积；

1. 基于函数的 API 每一个函数都可以作为 named ES export 被单独引入，使得它们对 tree-shaking 非常友好；
2. 没有被使用的 API 的相关代码可以在最终打包时被移除；
3. 基于函数 API 所写的代码也有更好的压缩效率，因为所有的函数名和 setup 函数体内部的变量名都可以被压缩，但对象和 class 的属性/方法名却不可以；

```javascript
// utils
const obj = {
  method1() {},
  method2() {}
};
export function method1() {}
export function method2() {}
export default obj;

// 调用
import util from '@/utils';
import { method1 } from '@/utils';
method1();
util.method1();

// 经过webpack打包压缩之后
function a(){}a();
a={method1:function(){},method2:function(){}};a.method1();
```

因为 Vue3 用的是基于函数形式的 API，能更好的 tree-shaking；

```javascript
import {
  defineComponent,
  reactive,
  ref,
  watchEffect,
  watch,
  onMounted,
  toRefs,
  toRef
} from 'vue';
```

## TS

### defineComponent

> 在 TS 下，需要用 Vue 暴露的方法 defineComponent，它单纯为了类型推导而存在的；

### 类型推导

为了能够在 TS 中提供正确的类型推导，需要通过一个函数来定义组件；



```javascript
import { defineComponent, ref } from 'vue'

const MyComponent = defineComponent({
  // props declarations are used to infer prop types
  props: {
    msg: String
  },
  setup(props) {
    props.msg // string | undefined

    const count = ref(0)
    return {
      count
    }
  }
})

// 如果使用手写 render 函数或是 TSX，可以在 setup() 当中直接返回一个渲染函数
const MyComponent = defineComponent({
  props: {
    msg: String
  },
  setup(props) {
    const count = ref(0)
    return () => h('div', [
      h('p', `msg is ${props.msg}`),
      h('p', `count is ${count.value}`)
    ])
  }
}) 
```

### PropType

> props 定义的类型，如果是一个复杂对象，我们就要用 PropType 来进行强转声明；

```javascript
interface IObj {
  id: number;
  name: string;
}

obj: {
  type: Object as PropType<IObj>,
  default: (): IObj => ({ id: 1, name: '张三' })
},
  
// 联合类型
type: {
  type: String as PropType<'success' | 'error' | 'warning'>,
  default: 'warning'
},
```

## 与React Hooks比较

React Hooks 在每次组件渲染时都会调用，通过隐式地将状态挂载在当前的内部组件节点上，在下一次渲染时根据调用顺序取出；

Vue 的 setup() 每个组件实例只会在初始化时调用一次 ，状态通过引用储存在 setup() 的闭包内，意味着基于 Vue 的函数 API 的代码：

1. 整体上更符合 JavaScript 的直觉；
2. 不受调用顺序的限制，可以有条件地被调用；
3. 不会在后续更新时不断产生大量的内联函数而影响引擎优化或是导致 GC 压力；
4. 不需要总是使用 useCallback 来缓存传给子组件的回调以防止过度更新；
5. 不需要担心传了错误的依赖数组给 useEffect/useMemo/useCallback 从而导致回调中使用了过期的值 —— Vue 的依赖追踪是全自动的；

