this是在运行时进行绑定的（这里可以参考[执行栈这篇文章](%E6%89%A7%E8%A1%8C%E4%B8%8A%E4%B8%8B%E6%96%87%E4%B8%8E%E6%89%A7%E8%A1%8C%E6%A0%88.md)，我理解的是因为执行栈是动态执行的，this指向也是在创建词法环境的创建的），不是在编写时绑定，它的上下文取决于函数调用时的各种条件；

所以this的绑定的函数声明的位置没有任何关系，只取决于函数的调用方式；

### 绑定规则

#### 默认绑定

最常用的函数调用类型：独立函数调用；函数调用时应用的this的默认绑定，指向全局对象；

如果使用严格模式，则不能将全局对象用于默认绑定，this会绑定到 undefined;

```javascript
// 严格模式
function foo(){
  'use strict'
  console.log(this.a)
}
var a = 2;
foo(); // TypeError:this is undefined;

// 非严格模式
function foo(){
  console.log(this.a)
}
var a= 2;
(function(){
  'use strict'
  foo();  // 2
})();

```

#### 隐式绑定

判断是否为隐式绑定一般是考虑**调用位置是否有上下文对象，或者是否被某个对象拥有/包含**；

- 在一个对象内部包含一个指向函数的属性，并通过这个属性间接引用函数，从而把this隐式绑定在这个对象上；
- **对象属性引用链中只有上一层或最后一层在调用位置中起作用**，比如通过obj1.obj2.fn()，只会在obj2起作用；
- 当函数引用有上下文对象时，隐式绑定会把函数调用中的this绑定到这个上下文对象；

```javascript
function foo(){
  console.log(this.a)
}
var obj2 = {
  a:42,
  foo:foo  // 对象属性引用
}
var obj1 = {
  a: 2,
  obj2:obj2
}
obj1.obj2.foo()  // 42， 只有在上一层调用位置起作用
```

##### 隐式丢失

被隐式绑定的函数会丢失绑定对象，也就会应用默认绑定，从而把this绑定到全局对象或undefined上（取决于严格模式）；

```javascript
function foo(){
  console.log(this.a)
}
var obj = {
  a:2,
  foo:foo
}
var bar = obj.foo; // 函数别名
var a = 'oops，global';

bar(); // 'oops，global'
```

虽然bar是对obj.foo的引用，但它引用的是foo函数本身，此时的bar（）是一个不带任何修饰的函数的调用，应用了默认绑定；

丢失绑定对象也会发生在传入回调函数中：

```javascript
function foo(){
  console.log(this.a)
}
function doFoo(fn){
  fn() // <--调用位置
}
var obj = {
  a:2,
  foo:foo
}
var a="global";
doFoo(obj.foo); // "global"
```

参数传递是一种隐式赋值，传入的函数会被隐式赋值，所以应用默认绑定；

#### 显示绑定

在某个对象上强制调用函数，使用call（..）和 apply（..）方法；

- 第一个参数是一个对象，给this准备，在调用函数时将其绑定到this；
- 如果 传入的是一个原始值（String、Boolean、Number）来当作this绑定对象，这个原始值 就会被转换成它的对象形式（new String（...）、new Boolean（...）、new Number(...)），这通常被称为装箱；
- 从绑定的角度来说，这两个方法没有区别；

##### 硬绑定

典型应用场景就是创建一个包裹函数 ，负责接收参数并返回值；

```javascript
function foo(something){
  console.log(this.a,something);
  return this.a + something;
}
var obj = {
  a:2
}
var bar = function(){
  return foo.apply(obj, arguments)
}
var b = bar(3); // 2 3
console.log(b); // 5
```

另一种场景就是创建一个可以重复使用的辅助函数；

```javascript
function foo(something){
  console.log(this.a,something);
  return this.a + something;
}

// 辅助绑定函数 
function bind(fn, obj){
  return function(){
    return fn.apply(obj, arguments)
  }
}
var obj = {
  a:2
}
var bar = bind(foo, obj)
var b = bar(3); // 2 3
console.log(b); // 5
```

ES5提供的内置方法 Function.prototype.bind方法；

```javascript
function foo(something){
  console.log(this.a,something);
  return this.a + something;
}

var obj = {
  a:2
}
var bar = foo.bind(obj)
var b = bar(3); // 2 3
console.log(b); // 5
```

bind（...）会返回一个硬编码的新函数，会把指定的参数设置为 this的上下文并调用原始函数；

#### new绑定

new 运算符创建一个定义的对象类型的实例或具有构造函数的内置对象类型之一；

使用 new 来调用函数，或者发生构造函数调用时，会自动执行下面的操作：

