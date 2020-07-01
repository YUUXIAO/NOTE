

1. Vue-router的工作流程

   url的改变 =》触发监听事件 =》改变vue-router的current变量 =》监视current变量的监听者 =》获取新的组件 =》render新组件

2. vue插件

   实际就是对一个install方法执行一遍

   可接收方法，单纯执行

   如果有install,执行install方法或者属性

3. init()方法：监听路由改变事件

   根据mode

   设置默认‘/’

   history有个current属性

   load也要设置current

4. vue.utils.defineReative()

   将一个外部变量关联到vue

   # VueRouter源码解析

   ​

   ### 前端路由原理

   前端路由的本质就是监听URL的变化，然后匹配路由规则，显示出对应的页面，无须刷新。

   路由模式

   - Hash模式
   - History模式


## 路由注册

对于路由注册来说，核心就是调用Vue.use(VueRouter),使得VueRouter可以使用Vue,然后通过Vue来调用VueRouter的install函数；

在Install方法中，核心就是给组件混入beforeCreate钩子函数和全局注册两个路由组件

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
        // 根路由设置为自己
        this._routerRoot = this
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
  // 注入 router route
  Object.defineProperty(Vue.prototype, '$router', {
    get () { return this.routerRoot.router }
  })
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

## VueRouter 实例化

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

## 创建路由匹配对象

createMathcher函数的作用就是创建路由映射表，然后通过闭包的方式让addRouters 和 match函数能够使用路由映射表的几个对象,最后返回 一个Matcher对象

```javascript
export function createMatcher (
  routes: Array<RouteConfig>,
  router: VueRouter
): Matcher {
    // 创建路由映射表
  const { pathList, pathMap, nameMap } = createRouteMap(routes)
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
  // 创建映射表
  const pathList: Array<string> = oldPathList || []
  const pathMap: Dictionary<RouteRecord> = oldPathMap || Object.create(null)
  const nameMap: Dictionary<RouteRecord> = oldNameMap || Object.create(null)
  // 遍历路由配置，为每个配置添加路由记录
  routes.forEach(route => {
    addRouteRecord(pathList, pathMap, nameMap, route)
  })
  // 确保通配符在最后
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
  // 格式化 url，替换 / 
  const normalizedPath = normalizePath(
    path,
    parent,
    pathToRegexpOptions.strict
  )
  // 生成记录对象
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
 
  if (route.children) {
    // 递归路由配置的 children 属性，添加路由记录
    route.children.forEach(child => {
      const childMatchAs = matchAs
        ? cleanPath(`${matchAs}/${child.path}`)
        : undefined
      addRouteRecord(pathList, pathMap, nameMap, child, record, childMatchAs)
    })
  }
  // 如果路由有别名的话
  // 给别名也添加路由记录
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
  // 更新映射表
  if (!pathMap[record.path]) {
    pathList.push(record.path)
    pathMap[record.path] = record
  }
  // 命名路由添加记录
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

## 路由初始化

当根组件调用beforeCreate钩子函数时，会执行` this._router.init(this)`初始化路由;

在路由初始化时,核心就是添加路由跳转，改变URL然后渲染对应的组件

```javascript
 init (app: any /* Vue component instance */) {
    // 保存组件实例
    this.apps.push(app)

    // 如果根组件已经有了就返回
    if (this.app) {
      return
    }
    this.app = app
    // 赋值路由模式
    const history = this.history

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

## 路由跳转

