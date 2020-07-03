



# VueRouter源码解析

![img](https://user-gold-cdn.xitu.io/2018/5/8/1633d8c30a032a2d?imageslim)



### 前端路由原理

前端路由的本质就是监听URL的变化，然后匹配路由规则，显示出对应的页面，无须刷新。

路由模式

- Hash模式
- History模式

# 应用初始化

构建一个Vue应用时，会使用Vue.use以插件的形式安装VueRouter,同时在Vue实例上挂载router实例

```javascript

import Vue from 'vue'
import Router from 'vue-router'
import Home from './views/Home.vue'

Vue.use(Router)

export default new Router({
  mode: 'history',
  base: process.env.BASE_URL,
  routes: [
    {
      path: '/',
      name: 'home',
      component: Home
    },
    {
      path: '/about',
      name: 'about',
      component: () => import(/* webpackChunkName: "about" */ './views/About.vue')
    }
  ]
})

```

```javascript
import Vue from 'vue'
import App from './App.vue'
import router from './router'

Vue.config.productionTip = false

let a = new Vue({
  router,
  render: h => h(App)
}).$mount('#app')

```



## 路由注册

对于路由注册来说，核心就是调用Vue.use(VueRouter),使得VueRouter可以使用Vue,然后通过Vue来调用VueRouter的install函数；

在Install方法中，核心就是给组件混入beforeCreate钩子函数和全局注册两个路由组件

- 在全局混入mixin
- 在Vue的实例上初始化了一些私有属性
- 在Vue的prototype上初始化一些getter
- 全局注册RouterView,RouterLink组件

```javascript
export function install (Vue) {
  if (install.installed && _Vue === Vue) return
  install.installed = true
  _Vue = Vue
  const isDef = v => v !== undefined
  const registerInstance = (vm, callVal) => {
    let i = vm.$options._parentVnode
    if (isDef(i) && isDef(i = i.data) && isDef(i = i.registerRouteInstance)) {
      i(vm, callVal)
    }
  }
  // beforeCreate mixin
  Vue.mixin({
    beforeCreate () {
      // 判断组件是否存在 router 对象，该对象只在根组件上有
      if (isDef(this.$options.router)) {
        // _routerRoot, 指向了Vue的实例
        this._routerRoot = this
        // _router, 指向了VueRouter的实例
        this._router = this.$options.router
        // 初始化路由
        this._router.init(this)
        // 为 _route 属性实现双向绑定
        Vue.util.defineReactive(this, 'route', this.router.history.current)
      } else {
        // 用于 router-view 层级判断找到根路由
        this.routerRoot = (this.parent && this.parent.routerRoot) || this
      }
      registerInstance(this, this)
    },
    destroyed () {
      registerInstance(this)
    }
  })
  // $router, 当前Router的实例
  Object.defineProperty(Vue.prototype, '$router', {
    get () { return this.routerRoot.router }
  })
  // $route, 当前Router的信息
  Object.defineProperty(Vue.prototype, '$route', {
    get () { return this.routerRoot.route }
  })
  //  全局注册组件 router-link 和 router-view
  Vue.component('RouterView', View)
  Vue.component('RouterLink', Link)
  const strats = Vue.config.optionMergeStrategies
  // use the same hook merging strategy for route hooks
  strats.beforeRouteEnter = strats.beforeRouteLeave = strats.beforeRouteUpdate = strats.created
}

```

## VueRouter 实例

在实例化VueRouter的过程中，核心是创建一个路由匹配对象 ，并且根据mode来采取不同的路由方式 

```javascript
constructor (options: RouterOptions = {}) {
    this.app = null
    this.apps = []
    this.options = options
    this.beforeHooks = []
    this.resolveHooks = []
    this.afterHooks = []
    // 创建路由匹配对象
    this.matcher = createMatcher(options.routes || [], this)
    // 根据 mode 采取不同的路由方式
    let mode = options.mode || 'hash'
    // fallback会在不支持history环境的情况下, 回退到hash模式
    this.fallback = mode === 'history' && !supportsPushState && options.fallback !== false
    if (this.fallback) {
      mode = 'hash'
    }
    if (!inBrowser) {
      mode = 'abstract'
    }
    this.mode = mode

    switch (mode) {
      case 'history':
        this.history = new HTML5History(this, options.base)
        break
      case 'hash':
        this.history = new HashHistory(this, options.base, this.fallback)
        break
      case 'abstract':
        this.history = new AbstractHistory(this, options.base)
        break
      default:
        if (process.env.NODE_ENV !== 'production') {
          assert(false, `invalid mode: ${mode}`)
        }
    }
  }
```

## matcher

matcher对象中包含了两个属性, addRoutes, match。

- pathList：路径的列表
- pathMap：路径和路由对象的映射
- nameMap：路由名称和路由对象的映射

## 创建路由匹配对象

createMathcher函数的作用就是创建路由映射表，然后通过闭包的方式让addRouters 和 match函数能够使用路由映射表的几个对象,最后返回 一个Matcher对象

```javascript
export function createMatcher (
  routes: Array<RouteConfig>,
  router: VueRouter
): Matcher {
  // 创建路由映射表
  const { pathList, pathMap, nameMap } = createRouteMap(routes)
  
  // 添加路由
  function addRoutes (routes) {
    createRouteMap(routes, pathList, pathMap, nameMap)
  }
  // 路由匹配
  function match (
    raw: RawLocation,
    currentRoute?: Route,
    redirectedFrom?: Location
  ): Route {
    //...
  }
  return {
    match,
    addRoutes
  }
}
```

createMatcher函数创建映射表

```javascript
export function createRouteMap (
  routes: Array<RouteConfig>,
  oldPathList?: Array<string>,
  oldPathMap?: Dictionary<RouteRecord>,
  oldNameMap?: Dictionary<RouteRecord>
): {
  pathList: Array<string>;
  pathMap: Dictionary<RouteRecord>;
  nameMap: Dictionary<RouteRecord>;
} {
  // pathList，pathMap，nameMap支持后续的动态添加
  const pathList: Array<string> = oldPathList || []
  const pathMap: Dictionary<RouteRecord> = oldPathMap || Object.create(null)
  const nameMap: Dictionary<RouteRecord> = oldNameMap || Object.create(null)
  
  // 遍历路由配置，为每个配置添加路由记录
  routes.forEach(route => {
    addRouteRecord(pathList, pathMap, nameMap, route)
  })
  
  // 将通配符的路径, push到pathList的末尾
  for (let i = 0, l = pathList.length; i < l; i++) {
    if (pathList[i] === '*') {
      pathList.push(pathList.splice(i, 1)[0])
      l--
      i--
    }
  }
  return {
    pathList,
    pathMap,
    nameMap
  }
}

```

routes为一组路由, 所以我们循环routes, 但是route可能存在children所以我们通过递归的形式创建route。返回一个route的树

```javascript
// 添加路由记录
function addRouteRecord (
  pathList: Array<string>,
  pathMap: Dictionary<RouteRecord>,
  nameMap: Dictionary<RouteRecord>,
  route: RouteConfig,
  parent?: RouteRecord,
  matchAs?: string
) {
  // 获得路由配置下的属性
  const { path, name } = route
  const pathToRegexpOptions: PathToRegexpOptions = route.pathToRegexpOptions || {}
    
  // normalizePath, 会对path进行格式化
  // 会删除末尾的/，如果route是子级，会连接父级和子级的path，形成一个完整的path
  const normalizedPath = normalizePath(
    path,
    parent,
    pathToRegexpOptions.strict
  )
  
  // 创建一个完整的路由对象
  const record: RouteRecord = {
    path: normalizedPath,
    regex: compileRouteRegex(normalizedPath, pathToRegexpOptions),
    components: route.components || { default: route.component },
    instances: {},
    name,
    parent,
    matchAs,
    redirect: route.redirect,
    beforeEnter: route.beforeEnter,
    meta: route.meta || {},
    props: route.props == null
      ? {}
      : route.components
        ? route.props
        : { default: route.props }
  }
 
  // 如果route存在children, 我们会递归的创建路由对象
  if (route.children) {
    // 递归路由配置的 children 属性，添加路由记录
    route.children.forEach(child => {
      const childMatchAs = matchAs
        ? cleanPath(`${matchAs}/${child.path}`)
        : undefined
      addRouteRecord(pathList, pathMap, nameMap, child, record, childMatchAs)
    })
  }
    
  // 如果路由有别名的话，给别名也添加路由记录
  if (route.alias !== undefined) {
    const aliases = Array.isArray(route.alias)
      ? route.alias
      : [route.alias]
 
    aliases.forEach(alias => {
      const aliasRoute = {
        path: alias,
        children: route.children
      }
      addRouteRecord(
        pathList,
        pathMap,
        nameMap,
        aliasRoute,
        parent,
        record.path || '/' // matchAs
      )
    })
  }
    
  //  填充pathMap，nameMap，pathList，可以理解为动态添加的时候 
  if (!pathMap[record.path]) {
    pathList.push(record.path)
    pathMap[record.path] = record
  }
  if (name) {
    if (!nameMap[name]) {
      nameMap[name] = record
    } else if (process.env.NODE_ENV !== 'production' && !matchAs) {
      warn(
        false,
        `Duplicate named routes definition: ` +
        `{ name: "${name}", path: "${record.path}" }`
      )
    }
  }
}
```

### addRoutes

动态添加更多的路由规则, 并动态的修改pathList，pathMap，nameMap

```javascript
function addRoutes (routes) {
  createRouteMap(routes, pathList, pathMap, nameMap)
}
```

### match

match方法根据参数raw(可以是字符串也可以Location对象), 以及currentRoute（当前的路由对象返回Route对象)，在nameMap中查找对应的Route，并返回。

