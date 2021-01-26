## new Vue过程

1. 调用 _init 合并配置，初始化生命周期，初始化事件中心，初始化渲染，初始化 data、props、computed、watcher 等；
2. 通过 Object.defineProperty 设置 setter 与 getter 函数，用来实现响应式以及依赖收集；

## 构造函数变化过程

1. 通过 instance 对 Vue.prototype 进行属性和方法的挂载；
2. 通过 core 对 Vue 进行静态属性和方法的挂载；
3. 通过 runtime 添加了对 platform === 'web'的情况下特有的配置、组件和指令；
4. 通过 entry 来为 $mount 方法增加编译 template 的能力；

### instance

> instance 主要是对 Vue 原型上挂载一系列方法的操作；

```javascript
// src/core/instance/index.js
import { initMixin } from './init'
import { stateMixin } from './state'
import { renderMixin } from './render'
import { eventsMixin } from './events'
import { lifecycleMixin } from './lifecycle'
import { warn } from '../util/index'

function Vue (options) {
  if (process.env.NODE_ENV !== 'production' &&
    !(this instanceof Vue)
  ) {
  	warn('Vue is a constructor and should be 
  	called with the `new` keyword')
  }
  this._init(options)
}

initMixin(Vue)
stateMixin(Vue)
eventsMixin(Vue)
lifecycleMixin(Vue)
renderMixin(Vue)

export default Vue
```

#### initMixin

> initMixin 方法主要的作用是在 Vue.prototype上定义 _init 方法；

```javascript
// src/core/instance/init.js
export function initMixin (Vue) {
  Vue.prototype._init = function (options) {
    const vm = this
    // 合并配置
    if (options && options._isComponent) {
      initInternalComponent(vm, options)
    } else {
      vm.$options = mergeOptions(
        resolveConstructorOptions(vm.constructor),
        options || {},
        vm
      )
    }

    // 2.render代理
    if (process.env.NODE_ENV !== 'production') {
      initProxy(vm)
    } else {
      vm._renderProxy = vm
    }

    // 初始化生命周期、初始化事件中心、初始化inject，初始化state、初始化provide、调用生命周期
    vm._self = vm
    initLifecycle(vm)
    initEvents(vm)
    initRender(vm)
    callHook(vm, 'beforeCreate')
    initInjections(vm)
    initState(vm)
    initProvide(vm)
    callHook(vm, 'created')

    // 挂载
    if (vm.$options.el) {
      vm.$mount(vm.$options.el)
    }
  }
}
```

#### stateMixin

> stateMixin 方法主要是处理跟实例相关的属性和方法，它会在 Vue.prototype 上定义实例会使用到的属性和方法；

```javascript
// src/core/instance/state.js
import { set, del } from '../observer/index'

export function stateMixin (Vue) {
  // 定义$data, $props, 这里只提供了get方法，set方法再非生产环境时会给予警告
  const dataDef = {}
  dataDef.get = function () { return this._data }
  const propsDef = {}
  propsDef.get = function () { return this._props }
  Object.defineProperty(Vue.prototype, '$data', dataDef)
  Object.defineProperty(Vue.prototype, '$props', propsDef)

  // 定义$set, $delete, $watch
  Vue.prototype.$set = set
  Vue.prototype.$delete = del
  // $watch方法实现的核心是通过一个 watcher 实例来监听
  Vue.prototype.$watch = function (
    expOrFn: string | Function,
    cb: any,
    options?: Object
  ): Function {
    const vm: Component = this
    if (isPlainObject(cb)) {
      return createWatcher(vm, expOrFn, cb, options)
    }
    options = options || {}
    options.user = true
    const watcher = new Watcher(vm, expOrFn, cb, options)
    if (options.immediate) {
      try {
        cb.call(vm, watcher.value)
      } catch (error) {
        handleError(error, vm, `callback for immediate watcher "${watcher.expression}"`)
      }
    }
    return function  () {
      watcher.teardownunwatchFn()
    }
  }
}
```

