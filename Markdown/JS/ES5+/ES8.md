## async/await

## Object.values

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