如果location包含name, 我通过nameMap找到了对应的Route, 但是此时path中可能包含params, 所以我们会通过fillParams函数将params填充到patch，返回一个真实的路径path。

```javascript

function match (
  raw,
  currentRoute,
  redirectedFrom
) {
  // 会对raw，currentRoute处理，返回格式化后path, hash, 以及params
  const location = normalizeLocation(raw, currentRoute, false, router)

  const { name } = location

  if (name) {
    const record = nameMap[name]
    if (!record) return _createRoute(null, location)
    
    // 获取所有必须的params。如果optional为true说明params不是必须的
    const paramNames = record.regex.keys
      .filter(key => !key.optional)
      .map(key => key.name)

    if (typeof location.params !== 'object') {
      location.params = {}
    }

    if (currentRoute && typeof currentRoute.params === 'object') {
      for (const key in currentRoute.params) {
        if (!(key in location.params) && paramNames.indexOf(key) > -1) {
          location.params[key] = currentRoute.params[key]
        }
      }
    }

    if (record) {
      // 使用params对path进行填充返回一个真实的路径
      location.path = fillParams(record.path, location.params, `named route "${name}"`)
      // 创建Route对象
      return _createRoute(record, location, redirectedFrom)
    }
  } else if (location.path) {
    location.params = {}
    for (let i = 0; i < pathList.length; i++) {
      const path = pathList[i]
      const record = pathMap[path]
      // 使用pathList中的每一个regex，对path进行匹配
      if (matchRoute(record.regex, location.path, location.params)) {
        return _createRoute(record, location, redirectedFrom)
      }
    }
  }
  return _createRoute(null, location)
}

```

