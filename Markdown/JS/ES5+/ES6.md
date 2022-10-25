## let和const

### let

1. 不存在变量提升【参考---执行上下文与执行栈】;
2. 暂时性死区——只要块级作用域内存在let命令，它所声明的变量就“绑定”（binding）这个区域，不再受外部的影响;
3. 不允许重复声明;
4. 块级作用域——被{}包裹的，外部不能访问内部;
5. let和var全局声明时，var可以通过window的属性访问而let不能【顶层对象不一样】

### const 

1. const保证的是变量指向的那个内存地址所保存的数据不得改动;
2. 对于复合类型的数据（主要是对象和数组），变量指向的内存地址，保存的只是一个指向实际数据的指针，只能保证这个指针是固定的,它指向的数据结构是不是可变的，就不能控制了;

## Class

```javascript
// ES6之前
function Person(name, age) {
    this.name = name
}
Person.prototype.information = function () {
    return 'My name is ' + this.name 
}

// ES6之后
class Person {
    constructor(name, age) {
        this.name = name
    }
    information() {
        return 'My name is ' + this.name 
    }
}
```

## 箭头函数

## 解构赋值

> 通过解构赋值，可以将属性/值从对象/数组中取出，赋值给其它变量；

```javascript
let a = 10
let b = 20
[a, b] = [b, a]
```



## 模块化

## Promise

## Symbol

> 新的原始数据类型Symbol，表示唯一的值。它是 JavaScript 语言的第七种数据类型；

1.  Symbol 值不是对象，不能添加属性。本质上，它是一种类似于字符串的数据类型；
2. Symbol函数可以接受一个字符串作为参数，表示对 Symbol 实例的描述，方便区分，，因此相同参数的Symbol函数的返回值是不相等的；
3. 在对象的内部，使用 Symbol 值定义属性时，Symbol 值必须放在方括号之中，不能用点运算符；
4. Symbol 作为属性名时属性不会出现在for...in、for...of循环中，也不会被Object.keys()、Object.getOwnPropertyNames()、JSON.stringify()返回。但它也不是私有属性，使用Object.getOwnPropertySymbols方法可以获取指定对象的所有 Symbol 属性名；

#### Symbol.for()，Symbol.keyFor()

Symbol.for()接受一个字符串作为参数，然后搜索有没有以该参数作为名称的 Symbol 值。如果有，就返回这个 Symbol 值，否则就新建并返回一个以该字符串为名称的 Symbol 值；

Symbol.keyFor方法返回一个已登记的 Symbol 类型值的key；

```javascript
let a1 = Symbol.for('123');
let a2 = Symbol.for('123');

a1 === a2 // true

// Symbol.for()与Symbol()这两种写法，都会生成新的Symbol。
// 它们的区别是，前者会被登记在全局环境中供搜索，后者不会
Symbol.keyFor(a1) // "123"
let c2 = Symbol("f");
Symbol.keyFor(c2) // undefined
```

## 迭代器/生成器

### 迭代器

1. 迭代器是一种特殊的对象，所有的的迭代器对象都有一个 next 方法；
2. 每次调用都返回一个结果对象，结果对象包含两个属性：
   - value：表示下一个将要返回的值；
   - done：一个布尔值，当没有更多数据返回时返回 true；

#### ES5实现

```javascript
function createIterator(items) {
  var i = 0
  return {
    next: function () {
      var done = i >= items.length;
      var value = !done ? items[i++] : undefined;
      return {
        done: done,
        value: value,
      }
    }
  }
}
```

### 生成器

> Generator 函数可以通过 yield 关键字，把函数的执行流挂起，通过next()方法可以切换到下一个状态，为改变执行流程提供了可能；

1. 生成器是一种返回迭代器的函数，通过 function 关键字后面的星号（*）来表示，函数中会用到关键字 yield；
2. 星号可以紧挨着 function 关键字，也可以在中间添加一个空格；
3. 生成器调用方式与普通函数相同，只不过返回的是一个迭代器；
4. yeild 关键字只可在生成器内部使用，在其它地方使用会导致程序抛出错误，

```javascript
function *createIterator(items){
  for(let i =0;i<items.length;i++){
    yield items[i]
  }
}

let iterator = createIterator([1,2,3])

iterator.next() // '{value:1,done:false}'
iterator.next() // '{value:2,done:false}'
iterator.next() // '{value:3,done:false}'
iterator.next() // '{value:undefine,done:true}'
```

## for...of

