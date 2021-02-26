> ref 最重要的作用是提供了一套 Ref 类型；

```javascript
// 生成一个唯一key，开发环境下增加描述符 'refSymbol'
export const refSymbol = Symbol(__DEV__ ? 'refSymbol' : undefined)

// 声明Ref接口
export interface Ref<T = any> {
  // 用此唯一key，来做Ref接口的一个描述符，让isRef函数做类型判断
  [refSymbol]: true
  // value值，存放真正的数据的地方。关于UnwrapNestedRefs这个类型，我后续单独解释
  value: UnwrapNestedRefs<T>
}

// 判断是否是Ref数据的方法
export function isRef(v: any): v is Ref {
  return v ? v[refSymbol] === true : false
}

// 见下文解释
export type UnwrapNestedRefs<T> = T extends Ref ? T : UnwrapRef<T>
```

