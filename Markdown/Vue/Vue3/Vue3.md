问题合集

- setup 同步还是异步，使用异步的 customRef



## vue3 响应式主要功能

- proxy  对象实现属性监听
- 多层属性嵌套，在访问属性过程中处理下一级属性
- 默认监听动态添加的属性
- 默认监听属性的删除操作
- 默认监听数组索引和length属性
- 可以作为单独的模块使用







## 设计目标

- 更小：移除了一些不常用的 API；引入 tree-shaking，可以将无用模块剪辑，仅打包需要的，使打包体积更小
- 更快：diff算法更新；静态提升；事件监听缓存；SSR优化
- 更友好：增加了 composition API，增加了代码的逻辑组织和复用能力（更像是逻辑方法化）；基于 typescript 编写，可以享受自动的类型定义提示

## 优化方案

- 源码

  - 源码管理：monorepo 的方式维护，可以根据功能模块的不同拆分到 packages 目录下的子目录中，模块拆分更细化，职责更明确，一些package 是可以单独引入使用的（reactive库）
  - Typesctipt：基于 typeScript 编写的，提供了更好的类型检查，能支持复杂的类型推导

- 性能

  - 体积优化：

  - 编译优化：

    - **静态提升 vdom**： Vue3 优化了 vdom 的更新性能，静态提升包含静态节点和静态属性的提升，把一些静态的不会变的节点用变量缓存起来，提供下次 re-render 直接调用（不会再参与 diff 计算，vue2 是进行标记，但是还是要计算）；针对只是 class 等样式的响应式也会单独处理

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

  - 数据劫持优化：

    - 使用 proxy 监听整个对象，Proxy 并不能监听到内部深层次的对象变化，而 `Vue3` 的处理方式是在` getter` 中去递归响应式，这样的好处是真正访问到的内部对象才会变成响应式，而不是无脑递归

    - Proxy 代理的是整个对象而不是对象的属性（区分 Object.defineProperty 属性），对于整个对象进行操作；可以说解决了 Object.defineProperty 的痛点

      - 用 Proxy 对 对象动态添加的属性也会被拦截到；
      - 可以监听数组变化；

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

- 语法API（主要指 composition  Api）

  - 优化逻辑组织：相同功能可以先在一块地方
  - 优化逻辑复用：
    - 对比 vue2 的mixins 复用会发现：一般业务重、逻辑复杂且繁琐，多功能重合的情况下会抽离 mixins 来实现代码复用，但是在后期的维护和修改成本其实是增加的（命名冲突、数据/方法来源不明确，牵一发而动全身（好处也是坏处））
    - vue3 setup 里面更像是主逻辑抽离，把逻辑功能做成方法复用，使用的时候直接调用

## Tree-shaking

tree-sharking 的目的是在构建后消除程序中无用的代码，来减少包的体积；

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

## Typescript

### defineComponent

在 TS 下，需要用 Vue 暴露的方法 defineComponent，它单纯为了类型推导而存在的；

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

