1. vuex 的使用基本真的分为以下几个部分：

- state：数据定义层，一个返回所有数据的函数
- gettters：定义获取 state 状态数据的计算属性，可以认为定义在这里的值都是 state 的派生值
- mutations：定义修改 state 数据的方法
- actions：定义异步操作的方法

# 解析Store类里的流程

```javascript
import { reactive, inject } from 'vue'
// 定义了一个全局的 key，用于在 Vue 组件中通过 inject API 访问 store 对象
const STORE_KEY = '__store__'
// 用于获取当前组件的 store 对象
function useStore() {
  return inject(STORE_KEY)
}
// Store 类，用于管理应用程序状态
// 构造函数，接收一个包含 state、mutations、actions 和 getters 函数的对象 options，然后将它们保存到实例属性中
class Store {
  constructor(options) {
    this.$options = options
    this._state = reactive({
      data: options.state() // 使用 reactive API 将 state 数据转换为响应式对象，并保存到实例属性 _state 中
    })
    this._mutations = options.mutations // 将 mutations 和 actions 函数保存到实例属性中
    this._actions = options.actions
    this.getters = {} // 初始化 getters 属性为空对象
    // 遍历所有的 getters 函数，将其封装成 computed 属性并保存到实例属性 getters 中
    Object.keys(options.getters).forEach((name) => {
      const fn = options.getters(name)
      this.getters[name] = computed(() => fn(this.state))
    })
  }
  // 用于获取当前状态数据
  get state() {
    return this._state.data
  }
  // 获取mutation内定义的函数并执行
  commit = (type, payload) => {
    const entry = this._mutations[type]
    entry && entry(this.state, payload)
  }
  // 获取actions内定义的函数并返回函数执行结果
  dispatch = (type, payload) => {
    const entry = this._actions[type]
    return entry && entry(this, payload)
  }
  // 将当前 store 实例注册到 Vue.js 应用程序中
  install(app) {
    app.provide(STORE_KEY, this)
  }
}
// 创建一个新的 Store 实例并返回
function createStore(options) {
  return new Store(options)
}
// 导出 createStore 和 useStore 函数，用于在 Vue.js 应用程序中管理状态
export { createStore, useStore }
```

## commit 和 dispatch 的具体实现

```javascript
Store.prototype.commit = function commit(_type, _payload, _options) {
  const this$1$1 = this
  // check object-style commit
  const ref = unifyObjectStyle(_type, _payload, _options)
  const type = ref.type
  const payload = ref.payload
  const options = ref.options
  const mutation = { type: type, payload: payload }
  const entry = this._mutations[type]
  if (!entry) {
    if (process.env.NODE_ENV !== 'production') {
      console.error('[vuex] unknown mutation type: ' + type)
    }
    return
  }
  this._withCommit(function () {
    entry.forEach(function commitIterator(handler) {
      handler(payload)
    })
  })
  this._subscribers
    .slice() // shallow copy to prevent iterator invalidation if subscriber synchronously calls unsubscribe
    .forEach(function (sub) {
      return sub(mutation, this$1$1.state)
    })
  if (process.env.NODE_ENV !== 'production' && options && options.silent) {
    console.warn(
      '[vuex] mutation type: ' +
        type +
        '. Silent option has been removed. ' +
        'Use the filter functionality in the vue-devtools'
    )
  }
}

```



```javascript
Store.prototype.dispatch = function dispatch(_type, _payload) {
  const this$1$1 = this
  // check object-style dispatch
  const ref = unifyObjectStyle(_type, _payload)
  const type = ref.type
  const payload = ref.payload
  const action = { type: type, payload: payload }
  const entry = this._actions[type]
  if (!entry) {
    if (process.env.NODE_ENV !== 'production') {
      console.error('[vuex] unknown action type: ' + type)
    }
    return
  }
  try {
    this._actionSubscribers
      .slice() // shallow copy to prevent iterator invalidation if subscriber synchronously calls unsubscribe
      .filter(function (sub) {
        return sub.before
      })
      .forEach(function (sub) {
        return sub.before(action, this$1$1.state)
      })
  } catch (e) {
    if (process.env.NODE_ENV !== 'production') {
      console.warn('[vuex] error in before action subscribers: ')
      console.error(e)
    }
  }
  const result =
    entry.length > 1
      ? Promise.all(
          entry.map(function (handler) {
            return handler(payload)
          })
        )
      : entry[0](payload)
  return new Promise(function (resolve, reject) {
    result.then(
      function (res) {
        try {
          this$1$1._actionSubscribers
            .filter(function (sub) {
              return sub.after
            })
            .forEach(function (sub) {
              return sub.after(action, this$1$1.state)
            })
        } catch (e) {
          if (process.env.NODE_ENV !== 'production') {
            console.warn('[vuex] error in after action subscribers: ')
            console.error(e)
          }
        }
        resolve(res)
      },
      function (error) {
        try {
          this$1$1._actionSubscribers
            .filter(function (sub) {
              return sub.error
            })
            .forEach(function (sub) {
              return sub.error(action, this$1$1.state, error)
            })
        } catch (e) {
          if (process.env.NODE_ENV !== 'production') {
            console.warn('[vuex] error in error action subscribers: ')
            console.error(e)
          }
        }
        reject(error)
      }
    )
  })
}
```



## 问题

**1、为什么只能通过  commit 方法才能触发 state的更新，为啥不直接调用 mutations 的方法操作**

看 Store 的实现方式就能明白，Store 类里没有实现 mutation 方法，只提供了一个 commit 方法去触发更新

**2、为什么不能直接对 state 的数据进行状态更新，只能通过 commit 的方法**

- 强制限制对 store 数据的修改可以确保状态变化的可追踪性
- 也可以避免无意的从不同的组件直接修改 store 的数据导致代码难以维护和调试

**3、为什么存在异步调用的地方需要通过 store.dispath( ) 方法，不能直接调用 store.commit 来处理**

dispatch方法返回的是一个Promise对象，而commit方法没有返回值，完全进行的是同步代码的操作

**4、createStore() 和 useStore() 发生了什么**

- createStore() 方法，接收一个包含 state、mutations、actions 和 getters  函数的对象 options，然后将它们保存到实例属性中，此时 state 中的值都会转换成响应式对象，同时遍历所有的 getters 函数，将它们封装成 comuted 属性，并保存到实例属性 getters 上，
- vue 项目在 main.js 文件中调用了 app.use（Store），install 方法自动执行，将当前 store 实例注册到 VUE.js 应用程序中，只需要调用 useStore( ) 就可以拿到全局状态管理的 Store 实例，可以靠 inject 和 provide 实现全局共享