### _createRoute

```javascript

function _createRoute (
  record: ?RouteRecord,
  location: Location,
  redirectedFrom?: Location
): Route {
  if (record && record.redirect) {
    return redirect(record, redirectedFrom || location)
  }
  if (record && record.matchAs) {
    return alias(record, location, record.matchAs)
  }
  return createRoute(record, location, redirectedFrom, router)
}

```

其中redirect，alias最终都会调用createRoute方法。

createRoute函数会返回一个冻结的Router对象。

其中matched属性为一个数组，包含当前路由的所有嵌套路径片段的路由记录。数组的顺序为从外向里(树的外层到内层)。

```javascript
export function createRoute (
  record: ?RouteRecord,
  location: Location,
  redirectedFrom?: ?Location,
  router?: VueRouter
): Route {
  const stringifyQuery = router && router.options.stringifyQuery

  let query: any = location.query || {}
  try {
    query = clone(query)
  } catch (e) {}

  const route: Route = {
    name: location.name || (record && record.name),
    meta: (record && record.meta) || {},
    path: location.path || '/',
    hash: location.hash || '',
    query,
    params: location.params || {},
    fullPath: getFullPath(location, stringifyQuery),
    matched: record ? formatMatch(record) : []
  }
  if (redirectedFrom) {
    route.redirectedFrom = getFullPath(redirectedFrom, stringifyQuery)
  }
  return Object.freeze(route)
}

```



## 路由初始化

当根组件调用beforeCreate钩子函数时，会执行` this._router.init(this)`初始化路由;

在路由初始化时,核心就是添加路由跳转，改变URL然后渲染对应的组件

