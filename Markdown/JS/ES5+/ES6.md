## Class

## 箭头函数

## 解构赋值

## 扩展操作符

## 模块化

## Promise

## Symbol

## Iterator/Generator

## Set/WeakSet

> Set 对象允许存储任何类型的唯一值，无论是原始值或引用对象；

WeakSet 结构与 Set 类似，主要区别是：

1. WeakSet 对象中只能存放对象引用，不能存放值，Set 对象都可以；
2. WeakSet 对象中的值都是被弱引用的，如果没有其它变量引用这个属性值，则这个对象会被当成垃圾回收掉；
3. WeakSet 对象是无法被枚举的，没有办法拿到它包含的所有元素；

## Map/WeakMap

> Map 对象保存键值对，对象或者原始值都可以作为一个键或一个值；
>
> WeakMap 对象是一组键/值对的集合，键是弱引用的，其键必须是对象，而值可以是任意的；

## Proxy/Reflect

> Proxy 对象用于定义基本操作的自定义行为（如属性查找，赋值，枚举，函数调用等）；
>
> Reflect 是一个内置的对象，它提供拦截 JS 操作的方法，这些方法与 Proxy 的方法相同；

1. Reflect 不是一个函数对象，因此它是不可构造的；

## Array扩展

### Array.prototype.from

> 转换具有 Iterator接口 的数据结构为真正数组，返回新数组；

```javascript
console.log(Array.from('foo')) // ["f", "o", "o"]
console.log(Array.from([1, 2, 3], x => x + x)) // [2, 4, 6]
```

### Array.prototype.of

> 转换一组值为真正数组，返回新数组；

```javascript
Array.of(7)       // [7] 
Array.of(1, 2, 3) // [1, 2, 3]

Array(7)          // [empty, empty, empty, empty, empty, empty]
Array(1, 2, 3)    // [1, 2, 3]
```

### Array.prototype.copyWithin

> 把指定位置的成员复制到其他位置，返回原数组；

```javascript
const array1 = ['a', 'b', 'c', 'd', 'e']

console.log(array1.copyWithin(0, 3, 4)) 
// ["d", "b", "c", "d", "e"]

console.log(array1.copyWithin(1, 3)) 
// ["d", "d", "e", "d", "e"]
```

### Array.prototype.find

> 返回第一个符合条件的成员；

```javascript
const array1 = [5, 12, 8, 130, 44]

const found = array1.find(element => element > 10)

console.log(found) // 12
```

### Array.prototype.findIndex

> 返回第一个符合条件的成员索引值；

```javascript
const array1 = [5, 12, 8, 130, 44]

const isLargeNumber = (element) => element > 13

console.log(array1.findIndex(isLargeNumber)) // 3
```

### Array.prototype.fill

> 根据指定值填充整个数组，返回原数组；

```javascript
const array1 = [1, 2, 3, 4]

console.log(array1.fill(0, 2, 4)) // [1, 2, 0, 0]

console.log(array1.fill(5, 1)) // [1, 5, 5, 5]

console.log(array1.fill(6)) // [6, 6, 6, 6]
```

### Array.prototype.keys

> 返回以索引值为遍历器的对象；

```javascript
const array1 = ['a', 'b', 'c']
const iterator = array1.keys()

for (const key of iterator) {
      console.log(key) // 0,1,2
}
```

### Array.prototype.values

> 返回以属性值为遍历器的对象；

```javascript
const array1 = ['a', 'b', 'c']
const iterator = array1.values()

for (const key of iterator) {
      console.log(key)  // a,b,c
}
```

### Array.prototype.entries

> 返回以索引值和属性值为遍历器的对象；

```javascript
const array1 = ['a', 'b', 'c']
const iterator = array1.entries()

console.log(iterator.next().value) // [0, "a"]
console.log(iterator.next().value) // [1, "b"]
```