> for...of 语句在可迭代对象（包括 Array，Map，Set，String，TypedArray，arguments对象等）上创建一个迭代循环，调用自定义迭代钩子，并为每个不同属性的值执行语句；

```javascript
const array1 = ['a', 'b', 'c'];

for (const element of array1) {
      console.log(element)
}

// "a"
// "b"
// "c"
```

## Set/WeakSet

> 新的数据结构Set,类似于数组，但是成员的值都是唯一的，没有重复的值。Set本身是一个构造函数，用来生成Set数据结构；

Set的遍历顺序就是插入顺序，这个特性有时非常有用，比如使用Set保存一个回调函数列表，调用时就能保证按照添加顺序调用；

### 实例属性和方法

- add(value)：添加某个值，返回Set结构本身；
- delete(value)：删除某个值，返回一个布尔值，表示删除是否成功；
- has(value)：返回一个布尔值，表示该值是否为Set的成员；
- clear()：清除所有成员，没有返回值；

遍历操作：

- keys()：返回键名的遍历器；
- values()：返回键值的遍历器；
- entries()：返回键值对的遍历器；
- forEach()：使用回调函数遍历每个成员；

WeakSet 结构与 Set 类似，主要区别是：

1. WeakSet 对象中只能存放对象引用，不能存放值，Set 对象都可以；
2. WeakSet 对象中的值都是被弱引用的，如果没有其它变量引用这个属性值，则这个对象会被当成垃圾回收掉；
3. WeakSet 对象是无法被枚举的，没有办法拿到它包含的所有元素；



## Map/WeakMap

> Map 对象保存键值对，对象或者原始值都可以作为一个键或一个值；
>
> WeakMap 对象是一组键/值对的集合，键是弱引用的，其键必须是对象，而值可以是任意的；

### 实例属性和方法

- size属性: 返回Map结构的成员总数；
- set(key, value): set方法设置key所对应的键值，然后返回整个Map结构。如果key已经有值，则键值会被更新，否则就新生成该键,set方法返回的是Map本身，因此可以采用链式写法；
- get(key) : get方法读取key对应的键值，如果找不到key，返回undefined；
- has(key) : has方法返回一个布尔值，表示某个键是否在Map数据结构中；
- delete(key) : delete方法删除某个键，返回true。如果删除失败，返回false；
- clear() : clear方法清除所有成员，没有返回值；

## Object.assign（浅拷贝）

> 用于对象的合并，将源对象的所有可枚举属性，复制到目标对象; 

1. 如果只有一个参数，Object.assign会直接返回该参数; 
2. 由于undefined和null无法转成对象，如果它们作为参数，就会报错;
3. 其他类型的值（即数值、字符串和布尔值）不在首参数，也不会报错。但是，除了字符串会以数组形式，拷贝入目标对象，其他值都不会产生效果;

```javascript
// 合并对象
const target = { a: 1, b: 1 };

const source1 = { b: 2, c: 2 };
const source2 = { c: 3 };

Object.assign(target, source1, source2);
target // {a:1, b:2, c:3}

// 非对象和字符串的类型将忽略；
const a1 = '123';
const a2 = true;
const a3 = 10;

const obj = Object.assign({}, a1, a2, a3);
console.log(obj); // { "0": "1", "1": "2", "2": "3" }
```

4. 处理数组时会把数组视为对象;

```javascript
Object.assign([1, 2, 3], [4, 5])
// [4, 5, 3]
```

5. Object.assign只能进行值的复制，如果要复制的值是一个取值函数，那么将求值后再复制；

```javascript
const a = {
  get num() { return 1 }
};
const target = {};

Object.assign(target, a)
// { num: 1 }
```

### 应用场景

1. 为对象添加属性和方法；
2. 克隆/合并对象；
3. 为属性指定默认值；

### 

## Array相关

### 扩展操作符(浅拷贝)

扩展运算符（spread）是三个点（...），将一个数组转为用逗号分隔的参数序列；

- 复制数组；

  ```javascript
  const a1 = [1, 2];
  const a2 = [...a1];
  ```

- 合并数组;

  ```javascript
  const arr1 = ['1', '2'];
  const arr2 = ['c', {a:1} ];

  // ES6 的合并数组
  [...arr1, ...arr2]
  ```

- 将字符串转化为数组;

  ```javascript
  [...'xuxi']
  // [ "x", "u", "x", "i" ]
  ```

- 实现了 Iterator 接口的对象;

  ```javascript
  let nodeList = document.querySelectorAll('div');
  let arr = [...nodeList];
  ```

