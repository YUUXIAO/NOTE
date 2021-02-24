## Vue3.0

1. Vue3.0 都有哪些重要新特性？
2. 相比 2.0 的优势
3. Vue3.0 和  React 16.X 都有哪些区别和相似处？
4. Vue3.0 是如何实现代码逻辑复用的？

## React

1. React 16.X  的 Fiber 原理；
2. setState 原理，什么时候是同步的？
3. React Hooks 相对高阶组件和 Class 组件有什么优势/缺点？
4. React 16.X 的生命周期，以及为何要替换掉以前的？
5. React 跨平台的实现原理；
6. 说一说 redux，以及比 flux 先进的原因；

## SPA单页面

> SPA 仅在 web 页面初始化时加载相应的 HTML，Javascript 和 css，一旦页面加载完成，SPA 不会因为用户的操作而进行页面的重新加载或跳转，主要是利用路由机制实现 HTML 内容的变换，避免页面的重新加载；

### 优点

1. 用户体验好、快，内容的改变不不需要重新加载整个页面，避免了不必要的跳转和重复渲染；
2. SPA 相对服务器压力小；
3. 前后端职责分离，架构清晰，前端进行交互逻辑，后端负责数据处理；

### 缺点

1. 初次加载耗时多：为实现单页 Web 应用功能及显示效果，需要在加载页面的时候将 JavaScript、CSS 统一加载，部分页面按需加载；
2. 前进后退路由管理：单页应用在一个页面中显示所有的内容，所以不能使用浏览器的前进后退功能，所有页面切换需要自己建立堆栈管理；
3. SEO 难度较大：由于所有的内容都在一个页面中动态替换显示，所以在 SEO 上其有着天然的弱势；

## MVVM模式

- Model 表示数据模型层；
- view 表示视图层；
- ViewModel 是 View 和 Model 层的桥梁，数据绑定到 viewModel 层并自动渲染到页面中，视图变化通知 viewModel 层更新数据；

## 为什么Vue采用异步渲染

Vue 是组件级更新，如果不采用异步更新，那么每次更新数据都会对当前组件进行重新渲染，所以为了性能，Vue 会在本轮数据更新后，再异步更新视图，核心的思想是 nextTick；

dep.notify（）通知 watcher 进行更新，subs[i].update 依次调用 watcher 的 update，queueWatcher 将 watcher 去重放入队列，nextTick 在下一tick中刷新 watcher 队列；

## v-if & v-show

### v-if

> v-if 指令在编译的阶段就会编译成一个三元运算符条件渲染；

对于 v-if 渲染的节点，由于新旧节点 vnode 不一致，在核心 diff 算法对比过程中，会移除旧的 vnode 节点，创建新的 vnode 节点，又会经历组件自身初始化、渲染 vnode 和 patch 的过程；

使用 v-if 每次更新组件都会创建新的子组件，当更新的组件多了会造成性能压力；

```javascript
<template functional>
  <div class="cell">
    <div v-if="props.value" class="on">
      <Heavy :n="10000"/>
    </div>
    <section v-else class="off">
      <Heavy :n="10000"/>
    </section>
  </div>
</template>

// 生成的渲染函数
function render() {
  with(this) {
    return _c('div', {
      staticClass: "cell"
    }, [(props.value) ? _c('div', {
      staticClass: "on"
    }, [_c('Heavy', {
      attrs: {
        "n": 10000
      }
    })], 1) : _c('section', {
      staticClass: "off"
    }, [_c('Heavy', {
      attrs: {
        "n": 10000
      }
    })], 1)])
  }
}
```

### v-show

> v-show 指令渲染的节点，只需要一直 patchVnode 即可；

在 patchVnode 过程中，内部会对执行 v-show 指令对应的钩子函数 update，然后根据 v-show 指令绑定的值来设置它作用的 DOM 元素的 style.display 的值控制显示隐；

在使用 v-show 的时候，所有分支内部的组件都会被渲染，所以对应的生命周期钩子函数都会执行；

```javascript
<template functional>
  <div class="cell">
    <div v-show="props.value" class="on">
      <Heavy :n="10000"/>
    </div>
    <section v-show="!props.value" class="off">
      <Heavy :n="10000"/>
    </section>
  </div>
</template>

// 生成的渲染函数
function render() {
  with(this) {
    return _c('div', {
      staticClass: "cell"
    }, [_c('div', {
      directives: [{
        name: "show",
        rawName: "v-show",
        value: (props.value),
        expression: "props.value"
      }],
      staticClass: "on"
    }, [_c('Heavy', {
      attrs: {
        "n": 10000
      }
    })], 1), _c('section', {
      directives: [{
        name: "show",
        rawName: "v-show",
        value: (!props.value),
        expression: "!props.value"
      }],
      staticClass: "off"
    }, [_c('Heavy', {
      attrs: {
        "n": 10000
      }
    })], 1)])
  }
}
```

