## async/await

1. Promise 可以解决回调地狱的问题，但是链式调用太多，也会变成另一种形式的回调地狱；
2. async/await 是 promise 的语法糖，实质上是对 generator 的封装；
3. async 声明一个异步函数；
   - 自动将常规函数转换成 Promise，返回一个 Promise 对象；
   - 只有 async 函数内部的异步操作执行完，才会执行 then 方法指定的回调函数；
   - 异步函数内部可使用 await；
   - 只要其中一个 Promise 对象变为 reject 状态，整个 async 函数都会中断操作；
4. await 暂停异步的功能执行；
   - 放在 Promise 调用之前，await 强制其它代码等待，直至 Promise 完成并返回结果；
   - 只能与 Promise 一起使用；
   - 只能在 async 函数内部使用，否则会报错；
   - ​

```javascript
async function myFetch() {
  let response = await fetch('coffee.jpg')
  return await response.blob()
}
```

### 原理

async/await 是 promise 的语法糖，实质上是对 generator 的封装，它们都提供了暂停执行的功能，区别有：

1. async/await自带执行器，不需要手动调用 next() 就能自动执行下一步；
2. async 函数返回 Promise 对象，而 Generator 返回生成器对象；
3. await 返回 Promise 的 resolve/reject 的值；

## Object.values

> 返回一个给定对象自身的所有可枚举属性值的数组；

其排列与使用 for...in 循环遍历该对象时返回的顺序一致，区别在于 for-in 循环还会枚举原型链中的属性；

```javascript
const object1 = {
      a: 'somestring',
      b: 42,
      c: false
}
console.log(Object.values(object1)) 
// ["somestring", 42, false]
```

## Object.entries

> 返回一个给定对象自身可枚举属性的键值对数组；

其排列与使用 for...in 循环遍历该对象时返回的顺序一致，区别在于 for-in 循环还会枚举原型链中的属性；

```javascript
const object1 = {
      a: 'somestring',
      b: 42
}

for (let [key, value] of Object.entries(object1)) {
      console.log(`${key}: ${value}`)
}

// "a: somestring"
// "b: 42"
```

## padStart

用另一个字符串从当前字符串的开始(左侧)填充当前字符串，以便产生的字符串达到给定的长度；

```javascript
const str1 = '5'
console.log(str1.padStart(2, '0')) // "05"

const fullNumber = '2034399002125581'
const last4Digits = fullNumber.slice(-4)
const maskedNumber = last4Digits.padStart(fullNumber.length, '*') 
console.log(maskedNumber) // "************5581"
```

## padEnd

> 方法会用一个字符串从当前字符串的末尾（右侧）开始填充当前字符串，返回填充后达到指定长度的字符串；

```javascript
const str1 = 'Breaded Mushrooms'
console.log(str1.padEnd(25, '.')) // "Breaded Mushrooms........"
const str2 = '200'
console.log(str2.padEnd(5)) // "200  "
```

## Object.getOwnPropertyDescriptors

> 用来获取一个对象的所有自身属性的描述符；

```javascript
const object1 = {
  property1: 42
}

const descriptors1 = Object.getOwnPropertyDescriptors(object1)

console.log(descriptors1.property1.writable) // true

console.log(descriptors1.property1.value) // 42

// 浅拷贝一个对象
Object.create(
  Object.getPrototypeOf(obj), 
  Object.getOwnPropertyDescriptors(obj) 
)

// 创建子类
function superclass() {}
superclass.prototype = {
  // 在这里定义方法和属性
}
function subclass() {}
subclass.prototype = Object.create(superclass.prototype, Object.getOwnPropertyDescriptors({
  // 在这里定义方法和属性
}))
```