### Array.prototype.from

> Array.from方法用于将类对象转为真正的数组：类似数组的对象和可遍历的对象（包括 ES6 新增的数据结构 Set 和 Map）。

实际应用中我们更多的是将Array.from用于DOM 操作返回的 NodeList 集合，以及函数内部的arguments对象。

Array.from还可以接受第二个参数，作用类似于数组的map方法，用来对每个元素进行处理，将处理后的值放入返回的数组。

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

> 在当前数组内部，将指定位置的成员复制到其他位置（会覆盖原有成员），然后返回当前数组，使用这个方法，会修改当前数组。

它接受三个参数:

- target（必需）：从该位置开始替换数据。如果为负值，表示倒数。
- start（可选）：从该位置开始读取数据，默认为 0。如果为负值，表示从末尾开始计算。
- end（可选）：到该位置前停止读取数据，默认等于数组长度。如果为负值，表示从末尾开始计算。

```javascript
const array1 = ['a', 'b', 'c', 'd', 'e']

console.log(array1.copyWithin(0, 3, 4)) 
// ["d", "b", "c", "d", "e"]

console.log(array1.copyWithin(1, 3)) 
// ["d", "d", "e", "d", "e"]
```

### Array.prototype.find

> 用于找出第一个符合条件的数组成员。它的参数是一个回调函数，所有数组成员依次执行该回调函数，直到找出第一个返回值为true的成员，然后返回该成员。如果没有符合条件的成员，则返回undefined；
>
> 可以接受第二个参数，用来绑定回调函数的this对象。

```javascript
const array1 = [5, 12, 8, 130, 44]

const found = array1.find(element => element > 10)

console.log(found) // 12
```

### Array.prototype.findIndex

> 返回第一个符合条件的数组成员的位置，如果所有成员都不符合条件，则返回-1；
>
> 可以接受第二个参数，用来绑定回调函数的this对象。

```javascript
const array1 = [5, 12, 8, 130, 44]

const isLargeNumber = (element) => element > 13

console.log(array1.findIndex(isLargeNumber)) // 3
```

### Array.prototype.fill

> fill方法使用给定值，填充一个数组。fill方法还可以接受第二个和第三个参数，用于指定填充的起始位置和结束位置。如果填充的类型为对象，那么被赋值的是同一个内存地址的对象，而不是深拷贝对象。

```javascript
const array1 = [1, 2, 3, 4]

console.log(array1.fill(0, 2, 4)) // [1, 2, 0, 0]

console.log(array1.fill(5, 1)) // [1, 5, 5, 5]

console.log(array1.fill(6)) // [6, 6, 6, 6]
```

### Array.prototype.includes

> Array.prototype.includes方法返回一个布尔值，表示某个数组是否包含给定的值。该方法的第二个参数表示搜索的起始位置，默认为0。如果第二个参数为负数，则表示倒数的位置，如果这时它大于数组长度（比如第二个参数为-4，但数组长度为3），则会重置为从0开始。

```javascript
[1, 4, 3].includes(3)     // true
[1, 2, 4].includes(3)     // false
[1, 5, NaN, 6].includes(NaN) // true
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

### Array.prototype.flat()、flatMap()

> flat()用于将嵌套的数组“拉平”，变成一维的数组。该方法返回一个新数组，对原数据没有影响。flat()默认只会“拉平”一层，如果想要“拉平”多层的嵌套数组，可以将flat()方法的参数写成一个整数，表示想要拉平的层数，默认为1。如果不管有多少层嵌套，都要转成一维数组，可以用Infinity关键字作为参数。flatMap()方法对原数组的每个成员执行一个函数，然后对返回值组成的数组执行flat()方法。该方法返回一个新数组，不改变原数组。flatMap()只能展开一层数组。flatMap()方法还可以有第二个参数，用来绑定遍历函数里面的this；

```javascript
[1, 2, [3, [4, 5]]].flat()
// [1, 2, 3, [4, 5]]

[1, 2, [3, [4, 5]]].flat(2)
// [1, 2, 3, 4, 5]

[1, [2, [3]]].flat(Infinity)
// [1, 2, 3]

// 如果原数组有空位，flat()方法会跳过空位
[1, 2, , 4, 5].flat()
// [1, 2, 4, 5]

// flatMap
[2, 3, 4].flatMap((x) => [x, x * 2])
// [2, 4, 3, 6, 4, 8]

```

## class

## 修饰器Decorator

## module