### 比较

1. 在组件的更新阶段，相比 v-if 不断删除和创建函数新的 DOM，v-show 仅仅是在更新现有 DOM 的显隐值，所以 v-show 的开销小得多；
2. 在组件的初始化阶段，v-if 只会渲染一个分支，v-show 会把两个分支渲染，所以 v-if 的性能高于 v-show；

## Vue中key的作用

> key 是 Vue 中 vnode 的唯一标记，通过这个 key，diff 操作可以更准确、迅速；

1. 更准确：因为带 key 就不是就地复用了，在 sameNode 函数 a.key === b.key 对比中可以避免就地复用的情况；
2. 更快速：利用 key 的唯一性生成 map 对象来获取对应节点，比遍历更快；

```javascript
function createKeyToOldIdx (children, beginIdx, endIdx) {
  let i, key
  const map = {}
  for (i = beginIdx; i <= endIdx; ++i) {
    key = children[i].key
    if (isDef(key)) map[key] = i
  }
  return map
}
```

## Vue项目优化

### 代码层面 

1. v-if 和 v-show 区分使用场景；

2. computed 和 watch 区分使用场景；

3. v-for 遍历必须为 item 添加 key，且避免同时使用 v-if；

4. 长列表性能优化；

5. 事件的销毁；

6. 图片资源懒加载；

7. 路由懒加载、异步组件；

8. 第三方插件的按需引入；

9. 优化无限列表性能；

10. 服务端渲染 SSR or 预渲染；

11. keep-alive 缓存组件 DOM；

12. 使用函数式组件，减少渲染开销；

13. 第三方模块按需导入（babel-plugin-component ）；

14. 子组件拆分耗时任务（或者计算属性缓存特性），避免组件重新渲染时重新执行耗时任务；

    ```javascript
    export default {
      props: ['number']
      components: {
        // 每次 number数据修改导致了父组件的重新渲染，但是 ChildComp 却不会重新渲染，因为它的内部也没有任何响应式数据的变化
        ChildComp: {
          methods: {
            heavy () {
              const n = 100000
              let result = 0
              for (let i = 0; i < n; i++) {
                result += Math.sqrt(Math.cos(Math.sin(42)))
              }
              return result
            },
          },
          render (h) {
            return h('div', this.heavy())
          }
        }
      },
    }
    ```

15. 使用局部变量缓存 this.xxx，避免直接访问响应式对象触发 getter 进行依赖收集相关代码；

    ```javascript
    export default {
      props: ['start'],
      computed: {
        base () {
          return 42
        },
        result ({ base, start }) {
          let result = start
          for (let i = 0; i < 1000; i++) {
            result += Math.sqrt(Math.cos(Math.sin(base))) + base * base + base + base * 2 + base * 3
          }
          return result
        },
      },
    }
    ```

16. 对象设置非响应式数据

    ```javascript
    function optimizeItem (item) {
      const itemData = {
        id: uid++,
        vote: 0
      }
      Object.defineProperty(itemData, 'data', {
        // Mark as non-reactive
        configurable: false,
        value: item
      })
      return itemData
    }

    // 有些数据不使用在模板时，可以直接在组件中定义；
    created() {
      this.scroll = null
    },
    mounted() {
      this.scroll = new BScroll(this.$el)
    }
    ```

    ​

### webpack层面

1. Webpack 对图片进行压缩；
2. 减少 ES6 转为 ES5 的冗余代码；
3. 提取公共代码；
4. 模板预编译；
5. 提取组件的 CSS；
6. 优化 SourceMap；
7. 构建结果输出分析；
8. Vue 项目的编译优化；

### 基础Web 技术

1. 开启 gzip 压缩；
2. 浏览器缓存；
3. CDN 的使用
4. 使用 Chrome Performance 查找性能瓶颈；

## Vue2源码的性能优化

1. cache 函数，利用闭包实现缓存；
2. 二次依赖收集时，cleanupDeps 方法清除上次存在但本次渲染不存在的依赖；
3. traverse，处理深度监听数据，解除循环引用；
4. 编译优化阶段，optimize 方法标记静态节点；
5. keep-alive 组件利用 LRU 缓存淘汰算法；
6. 异步组件，分两次渲染；