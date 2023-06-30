## effect

```javascript
  let activeEffect = null
    const targetMap = new WeakMap()
    export function effect(fn, options = {}) { 
        // effect嵌套，通过队列管理
        const effectFn = () => {
            try {
                activeEffect = effectFn
                //fn执行的时候，内部读取响应式数据的时候，就能在get配置里读取到activeEffect
                return fn()
            } finally {
                activeEffect = null 
            }
        }
        // 第二个参数options，传递lazy和scheduler来控制函数执行的时机
        if (!options.lazy) {
            /没有配置lazy 直接执行
            effectFn() // proxy实例对象发起拦截操作
        }
        effectFn.scheduler = options.scheduler // 调度时机 watchEffect会用到
        return effectFn
    }

    export function track(target, type, key) {
        let depsMap = targetMap.get(target) 
        if (!depsMap) { // 防止重复注册
            targetMap.set(target, (depsMap = new Map()))
        }
        let deps = depsMap.get(key)
        if (!deps) { // 防止重复注册
            deps = new Set() 
        }
        if (!deps.has(activeEffect) && activeEffect) { // 防止重复注册
            deps.add(activeEffect)
        }
        depsMap.set(key, deps)
    }

    export function trigger(target, type, key) {
        const depsMap =  targetMap.get(target)
        if (!depsMap) {
            return
        }
        const deps = depsMap.get(key)
        if (!deps) {
            return
        }
        deps.forEach((effectFn) => { // 挨个执行effect函数
            effectFn()
        })
    }

```

从上面可以看出：

- 定义的注册全局地图依赖是使用的 **WeakMap** 数据类型
  - WeakMap 数据类型的键名所引用的对象都是弱引用（垃圾回收机制不将该引用考虑在内），所以只要所引用 的对象的其它引用都被清除，垃圾回收机制就会释放该对象所占用的内存，可以理解为，一旦不再需要， WeakMap 里面的键名对象和所对应的键值对会自动消失，不需要手动删除引用 
- effect 传递的函数，可以通过传递 lazy 和 scheduler 来控制函数执行的时机，默认是同步执行
  - scheduler 存在的意义就是我们可以手动控制函数执行的时机，方便应对一些性能优化的场景，比如数据在一次交互中可能会被修改很多次，我们不想每次修改都重新执行依次 effect 函数，而是合并最终的状态之后，最后统一修改一次。
- track 函数的作用就是把effect 注册到依赖地图中，其中用 Set 数据类型存储 effect ，防止重复注册相同的 effect，无需额外的代码处理
- trigger 函数的作用就是把依赖地图中对应的 effect 函数数组 全部执行一遍