```javascript
 init (app: any /* Vue component instance */) {
    // 保存组件实例，app为Vue的实例
    this.apps.push(app)

    // 如果根组件已经有了就返回
    if (this.app) {
      return
    }
   
    // 在VueRouter上挂载app属性
    this.app = app
   
    // 赋值路由模式
    const history = this.history

    // 初始化当前的路由，完成第一次导航，在hash模式下会在transitionTo的回调中调用setupListeners
    // setupListeners里会对hashchange事件进行监听
    //  transitionTo是进行路由导航的函数
    
    if (history instanceof HTML5History || history instanceof HashHistory) {
      // scrollBehavior滚动行为,只支持在history.pushState的浏览器中使用
      const handleInitialScroll = (routeOrError) => {
        const from = history.current
        const expectScroll = this.options.scrollBehavior
        const supportsScroll = supportsPushState && expectScroll
        if (supportsScroll && 'fullPath' in routeOrError) {
          handleScroll(this, routeOrError, from, false)
        }
      }
      // 添加路由监听
      const setupListeners = (routeOrError) => {
        history.setupListeners()
        handleInitialScroll(routeOrError)
      }
      // 路由跳转
      history.transitionTo(history.getCurrentLocation(), setupListeners, setupListeners)
    }
   
    // 该回调会在 transitionTo 中调用
    // 对组件的 _route 属性进行赋值，触发组件渲染
    history.listen(route => {
      this.apps.forEach((app) => {
        app._route = route
      })
    })
  }
```

## history

history一共有三个模式：hash，history，abstract,这三个类都继承History类。

### base

base的构造函数

- router：是VueRouter的实例
- base：是路由的基础路径
- current：是当前的路由，默认为/
- ready：是路由的状态
- readyCbs：是ready的回调的集合
- readyErrorCbs：是ready失败的回调
- errorCbs：是导航出错的回调的集合
- listeners

```javascript

export class History {
  constructor (router: Router, base: ?string) {
    this.router = router
    // normalizeBase会对base路径做出格式化的处理，会为base开头自动添加‘/’，删除结尾的‘/’，默认返回’/‘
    this.base = normalizeBase(base)
    // 初始化的当前路由对象
    this.current = START
    this.pending = null
    this.ready = false
    this.readyCbs = []
    this.readyErrorCbs = []
    this.errorCbs = []
  }
}

```

```javascript
export const START = createRoute(null, {
  path: '/'
})
```

```javascript

function normalizeBase (base: ?string): string {
  if (!base) {
    // inBrowser判断是否为浏览器环境
    if (inBrowser) {
      const baseEl = document.querySelector('base')
      base = (baseEl && baseEl.getAttribute('href')) || '/'
      base = base.replace(/^https?:\/\/[^\/]+/, '')
    } else {
      base = '/'
    }
  }
  if (base.charAt(0) !== '/') {
    base = '/' + base
  }
  return base.replace(/\/$/, '')
}

```

base中的listen方法，会在VueRouter的init方法中使用到,listen会给每一次的路由的更新添加回调

### HashRouter

#### 构造函数

在HashHistory的构造函数中。我们会判断当前的fallback是否为true。如果为true，使用checkFallback，添加’#‘，并使用window.location.replace替换文档。

如果fallback为false，我们会调用ensureSlash，ensureSlash会为没有“#”的url，添加“#”，并且使用histroy的API或者replace替换文档。

```javascript

export class HashHistory extends History {
  constructor (router: Router, base: ?string, fallback: boolean) {
    super(router, base)
    // 如果是回退hash的情况，并且判断当前路径是否有/#/。如果没有将会添加'/#/'
    if (fallback && checkFallback(this.base)) {
      return
    }
    ensureSlash()
  }
}

```

checkFallback

```javascript
// 检查url是否包含‘/#/’
function checkFallback (base) {
  // 获取hash值
  const location = getLocation(base)
  // 如果location不是以/#，开头。添加/#，使用window.location.replace替换文档
  if (!/^\/#/.test(location)) {
    window.location.replace(
      cleanPath(base + '/#' + location)
    )
    return true
  }
}
```

```javascript
// 返回hash
export function getLocation (base) {
  let path = decodeURI(window.location.pathname)
  if (base && path.indexOf(base) === 0) {
    path = path.slice(base.length)
  }
  return (path || '/') + window.location.search + window.location.hash
}

```

```javascript
// 删除 //, 替换为 /
export function cleanPath (path) {
  return path.replace(/\/\//g, '/')
}
```

ensureSlash

```javascript

function ensureSlash (): boolean {
  // 判断是否包含#，并获取hash值。如果url没有#，则返回‘’
  const path = getHash()
  // 判断path是否以/开头
  if (path.charAt(0) === '/') {
    return true
  }
  // 如果开头不是‘/’, 则添加/
  replaceHash('/' + path)
  return false
}

```



