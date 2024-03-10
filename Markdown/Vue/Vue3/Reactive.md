## reactive方法

reactive方法是vue3内部用来创建响应式数据的入口方法，实现很简单：

- 区分只读数据类型返回；
- 否则调用 createReactiveObject 方法创建响应式数据

```typescript
export function reactive(target: object) {
  // if trying to observe a readonly proxy, return the readonly version.
  if (target && (target as Target)[ReactiveFlags.IS_READONLY]) {
    return target
  }
  return createReactiveObject(
    target,
    false,
    mutableHandlers, 
    mutableCollectionHandlers,
    reactiveMap
  )
}
```

## createReactiveObject

createReactiveObject 方法保存了**代理的数据**和**原始数据**，返回 new Proxy 代理后的对象；

1. **防止重复劫持代理：**数据类型判断、是只读/proxy数据就直接返回，已经是代理对象就直接返回代理对象
2. **区分对象和数据类型选择不同的方式代理**；
3. rawToReactive 和 reactiveToRaw 是两个weakMap 结构，这两个 Map 用来保存原始数据和可响应数据，toProxy  和 toRaw就是各自用来处理这个数据的映射表；

```typescript
function createReactiveObject(
  target: Target,
  isReadonly: boolean,
  baseHandlers: ProxyHandler<any>, // 为普通对象创建proxy是的第二个参数handers
  collectionHandlers: ProxyHandler<any>, // 为collection（map、set）类型对象创建proxy时的第二个参数handler    
  proxyMap: WeakMap<Target, any> // 全局用来记录 target和它proxy之间的映射
) {
  if (!isObject(target)) {
    if (__DEV__) {
      console.warn(`value cannot be made reactive: ${String(target)}`)
    }
    return target
  }
  // target is already a Proxy, return it.
  // exception: calling readonly() on a reactive object
  if (
    target[ReactiveFlags.RAW] &&
    !(isReadonly && target[ReactiveFlags.IS_REACTIVE])
  ) {
    return target
  }
  // 已经是proxy代理对象了
  const existingProxy = proxyMap.get(target)
  if (existingProxy) {
    return existingProxy
  }
  // only a whitelist of value types can be observed.
  const targetType = getTargetType(target)
  if (targetType === TargetType.INVALID) {
    return target
  }
  const proxy = new Proxy(
    target,
    targetType === TargetType.COLLECTION ? collectionHandlers : baseHandlers
  )
  proxyMap.set(target, proxy)
  return proxy
}


// 类型说明一下
function targetTypeMap(rawType: string) {
  switch (rawType) {
    case 'Object':
    case 'Array':
      return TargetType.COMMON
    case 'Map':
    case 'Set':
    case 'WeakMap':
    case 'WeakSet':
      return TargetType.COLLECTION
    default:
      return TargetType.INVALID
  }
}
```

## baseHandles

baseHandles 为普通对象创建proxy时的第二个参数handlers的一些操作方法；

```javascript
// baseHandles主要包含4种handler，
// mutableHandlers、readonlyHandlers、shallowReactiveHandlers、 shallowReadonlyHandlers
export const mutableHandlers: ProxyHandler<object> = {
  get,
  set,
  deleteProperty,
  has,
  ownKeys
}

```

### createGetter方法

1. 判断是否是特定的一些值处理掉
2. 区分数组和对象，如果是数组hasOwn(arrayInstrumentations, key) ，调用  arrayInstrumentations 获取值

   - 数组的查找方法：includes、indexOf、lastIndexOf
   - 修改数组的方法：push、pop、unshift、shift、splice

3. 浅响应式的对象就直接返回
4. 非只读对象就收集依赖

