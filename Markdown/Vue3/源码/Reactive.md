## reactive&readonly

两个方法主要逻辑都封装到了 createReactiveObject，主要作用是：

1. 透传给 createReactiveObject 相应的代理数据与响应数据的双向映射 map；
2. reactive 会做 readonly 的相关校验，反之 readonly 方法也是；

```typescript
// 函数类型声明，接受一个对象，返回不会深度嵌套的Ref数据
export function reactive<T extends object>(target: T): UnwrapNestedRefs<T>
// 函数实现
export function reactive(target: object) {
  // 如果传递的是一个只读响应式数据，则直接返回
  if (readonlyToRaw.has(target)) {
    return target
  }
  // 如果是被用户标记的只读数据，那通过readonly函数去封装
  if (readonlyValues.has(target)) {
    return readonly(target)
  }

  // 创建响应式对象数据
  return createReactiveObject(
    target, // 原始数据
    rawToReactive, // 原始数据 -> 响应式数据映射
    reactiveToRaw, // 响应式数据 -> 原始数据映射
    mutableHandlers, // 可变数据的代理劫持方法
    mutableCollectionHandlers // 可变集合数据的代理劫持方法
  )
}

// 函数声明+实现，接受一个对象，返回一个只读的响应式数据。
export function readonly<T extends object>(
  target: T
): Readonly<UnwrapNestedRefs<T>> {
  // 如果本身是响应式数据，获取其原始数据并将target入参赋值为原始数据
  if (reactiveToRaw.has(target)) {
    target = reactiveToRaw.get(target)
  }
  // 创建响应式数据
  return createReactiveObject(
    target,
    rawToReadonly,
    readonlyToRaw,
    readonlyHandlers,
    readonlyCollectionHandlers
  )
}
```

## createReactiveObject

> createReactiveObject 方法保存了代理的数据和原始数据，返回了 new Proxy 代理后的对象；

rawToReactive 和 reactiveToRaw 是两个弱引用的 Map 结构，这两个 Map 用来保存原始数据和可响应数据，toProxy  和 toRaw 传入的是这两个 Map；

```typescript
function createReactiveObject(
  target: any,
  toProxy: WeakMap<any, any>,
  toRaw: WeakMap<any, any>,
  baseHandlers: ProxyHandler<any>,
  collectionHandlers: ProxyHandler<any>
) {
  // 不是一个对象就直接返回原始数据，在开发环境下会打警告
  if (!isObject(target)) {
    if (__DEV__) {
      console.warn(`value cannot be made reactive: ${String(target)}`)
    }
    return target
  }
  // 通过原始数据 -> 响应数据的映射，获取响应数据
  let observed = toProxy.get(target)
  // 如果原始数据已经是响应式数据，则直接返回此响应数据
  if (observed !== void 0) {
    return observed
  }
  // 如果原始数据本身就是个响应数据了，直接返回自身
  if (toRaw.has(target)) {
    return target
  }
  // 如果是不可观察的对象，则直接返回原对象
  if (!canObserve(target)) {
    return target
  }
  // 集合数据与(对象/数组) 两种数据的代理处理方式不同。
  const handlers = collectionTypes.has(target.constructor)
    ? collectionHandlers
    : baseHandlers
  // 声明一个代理对象，也即是响应式数据
  observed = new Proxy(target, handlers)
  // 设置好原始数据与响应式数据的双向映射
  toProxy.set(target, observed)
  toRaw.set(observed, target)

  // 在这里用到了targetMap，但是它的value值存放什么我们依旧不知道
  if (!targetMap.has(target)) {
    targetMap.set(target, new Map())
  }
  return observed
}
```

## baseHandles

1. 对于原始对象数据，会通过 Proxy 劫持，返回新的响应式数据(代理数据)；
2. 对于代理数据的任何读写操作，都会通过 Refelct 反射到原始对象上；
3. 对于读操作会执行收集依赖的逻辑，对于写操作会触发监听函数的逻辑；

### get

> createGetter 接收一个参数：isReadonly，在 readonlyHandlers 中就是传入 true；