## 路由跳转

```javascript
  transitionTo (
    location: RawLocation,
    onComplete?: Function,
    onAbort?: Function
  ) {
    // 获取匹配的路由信息
    const route = this.router.match(location, this.current)
    // 确认切换路由
    this.confirmTransition(
      route,
      () => {
        const prev = this.current
        // 更新路由信息，对组件的 _route 属性进行赋值，触发组件渲染
        this.updateRoute(route)
        // 添加事件监听
        onComplete && onComplete(route)
        // 更新URL
        this.ensureURL()
        this.router.afterHooks.forEach(hook => {
          hook && hook(route, prev)
        })

        // 只执行一次 ready 回调
        if (!this.ready) {
          this.ready = true
          this.readyCbs.forEach(cb => {
            cb(route)
          })
        }
      },
      // 错误处理
      err => {
        if (onAbort) {
          onAbort(err)
        }
        if (err && !this.ready) {
          this.ready = true
          // Initial redirection should still trigger the onReady onSuccess
          // https://github.com/vuejs/vue-router/issues/3225
          if (!isRouterError(err, NavigationFailureType.redirected)) {
            this.readyErrorCbs.forEach(cb => {
              cb(err)
            })
          } else {
            this.readyCbs.forEach(cb => {
              cb(route)
            })
          }
        }
      }
    )
  }
```

在路由跳转中，需要先获取匹配的路由信息，看下如何获取匹配的路由信息

### 匹配路由信息

```javascript
 function match (
    raw: RawLocation,
    currentRoute?: Route,
    redirectedFrom?: Location
  ): Route {
    // 序列化 url
    // 比如对于该 url 来说 /abc?foo=bar&baz=qux#hello
    // 会序列化路径为 /abc
    // 哈希为 #hello
    // 参数为 foo: 'bar', baz: 'qux'
    const location = normalizeLocation(raw, currentRoute, false, router)
    const { name } = location

    if (name) {
      // 如果是命名路由，就判断记录中是否有该命名路由配置
      const record = nameMap[name]
      if (process.env.NODE_ENV !== 'production') {
        warn(record, `Route with name '${name}' does not exist`)
      }
      // 没找到表示没有匹配的路由
      if (!record) return _createRoute(null, location)
      const paramNames = record.regex.keys
        .filter(key => !key.optional)
        .map(key => key.name)
      // 参数处理
      if (typeof location.params !== 'object') {
        location.params = {}
      }

      if (currentRoute && typeof currentRoute.params === 'object') {
        for (const key in currentRoute.params) {
          if (!(key in location.params) && paramNames.indexOf(key) > -1) {
            location.params[key] = currentRoute.params[key]
          }
        }
      }

      location.path = fillParams(record.path, location.params, `named route "${name}"`)
      return _createRoute(record, location, redirectedFrom)
    } else if (location.path) {
      // 非命名路由处理
      location.params = {}
      for (let i = 0; i < pathList.length; i++) {
        // 查找记录
        const path = pathList[i]
        const record = pathMap[path]
        // 如果匹配路由，则创建路由
        if (matchRoute(record.regex, location.path, location.params)) {
          return _createRoute(record, location, redirectedFrom)
        }
      }
    }
    // 没有匹配的路由
    return _createRoute(null, location)
  }
```

### 创建路由

```javascript
// 根据条件创建不同的路由
function _createRoute(
  record: ?RouteRecord,
  location: Location,
  redirectedFrom?: Location
): Route {
  if (record && record.redirect) {
    return redirect(record, redirectedFrom || location)
  }
  if (record && record.matchAs) {
    return alias(record, location, record.matchAs)
  }
  return createRoute(record, location, redirectedFrom, router)
}
 
export function createRoute (
  record: ?RouteRecord,
  location: Location,
  redirectedFrom?: ?Location,
  router?: VueRouter
): Route {
  const stringifyQuery = router && router.options.stringifyQuery
  // 克隆参数
  let query: any = location.query || {}
  try {
    query = clone(query)
  } catch (e) {}
  // 创建路由对象
  const route: Route = {
    name: location.name || (record && record.name),
    meta: (record && record.meta) || {},
    path: location.path || '/',
    hash: location.hash || '',
    query,
    params: location.params || {},
    fullPath: getFullPath(location, stringifyQuery),
    matched: record ? formatMatch(record) : []
  }
  if (redirectedFrom) {
    route.redirectedFrom = getFullPath(redirectedFrom, stringifyQuery)
  }
  // 让路由对象不可修改
  return Object.freeze(route)
}
// 获得包含当前路由的所有嵌套路径片段的路由记录
// 包含从根路由到当前路由的匹配记录，从上至下
function formatMatch(record: ?RouteRecord): Array<RouteRecord> {
  const res = []
  while (record) {
    res.unshift(record)
    record = record.parent
  }
  return res
}

```

