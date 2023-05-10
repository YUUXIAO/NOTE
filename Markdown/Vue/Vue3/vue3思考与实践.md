区别于2.0的 computed、watch，3.0针对响应式进行了更细化的处理和使用场景区分（我个人觉得是性能优化入手的重头）



https://www.cnblogs.com/zdsdididi/p/16396088.html 【vue3特性】

- watch和watchEffect的区别？

  - https://zhuanlan.zhihu.com/p/528715632?utm_id=0
  - https://blog.csdn.net/weixin_52148548/article/details/125073998
  - computed和watch所依赖的数据必须是响应式的。Vue3引入了watchEffect,watchEffect 相当于将 watch 的依赖源和回调函数合并，当任何你有用到的响应式依赖更新时，该回调函数便会重新执行。不同于 watch的是watchEffect的回调函数会被立即执行，即（{ immediate: true }）
    https://blog.csdn.net/weixin_52148548/article/details/125055677?spm=1001.2014.3001.5502 【reactive、ref、toRef、toRefs】

- **defineProps** 不能引用外部的 Ts？

- ### attrs和listeners

  - Vue3中使用attrs调用父组件方法时，方法前需要加上on；如parentFun->onParentFun

- 组件通信

  - 在组合式API中，如果想在子组件中用其它变量接收props的值时需要使用toRef将props中的属性转为响应式。

- build.polyfillDynamicImport ？？？ 已经废弃

- inheritAttrs 属性

- `Attribute`强制策略


## watch

这个 api  提供了**基于观察状态**的变化来执行副作用的能力；

第二个参数是回调函数，回调函数只有当数据源发生变动时才会被触发

有时当观察的数据源变化后可能需要对之前所执行的副作用进行清理，比如 接口请求要取消上一次的，watcher 的回调会接收到第三个参数是一个用来注册清理操作的函数，调用这个函数可以注册一个清理函数；

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


## v-memo 指令

提供了记忆一部分模板树的能力。 v-memo 指令使得这部分模板可以跳过虚拟 DOM 的 diff 比较，同时还完全跳过新 VNode 的创建。 虽然很少需要，但它提供了一种在某些情况下想要得到最大性能的方案，例如大型 v-for 列表

## 响应式 API

https://cn.vuejs.org/api/reactivity-advanced.html 【官方组件】

### ref 和 shallowRef 

#### shallowRef

shallowRef 的内部值将会原样存储和暴露，并不会被深层递归地转为响应式，只处理基本数据类型的响应式，不进行对象的响应式处理

可以理解为只响应式.value返回的值，打印的包裹的对象类型数据是 Object 不是 proxy，针对ref包裹的对象类型数据，结果打印是Proxy，所以是响应式的

可优化场景（用于对大型数据结构的性能优化或是与外部的状态管理系统集成）

- refs 组件引用，典型的 ref.value.validate（表单、表格引用）
- ​

```javascript
const state = shallowRef({ count: 1 })

// shallowRef 不会触发更改, ref 会触发页面更改
state.value.count = 2

// shallowRef 和  ref 都会触发更改
state.value = { count: 2 }
```

#### triggerRef