1. 用 Reflect，可方便的把现有操作行为原模原样地反射到目标对象上，又保证真实的作用域（通过第三个参数 receiver）；
2. 内置方法不需要收集依赖，因为调用内置方法时已经调用了一次 get 的 trap，没有必要重复收集依赖；
3. 属性值是对象时，需要延迟的使用 reactive|readonly 来避免循环依赖；

```typescript
// 入参只有一个是否只读
function createGetter(isReadonly: boolean) {
  // receiver即是被创建出来的代理对象
  return function get(target: any, key: string | symbol, receiver: any) {
    // 获取原始数据的相应值
    const res = Reflect.get(target, key, receiver)
    // 如果是js的内置方法，不做依赖收集
    if (isSymbol(key) && builtInSymbols.has(key)) {
      return res
    }
    // 如果是Ref类型数据，说明已经被收集过依赖，不做依赖收集，直接返回其value值。
    if (isRef(res)) {
      return res.value
    }
    // 收集依赖
    track(target, OperationTypes.GET, key)
    // 通过get获取的值不是对象的话，则直接返回即可,否则根据isReadyonly返回响应数据
    return isObject(res)
      ? isReadonly
        ? readonly(res)
        : reactive(res)
      : res
  }
```

### set

1. 数组的 push 的内部逻辑就是先给下标赋值，然后设置 length，触发了两次 set，但走到 length 逻辑时，获取老的 length 也已经是新的值了，所以由于 value === oldValue，实际只会走到一次 trigger；
2. 数组的  shift|unshift 以及 splice，会带来多次的 effect 触发；

```typescript
function set(
  target: any,
  key: string | symbol,
  value: any,
  receiver: any
): boolean {
  // 如果value是响应式数据，则返回其映射的源数据
  value = toRaw(value)
  // 获取旧值
  const oldValue = target[key]
  // 如果旧值是Ref数据，但新值不是，那更新旧的值的value属性值，返回更新成功
  if (isRef(oldValue) && !isRef(value)) {
    oldValue.value = value
    return true
  }
  // 代理对象中，是不是真的有这个key，没有说明操作是新增
  const hadKey = hasOwn(target, key)
  // 将本次设置行为，反射到原始对象上
  const result = Reflect.set(target, key, value, receiver)
  // 如果是原始数据原型链上的数据操作，不做任何触发监听函数的行为。
  if (target === toRaw(receiver)) {
    if (__DEV__) {
      // 开发环境下，会传给trigger一个扩展数据，包含了新旧值。明显的是便于开发环境下做一些调试。
      const extraInfo = { oldValue, newValue: value }
      // 如果不存在key时，说明是新增属性，操作类型为ADD
      // 存在key，则说明为更新操作，当新值与旧值不相等时，才是真正的更新，进而触发trigger
      if (!hadKey) {
        trigger(target, OperationTypes.ADD, key, extraInfo)
      } else if (value !== oldValue) {
        trigger(target, OperationTypes.SET, key, extraInfo)
      }
    } else {
      // 同上述逻辑，只是少了extraInfo
      if (!hadKey) {
        trigger(target, OperationTypes.ADD, key)
      } else if (value !== oldValue) {
        trigger(target, OperationTypes.SET, key)
      }
    }
  }
  return result
}

```

### 其它traps

```typescript
// 劫持属性删除
function deleteProperty(target: any, key: string | symbol): boolean {
  const hadKey = hasOwn(target, key)
  const oldValue = target[key]
  const result = Reflect.deleteProperty(target, key)
  if (result && hadKey) {
    /* istanbul ignore else */
    if (__DEV__) {
      trigger(target, OperationTypes.DELETE, key, { oldValue })
    } else {
      trigger(target, OperationTypes.DELETE, key)
    }
  }
  return result
}
// 劫持 in 操作符
function has(target: any, key: string | symbol): boolean {
  const result = Reflect.has(target, key)
  track(target, OperationTypes.HAS, key)
  return result
}
// 劫持 Object.keys
function ownKeys(target: any): (string | number | symbol)[] {
  track(target, OperationTypes.ITERATE)
  return Reflect.ownKeys(target)
}
```

## collectionHandlers

