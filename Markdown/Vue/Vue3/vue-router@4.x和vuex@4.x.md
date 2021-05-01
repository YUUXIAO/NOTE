## vue-router@4.x

### 创建实例

```javascript
// 2.x-router
import Vue from 'vue'
import Router from 'vue-router'

const routes = [
  {
    path: '/',
    name: 'Home',
    component: Home
  },
  {
    path: '/about',
    name: 'About',
    component: () => import(/* webpackChunkName: "about" */ '../views/About.vue')
  }
]

Vue.use(Router)

const router = new Router({
  base: process.env.BASE_URL,
  mode: 'history',
  scrollBehavior: () => ({ y: 0 }),
  routes
})

export default router

// 3.x-router
import { createRouter, createWebHistory } from 'vue-router'
import Home from '../views/Home.vue'

const routes = [
  {
    path: '/',
    name: 'Home',
    component: Home
  },
  {
    path: '/about',
    name: 'About',
    component: () => import(/* webpackChunkName: "about" */ '../views/About.vue')
  }
]

const router = createRouter({
  history: createWebHistory(process.env.BASE_URL),
  routes
})

export default router
```

### scrollBehavior

```javascript
// 2.x-router
const router = new Router({
  base: process.env.BASE_URL,
  mode: 'history',
  scrollBehavior: () => ({ x: 0, y: 0 }),
  routes
})

// 3.x-router
const router = createRouter({
  history: createWebHistory(process.env.BASE_URL),
  routes,
  scrollBehavior(to, from, savedPosition) {
    // scroll to id `#app` + 200px, and scroll smoothly (when supported by the browser)
    return {
      el: '#app',
      top: 0,
      left: 0,
      behavior: 'smooth'
    }
  }
})
```

### 路由组件跳转

> 2.x 使用路由选项 redirect 设置路由自动调整，3.x中移除了这个选项，将在子路由中添加一个空路径路由来匹配跳转；

```javascript
// 2.x-router
[
  {
    path: '/',
    component: Layout,
    name: 'WebHome',
    meta: { title: '首页' },
    redirect: '/dashboard', // 这里写跳转
    children: [
      {
        path: 'dashboard',
        name: 'Dashboard',
        meta: { title: '工作台' },
        component: () => import('../views/dashboard/index.vue')
      }
    ]
  }
]

// 3.x-router
[
  {
    path: '/',
    component: Layout,
    name: 'WebHome',
    meta: { title: '首页' },
    children: [
      { path: '', redirect: 'dashboard' }, // 这里写跳转
      {
        path: 'dashboard',
        name: 'Dashboard',
        meta: { title: '工作台' },
        component: () => import('../views/dashboard/index.vue')
      }
    ]
  }
]
```

### 捕获所有路由 /:catchAll(.*)

> 捕获所有路由 ( /* )  时，必须使用带有自定义正则表达式的参数进行定义：/:catchAll(.*)；

```javascript
// 2.x-router
const router = new VueRouter({
  mode: 'history',
  routes: [
    { path: '/user/:a*' },
  ]
})

// 3.x-router
const router = createRouter({
  history: createWebHistory(),
  routes: [
    { path: '/user/:a:catchAll(.*)', component: component },
  ]
})

// 当路由为 /user/a/b 时，捕获到的 params 为 {"a": "a", "catchAll": "/b"}
```

### router.resolve

```javascript
// 2.x-router
...
resolve ( to: RawLocation, current?: Route, append?: boolean) {
  ...
  return {
    location,
    route,
    href,
    normalizedTo: location,
    resolved: route
  }
}

// 3.x-router
function resolve(
    rawLocation: Readonly<RouteLocationRaw>,
    currentLocation?: Readonly<RouteLocationNormalizedLoaded>
  ): RouteLocation & { href: string } {
  ...
  let matchedRoute = matcher.resolve(matcherLocation, currentLocation)
  ...
  return {
    fullPath,
    hash,
    query: normalizeQuery(rawLocation.query),
    ...matchedRoute,
    redirectedFrom: undefined,
    href: routerHistory.base + fullPath,
  }
}
```

### 不存在的命名路由

2.x-router 中，当 push 一个不存在的命名路由时，路由会导航到根路由 "/" 下，并且不会渲染任何内容，浏览器控制台只会打印警告；

3.x-router 中，同样做法会引发错误；

```javascript
// 2.x-router
const router = new VueRouter({
  mode: 'history',
  routes: [{ path: '/', name: 'foo', component: Foo }]
}
this.$router.push({ name: 'baz' }) // 导航到根路由 "/" 下

// 3.x-router
const router = createRouter({
  history: routerHistory(),
  routes: [{ path: '/', name: 'foo', component: Foo }]
})
...
import { useRouter } from 'vue-next-router'
...
const router = userRouter()
router.push({ name: 'baz' })) // 这行代码会报错
```

### 获取路由

```javascript
import { getCurrentInstance } from 'vue'
import { useRouter, useRoute } from 'vue-router'

export default {
  setup () {
    const router = useRouter()
    const route = useRoute()
    console.log(router, route)
    console.log(router.currentRoute.value)
  }
}
```

## vuex@4.x

### 创建实例

```javascript
// 2.x-vuex
import Vue from 'vue'
import Vuex from 'vuex'

Vue.use(Vuex)

export default new Vuex.Store({
  state: {},
  mutations: {},
  actions: {},
  getters: {},
  modules: {}
}

// 3.x-vuex
import Vuex from 'vuex'

export default Vuex.createStore({
  state: {},
  mutations: {},
  actions: {},
  getters: {},
  modules: {}
})
```

### 获取store

```javascript
import { getCurrentInstance } from 'vue'
import { useStore } from 'vuex'

export default {
  setup () {
    const store = userStore()
    const userId = computed(() => store.state.userId)
    return {
      userId
    }
  }
}
```