###  confirmTransition确认跳转函数

```javascript
transitionTo (location: RawLocation, onComplete?: Function, onAbort?: Function) {
  // 确认切换路由
  this.confirmTransition(route, () => {}
}
confirmTransition(route: Route, onComplete: Function, onAbort?: Function) {
  const current = this.current
  // 中断跳转路由函数
  const abort = err => {
    if (isError(err)) {
      if (this.errorCbs.length) {
        this.errorCbs.forEach(cb => {
          cb(err)
        })
      } else {
        warn(false, 'uncaught error during route navigation:')
        console.error(err)
      }
    }
    onAbort && onAbort(err)
  }
  // 如果是相同的路由就不跳转
  if (
    isSameRoute(route, current) &&
    route.matched.length === current.matched.length
  ) {
    this.ensureURL()
    return abort()
  }
  // 通过对比路由解析出可复用的组件，需要渲染的组件，失活的组件
  const { updated, deactivated, activated } = resolveQueue(
    this.current.matched,
    route.matched
  )
  
  function resolveQueue(
      current: Array<RouteRecord>,
      next: Array<RouteRecord>
    ): {
      updated: Array<RouteRecord>,
      activated: Array<RouteRecord>,
      deactivated: Array<RouteRecord>
    } {
      let i
      const max = Math.max(current.length, next.length)
      for (i = 0; i < max; i++) {
        // 当前路由路径和跳转路由路径不同时跳出遍历
        if (current[i] !== next[i]) {
          break
        }
      }
      return {
        // 可复用的组件对应路由
        updated: next.slice(0, i),
        // 需要渲染的组件对应路由
        activated: next.slice(i),
        // 失活的组件对应路由
        deactivated: current.slice(i)
      }
  }
  // 导航守卫数组
  const queue: Array<?NavigationGuard> = [].concat(
    // 失活的组件钩子
    extractLeaveGuards(deactivated),
    // 全局 beforeEach 钩子
    this.router.beforeHooks,
    // 在当前路由改变，但是该组件被复用时调用
    extractUpdateHooks(updated),
    // 需要渲染组件 enter 守卫钩子
    activated.map(m => m.beforeEnter),
    // 解析异步路由组件
    resolveAsyncComponents(activated)
  )
  // 保存路由
  this.pending = route
  // 迭代器，用于执行 queue 中的导航守卫钩子
  const iterator = (hook: NavigationGuard, next) => {
  // 路由不相等就不跳转路由
    if (this.pending !== route) {
      return abort()
    }
    try {
    // 执行钩子
      hook(route, current, (to: any) => {
        // 只有执行了钩子函数中的 next，才会继续执行下一个钩子函数
        // 否则会暂停跳转
        // 以下逻辑是在判断 next() 中的传参
        if (to === false || isError(to)) {
          // next(false) 
          this.ensureURL(true)
          abort(to)
        } else if (
          typeof to === 'string' ||
          (typeof to === 'object' &&
            (typeof to.path === 'string' || typeof to.name === 'string'))
        ) {
        // next('/') 或者 next({ path: '/' }) -> 重定向
          abort()
          if (typeof to === 'object' && to.replace) {
            this.replace(to)
          } else {
            this.push(to)
          }
        } else {
        // 这里执行 next
        // 也就是执行下面函数 runQueue 中的 step(index + 1)
          next(to)
        }
      })
    } catch (e) {
      abort(e)
    }
  }
  // 经典的同步执行异步函数
  runQueue(queue, iterator, () => {
    const postEnterCbs = []
    const isValid = () => this.current === route
    // 当所有异步组件加载完成后，会执行这里的回调，也就是 runQueue 中的 cb()
    // 接下来执行 需要渲染组件的导航守卫钩子
    const enterGuards = extractEnterGuards(activated, postEnterCbs, isValid)
    const queue = enterGuards.concat(this.router.resolveHooks)
    runQueue(queue, iterator, () => {
    // 跳转完成
      if (this.pending !== route) {
        return abort()
      }
      this.pending = null
      onComplete(route)
      if (this.router.app) {
        this.router.app.$nextTick(() => {
          postEnterCbs.forEach(cb => {
            cb()
          })
        })
      }
    })
  })
}
export function runQueue (queue: Array<?NavigationGuard>, fn: Function, cb: Function) {
  const step = index => {
  // 队列中的函数都执行完毕，就执行回调函数
    if (index >= queue.length) {
      cb()
    } else {
      if (queue[index]) {
      // 执行迭代器，用户在钩子函数中执行 next() 回调
      // 回调中判断传参，没有问题就执行 next()，也就是 fn 函数中的第二个参数
        fn(queue[index], () => {
          step(index + 1)
        })
      } else {
        step(index + 1)
      }
    }
  }
  // 取出队列中第一个钩子函数
  step(0)
}
```