Set|Map｜WeakMap|WeakSet 这几个数据需要特殊处理，因为只要劫持了 set 或者直接引入 Reflect，反射行为到 target 上就会报错，因为 Map|Set 的内部存储数据必须通过 this 来访问，而通过代理对象去操作时，this 是 Proxy，于是无法访问其内部数据；

1. 由于 Set|Map 等集合数据的底层设计问题，Proxy 无法直接劫持 set 或直接反射行为；
2. 劫持原始集合数据的 get，对于它的原始方法或属性，Reflect 反射到插桩器上，否则反射原始对象；
3. 插装器上的方法，会先通过 toRaw 获取代理数据的原始数据，再获取原始数据的原型方法，然后绑定 this 为原始数据，调取相应方法；
4. 对于 getter|has 这类查询方法，插入收集依赖的逻辑，并将返回值转为响应式数据（has 返回 boolean 值故不需要转换）；
5. 对于迭代器相关的查询方法，插入收集依赖逻辑，并将迭代过程的数据转为响应式数据；
6. 对于写操作相关方法，插入触发监听的逻辑；

### 插桩

> 插桩指的是向某个方法注入一段有其它作用的代码，目的就是为了劫持这些方法，增加相应逻辑；

```typescript
// proxy handlers
export const mutableCollectionHandlers: ProxyHandler<any> = {
  // 创建一个插桩getter
  get: createInstrumentationGetter(mutableInstrumentations)
}
```

#### mutableInstrumentations

1. 创建了一个新的对象，它具有 Set|Map 一样的方法名；
2. 这些方法名对应的就是插桩后，注入了依赖收集跟响应触发的方法；
3. 然后通过 Reflect 反射到这个插桩对象上，获取的是插桩后的数据，调用的是插桩后的方法；

```typescript
// 可变数据插桩对象，以及一系列相应的插桩方法
const mutableInstrumentations: any = {
  get(key: any) {
    return get(this, key, toReactive)
  },
  get size() {
    return size(this)
  },
  has,
  add,
  set,
  delete: deleteEntry,
  clear,
  forEach: createForEach(false)
}
// 迭代器相关的方法
const iteratorMethods = ['keys', 'values', 'entries', Symbol.iterator]
iteratorMethods.forEach(method => {
  mutableInstrumentations[method] = createIterableMethod(method, false)
  readonlyInstrumentations[method] = createIterableMethod(method, true)
})
// 创建getter的函数
function createInstrumentationGetter(instrumentations: any) {
  // 返回一个被插桩后的get
  return function getInstrumented(
    target: any,
    key: string | symbol,
    receiver: any
  ) {
    // 如果有插桩对象中有此key，且目标对象也有此key，
    // 那就用这个插桩对象做反射get的对象，否则用原始对象
    target =
      hasOwn(instrumentations, key) && key in target ? instrumentations : target
    return Reflect.get(target, key, receiver)
  }
}
```

### 插桩读操作

> 本质上就是通过原始数据的原型方法 + call this，返回真正的数据；

- 在 get 方法中，第一个入参 target 其实是被代理后的数据，也就是 Reflect.get(target, key, receiver) 中的 receiver；

```typescript
const mutableInstrumentations: any = {
  get(key: any) {
    // this 即是调用get的对象，现实情况就是Proxy代理对象
    // toReactive是一个将数据转为响应式数据的方法
    return get(this, key, toReactive)
  }
  // ...省略其他
}
function get(target: any, key: any, wrap: (t: any) => any): any {
  // 获取原始数据
  target = toRaw(target)
  // 由于Map可以用对象做key，所以key也有可能是个响应式数据，先转为原始数据
  key = toRaw(key)
  // 获取原始数据的原型对象
  const proto: any = Reflect.getPrototypeOf(target)
  // 收集依赖
  track(target, OperationTypes.GET, key)
  // 使用原型方法，通过原始数据去获得该key的值。
  const res = proto.get.call(target, key)
  // wrap 即传入的toReceive方法，将获取的value值转为响应式数据
  return wrap(res)
}
```

- size 是一个属性不是方法，所以需要以 get size() 的方式去劫持；
- has 是个方法，不需要专门绑定 this；

