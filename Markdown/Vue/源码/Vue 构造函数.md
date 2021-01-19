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

在传入的 Vue 对象的原型上挂载了 _init 方法；

```javascript
// src/core/instance/init.js
Vue.prototype._init = function (options?: Object) {}
```

#### stateMixin

```javascript
// src/core/instance/state.js
// Object.defineProperty(Vue.prototype, '$data', dataDef)
// 这里$data只提供了get方法，set方法再非生产环境时会给予警告
Vue.prototype.$data = undefined;

// Object.defineProperty(Vue.prototype, '$props', propsDef)
// 这里$props只提供了get方法，set方法再非生产环境时会给予警告
Vue.prototype.$props = undefined;

Vue.prototype.$set = set
Vue.prototype.$delete = del

Vue.prototype.$watch = function() {}
```

#### eventsMixin

```javascript
// src/core/instance/events.js
Vue.prototype.$on = function() {}
Vue.prototype.$once = function() {}
Vue.prototype.$off = function() {}
Vue.prototype.$emit = function() {}
```

#### lifecycleMixin

```javascript
// src/core/instance/lifecycle.js
Vue.prototype._update = function() {}
Vue.prototype.$forceUpdate = function () {}
Vue.prototype.$destroy = function () {}
```

#### renderMixin

```javascript
// src/core/instance/render.js
// installRenderHelpers 
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
  // 合并选项
  if (options && options._isComponent) {
    initInternalComponent(vm, options)
  } else {
    // 将options进行合并
    vm.$options = mergeOptions(
      resolveConstructorOptions(vm.constructor),
      options || {},
      vm
    )
  }
  // 数据代理，可以通过 this.xxx 访问数据
  if (process.env.NODE_ENV !== 'production') {
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

```javascript
// ......
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



