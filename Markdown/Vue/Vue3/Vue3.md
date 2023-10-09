问题合集

- setup 同步还是异步，使用异步的 customRef


## 组件相关内容

### $attrs（Object）

$attr 对象不包含的有：

- vue 内置的特殊的 attribute： key、ref
- vue 所有的内置指令（v-on、v-bind 除外）
- 所有自定义的 directive 指令
- 当前组件的 props 内声明的所有 prop 名称 
- 当前组件的 emits 内声明的所有自定义事件名、



### inheritAttrs（Boolean）

这个属性感觉继承的概念比较像，主要与子组件单、多节点影响，

会被继承的有：



当template 有根元素的时候，绑定到组件上的属性和事件会自动继承到根元素上

当组件返回单个根节点时，非







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

## 项目结构

![vue1](F:\Yabby\NOTE\images\vue\vue1.png)

V3 源码主要由下面几个模块组成：

### Compiler 模块

主要用来解析 Vue 模板，将其转换为渲染函数

首先将 vue 模版编译成渲染函数，其中调用了 parse 函数将模版解析成 AST 语法树，再调用了 generate 函数将 AST 转换成 js 代码

### Renderer 模块

主要将组件渲染为真实的 Dom 元素，这里会区分**浏览器渲染器**（Dom API）还是**服务端渲染器** （Node.js 的流式渲染），两种渲染器的实现方式不同，但是主要渲染流程差不多：

- **创建根组建实例：** 创建根组件实例，并将其挂载到指定的 DOM 元素上
- **渲染组件：** 从根组件开始，递归渲染组件树，会先执行组件的 setup 函数，然后 render 函数，再生成虚拟Dom 树
- **更新虚拟 DOM：** 当组件状态发生变化时，会重新执行 render 函数，生成新的虚拟 DOM
- **比较新旧虚拟 DOM：**进行比较，找出需要更新的 DOM 元素
- **更新 DOM**

在 V3 中，Renderer 模块的实现主要使用了以下功能：

1. Diff 算法