#### eventsMixin

> eventsMixin 方法主要是在 Vue.prototype 上定义下面四个实例方法；

```javascript
// src/core/instance/events.js
// 定义$on
Vue.prototype.$on = function (event: string | Array<string>, fn: Function): Component {
  const vm: Component = this
  if (Array.isArray(event)) {
    for (let i = 0, l = event.length; i < l; i++) {
      vm.$on(event[i], fn)
    }
  } else {
    (vm._events[event] || (vm._events[event] = [])).push(fn)
    if (hookRE.test(event)) {
      vm._hasHookEvent = true
    }
  }
  return vm
}

// 定义$once
Vue.prototype.$once = function (event: string, fn: Function): Component {
  const vm: Component = this
  function on () {
    vm.$off(event, on)
    fn.apply(vm, arguments)
  }
  on.fn = fn
  vm.$on(event, on)
  return vm
}

// 定义$off
Vue.prototype.$off = function (event?: string | Array<string>, fn?: Function): Component {
  const vm: Component = this
  // 没有提供任何参数时，移除全部事件监听
  if (!arguments.length) {
    vm._events = Object.create(null)
    return vm
  }
  
  // 只提供event参数时，只移除此event对应的监听器
  if (Array.isArray(event)) {
    for (let i = 0, l = event.length; i < l; i++) {
      vm.$off(event[i], fn)
    }
    return vm
  }
  
  // 传递了未监听的event
  const cbs = vm._events[event]
  if (!cbs) {
    return vm
  }
  // 没有传递fn
  if (!fn) {
    vm._events[event] = null
    return vm
  }
  // 同时提供event参数和fn回调，则只移除此event对应的fn这个监听器
  let cb
  let i = cbs.length
  while (i--) {
    cb = cbs[i]
    if (cb === fn || cb.fn === fn) {
      cbs.splice(i, 1)
      break
    }
  }
  return vm
}

// 定义$emit
Vue.prototype.$emit = function (event: string): Component {
  const vm: Component = this
  // ......
  let cbs = vm._events[event]
  if (cbs) {
    cbs = cbs.length > 1 ? toArray(cbs) : cbs
    const args = toArray(arguments, 1)
    const info = `event handler for "${event}"`
    for (let i = 0, l = cbs.length; i < l; i++) {
      // invokeWithErrorHandling 方法使用try/catch把函数调用并执行的地方包裹起来，当函数调用出错时，执行Vue的handleError()方法
      invokeWithErrorHandling(cbs[i], vm, args, vm, info)
    }
  }
  return vm
}

```

#### lifecycleMixin

> lifecycleMixin 方法主要是定义实例方法和生命周期；

- _ update 方法会在组件渲染时调用；
- $forceUpdate 方法是一个强制 Vue 实例重新渲染的方法，它的内部调用了 _update 方法，也就是强制组件重新编译挂载；
- $destory 方法是组件销毁方法，会处理父子组件的关系，事件监听和触发生命周期等操作；

```javascript
// src/core/instance/lifecycle.js
export function lifecycleMixin (Vue) {
  // 私有方法
  Vue.prototype._update = function () {}
  // 实例方法
  Vue.prototype.$forceUpdate = function () {
    if (this._watcher) {
      this._watcher.update()
    }
  }
  Vue.prototype.$destroy = function () {}
}
```

#### renderMixin

> renderMixin 方法主要是在 Vue.prototype 上定义各种私有方法和 $nextTick 方法；

