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


## 响应式 API

https://cn.vuejs.org/api/reactivity-advanced.html 【官方组件】

~~https://zhuanlan.zhihu.com/p/528715632?utm_id=0【vue3的watch和watchEffect】~~



### watch 和 watchEffect()

#### watch()

watch 方法一般用于侦听一个或多个响应式数据源，并在数据源变化时调用所给的回调函数。

`watch()` 默认是懒侦听的，即仅在侦听源发生变化时才执行回调函数。

**watch的监听类型**

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
  onCleanup: (cleanupFn: () => void) => void
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
  - 一个函数，返回一个值
  - 一个 ref
  - 一个响应式对象
  - ...或是由以上类型的值组成的数组
- 第二个参数是在发生变化时要调用的回调函数。这个回调函数接受三个参数：新值、旧值，以及一个用于注册副作用清理的回调函数。该回调函数会在副作用下一次重新执行前调用，可以用来清除无效的副作用（同watchEffect 的使用）；
  - 当侦听多个来源时，回调函数接受两个数组，分别对应来源数组中的新值和旧值。
- 第三个可选的参数是一个对象，支持以下这些选项：
  - **immediate**：在侦听器创建时立即触发回调。第一次调用时旧值是 `undefined`（watchEffect 自带，无须配置）
  - **deep**：如果源是对象，强制深度遍历，以便在深层级变更时触发回调
  - **flush**：调整回调函数的刷新时机（同 watchEffect ）
  - **onTrack / onTrigger**：调试侦听器的依赖（同 watchEffect ）

#### watchEffect()

立即运行一个函数，同时响应式地追踪其依赖，并在依赖更改时重新执行

比watch不同的是，这个方法是默认初始化会之执行一次副作用函数，不需要添加属性 immediate: true

使用场景：

- 不需要持续监听，比较适合初始化根据某部分数据执行方法
- 比 watch 方便，immediate 方向的使用
- 不需要关心监听的数据具体变化的值，只关注结果

看下watchEffect 的ts 类型定义：

```typescript
function watchEffect(
  effect: (onCleanup: OnCleanup) => void,
  options?: WatchEffectOptions
): StopHandle

type OnCleanup = (cleanupFn: () => void) => void

interface WatchEffectOptions {
  flush?: 'pre' | 'post' | 'sync' // 默认：'pre'
  onTrack?: (event: DebuggerEvent) => void
  onTrigger?: (event: DebuggerEvent) => void
}

type StopHandle = () => void
```

**参数信息：**

- effect：要运行的副作用函数，用来注册清理回调。清理回调会在该副作用下一次执行前被调用，可以用来清理无效的副作用，例如等待中的异步请求

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

- options：一个可选的选项，一般用来调整副作用的刷新时机或调试副作用的依赖

  - 默认情况下，watch的回调函数会在组件重新渲染之前执行，flush 属性可以设置回调函数的触发时机（post：在组件渲染之后再执行；sync：在响应式依赖发生改变时立即触发侦听器），使用时可能会导致页面性能和数据不一致的情况 

  ```

  ```

  ​

- 返回一个用来停止该副作用的函数（停止监听）

  ```javascript
  const stop = watchEffect((onCleanup) => {
    if (props.id) {
      getData() // 获取接口信息
      stop() // 达到条件，停止监听
    }
  })
  ```

#### 手动停止监听器

通常来说，在组件被销毁或者卸载后，监听器也会跟着被销毁。但是总是有一些特殊情况，即使组件卸载了，但是监听器依然存在，这个时候其实式需要我们手动关闭它的，否则容易造成内存泄漏

```javascript
<script setup>
import { watchEffect } from 'vue'
// 它会自动停止
watchEffect(() => {})
// ...这个则不会！
setTimeout(() => {
  watchEffect(() => {})
}, 100)
</script>
```

上段代码中我们采用异步的方式创建了一个监听器，这个时候监听器没有与当前组件绑定，所以即使组件销毁了，监听器依然存在。

我们需要用一个变量接收监听器函数的返回值，其实就是返回的一个函数，然后我们调用该函数，即可关闭当前监听器。

```javascript
const unwatch = watchEffect(() => {})
// ...当该侦听器不再需要时
unwatch()
```

#### 回调中的 DOM

如果我们在监听器的回调函数中或取 DOM，这个时候的 DOM 是更新前的，可以通过给监听器多传递一个参数选项：flush: 'post' 来调整获取对应状态的 DOM

#### 使用场景区分

相比watchEffect而言，watch 可以 

- **执行时机：**watch 是懒执行副作用（可配置 immediate：true），watchEffect 是默认初始化先执行一次，无需配置，如果关注这个使用 watchEffect 更方便
- **数据源的变化：**watch 更明确、关注由哪个/哪些数据触发的侦听器执行（过程，可以访问所侦听状态的前一个值和当前值），watchEffect 只关注数据改变的状态-触发侦听器方法执行（结果）

## shallowRef()

场景：长列表数据，常常用于对大型数据结构的性能优化或是与外部的状态管理系统集成

## dev 调试 Api

### onRenderTracked 状态跟踪

当组件渲染过程中追踪到响应式依赖时调用

onRenderTracked 会跟踪页面上所有响应式变量和方法的状态（可以理解为用 return 返回的值都会跟踪），只要页面有 update 的情况，就会跟踪，生成一个 event 对象



### onRenderTriggered 状态触发

当响应式依赖的变更触发了组件渲染时调用

onRenderTriggered 不会跟踪每一个值，而是给你变化值的信息，并且新值和旧值都会给你明确的展示出来。

可以理解为就是跟踪渲染页面的变量，js 变量不会跟踪，更精准，会返回：

- 变化变量的 key 
- newValue 更新后变量的值
- oldValue 更新前变量的值
- target 目前页面中的响应变量和函数

感觉和watch 很像，而且是 immidate= true