```typescript
function createGetter(isReadonly = false, shallow = false) {
  return function get(target: Target, key: string | symbol, receiver: object) {
    // 判断返回一些特定的值 例如 是 readonly 的就返回 readonlyMap，是 reactive 的就返回 reactiveMap 等等
    if (key === ReactiveFlags.IS_REACTIVE) {
      return !isReadonly
    } else if (key === ReactiveFlags.IS_READONLY) {
      return isReadonly
    } else if (key === ReactiveFlags.IS_SHALLOW) {
      return shallow
    } else if (
      key === ReactiveFlags.RAW &&
      receiver ===
        (isReadonly
          ? shallow
            ? shallowReadonlyMap
            : readonlyMap
          : shallow
          ? shallowReactiveMap
          : reactiveMap
        ).get(target)
    ) {
      return target
    }
 
    // 如果是数组要进行一些特殊处理
    const targetIsArray = isArray(target)
    if (!isReadonly && targetIsArray && hasOwn(arrayInstrumentations, key)) {
      //  重写数组的方法
      return Reflect.get(arrayInstrumentations, key, receiver)
    }
 
    // 非数组操作
    const res = Reflect.get(target, key, receiver)
    if (isSymbol(key) ? builtInSymbols.has(key) : isNonTrackableKeys(key)) {
      return res
    }
 
    // 判断不是只读属性 才进行依赖收集
    if (!isReadonly) {
      track(target, TrackOpTypes.GET, key)
    }
 
    // 浅层响应则直接返回对应的值，shallowRelative 
    if (shallow) {
      return res
    }
 
 
    // 如果是ref 则自动进行脱ref.value
    if (isRef(res)) {
      // ref unwrapping - does not apply for Array + integer key.
      const shouldUnwrap = !targetIsArray || !isIntegerKey(key)
      return shouldUnwrap ? res.value : res
    }
 
    // 返回值是对象,如果是只读就用 readonly 包裹返回数据
    // 否则则进行递归深层包裹 reactive 返回 Proxy 代理对象
    if (isObject(res)) {
      return isReadonly ? readonly(res) : reactive(res)
    }
 
    // 如都不是上面的判断 则返回这个数据
    return res
  }
}
```

### createSetter方法

1. 取出oldVlaue和即将set的新值 newValue；
2. 如果是 ref 类型直接ref.value更新
3. 根据 oldValue 和 newValue 判断是新增和修改属性，触发dep里的函数；

   - 比如数组的 push 的内部逻辑就是先给下标赋值再设置 length，触发了两次 set，但是第二次触发时数据已跟新，所以由于 value === oldValue，实际只会走到一次 trigger；
   - 数组的  shift|unshift 以及 splice，会带来多次的 effect 触发；


```typescript
function createSetter(shallow = false) {
  return function set( 
    target: object,
    key: string | symbol,
    value: unknown,
    receiver: object
  ): boolean {
    // 缓存旧值
    let oldValue = (target as any)[key]
 
    if (!shallow && !isReadonly(value)) {
      if (!isShallow(value)) {
        value = toRaw(value)
        oldValue = toRaw(oldValue)
      }
 
      // 若是 ref 并且非只读 则直接修改 ref的值
      if (!isArray(target) && isRef(oldValue) && !isRef(value)) {
        oldValue.value = value
        return true
      }
    } else {
      // in shallow mode, objects are set as-is regardless of reactive or not
    }
 
    // 判断是否有对应的key，这里判断了数组的索引取值
    const hadKey =
      isArray(target) && isIntegerKey(key)
        ? Number(key) < target.length
        : hasOwn(target, key)
 
    // 修改对应的值
    const result = Reflect.set(target, key, value, receiver)
    // don't trigger if target is something up in the prototype chain of original
 
    // 若目标是原型链上的内容就不触发依赖
    if (target === toRaw(receiver)) {
 
      // 这里主要是判断是 新增属性 还是修改属性的操作
      if (!hadKey) {
        trigger(target, TriggerOpTypes.ADD, key, value)
      } else if (hasChanged(value, oldValue)) {
        trigger(target, TriggerOpTypes.SET, key, value, oldValue)
      }
    }
 
    // 最终返回结果
    return result
  }
}

```

### deleteProperty & has & in

```typescript
// 劫持属性删除
function deleteProperty(target: object, key: string | symbol): boolean {
  const hadKey = hasOwn(target, key)
  const oldValue = (target as any)[key]
  const result = Reflect.deleteProperty(target, key)
  if (result && hadKey) {
    trigger(target, TriggerOpTypes.DELETE, key, undefined, oldValue)
  }
  return result
}

// 劫持 in 操作符
function has(target: object, key: string | symbol): boolean {
  const result = Reflect.has(target, key)
  if (!isSymbol(key) || !builtInSymbols.has(key)) {
    track(target, TrackOpTypes.HAS, key)
  }
  return result
}
// 劫持 Object.keys
function ownKeys(target: any): (string | number | symbol)[] {
  track(target, OperationTypes.ITERATE)
  return Reflect.ownKeys(target)
}
```

## collectionHandlers

collectionHandlers 为特殊类型（**Set/WeakSet、Map/WeakMap** ）创建proxy是的第二个参数handles：

- 由于 Set、Map 等集合数据的底层设计问题，Proxy 无法直接劫持 set 或直接反射行为
- 只要劫持了 set 或者直接引入 Reflect，反射行为到 target 上就会报错，因为 **Map或Set 的内部存储数据必须通过 this 来访问，而通过代理对象去操作时，this 是 Proxy**，于是无法访问其内部数据；

