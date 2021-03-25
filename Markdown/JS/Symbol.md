## 概述

> symbol 是 ES6 新引入的一种基础数据类型，该类型具有静态属性和静态方法；

1. 作为构造函数来它是不完整的，所以不支持“new Symbol()”语法；
2. 每个从 Symbol() 返回的symbol值都是唯一的；
3. 一个 symbol 值能作为对象属性的标志符，这是该数据类型仅有的目的；

###  Symbol()

```javascript
let sym1 = Symbol("foo");
let sym2 = Symbol("foo");

sym1 === sym2; // false

let sym = new Symbol(); // TypeError
```

### Symbol.for()

它接受一个字符串作为参数，然后全局环境中搜索是否有以该参数注册的Symbol值；

1. 如果有，就返回这个Symbol值；
2. 没有就创建并返回一个以该字符串作为名称的 Symbol 值；

```javascript
let sym1 = Symbol.for("foo");
let sym2 = Symbol.for("foo");

sym1 === sym2; // true
```

### Symbol.keyFor()

在全局注册表中搜索查找改symbol；

1. 如果有返回该 symbol 的 key值，形式为string；
2. 如果没有返回 undefined；

```javascript
let sym= Symbol.for("foo");

console.log(Symbol.keyFor(sym)); // foo
```

## 应用场景

### 模拟私有变量

symbol 不会被常规的方法，除了 Object.getOwnPropertySymbols 外的方法遍历到，可以用来模拟私有变量；

```javascript
let s_name= Symbol("name");

let obj= {
    [s_name]: "lle",
    age: 18,
    title: "Engineer"
};

console.log(Object.keys(obj)); // ["age", "title"]

for(let key in obj) {
    console.log(key); // age, title
}

console.log(Object.getOwnPropertyNames(obj)); 
// ["age", "title"]

JSON.stringify(obj);  // {"age":18,"title":"Engineer"}
```

如果要获取可以用个以下三种方式：

```javascript
obj[s_name]；// lle
Object.getOwnPropertySymbols(obj); // [Symbol(name)]
Reflect.ownKeys(obj); // ["age", "title", Symbol(name)]
```

### 替代常量

```javascript
const NAME = Symbol();
const Age = Symbol()
```