### 导航守卫

```javascript

const queue: Array<?NavigationGuard> = [].concat(
    // 失活的组件钩子
    extractLeaveGuards(deactivated),
    // 全局 beforeEach 钩子
    this.router.beforeHooks,
    // 在当前路由改变，但是该组件被复用时调用
    extractUpdateHooks(updated),
    // 需要渲染组件 enter 守卫钩子
    activated.map(m => m.beforeEnter),
    // 解析异步路由组件
    resolveAsyncComponents(activated)

```

第一步是先执行失活组件的钩子函数

```javascript
function extractLeaveGuards(deactivated: Array<RouteRecord>): Array<?Function> {
// 传入需要执行的钩子函数名
  return extractGuards(deactivated, 'beforeRouteLeave', bindGuard, true)
}
function extractGuards(
  records: Array<RouteRecord>,
  name: string,
  bind: Function,
  reverse?: boolean
): Array<?Function> {
  const guards = flatMapComponents(records, (def, instance, match, key) => {
   // 找出组件中对应的钩子函数
    const guard = extractGuard(def, name)
    if (guard) {
    // 给每个钩子函数添加上下文对象为组件自身
      return Array.isArray(guard)
        ? guard.map(guard => bind(guard, instance, match, key))
        : bind(guard, instance, match, key)
    }
  })
  // 数组降维，并且判断是否需要翻转数组
  // 因为某些钩子函数需要从子执行到父
  return flatten(reverse ? guards.reverse() : guards)
}
export function flatMapComponents (
  matched: Array<RouteRecord>,
  fn: Function
): Array<?Function> {
// 数组降维
  return flatten(matched.map(m => {
  // 将组件中的对象传入回调函数中，获得钩子函数数组
    return Object.keys(m.components).map(key => fn(
      m.components[key],
      m.instances[key],
      m, key
    ))
  }))
}
```

第二步执行全局 beforeEach 钩子函数

```javascript
beforeEach(fn: Function): Function {
    return registerHook(this.beforeHooks, fn)
}
function registerHook(list: Array<any>, fn: Function): Function {
  list.push(fn)
  return () => {
    const i = list.indexOf(fn)
    if (i > -1) list.splice(i, 1)
  }
}

```

第三步执行 `beforeRouteUpdate` 钩子函数，调用方式和第一步相同，只是传入的函数名不同，在该函数中可以访问到 `this` 对象。

第四步执行 `beforeEnter` 钩子函数，该函数是路由独享的钩子函数。

第五步是解析异步组件

```javascript
export function resolveAsyncComponents (matched: Array<RouteRecord>): Function {
  return (to, from, next) => {
    let hasAsync = false
    let pending = 0
    let error = null
    // 该函数作用之前已经介绍过了
    flatMapComponents(matched, (def, _, match, key) => {
    // 判断是否是异步组件
      if (typeof def === 'function' && def.cid === undefined) {
        hasAsync = true
        pending++
        // 成功回调
        // once 函数确保异步组件只加载一次
        const resolve = once(resolvedDef => {
          if (isESModule(resolvedDef)) {
            resolvedDef = resolvedDef.default
          }
          // 判断是否是构造函数
          // 不是的话通过 Vue 来生成组件构造函数
          def.resolved = typeof resolvedDef === 'function'
            ? resolvedDef
            : _Vue.extend(resolvedDef)
        // 赋值组件
        // 如果组件全部解析完毕，继续下一步
          match.components[key] = resolvedDef
          pending--
          if (pending <= 0) {
            next()
          }
        })
        // 失败回调
        const reject = once(reason => {
          const msg = `Failed to resolve async component ${key}: ${reason}`
          process.env.NODE_ENV !== 'production' && warn(false, msg)
          if (!error) {
            error = isError(reason)
              ? reason
              : new Error(msg)
            next(error)
          }
        })
        let res
        try {
        // 执行异步组件函数
          res = def(resolve, reject)
        } catch (e) {
          reject(e)
        }
        if (res) {
        // 下载完成执行回调
          if (typeof res.then === 'function') {
            res.then(resolve, reject)
          } else {
            const comp = res.component
            if (comp && typeof comp.then === 'function') {
              comp.then(resolve, reject)
            }
          }
        }
      }
    })
    // 不是异步组件直接下一步
    if (!hasAsync) next()
  }
}
```

