## 类数组对象

> 类数组对象是拥有一个 length 属性和若干索引属性的对象；

## 调用数组方法

> 类数组对象可以用 Function.call 间接调用数组的方法；

```javascript
var arrayLike = {0: 'name', 1: 'age', 2: 'sex', length: 3 }
Array.prototype.join.call(arrayLike, '&'); // name&age&sex
// slice可以做到类数组转数组
Array.prototype.slice.call(arrayLike, 0); // ["name", "age", "sex"] 
Array.prototype.map.call(arrayLike, function(item){
    return item.toUpperCase();
}); 
// ["NAME", "AGE", "SEX"]
```

## 类数组转数组

```javascript
var arrayLike = {0: 'name', 1: 'age', 2: 'sex', length: 3 }
// 1. slice
Array.prototype.slice.call(arrayLike); // ["name", "age", "sex"] 
// 2. splice
Array.prototype.splice.call(arrayLike, 0); // ["name", "age", "sex"] 
// 3. ES6 Array.from
Array.from(arrayLike); // ["name", "age", "sex"] 
// 4. apply
Array.prototype.concat.apply([], arrayLike)
```

### 类数组检测

```javascript
// 字符串和函数有length属性，但是它们可以用 typeof 检测排除
function isArrayLike(o) {
  if (o &&                                // o is not null, undefined, etc.
      typeof o === 'object' &&            // o is an object
      isFinite(o.length) &&               // o.length is a finite number
      o.length >= 0 &&                    // o.length is non-negative
      o.length===Math.floor(o.length) &&  // o.length is an integer
      o.length < 4294967296)              // o.length < 2^32
      return true;                        // Then o is array-like
  else
      return false;                       // Otherwise it is not
}
```

## Arguments对象

> Arguments 对象只定义在函数体中，包括了函数的参数和其他属性。在函数体中，arguments 指代该函数的 Arguments 对象；

### length属性

> Arguments对象的length属性，表示实参的长度；

```javascript
function foo(b, c, d){
    console.log("实参的长度为：" + arguments.length)	// 实参的长度为：1
}
console.log("形参的长度为：" + foo.length)			 	// 形参的长度为：3
```

### callee属性

> Arguments 对象的 callee 属性，通过它可以调用函数自身；

```javascript
var data = [];
for (var i = 0; i < 3; i++) {
    (data[i] = function () {
       console.log(arguments.callee.i) 
    }).i = i;
}
data[0]();			// 0
data[1]();			// 1
data[2]();			// 2
```

### arguments 和对应参数的绑定

1. 传入的参数，实参和 arguments 的值会共享，当没有传入时，实参与 arguments 值不会共享；
2. 如果是在严格模式下，实参和 arguments 是不会共享的；

```javascript
function foo(name, age, sex, hobbit) {
    console.log(name, arguments[0]); // name name
  
    // 改变形参
    name = 'new name';
    console.log(name, arguments[0]); // new name new name
  
    // 改变arguments
    arguments[1] = 'new age';
    console.log(age, arguments[1]); // new age new age
  
    // 测试未传入的是否会绑定
    console.log(sex); // undefined
    sex = 'new sex';
    console.log(sex, arguments[2]); // new sex undefined
    arguments[3] = 'new hobbit';
    console.log(hobbit, arguments[3]); // undefined new hobbit
}
foo('name', 'age')
```

### 传递参数

> 将参数从一个函数传递到另一个函数；

```javascript
function foo() {
    bar.apply(this, arguments);
}
function bar(a, b, c) {
   console.log(a, b, c);
}

foo(1, 2, 3)
```

### arguments转数组

```javascript
// 使用ES6的 ... 运算符，我们可以轻松将 arguments 转成数组；
function func(...arguments) {
    console.log(arguments); // [1, 2, 3]
}

func(1, 2, 3);
```

### 应用场景

1. 参数不定长；
2. 函数柯里化；
3. 递归调用；
4. 函数重载；

