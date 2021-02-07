## 语法

> reduce（）方法对数组中的每个元素执行一个由我们提供的 reducer（升序执行），将其结果汇总为单个返回值；

```javascript
arr.reduce(callback(accumulator, currentValue[, index[, array]])[, initialValue])
```

reduce 接收2个参数：第一个是回调函数（必选），第二个是初始值（可选）；

第一个参数（回调函数）接收四个参数：

- Accumulator (累计器)；
- Current Value (当前值)；
- Current Index  (当前索引)；
- Source Array (源数组)

第二个参数 initialValue  可以是任意类型；

- 如果没有提供 initialValue，reduce 会从索引 1 的地方开始执行 callback，跳过第1个索引；
- 如果提供 initialValue，从索引 0 开始；

## 转化

### map

> map 方法接收一个回调函数，函数接收三个参数：当前项，索引和原数组，返回一个新数组，并且数组长度不变；

```javascript
Array.prototype.reduceMap = function(callback) {
  return this.reduce((acc, cur, index, array) => {
    const item = callback(cur, index, array)
    acc.push(item)
    return acc
  }, [])
}
```

### forEach

> forEach 方法接收一个回调函数作为参数，函数接收四个参数：当前项，索引，原函数和当执行回调函数时用作 this 的值，没有返回值；

```javascript
Array.prototype.reduceForEach = function(callback) {
  this.reduce((acc, cur, index, array) => {
    callback(cur, index, array)
  }, [])
}
```

### filter

> filter 方法接收一个回调函数，函数接收四个参数：当前项，索引，原函数和当执行回调函数时用作 this 的值，回调函数返回 true 则返回当前项，反之则不返回；

```javascript
Array.prototype.reduceFilter = function (callback) {
   return this.reduce((acc, cur, index, array) => {
    if (callback(cur, index, array)) {
      acc.push(cur)
    }
    return acc
  }, [])
}
```

### find

> find 方法方法接收一个回调函数，函数接收四个参数：当前项，索引，原函数和当执行回调函数时用作 this 的值，返回数组中满足回调函数的第一个元素的值，否则返回 undefined；

```javascript
const testArr = [1, 2, 3, 4]
const testObj = [{ a: 1 }, { a: 2 }, { a: 3 }, { a: 4 }]
Array.prototype.reduceFind = function (callback) {
  return this.reduce((acc, cur, index, array) => {
    if (callback(cur, index, array)) {
      if (acc instanceof Array && acc.length == 0) {
      	acc = cur
      }
    }
    // 循环到最后若 acc 还是数组，且长度为 0，代表没有找到想要的项，则 acc = undefined
    if ((index == array.length - 1) && acc instanceof Array && acc.length == 0) {
      acc = undefined
    }
    return acc
  }, [])
}
```

## 使用场景

### 将二维数组转为一维数组

```javascript
const testArr = [[1,2], [3,4], [5,6]]
testArr.reduce((acc, cur) => {
  return acc.concat(cur)
}, [])
```

### 计算数组中元素出现的个数

```javascript
const testArr = [1, 3, 4, 1, 3, 2, 9, 8, 5, 3, 2, 0, 12, 10]
testArr.reduce((acc, cur) => {
  if (!(cur in acc)) {
    acc[cur] = 1
  } else {
    acc[cur] += 1
  }
  return acc
}, {})
```

### 按属性给数组分类

```javascript
const bills = [
  { type: 'shop', momey: 223 },
  { type: 'study', momey: 341 },
  { type: 'shop', momey: 821 },
  { type: 'transfer', momey: 821 },
  { type: 'study', momey: 821 }
];
bills.reduce((acc, cur) => {
  // 如果不存在这个键，则设置它赋值 [] 空数组
  if (!acc[cur.type]) {
    acc[cur.type] = [];
  }
  acc[cur.type].push(cur)
  return acc
}, {})
```

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4a477b0481c14f08868f3bce17f888e2~tplv-k3u1fbpfcp-zoom-1.image)

### 数组去重

```javascript
const testArr = [1,2,2,3,4,4,5,5,5,6,7]
testArr.reduce((acc, cur) => {
  if (!(acc.includes(cur))) {
    acc.push(cur)
  }
  return acc
}, [])
```