1. 创建了一个全新的对象
2. 它会被执行[[Prototype]]（也就是__proto__），
3. 把this指向新创建的对象
4. 通过new创建的每个对象将最终被[[Prototype]]链接到这个函数的prototype对象上
5. 如果函数没有返回对象类型Object(包含Functoin, Array, Date, RegExg, Error)，那 么new表达式中的函数调用将返回该对象引用

```javascript

function createNew() {
    let obj = {}  
    let constructor = [].shift.call(arguments)   
    obj.__proto__ = constructor.prototype  
    let result = constructor.apply(obj, arguments)  
    return typeof result === 'object' ? result : obj  
}
```

### 优先级

如果某个调用位置可以应用多条规则，这时就必须给规则设置优先级；

**new > 显示绑定 > 隐式绑定 > 默认绑定**

- 默认绑定的优先级最低；
- 显示绑定的优先级高于隐式绑定；
- new绑定的优先级高于隐式绑定；
- new绑定的优先级大于显示绑定；

### 绑定例外

#### 被忽略的this

如果把null或undefined作为this的绑定对象传入 call、apply或bind，这些值在调用时会被忽略，实际应用的是默认绑定规则；

#### 间接引用

间接引用在赋值时发生：

```javascript
function foo(){
  console.log(this.a)
}
var a=2;
var o = {
  a:3,
  foo:foo
}
var p ={
  a:4
}
o.foo(); // 3
// 赋值表达式 p.foo = o.foo 的返回值是目标函数的引用，所以调用位置是 foo（），会应用默认绑定
(p.foo = o.foo)(); // 2
```

#### 软绑定

给默认绑定指定一个全局对象和undefined以外的值，可以实现和硬绑定相同的效果，同时保留隐式绑定或显示绑定修改this的能力；

```javascript
if(!Function.prototype.softBind){
  Function.prototype.softBind = function(obj){
    var fn = this
    // 捕获所有 curried 参数
    var curried = [].prototype.slice.call(arguments,1);
    var bound = function(){
      const _this = !this || this === (window||global) ? obj :this;
      const args = curried.concat.apply(curried, arguments)
      return fn.apply(_this, args)
    }
    bound.prototype = Object.creat(fn, prototype)
    return bound
  }
}
```

### this词法

箭头函数不是使用 function 关键字来定义的，它是根据外层 （函数或全局）作用域来决定this；

箭头函数的绑定无法被修改！

```javascript
function foo(){
  // 返回一个箭头函数
  return (a) =>{
    // this 继承于 foo()
    console.log(this.a)
  }
}

var obj1 = {
  a:2
}

var obj2 = {
  a:3
}
var bar = foo.call(obj1);
bar.call(obj2); // 2
```

foo（）内部创建的箭头函数会捕获调用时 foo()的this，由于 foo()的this 绑定到 obj1，bar（引用箭头函数）的this 也会绑定到 obj1，箭头函数的绑定无法被修改；

## 箭头函数

在方法中定义函数应该是指向 window，但是箭头函数没有自己的 this，所以指向上一层作用域中的 this；

1. 箭头函数没有 this；

   - 箭头函数中的 this、super、arguments 及 new.target 这些值由外围最近一层非箭头函数决定；
   - 没有 super，因为没有原型，所以也不能通过 super 来访问原型的属性；

2. 箭头函数没有自己的 arguments；

   - 箭头函数可以访问外围函数的 arguments 对象；
   - 要访问箭头函数的参数可以通过命名参数或 rest 参数的形一式访问；

   ```javascript
   // 访问外围函数的 arguments 对象
   function constant() {
       return () => arguments[0]
   }
   
   var result = constant(1);
   console.log(result()); // 1
   ```

   ```javascript
   // 通过命名参数或者 rest 参数的形式访问参数
   let nums = (...nums) => nums;
   ```

3. 不能通过 new 关键字调用；

   - 箭头函数没有 [[ Construct ]] 方法，所以不能被用作函数，如果通过 new 关键字调用，程序会报错；

4. 箭头函数没有原型；

   - 由于不可能通过 new 关键字调用，所以没有构建原型，箭头函数不存在 prototype 这个属性；

5. 不支持重复的命名参数；

```javascript

obj = {
a: 10,
c: function () {
b = () => {
console.log(this)           // obj
}
b()
}
}
obj.c()

document.getElementById('app').addEventListener('click', () => {
console.log(this)           // window
})

```

## 为什么要用this

- this提供了一种更优雅的方式来隐式“传递”一个对象引用，将API设计更加简洁和易于复用；
- 相对显示传递上下文对象更简洁，可以自动引用合适的上下文对象；

全局环境&普通函数调用&普通对象

```javascript
const obj={a:this}
obj.a===window  //true

function fn(){
    console.log(this)   //window
}
```