强制触发依赖于一个[浅层 ref](https://cn.vuejs.org/api/reactivity-advanced.html#shallowref) 的副作用，这通常在对浅引用的内部值进行深度变更后使用，一般搭配着 shallowRef 使用

```javascript
function triggerRef(ref: ShallowRef): void
```

```javascript
const shallow = shallowRef({
  greet: 'Hello, world'
})

// 触发该副作用第一次应该会打印 "Hello, world"
watchEffect(() => {
  console.log(shallow.value.greet)
})

// 这次变更不应触发副作用，因为这个 ref 是浅层的
shallow.value.greet = 'Hello, universe'

// 打印 "Hello, universe"
triggerRef(shallow)
```

#### customRef

创建一个自定义的 ref，显式声明对其依赖追踪和更新触发的控制方式

`customRef()` 预期接收一个工厂函数作为参数，这个工厂函数接受 `track` 和 `trigger` 两个函数作为参数，并返回一个带有 `get` 和 `set` 方法的对象。

一般来说，`track()` 应该在 `get()` 方法中调用，而 `trigger()` 应该在 `set()` 中调用。然而事实上，你对何时调用、是否应该调用他们有完全的控制权。

比如我们有一个值会频繁刷新调用，我们想手动控制它的刷新时机，创建一个防抖 ref

```javascript
import { customRef } from 'vue'

export function useDebouncedRef(value, delay = 200) {
  let timeout
  return customRef((track, trigger) => {
    return {
      get() {
        track() // 可以理解为添加响应式追踪(vue 一般起名为track 的都是这个作用)
        return value
      },
      set(newValue) {
        clearTimeout(timeout)
        timeout = setTimeout(() => {
          value = newValue
          trigger() // 可以理解为触发页面渲染更新函数
        }, delay)
      }
    }
  })
}

const text = useDebouncedRef('text')
```

#### shallowRef 和 shallowReactive

- shallowReactive：只处理**最外层** 属性的响应式（浅响应式）
  - 有一个对象数据（数据结构嵌套深），使用变化修改时只是外层属性的变化===>shallowReactive
- shallowRef：只处理基本数据类型的响应式，不进行对象的响应式处理
  - 一个对象数据，后续不会修改该对象中的属性，可能只是节点引用或生成新的对象来替换的===》shallowRef

### readonly 和 shallowReadonly

- readonly：让一个响应式数据变为只读的（深只读，所有嵌套结构）
- shallowReadonly：让一个响应式数据外层属性变为只读（浅只读，深层次嵌套可以修改）
- 应用场景：
  - 听起来很奇怪，竟然定义一个数据为响应式但是为什么又要包装为只读的
  - 比如一个场景：数据 A 是从别的组件传入的，但是这个数据只希望你在这个组件去引用别修改的情况，那就可以用 readonly 去处理

### toRaw 与 markRaw

- toRaw：将一个由 reactive （针对ref 缔造的响应式数据无效）生成的响应式对象转为普通对象，用于读取响应式对象对应的普通对象，对这个转换后的对象所有操作，不会引起页面更新
- markRaw：标记一个对象，使其永远不会再成为响应式对象
  - 有些值不应该设置为响应式的，比如复杂的第三方类库或 Vue 组件对象等
  - 当渲染具有不可变数据源的大列表时，跳过响应式转换可以提高性能

### watch 和 watchEffect()

#### watch()

watch 方法一般用于侦听一个或多个响应式数据源，并在数据源变化时调用所给的回调函数。

它本质上充当组件响应式数据的事件监听器，特别是当与异步 API 调用配合使用时

- 当 ID 改变时，从数据库中获取一个对象
- 当 `prop` 更改时重新运行动画
- 监听路由变化
- `input` 输入框值的特殊处理等

`watch` 在以下情况下很有用：

- 您需要控制哪些依赖项会触发该方法
- 您需要访问之前的值

`watch()` 默认是懒侦听的，即仅在侦听源发生变化时才执行回调函数。

- watch 侦听 reactive 定义的响应式数据（因为reactive 只能定义数组或对象类型的响应式）时，oldValue 无法正确获取**，会强制开启深度监视，此时 deep 配置无效**
- 侦听 reactive定义的响应式数据中的某个属性时，且该属性是一个对象，那么此时deep配置生效。 

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
  - 一个返回任意值的函数（getter 函数）:（）=> xxx；
  - 一个包装对象（响应式对象）；
  - ...一个包含上述两种数据源的数组；
- 第二个参数是在发生变化时要调用的回调函数。
  - 这个回调函数接受三个参数：新值、旧值，以及一个用于注册副作用清理的回调函数。
  - 该回调函数会在副作用下一次重新执行前调用，可以用来清除无效的副作用（同watchEffect）；
  - 当侦听多个来源时，回调函数接受两个数组，分别对应来源数组中的新值和旧值。
- 第三个可选的参数是一个对象，支持以下这些选项：
  - **immediate**：在侦听器创建时立即触发回调。第一次调用时旧值是 `undefined`（watchEffect 自带，无须配置）
  - **deep**：如果源是对象，强制深度遍历，以便在深层级变更时触发回调
  - **flush**：调整回调函数的刷新时机（同 watchEffect ）
  - **onTrack / onTrigger**：调试侦听器的依赖（同 watchEffect ）

#### watchEffect()

它立即执行传入的一个函数，同时响应式**追踪其依赖**，并在其依赖变更时重新运行该函数

比watch不同的是，这个方法是默认初始化会之执行一次副作用函数，不需要添加属性 immediate: true

##### [清除副作用](https://www.javascriptc.com/vue3js/guide/reactivity-computed-watchers.html#%E6%B8%85%E9%99%A4%E5%89%AF%E4%BD%9C%E7%94%A8)

**无效副作用**：每当响应性依赖项发生变化时，就进行某种异步 API 调用。但是如果依赖关系在第一个 API 调用完成之前再次发生变化，会发生什么呢？**这就是无效副作用出现的原因**

清楚副作用的时机

- 副作用即将重新执行时
- 侦听器被停止 (如果在 `setup()` 或生命周期钩子函数中使用了 `watchEffect`，则在组件卸载时)

`watchEffect` 方法还有一个 `onInvalidate` 方法，每当该方法要再次运行或监视程序停止时，该方法就会运行。

```javascript
export default {
  setup() {
    watchEffect(onInvalidate => {
      // 异步 API 调用
      const apiCall = someAsyncMethod(props.songID)
      onInvalidate(() => {
        // 取消 API 调用
        apiCall.cancel()
      })
    })
  }
}
```

**使用场景**：

- 不需要持续监听，【比较适合初始化根据某部分数据执行方法】
- 比 watch 方便，写法、immediate 方向的使用
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

##### watchEffect 和 watch

既然已经有了 `watch` 方法，为什么这个新的 `watchEffect` 方法还会存在呢？

- watchEffect 将在方法的**任何**依赖项发生更改时运行，`watch` 跟踪一个或多个**特定**的响应性属性，并且仅在该属性发生更改时运行。
- 默认情况下，`watch` 是惰性的，因此仅当依赖项更改时才会触发。`watchEffect` 在创建组件后立即运行，然后跟踪依赖关系。

##### watchEffect 和 computed

warchEffect 和 watch 的功能和写法都挺好理解的，反而觉得它和 computed 有点像，对比一下区别

- computed 主要注重的是计算出来的值（回调函数返回值），且必须要写返回值
- watchEffect 始终注重的是过程（函数体），不用写返回值
- computed 如果没有值被使用时是不会调用的，但是watchEffect 一定会至少调用一次（初始化那次，目的是自动获取依赖，这里又区分了watch 和 computed 本质区别）

##### watchEffect 和 useEffect（React）

其实我感觉它和React中的useEffect很像（写法的问题），只是不需要传入第二个参数依赖项数组

- 初始化立即执行一次逻辑处理函数
- 第一个参数返回一个回调函数，可以在该函数中将组件被摧毁之前和再一次触发更新时，将之前的副作用清除掉（不断循环的订阅（计时器，或者递归循环），和watchEffect写法不一样
- 依赖的改变触发函数的执行

#### 手动停止监听器（停止观察）

如果我们想观察一个依赖项，直到它达到某个值，然后停止它，这可能很有用。如果我们在它达到目标值后继续观察，我们只是在**浪费资源**。

在异步场景下，清理失效的回调，保证当前副作用有效，不会被覆盖；

通常来说，在组件被销毁或者卸载后，监听器也会跟着被销毁。但是总是有一些特殊情况（不断循环的订阅（计时器，或者递归循环），即使组件卸载了，但是监听器依然存在，这个时候其实式需要我们手动关闭它的，否则容易造成内存泄漏

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

watchEffect(onInvalidate => {
  // 异步 API 调用
  const apiCall = someAsyncMethod(props.songID)

  onInvalidate(() => {
    // 取消 API 调用
    apiCall.cancel()
  })
})
```

#### 回调中的 DOM

如果我们在监听器的回调函数中或取 DOM，这个时候的 DOM 是更新前的，可以通过给监听器多传递一个参数选项：flush: 'post' 来调整获取对应状态的 DOM

#### 使用场景区分

相比watchEffect而言，watch 可以 

- **执行时机：**watch 是懒执行副作用（可配置 immediate：true），watchEffect 是默认初始化先执行一次，无需配置，如果关注这个使用 watchEffect 更方便
- **数据源的变化：**watch 更明确、关注由哪个/哪些数据触发的侦听器执行（过程，可以访问所侦听状态的前一个值和当前值），watchEffect 只关注数据改变的状态-触发侦听器方法执行（结果）

## shallowRef()

场景：长列表数据，常常用于对大型数据结构的性能优化或是与外部的状态管理系统集成

如果不是页面上需要进行视图更新的，我们可以不用reactive，ref更进行声明，可以使用[`shallowRef()`](https://links.jianshu.com/go?to=https%3A%2F%2Fcn.vuejs.org%2Fapi%2Freactivity-advanced.html%23shallowref) 和 [`shallowReactive()`](https://links.jianshu.com/go?to=https%3A%2F%2Fcn.vuejs.org%2Fapi%2Freactivity-advanced.html%23shallowreactive) 浅层式响应进行声明（浅层式顶部是响应的，底部都不是响应数据）

```javascript
const shallowArray = shallowRef([
  /* 巨大的列表，里面包含深层的对象 */
])

// 这不会触发更新...
shallowArray.value.push(newObject)
// 这才会触发更新
shallowArray.value = [...shallowArray.value, newObject]

// 这不会触发更新...
shallowArray.value[0].foo = 1
// 这才会触发更新
shallowArray.value = [
  {
    ...shallowArray.value[0],
    foo: 1
  },
  ...shallowArray.value.slice(1)
]
```

###  Effect 作用域 API（v2.3）

## effectScope

创建一个 effect 作用域，可以捕获其中所创建的响应式副作用（computed 和  watchers），这样捕获到的副作用就可以 一起处理（直接销毁作用域）

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
function effectScope(detached?: boolean): EffectScope

interface EffectScope {
  run<T>(fn: () => T): T | undefined // 如果作用域不活跃就为 undefined
  stop(): void
}

// 使用示例
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

## dev 调试 Api

### 【组件调试钩子】onRenderTracked 状态跟踪

当组件渲染过程中**追踪到**响应式依赖时调用
当组件渲染过程中追踪到响应式依赖时调用，用来调试查看哪些依赖正在被使用（这是一个**生命周期钩子**）

onRenderTracked 会跟踪页面上所有响应式变量和方法的状态（可以理解为用 return 返回的值都会跟踪），只要页面有 update 的情况，就会跟踪，生成一个 event 对象



### 【组件调试钩子】onRenderTriggered 状态触发

当响应式依赖的**变更**触发了组件渲染时调用
当响应式依赖的变更触发了组件渲染时调用，用来确定哪个依赖正在触发更新

onRenderTriggered 不会跟踪每一个值，而是给你变化值的信息，并且新值和旧值都会给你明确的展示出来。

可以理解为就是跟踪渲染页面的变量，js 变量不会跟踪，更精准，会返回：

- 变化变量的 key 
- newValue 更新后变量的值
- oldValue 更新前变量的值
- target 目前页面中的响应变量和函数

感觉和watch 很像，而且是 immidate= true 的时候