2. [**PatchFlag：**](https://github.com/vuejs/core/blob/main/packages/shared/src/patchFlags.ts)在虚拟节点上添加 PatchFlag 标记，用于标记节点需要更新的类型，减少一定的比较时间

   ![vue_2](F:\Yabby\NOTE\images\vue\vue_2.png)

3. **静态提升：**将静态节点提升到父组件的渲染函数中，避免重复渲染静态节点的性能问题

4. **缓存事件处理函数：** 将事件处理函数缓存起来，避免会重复创建函数

5. **内置组件的优化：** 对内置组件（slot、keep-alive等）进行优化



### Reactivity 模块

主要实现响应式数据绑定，将，并在对象属性发生变化时自动更新相关视图

- reactive 函数用于将普通的 js 对象转化成响应式对象，其中使用了 createReactiveObject 函数来创建响应式对象
- createReactiveObject 函数使用了 Proxy 对象来监听对象属性的读写操作，同时根据 isReadonly 参数来选择不同的处理器（mutableHandlers 或 readonlyHandlers）
- mutableHandlers 使用了 get 和 set 函数来监听对象属性的读写，并在其中调用了 track 和 trigger 函数来进行依赖收集和派发更新
- readonlyHandlers 使用了 readonlyGet 和 readonlySet 函数来监听对象属性的读写，其中 readonlySet 函数返回了 false，避免了对只读对象进行修改

### Runtime-core 模块

主要实现了 Vue 组件的实例化、生命周期、事件等核心功能，主要的作用的将组件模版编译成渲染函数，并在组件状态发生变化时自动更新相关视图

- createComponentInstance 函数创建组件实例，其中包含了组件的状态、渲染函数等信息
- setupComponent 函数初始化组件实例，其中将组件模版编译成渲染函数，并将其赋值给 instance.render属性
- renderCompoentRoot 函数用来渲染组件的根节点，其中调用了 instance.render 来生成组件的虚拟 DOM，并将其赋值给 instance.subTree 属性
- updateComponent 用来更新组件的视图，其中调用了 renderCompoentRoot 来生成新的虚拟Dom，并将其与旧的虚拟DOM进行对比，最终更新需要更新的节点  

### Shared 模块

主要包含一些公共的工具函数，链接：https://github.com/vuejs/core/blob/main/packages/shared/src/index.ts



## 优化方案

V3 和 V2 在设计上的主要区别：

- **响应式系统：**使用 Proxy 替换了 defineProperty
- **组件实现：**采用编译时优化组件
- **编译器：** 将编译器独立出来，减少项目的体积，支持渲染函数以及Typescript类型声明
- **项目构建：** 默认使用了浏览器原生支持的 ES 模块，在打包时使用 Tree Sharking 去掉无用代码，减少项目体积



- 源码

  - **源码管理：**monorepo 的方式维护，可以根据功能模块的不同拆分到 packages 目录下的子目录中，模块拆分更细化，职责更明确，一些package 是可以单独引入使用的（reactive库）
  - **Typesctipt：**基于 typeScript 编写的，提供了更好的类型检查，能支持复杂的类型推导，其实感觉V3加入这个对于开发者也更容易阅读源码





### 打包体积

vue2 官方说明运行时打包 23k，这个是没有安装依赖的时候，随着依赖包和框架特性的增多，有时候不必要的、未使用的代码文件都被打包进去，后期项目大了，打包文件会很大

在 vue3 中，通过将大多数全局 API 和内部帮助程序移动到 javascript 的 module.exports 属性上实现这一点，module bundler 能够静态地分析模块依赖关系，并删除与未使用的 module.exports 属性相关的代码，模板编译器还生成了对 Tree Shaking 摇树优化友好的代码，只有在模板中实际使用某个特性时，该代码才导入该特性的帮助程序，尽管增加了许多新特性，但 Vue3 被压缩后的基线大小约为 10k

### diff算法

vue2 通过深度递归遍历两个虚拟 Dom 树，并比较每个节点上的每个属性，来确定实际 DOM 的哪些部分需要更新，这种方法比较暴力但快速，但是 DOM 的更新仍然设计许多不必要的 CPU 工作

vue3 通过编译器在分析模版并生成带有可优化提示的代码，这样在运行时尽可能获取提示并采用“快速路径”，主要有以下三个优化：

- 在 DOM 树级别，在没有动态改变节点结构的模板指令（比如  v-if 或者 v-for ）的情况下，节点结构保持完全静态，如果我们把一个模板分成由这些结构指令分隔的嵌套块，则每个块中的节点结构将再次完全静态，当我们更新块中的节点时，不需要再遍历递归 DOM 树，该块内的动态绑定可以在一个平面数组中跟踪，这种优化通过将需要执行的树遍历量减少了一个数量级
- 编译器积极地检测模版中的静态节点、子树或者数据对象，并在生成代码中将它们提升到渲染函数之外，这样可以避免在每次渲染时重新创建这些对象，从而大大提高内存使用率并减少垃圾回收的频率
- 在元素级别，编译器根据需要执行的更新类型 ，为每个具有动态绑定的元素生成一个优化标志，比如具有动态类绑定和许多静态属性的元素打上标志，提示只需要进行类检查，运行时将获取这些提示并采用专用的快速路径

- 性能

  - 体积优化：

  - 编译优化：

    - [**静态标记：**](https://github.com/vuejs/core/blob/main/packages/shared/src/patchFlags.ts) vue2 都是从根节点开始一层一层进行全量对比（不会区分静动态），vue3 新增了静态标记，在与上次虚拟dom 对比的时候，只对比带有 patchFlags 的节点，这样就可以跳过一些静态标记

    - **静态提升 vdom**： Vue3 优化了 vdom 的更新性能，静态提升包含静态节点和静态属性的提升，把一些不更新的节点用变量缓存起来，提供下次 re-render 直接调用（不会再参与 diff 计算，vue2 是进行标记，但是还是要计算）；针对只是 class 等样式的响应式也会单独处理

      这里推荐一个网站可以直接看到 tempalte 到编译后的内容： https://template-explorer.vuejs.org/

      ```javascript
      // template
      <div class="div">
        <div class="line">静态文字1</div>
        <div class="text">{{ text }}</div>
      </div>

      // 没有静态提升
      function render(_ctx, _cache, $props, $setup, $data, $options) {
        return (_openBlock(), _createBlock("div", { class: "div" }, [
          _createElementVNode("div", { class: "line" }, "静态文字1"),
          _createElementVNode("div", { class: "text" }, _toDisplayString(_ctx.text), 1 /* TEXT */)
        ]))
      }

      // 有静态提升 
      const _hoisted_1 = { class: "div" }
      const _hoisted_2 = /*#__PURE__*/_createVNode("div", null, "content", -1 /* HOISTED */)

      function render(_ctx, _cache, $props, $setup, $data, $options) {
        return (_openBlock(), _createBlock("div", _hoisted_1, [
          _hoisted_2,
          _createElementVNode("div", { class: "text" }, _toDisplayString(_ctx.text), 1 /* TEXT */)
        ]))
      }
      ```

    - **cacheHandles 事件缓存：** V2 里绑定事件都要重新生成新的 function 去更新；V3 会自动生成一个内联函数，同时生成一个静态节点，onClick 时会读取缓存，如果已有缓存，就更新

      ```vue
      <template>
        <div @click="triggerClick()">按钮1</div>
      </template>

      // 编译后
      import { createElementVNode as _createElementVNode, openBlock as _openBlock, createElementBlock as _createElementBlock } from "vue"

      export function render(_ctx, _cache, $props, $setup, $data, $options) {
        return (_openBlock(), _createElementBlock("template", null, [
          _createElementVNode("div", {
            onClick: _cache[0] || (_cache[0] = $event => (_ctx.triggerClick()))
          }, "按钮1")
        ]))
      }
      ```

### 数据劫持优化（proxy 代替 defindProperty）

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

### 语法API（主要指 composition  Api）

- 优化逻辑组织：相同功能可以先在一块地方
- 优化逻辑复用：
  - 对比 vue2 的mixins 复用会发现：一般业务重、逻辑复杂且繁琐，多功能重合的情况下会抽离 mixins 来实现代码复用，但是在后期的维护和修改成本其实是增加的（命名冲突、数据/方法来源不明确，牵一发而动全身（好处也是坏处））
  - vue3 setup 里面更像是主逻辑抽离，把逻辑功能做成方法复用，使用的时候直接调用

## Tree-shaking

tree-sharking 的目的是在打包时去掉多余的代码，只保留项目中用到的代码，V3 中采用了基于静态分析和 ES 模块的机制来进行摇树优化：

- 首先：使用  ESLint 和 Typescript 在编译期间进行静态分析，检测出没有被使用的代码，并删除这些代码
- 其次：使用 ES 模块（支持 Tree Shaking）实现按需加载的效果，仅加载需要的特定部分



1. 基于函数的 API 每一个函数都可以作为 named ES export 被单独引入，使得它们对 tree-shaking 非常友好（ES Module 支持 Tree Shaking）；
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