```typescript
const mutableInstrumentations: any = {
  // ...
  get size() {
    return size(this)
  },
  has
  // ...
}
function size(target: any) {
  // 获取原始数据
  target = toRaw(target)
  const proto = Reflect.getPrototypeOf(target)
  track(target, OperationTypes.ITERATE)
  return Reflect.get(proto, 'size', target)
}

function has(this: any, key: any): boolean {
  // 获取原始数据
  const target = toRaw(this)
  key = toRaw(key)
  const proto: any = Reflect.getPrototypeOf(target)
  track(target, OperationTypes.HAS, key)
  return proto.has.call(target, key)
}

```

### 插桩迭代器

核心就是劫持迭代器方法，将每次 next 返回的 value 用 reactive 转化；

```typescript
// 迭代器相关的方法
const iteratorMethods = ['keys', 'values', 'entries', Symbol.iterator]
iteratorMethods.forEach(method => {
  mutableInstrumentations[method] = createIterableMethod(method, false)
})
function createIterableMethod(method: string | symbol, isReadonly: boolean) {
  return function(this: any, ...args: any[]) {
    // 获取原始数据
    const target = toRaw(this)
    // 获取原型
    const proto: any = Reflect.getPrototypeOf(target)
    // 如果是entries方法，或者是map的迭代方法的话，isPair为true
    // 这种情况下，迭代器方法的返回的是一个[key, value]的结构
    const isPair =
      method === 'entries' ||
      (method === Symbol.iterator && target instanceof Map)
    // 调用原型链上的相应迭代器方法
    const innerIterator = proto[method].apply(target, args)
    // 获取相应的转成响应数据的方法
    const wrap = isReadonly ? toReadonly : toReactive
    // 收集依赖
    track(target, OperationTypes.ITERATE)
    // return a wrapped iterator which returns observed versions of the
    // values emitted from the real iterator
    // 给返回的innerIterator插桩，将其value值转为响应式数据
    return {
      // iterator protocol
      next() {
        const { value, done } = innerIterator.next()
        return done
          ? // 为done的时候，value是最后一个值的next，是undefined，没必要做响应式转换了
            { value, done }
          : {
              value: isPair ? [wrap(value[0]), wrap(value[1])] : wrap(value),
              done
            }
      },
      // iterable protocol
      [Symbol.iterator]() {
        return this
      }
    }
  }
}

```

forEach 的也是劫持了方法，将原本的传参数据转为响应式数据后返回；

```typescript
function createForEach(isReadonly: boolean) {
  // 这个this，我们已经知道了是假参数，也就是forEach的调用者
  return function forEach(this: any, callback: Function, thisArg?: any) {
    const observed = this
    const target = toRaw(observed)
    const proto: any = Reflect.getPrototypeOf(target)
    const wrap = isReadonly ? toReadonly : toReactive
    track(target, OperationTypes.ITERATE)
    // important: create sure the callback is
    // 1. invoked with the reactive map as `this` and 3rd arg
    // 2. the value received should be a corresponding reactive/readonly.
    // 将传递进来的callback方法插桩，让传入callback的数据，转为响应式数据
    function wrappedCallback(value: any, key: any) {
      // forEach使用的数据，转为响应式数据
      return callback.call(observed, wrap(value), wrap(key), observed)
    }
    return proto.forEach.call(target, wrappedCallback, thisArg)
  }
}

```

### 插桩写操作

```typescript
function add(this: any, value: any) {
  // 获取原始数据
  value = toRaw(value)
  const target = toRaw(this)
  // 获取原型
  const proto: any = Reflect.getPrototypeOf(this)
  // 通过原型方法，判断是否有这个key
  const hadKey = proto.has.call(target, value)
  // 通过原型方法，增加这个key
  const result = proto.add.call(target, value)
  // 原本没有key的话，说明真的是新增，则触发监听响应逻辑
  if (!hadKey) {
    /* istanbul ignore else */
    if (__DEV__) {
      trigger(target, OperationTypes.ADD, value, { value })
    } else {
      trigger(target, OperationTypes.ADD, value)
    }
  }
  return result
}
```

