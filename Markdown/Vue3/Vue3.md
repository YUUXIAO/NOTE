## 用法

### ref/reactive/toRef/toRefs

#### ref

> ref 用来包装原始值类型（确切的说是基本数据类型 int 或 string），返回的是一个包含 .value 属性的对象；

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

### props 推导

```javascript
import { defineComponent } from 'vue';
export default defineComponent({
  props: {
    val1: String,
    val2: { type: String, default: '' },
  },
  setup(props, context) {
    props.val1;
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