```javascript
export const mutableCollectionHandlers: ProxyHandler<CollectionTypes> = {
  get: createInstrumentationGetter(false, false)
}
 
export const shallowCollectionHandlers: ProxyHandler<CollectionTypes> = {
  get: createInstrumentationGetter(false, false)(false, true)
}
 
export const readonlyCollectionHandlers: ProxyHandler<CollectionTypes> = {
  get: createInstrumentationGetter(true, false)
}
```

### createInstrumentationGetter方法

```javascript
function createInstrumentationGetter(isReadonly: boolean, shallow: boolean) {
  const instrumentations = shallow
    ? shallowInstrumentations
    : isReadonly
      ? readonlyInstrumentations
      : mutableInstrumentations
 
  return (
    target: CollectionTypes,
    key: string | symbol,
    receiver: CollectionTypes
  ) => {
    if (key === ReactiveFlags.isReactive) {
      return !isReadonly
    } else if (key === ReactiveFlags.isReadonly) {
      return isReadonly
    } else if (key === ReactiveFlags.raw) {
      return target
    }
 
    return Reflect.get(
      hasOwn(instrumentations, key) && key in target
        ? instrumentations
        : target,
      key,
      receiver
    )
  }
}
```

### mutableInstrumentations方法

这个方法主要是用来解决**代理对象无法访问集合类型的属性和方法的问题，对集合的增删改查和迭代方法做了一些代理**

1. 创建了一个新的对象，它具有 Set|Map 一样的方法名；
2. 这些方法名对应的就是插桩后，注入了依赖收集跟响应触发的方法；
3. 然后通过 Reflect 反射到这个插桩对象上，获取的是插桩后的数据，调用的是插桩后的方法；

```javascript
const mutableInstrumentations: Record<string, Function> = {
  get(this: MapTypes, key: unknown) {
    return get(this, key, toReactive)
  },
  get size() {
    return size((this as unknown) as IterableCollections)
  },
  has,
  add,
  set,
  delete: deleteEntry,
  clear,
  forEach: createForEach(false, false)
}
const iteratorMethods = ['keys', 'values', 'entries', Symbol.iterator]
iteratorMethods.forEach(method => {
  mutableInstrumentations[method as string] = createIterableMethod(
    method,
    false,
    false
  )
  readonlyInstrumentations[method as string] = createIterableMethod(
    method,
    true,
    false
  )
  shallowInstrumentations[method as string] = createIterableMethod(
    method,
    true,
    true
  )
})
```

#### get方法

> 本质上就是通过原始数据的原型方法 + call（this），返回真正的数据；

1. 先对target和key进行toRaw获取原始数据，因为Map对象可以用对象做key，所以也是要toRaw
2. 如果key值不相等，说明key是reactive对象，要对key 和rawkey进行track
3. 调用原型上的has方法，如果为true，就把值转换为响应式数据

```typescript
function get(
  target: MapTypes,
  key: unknown,
  wrap: typeof toReactive | typeof toReadonly | typeof toShallow
) {
  target = toRaw(target) // 获取原始数据
  const rawKey = toRaw(key) // 由于Map可以用对象做key，所以key也有可能是个响应式数据，先转为原始数据
  if (key !== rawKey) {
    track(target, TrackOpTypes.GET, key)
  }
  track(target, TrackOpTypes.GET, rawKey)
  const { has, get } = getProto(target) // 获取原始数据的原型对象
  if (has.call(target, key)) {
    return wrap(get.call(target, key)) // 将获取的value值转为响应式数据
  } else if (has.call(target, rawKey)) {
    return wrap(get.call(target, rawKey)) // 将获取的value值转为响应式数据
  }
}
```

#### set方法

set方法主要是注意Map类型的数据：

1. Map上有没有同key的数据，来判断是要新增还是更新

```javascript
function set(this: MapTypes, key: unknown, value: unknown) {
  value = toRaw(value)
  const target = toRaw(this)
  const { has, get, set } = getProto(target)
 
  let hadKey = has.call(target, key)
  if (!hadKey) {
    key = toRaw(key)
    hadKey = has.call(target, key)
  } else if (__DEV__) {
    checkIdentityKeys(target, has, key) // 开发环境检查目标对象是不是同时存在rawkey和key，这样会数据不一致
  }
 
  const oldValue = get.call(target, key)
  const result = set.call(target, key, value)
  if (!hadKey) {
    trigger(target, TriggerOpTypes.ADD, key, value)
  } else if (hasChanged(value, oldValue)) {
    trigger(target, TriggerOpTypes.SET, key, value, oldValue)
  }
  return result
}
```

#### forEach方法

在调用原型上的 forEach 进行循环的时候，会对 key 和 value 都进行一层数据处理（就是 reactive）