```javascript
// src/core/instance/render.js
Vue.prototype._o = markOnce
Vue.prototype._n = toNumber
Vue.prototype._s = toString
Vue.prototype._l = renderList
Vue.prototype._t = renderSlot
Vue.prototype._q = looseEqual
Vue.prototype._i = looseIndexOf
Vue.prototype._m = renderStatic
Vue.prototype._f = resolveFilter
Vue.prototype._k = checkKeyCodes
Vue.prototype._b = bindObjectProps
Vue.prototype._v = createTextVNode
Vue.prototype._e = createEmptyVNode
Vue.prototype._u = resolveScopedSlots
Vue.prototype._g = bindObjectListeners

Vue.prototype.$nextTick = function() {}
// _render 方法会把模板编译成 VNode
Vue.prototype._render = function() {}
```

处理后的 Vue 的原型：

![img](https://user-gold-cdn.xitu.io/2018/4/25/162fad302d76df09?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

### Core

> Core 在 instance 的基础上，又对 Vue 的属性进行了一系列的操作；

```javascript
import Vue from './instance/index'
import { initGlobalAPI } from './global-api/index'
import { isServerRendering } from 'core/util/env'
import { 
	FunctionalRenderContext 
} from 'core/vdom/create-functional-component'

// 初始化全局API
initGlobalAPI(Vue)

Object.defineProperty(Vue.prototype, '$isServer', {
  get: isServerRendering
})

Object.defineProperty(Vue.prototype, '$ssrContext', {
  get () {
    /* istanbul ignore next */
    return this.$vnode && this.$vnode.ssrContext
  }
})

// expose FunctionalRenderContext for ssr runtime helper installation
Object.defineProperty(Vue, 'FunctionalRenderContext', {
  value: FunctionalRenderContext
})

Vue.version = '__VERSION__'

export default Vue
```

#### initGlobalAPI

```javascript
// Object.defineProperty(Vue, 'config', configDef)
Vue.config = { devtools: true, …}
Vue.util = {
	warn,
	extend,
	mergeOptions,
	defineReactive,
}
Vue.set = set
Vue.delete = delete
Vue.nextTick = nextTick
Vue.options = {
	components: {},
	directives: {},
	filters: {},
	_base: Vue,
}
// extend(Vue.options.components, builtInComponents)
Vue.options.components.KeepAlive = { name: 'keep-alive' …}
// initUse
Vue.use = function() {}
// initMixin
Vue.mixin = function() {}
// initExtend
Vue.cid = 0
Vue.extend = function() {}
// initAssetRegisters
Vue.component = function() {}
Vue.directive = function() {}
Vue.filter = function() {}
```

处理后的 Vue：

![img](https://user-gold-cdn.xitu.io/2018/4/25/162fad339a16328b?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

### runtime

1. 针对 web 平台，对 Vue.config 添加了一些方法；

   ```javascript
   Vue.config.mustUseProp = mustUseProp
   Vue.config.isReservedTag = isReservedTag
   Vue.config.isReservedAttr = isReservedAttr
   Vue.config.getTagNamespace = getTagNamespace
   Vue.config.isUnknownElement = isUnknownElement
   ```

2. 向 options 中的 directives 增加了 model 及 show 指令；

   ```javascript
   // extend(Vue.options.directives, platformDirectives)
   Vue.options.directives = {
   	model: { componentUpdated: ƒ …}
   	show: { bind: ƒ, update: ƒ, unbind: ƒ }
   }
   ```

3. 向 options 中的 components 增加了一些组件；

   ```javascript
   // extend(Vue.options.components, platformComponents)
   Vue.options.components = {
   	KeepAlive: { name: "keep-alive" …}
   	Transition: {name: "transition", props: {…} …}
   	TransitionGroup: {props: {…}, beforeMount: ƒ, …}
   }
   ```

4. 在原型中追加 _ patch _ 以及 $mount；

   ```javascript
   // 虚拟dom所用到的方法
   Vue.prototype.__patch__ = patch
   Vue.prototype.$mount = function() {}
   ```

5. 对 devtools 的支持；

### entry

1. 在 entry 中，覆盖了 $mount 方法；

2. 挂载 compile，compileToFunctions 方法就是将 template 编译为 render 函数；

   ```javascript
   Vue.compile = compileToFunctions
   ```

## 微观角度

### _init

1. 在当前实例中，添加 _ uid，_isVue 属性；
2. 在非生产环境时，使用 window.performance 标记 vue 初始化的开始；
3. 将 Vue.options 与传入的 options 进行合并；
4. 为当前实例添加 _ renderProxy，_ self 属性；
5. 初始化生命周期 initLifecycle；
6. 初始化render initRender；
7. 调用生命周期中的 beforeCreate；
8. 初始化注入值 initInjections；
9. 初始化状态 initState；
10. 调用生命周期的 created；
11. 非生产环境下，标识初始化结束，为当前实例增加 _name 属性；
12. 根据 options 传入的 el，调用当前实例的 $mount；

```javascript
Vue.prototype._init = function (options?: Object) {
  const vm: Component = this
  vm._uid = uid++

  let startTag, endTag
  // 浏览器环境&支持window.performance&非生产环境&配置了performance
  if (process.env.NODE_ENV !== 'production' 
      && config.performance && mark) {
    startTag = `vue-perf-start:${vm._uid}`
    endTag = `vue-perf-end:${vm._uid}`
    // 相当于 window.performance.mark(startTag)
    mark(startTag)
  }

  // a flag to avoid this being observed
  vm._isVue = true
  // 合并配置，区分component配置和constructor配置
  if (options && options._isComponent) {
    initInternalComponent(vm, options)
  } else {
    vm.$options = mergeOptions(
      resolveConstructorOptions(vm.constructor),
      options || {},
      vm
    )
  }
  // render的Prxoy代理,开发环境为 new Proxy() 实例，生产环境为自身
  if (process.env.NODE_ENV !== 'production') {
    // 对vm实例进行一层代理
    initProxy(vm)
  } else {
    vm._renderProxy = vm
  }
 
  vm._self = vm
  initLifecycle(vm)
  initEvents(vm)
  initRender(vm)
  callHook(vm, 'beforeCreate')
  initInjections(vm) 
  initState(vm)
  initProvide(vm)
  callHook(vm, 'created')

  if (process.env.NODE_ENV !== 'production' 
      && config.performance && mark) {
    vm._name = formatComponentName(vm, false)
    mark(endTag)
    measure(`vue ${vm._name} init`, startTag, endTag)
  }

  if (vm.$options.el) {
    vm.$mount(vm.$options.el)
  }
}
```

### initProxy

- 当浏览器支持 Proxy 时，vm._renderProxy 会代理 vm 实例，并且代理过程也会随着参数的不同呈现不同的效果；
- 当浏览器不支持Proxy 时，直接将 vm 赋值给 vm._renderProxy；

```javascript
var initProxy = function initProxy (vm) {
    // 判断浏览器是否支持原生的 proxy
    if (hasProxy) {
        var options = vm.$options;
        var handlers = options.render && options.render._withStripped
            ? getHandler
            : hasHandler;
        // 代理vm实例到vm属性_renderProxy
        vm._renderProxy = new Proxy(vm, handlers);
    } else {
        vm._renderProxy = vm;
    }
};
```

#### 支持proxy情况 

##### getHandler

> getHandler 方法主要是针对读取代理对象某个属性时进行的操作，当访问的属性不是 String 类型或者属性值在被代理的对象上不存在时，抛出错误提示，否则就返回属性值；

```javascript
const getHandler = {
  get (target, key) {
    if (typeof key === 'string' && !(key in target)) {
      warnNonPresent(target, key)
    }
    return target[key]
  }
}
```

##### hasHandler 

> hasHandler 方法可以在开发者错误的调用vm属性时，提供提示作用；

- allowedGlobals 定义了 javascript 保留的关键字，这些关键字是不允许作为用户变量存在的；
- warnReservedPrefix: 警告不能以$ _开头的变量；
- warnNonPresent: 警告模板出现的变量在vue实例中未定义；

```javascript
var hasHandler = {
    has: function has (target, key) {
        var has = key in target;
        // isAllowed用来判断模板上出现的变量是否合法。
        var isAllowed = allowedGlobals(key) ||
            (typeof key === 'string' && key.charAt(0) === '_' && !(key in target.$data));
        // _和$开头的变量不允许出现在定义的数据中，因为它是vue内部保留属性的开头
        // warnReservedPrefix: 警告不能以$ _开头的变量
        // warnNonPresent: 警告模板出现的变量在vue实例中未定义
        if (!has && !isAllowed) {
            if (key in target.$data) { warnReservedPrefix(target, key); }
            else { warnNonPresent(target, key); }
        }
        return has || !isAllowed
    }
};

// 模板中允许出现的非vue实例定义的变量
var allowedGlobals = makeMap(
    'Infinity,undefined,NaN,isFinite,isNaN,' +
    'parseFloat,parseInt,decodeURI,decodeURIComponent,encodeURI,encodeURIComponent,' +
    'Math,Number,Date,Array,Object,Boolean,String,RegExp,Map,Set,JSON,Intl,' +
    'require' // for Webpack/Browserify
);

// 模板使用未定义的变量
var warnNonPresent = function (target, key) {
    warn(
    "Property or method \"" + key + "\" is not defined on the instance but " +
    'referenced during render. Make sure that this property is reactive, ' +
    'either in the data option, or for class-based components, by ' +
    'initializing the property. ' +
    'See: https://vuejs.org/v2/guide/reactivity.html#Declaring-Reactive-Properties.',
    target
    );
};

// 使用$,_开头的变量
var warnReservedPrefix = function (target, key) {
    warn(
    "Property \"" + key + "\" must be accessed with \"$data." + key + "\" because " +
    'properties starting with "$" or "_" are not proxied in the Vue instance to ' +
    'prevent conflicts with Vue internals' +
    'See: https://vuejs.org/v2/api/#data',
    target
    );
};
```

#### 不支持proxy情况

- 在没有经过代理的情况下，使用 $,_ 开头的变量变成了 js 语言层面的错误，表示该变量没有被声明；

在初始化数据阶段，Vue 已经为数据进行了一层筛选的代理；

```javascript
function initData(vm) {
  vm._data = typeof data === 'function' ? getData(data, vm) : data || {}
  if (!isReserved(key)) {
      // 数据代理，用户可直接通过vm实例返回data数据
      proxy(vm, "_data", key);
  }
}

// isReserved会过滤以 $,_ 开头的变量
function isReserved (str) {
  var c = (str + '').charCodeAt(0);
  // 首字符是$, _的字符串
  return c === 0x24 || c === 0x5F
}
```

##### proxy

> proxy 方法的实现是基于 Object.defineProperty 来实现的；

```javascript
function proxy (target, sourceKey, key) {
    sharedPropertyDefinition.get = function proxyGetter () {
        // 当访问this[key]时，会代理访问this._data[key]的值
        return this[sourceKey][key]
    };
    sharedPropertyDefinition.set = function proxySetter (val) {
        this[sourceKey][key] = val;
    };
    Object.defineProperty(target, key, sharedPropertyDefinition);
}
```

### mergeOptions

> mergeOptions 方法用于处理 Vue 和当前组件实例的选项合并；

```javascript
// _init 方法中使用
vm.$options = mergeOptions(
	resolveConstructorOptions(vm.constructor),
	options || {},
	vm
)

function resolveConstructorOptions (Ctor: Class<Component>) {
  let options = Ctor.options
  if (Ctor.super) {
    const superOptions = resolveConstructorOptions(Ctor.super)
    const cachedSuperOptions = Ctor.superOptions
    if (superOptions !== cachedSuperOptions) {
      // super option changed,
      // need to resolve new options.
      Ctor.superOptions = superOptions
      // check if there are any late-modified/attached options 
      const modifiedOptions = resolveModifiedOptions(Ctor)
      // update base extend options
      if (modifiedOptions) {
        extend(Ctor.extendOptions, modifiedOptions)
      }
      options = Ctor.options = 
      		mergeOptions(superOptions, Ctor.extendOptions)
      if (options.name) {
        options.components[options.name] = Ctor
      }
    }
  }
  return options
}
```

在 runtime 中对 Vue 增加了一系统的配置、指令和组件：

```javascript
Vue.options = {
	components: {
   		KeepAlive: { name: "keep-alive" …}
    	Transition: {name: "transition", props: {…} …}
    	TransitionGroup: {props: {…}, beforeMount: ƒ, …}
	},
	directives: {
    	model: { componentUpdated: ƒ …}
		show: { bind: ƒ, update: ƒ, unbind: ƒ }
	},
	filters: {},
	_base: ƒ Vue
}
```

#### strats

> strats 是定义了一些标准合并策略，如果没有定义在其中，就使用默认合并策略 defaultStrat；

在引用 mergeOptions 方法时，会立即执行一段代码：

```javascript
const strats = config.optionMergeStrategies
strats.el = strats.propsData = function (parent, child, vm, key) {}
strats.data = function (parentVal, childVal, vm) {}
constants.LIFECYCLE_HOOKS.forEach(hook => strats[hook] = mergeHook)
constants.ASSET_TYPES.forEach(type => strats[type + 's'] = mergeAssets)
strats.watch = function(parentVal, childVal, vm, key) {}
strats.props = 
strats.methods = 
strats.inject = 
strats.computed = function(parentVal, childVal, vm, key) {}
strats.provide = mergeDataOrFn

// 默认合并策略
const defaultStrat = function (parentVal, childVal) {
  return childVal === undefined
    ? parentVal
    : childVal
}
```

后面还有一系列针对 strats 挂载方法和属性的操作，最终 strats 会变成：

![img](https://user-gold-cdn.xitu.io/2018/4/25/162fad3adc464ace?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

#### mergeOptions

```javascript
function mergeOptions (
  parent: Object,
  child: Object,
  vm?: Component
): Object {
    
  // 根据用户传入的options，检查合法性
  if (process.env.NODE_ENV !== 'production') {
    checkComponents(child)
  }

  if (typeof child === 'function') {
    child = child.options
  }
  // 标准化传入options中的props
  normalizeProps(child, vm)
  // 标准化注入
  normalizeInject(child, vm)
  // 标准化指令
  normalizeDirectives(child)
  const extendsFrom = child.extends
  if (extendsFrom) {
    parent = mergeOptions(parent, extendsFrom, vm)
  }
  if (child.mixins) {
    for (let i = 0, l = child.mixins.length; i < l; i++) {
      parent = mergeOptions(parent, child.mixins[i], vm)
    }
  }
  const options = {}
  let key
  for (key in parent) {
    mergeField(key)
  }
  // 循环子options时，仅合并父options中不存在的项，来提高合并效率
  for (key in child) {
    if (!hasOwn(parent, key)) {
      mergeField(key)
    }
  }
  // 合并策略
  function mergeField (key) {
    const strat = strats[key] || defaultStrat
    options[key] = strat(parent[key], child[key], vm, key)
  }
  return options
}
```

### initProxy

在合并选项后，判断如果是非生产环境时，会进入 initProxy 方法；

```javascript
if (process.env.NODE_ENV !== 'production') {
  initProxy(vm)
} else {
  vm._renderProxy = vm
}
vm._self = vm
```

### initLifecycle

> initLifecycle 方法初始化了与生命周期相关的属性；

```javascript
function initLifecycle (vm) {
  const options = vm.$options
  // .....
  vm.$parent = undefined
  vm.$root = vm

  vm.$children = []
  vm.$refs = {}

  vm._watcher = null
  vm._inactive = null
  vm._directInactive = false
  vm._isMounted = false
  vm._isDestroyed = false
  vm._isBeingDestroyed = false
}
```

### initEvents

```javascript
function initEvents (vm) {
  vm._events = Object.create(null)
  vm._hasHookEvent = false
  
  // ......
}
```

### initRender

```javascript
function initRender (vm: Component) {
  // the root of the child tree
  vm._vnode = null 
  // v-once cached trees
  vm._staticTrees = null 
  vm.$slots = {}
  vm.$scopedSlots = {}
  vm._c = (a, b, c, d) => createElement(vm, a, b, c, d, false)
  vm.$createElement= (a, b, c, d) => createElement(vm, a, b, c, d, true)
  vm.$attrs = {}
  vm.$listeners = {}
}
```

### initState

![img](https://user-gold-cdn.xitu.io/2018/4/25/162fad44310078ca?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

### vm.$mount

首先判断在 new Vue 中的 options 中是否传入 el，如果没有传入就不会触发实际的渲染，需要自己手动调用 $mount；

整个覆盖后的 $mount 方法就是将 template 转为 render 函数挂载至 vm 的 options，然后调用原有的 mount；

```javascript
// ......
// 先将原有的$mount保留到变量mount中，
const mount = Vue.prototype.$mount
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  el = el && query(el)

  const options = this.$options
  if (!options.render) {
    let template = getOuterHTML(el)
    if (template) {
      const { render, staticRenderFns } = compileToFunctions(template, {
        shouldDecodeNewlines,
        shouldDecodeNewlinesForHref,
        delimiters: options.delimiters,
        comments: options.comments
      }, this)
      options.render = render
      options.staticRenderFns = staticRenderFns
    }
  }
  return mount.call(this, el, hydrating)
}
```

原有的 $mount 方法主要是调用了生命周期中的 mountComponet；

```javascript
function mountComponent (
  vm: Component,
  el: ?Element,
  hydrating?: boolean
): Component {
  vm.$el = el
  callHook(vm, 'beforeMount')

  let updateComponent = () => {
  	vm._update(vm._render(), hydrating)
  }

  new Watcher(vm, updateComponent, noop, {
    before () {
      if (vm._isMounted) {
        callHook(vm, 'beforeUpdate')
      }
    }
  }, true /* isRenderWatcher */)
  hydrating = false

  if (vm.$vnode == null) {
    vm._isMounted = true
    callHook(vm, 'mounted')
  }
  return vm
}
```

Watcher 的构造函数：

```javascript
constructor (vm, expOrFn, cb, options, isRenderWatcher) {
  this.vm = vm
  vm._watcher = this
  vm._watchers.push(this)
  this.getter = expOrFn
  this.value = this.get()
}

get () {
  pushTarget(this)
  let value
  const vm = this.vm
  value = this.getter.call(vm, vm)
  popTarget()
  this.cleanupDeps()
  return value
}
```

1. 在 Watcher 的构造函数中，本次传入的 updateComponent 作为 Watcher 的 getter；
2. 在 get 方法调用时，又通过 pushTarget 方法将当前 Watcher 赋值给 Dep.target；
3. 调用 getter，相当于调用 vm._ update ，先调用 vm._ render，这时 vm._ render 此时会将已经准备好的 render 函数调用；
4. render 函数用到了 this.aaa，这时会调用 aaa 的 get 方法从而触发了 dep.depend（）；
5. dep.depend（）会调用 Watcher 的 addDep，这时 Watcher 记录当前的 dep 实例；
6. 继续调用 dep.addSub（this），dep 又记录了当前 Watcher 实例，将当前的 Watcher 存入 dep.subs 中；
7. 当 aaa 发生改变时，会触发 Observer 中的 set 方法，从而触发 dep,notify （）方法来进行 update 操作；