第五步完成后会执行第一个 `runQueue` 中回调函数

```javascript
// 该回调用于保存 `beforeRouteEnter` 钩子中的回调函数
const postEnterCbs = []
const isValid = () => this.current === route
// beforeRouteEnter 导航守卫钩子
const enterGuards = extractEnterGuards(activated, postEnterCbs, isValid)
// beforeResolve 导航守卫钩子
const queue = enterGuards.concat(this.router.resolveHooks)
runQueue(queue, iterator, () => {
  if (this.pending !== route) {
    return abort()
  }
  this.pending = null
  // 这里会执行 afterEach 导航守卫钩子
  onComplete(route)
  if (this.router.app) {
    this.router.app.$nextTick(() => {
      postEnterCbs.forEach(cb => {
        cb()
      })
    })
  }
})
```

第六步是执行 `beforeRouteEnter` 导航守卫钩子，`beforeRouteEnter` 钩子不能访问 `this` 对象，因为钩子在导航确认前被调用，需要渲染的组件还没被创建。但是该钩子函数是唯一一个支持在回调中获取 `this` 对象的函数，回调会在路由确认执行

```javascript

beforeRouteEnter (to, from, next) {
  next(vm => {
    // 通过 `vm` 访问组件实例
  })
}
```

看看是如何支持在回调中拿到 `this` 对象的

```javascript
function extractEnterGuards(
  activated: Array<RouteRecord>,
  cbs: Array<Function>,
  isValid: () => boolean
): Array<?Function> {
// 这里和之前调用导航守卫基本一致
  return extractGuards(
    activated,
    'beforeRouteEnter',
    (guard, _, match, key) => {
      return bindEnterGuard(guard, match, key, cbs, isValid)
    }
  )
}
function bindEnterGuard(
  guard: NavigationGuard,
  match: RouteRecord,
  key: string,
  cbs: Array<Function>,
  isValid: () => boolean
): NavigationGuard {
  return function routeEnterGuard(to, from, next) {
    return guard(to, from, cb => {
    // 判断 cb 是否是函数
    // 是的话就 push 进 postEnterCbs
      next(cb)
      if (typeof cb === 'function') {
        cbs.push(() => {
          // 循环直到拿到组件实例
          poll(cb, match.instances, key, isValid)
        })
      }
    })
  }
}
// 该函数是为了解决 issus #750
// 当 router-view 外面包裹了 mode 为 out-in 的 transition 组件 
// 会在组件初次导航到时获得不到组件实例对象
function poll(
  cb: any, // somehow flow cannot infer this is a function
  instances: Object,
  key: string,
  isValid: () => boolean
) {
  if (
    instances[key] &&
    !instances[key]._isBeingDestroyed // do not reuse being destroyed instance
  ) {
    cb(instances[key])
  } else if (isValid()) {
  // setTimeout 16ms 作用和 nextTick 基本相同
    setTimeout(() => {
      poll(cb, instances, key, isValid)
    }, 16)
  }
}
```

第七步是执行 `beforeResolve` 导航守卫钩子，如果注册了全局 `beforeResolve` 钩子就会在这里执行

第八步就是导航确认，调用 `afterEach` 导航守卫钩子了

以上都执行完成后，会触发组件的渲染

```javascript

history.listen(route => {
      this.apps.forEach(app => {
        app._route = route
      })
}
```

以上回调会在 `updateRoute` 中调用

```javascript

updateRoute(route: Route) {
    const prev = this.current
    this.current = route
    this.cb && this.cb(route)
    this.router.afterHooks.forEach(hook => {
      hook && hook(route, prev)
    })
}
```

路由跳转的核心就是判断需要跳转的路由是否存在于记录中，然后执行各种导航守卫函数，最后完成 URL 的改变和组件的渲染。