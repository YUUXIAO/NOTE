

目标对象标志：

```javascript
export const enum ReactiveFlags {
  SKIP = '__v_skip', 
  IS_REACTIVE = '__v_isReactive',
  IS_READONLY = '__v_isReadonly',
  RAW = '__v_raw'
}

export interface Target {
  [ReactiveFlags.SKIP]?: boolean   // 跳过，不对target作响应式处理
  [ReactiveFlags.IS_REACTIVE]?: boolean  // target是响应式的
  [ReactiveFlags.IS_READONLY]?: boolean  // target是只读的
  [ReactiveFlags.RAW]?: any  // target 对应的原始数据，未经过响应式代理
}
```



```javascript
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

1. 防止重复劫持；
2. 只读劫持；
3. 不同数据类型选择不同劫持方式；

```javascript
target: Target,  // 目标对象
isReadonly: boolean, // 是否只读
baseHandlers: ProxyHandler<any>,
collectionHandlers: ProxyHandler<any>,
proxyMap: WeakMap<Target, any>  // 全局proxyMap

function createReactiveObject(
  target: Target,
  isReadonly: boolean,
  baseHandlers: ProxyHandler<any>,
  collectionHandlers: ProxyHandler<any>,
  proxyMap: WeakMap<Target, any>
) {
  /**
   * ......
   * 数据类型判断、如果是只读/proxy数据 直接返回
   * 如果是已代理对象 直接返回
   * ......
   */
  
  // 创建代理对象
  // collectionHandlers：对引用类型的劫持
  // baseHandlers: 对基本类型的劫持
  const proxy = new Proxy(
    target,
    targetType === TargetType.COLLECTION ? collectionHandlers : baseHandlers
  )
  proxyMap.set(target, proxy)  // prxyMap 保存代理对象
  return proxy
}
```

### shallowReactive

创建一个响应式代理，该代理跟踪其自身 property 的响应性，但不执行嵌套对象的深度响应式转换

## baseHandlers

### createGetter

1. 如果是数组且 hasOwn(arrayInstrumentations, key) ，调用  arrayInstrumentations 获取值；
2. 调用 track；
3. 对象迭代 reactive；

```javascript
function createGetter(isReadonly = false, shallow = false) {
  return function get(target: Target, key: string | symbol, receiver: object) {
    if (key === ReactiveFlags.IS_REACTIVE) {
      return !isReadonly
    } else if (key === ReactiveFlags.IS_READONLY) {
      return isReadonly
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


    // 数组操作处理
    const targetIsArray = isArray(target)
    if (!isReadonly && targetIsArray && hasOwn(arrayInstrumentations, key)) {
      return Reflect.get(arrayInstrumentations, key, receiver)
    }


    // 非数组操作
    const res = Reflect.get(target, key, receiver)

    if (isSymbol(key) ? builtInSymbols.has(key) : isNonTrackableKeys(key)) {
      return res
    }

    // 如果可写，调用 track
    if (!isReadonly) {
      track(target, TrackOpTypes.GET, key)
    }

    if (shallow) {
      return res
    }

    if (isRef(res)) {
      const shouldUnwrap = !targetIsArray || !isIntegerKey(key)
      return shouldUnwrap ? res.value : res
    }

    // 是对象 递归      
    if (isObject(res)) {
      return isReadonly ? readonly(res) : reactive(res)
    }

    return res
  }
}
```

#### createArrayInstrumentations

### getters

```javascript
const get = /*#__PURE__*/ createGetter()
const shallowGet = /*#__PURE__*/ createGetter(false, true)
const readonlyGet = /*#__PURE__*/ createGetter(true)
const shallowReadonlyGet = /*#__PURE__*/ createGetter(true, true)
```

### createSetter

1. ​

```javascript
function createSetter(shallow = false) {
  return function set(
    target: object,
    key: string | symbol,
    value: unknown,
    receiver: object
  ): boolean {
    let oldValue = (target as any)[key]
    if (!shallow) {
      value = toRaw(value)
      oldValue = toRaw(oldValue)
      if (!isArray(target) && isRef(oldValue) && !isRef(value)) {
        oldValue.value = value
        return true
      }
    } else {
      // in shallow mode, objects are set as-is regardless of reactive or not
    }

    const hadKey =
      isArray(target) && isIntegerKey(key)
        ? Number(key) < target.length
        : hasOwn(target, key)
    const result = Reflect.set(target, key, value, receiver)
    // don't trigger if target is something up in the prototype chain of original
    if (target === toRaw(receiver)) {
      if (!hadKey) {
        // 数组 索引值大于数组长度 
        trigger(target, TriggerOpTypes.ADD, key, value)
      } else if (hasChanged(value, oldValue)) {
        // 修改对象
        trigger(target, TriggerOpTypes.SET, key, value, oldValue)
      }
    }
    return result
  }
}
```



### deleteProperty

1. deleteProperty 删除属性
2. 调用 trigger 方法

```javascript
function deleteProperty(target: object, key: string | symbol): boolean {
  const hadKey = hasOwn(target, key)
  const oldValue = (target as any)[key]
  const result = Reflect.deleteProperty(target, key)
  if (result && hadKey) {
    trigger(target, TriggerOpTypes.DELETE, key, undefined, oldValue)
  }
  return result
}
```

### has

> 判断对象是否有否属性

1.  Reflect.has 获取结果
2. 如果不是 Symbol 类型，触发 track 方法
3. 返回结果

```javascript
function has(target: object, key: string | symbol): boolean {
  const result = Reflect.has(target, key)
  if (!isSymbol(key) || !builtInSymbols.has(key)) {
    track(target, TrackOpTypes.HAS, key)
  }
  return result
}
```

### ownKeys

> 返回由对象自身的属性键组成的数组

```javascript
function ownKeys(target: object): (string | symbol)[] {
  track(target, TrackOpTypes.ITERATE, isArray(target) ? 'length' : ITERATE_KEY)
  return Reflect.ownKeys(target)
}
```



## baseHandlers

> handlers 是包含 Proxy 各种方法的常量 

### mutableHandlers

```javascript
export const mutableHandlers: ProxyHandler<object> = {
  get,
  set,
  deleteProperty,
  has,
  ownKeys
}
```

### readonlyHandlers

```javascript
export const readonlyHandlers: ProxyHandler<object> = {
  get: readonlyGet,  // createGetter(true)
  set(target, key) {
    if (__DEV__) {
      console.warn(
        `Set operation on key "${String(key)}" failed: target is readonly.`,
        target
      )
    }
    return true
  },
  deleteProperty(target, key) {
    if (__DEV__) {
      console.warn(
        `Delete operation on key "${String(key)}" failed: target is readonly.`,
        target
      )
    }
    return true
  }
}
```

### shallowReactiveHandlers

```javascript
export const shallowReactiveHandlers = /*#__PURE__*/ extend(
  {},
  mutableHandlers,
  {
    get: shallowGet,  //  createGetter(false, true)
    set: shallowSet  //  createSetter(true)
  }
)

```

### shallowReadonlyHandlers

```javascript
export const shallowReadonlyHandlers = /*#__PURE__*/ extend(
  {},
  readonlyHandlers,
  {
    get: shallowReadonlyGet  // createGetter(true, true)
  }
)
```