```javascript
function createForEach(isReadonly: boolean, shallow: boolean) {
  return function forEach(
    this: IterableCollections,
    callback: Function,
    thisArg?: unknown
  ) {
    const observed = this
    const target = toRaw(observed)
    const wrap = isReadonly ? toReadonly : shallow ? toShallow : toReactive
    !isReadonly && track(target, TrackOpTypes.ITERATE, ITERATE_KEY)
    // important: create sure the callback is
    // 1. invoked with the reactive map as `this` and 3rd arg
    // 2. the value received should be a corresponding reactive/readonly.
    function wrappedCallback(value: unknown, key: unknown) {
      return callback.call(thisArg, wrap(value), wrap(key), observed)
    }
    return getProto(target).forEach.call(target, wrappedCallback)
  }
}
```

createIterableMethod方法 主要是对集合中的迭代进行代理，['keys', 'values', 'entries', Symbol.iterator] 主要是这四个方法

```javascript
const iteratorMethods = ['keys', 'values', 'entries', Symbol.iterator]
iteratorMethods.forEach(method => {
  mutableInstrumentations[method as string] = createIterableMethod(
    method,
    false,
    false
  )
  readonlyInstrumentations[method as string] = createIterableMethod(
    method,
    true,
    false
  )
  shallowInstrumentations[method as string] = createIterableMethod(
    method,
    true,
    true
  )
})

function createIterableMethod(
  method: string | symbol,
  isReadonly: boolean,
  shallow: boolean
) {
  return function(this: IterableCollections, ...args: unknown[]) {
    const target = toRaw(this)
    const isMap = target instanceof Map
    const isPair = method === 'entries' || (method === Symbol.iterator && isMap)
    const isKeyOnly = method === 'keys' && isMap
    const innerIterator = getProto(target)[method].apply(target, args)
    const wrap = isReadonly ? toReadonly : shallow ? toShallow : toReactive
    !isReadonly &&
      track(
        target,
        TrackOpTypes.ITERATE,
        isKeyOnly ? MAP_KEY_ITERATE_KEY : ITERATE_KEY
      )
    // return a wrapped iterator which returns observed versions of the
    // values emitted from the real iterator
    return {
      // iterator protocol
      next() {
        const { value, done } = innerIterator.next()
        return done
          ? { value, done }
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



#### has & size & add方法

```javascript
function has(this: CollectionTypes, key: unknown): boolean {
  const target = toRaw(this)
  const rawKey = toRaw(key)
  if (key !== rawKey) {
    track(target, TrackOpTypes.HAS, key)
  }
  track(target, TrackOpTypes.HAS, rawKey)
  const has = getProto(target).has
  return has.call(target, key) || has.call(target, rawKey)
}

function size(target: IterableCollections) {
  target = toRaw(target)
  track(target, TrackOpTypes.ITERATE, ITERATE_KEY)
  return Reflect.get(getProto(target), 'size', target)
}


function add(this: SetTypes, value: unknown) {
  value = toRaw(value)
  const target = toRaw(this)
  const proto = getProto(target)
  const hadKey = proto.has.call(target, value)
  const result = proto.add.call(target, value)
  // 需要判断是否已存在
  if (!hadKey) {
    trigger(target, TriggerOpTypes.ADD, value, value)
  }
  return result
}
```

## ref数据

上面大概梳理了一下reactive对象的处理方式，ref原理差不多，下面先写一下

### createRef方法

```javascript
export function ref(value?: unknown) {
  return createRef(value, false)
}
 
function createRef(rawValue: unknown, shallow: boolean) {
  // 如果已经是一个ref 则直接返回
  if (isRef(rawValue)) {
    return rawValue
  }
  
  // 实例化 class
  return new RefImpl(rawValue, shallow)
}
 
class RefImpl<T> {
  private _value: T
  private _rawValue: T
 
  public dep?: Dep = undefined。// 收集当前ref的依赖
  
  // 用于区分 ref 的不可枚举属性 例如 isRef 方法就是直接判断这个属性
  public readonly __v_isRef = true
 
  // 构造函数
  constructor(value: T, public readonly __v_isShallow: boolean) {
    this._rawValue = __v_isShallow ? value : toRaw(value)
    this._value = __v_isShallow ? value : toReactive(value)
  }
 
  get value() {
    trackRefValue(this) // 收集依赖，和reactive处理方式查不多
    return this._value
  }
 
  set value(newVal) {
    newVal = this.__v_isShallow ? newVal : toRaw(newVal)
    
    // 判断是否有变化 如有才进行更新
    if (hasChanged(newVal, this._rawValue)) {
      this._rawValue = newVal
      this._value = this.__v_isShallow ? newVal : toReactive(newVal)
      // 依赖更新
      triggerRefValue(this, newVal)
    }
  }
}

```

