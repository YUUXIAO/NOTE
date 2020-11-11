## 全局环境、普通函数调用、普通对象

```javascript
const obj={a:this}
obj.a===window  //true

function fn(){
    console.log(this)   //window
}
```

## 构造函数

1. new 出来的对象，this 指向了即将 new 出来的对象；
2. 当做普通函数执行，this指向window；

```javascript
function a() {
  console.log(this)
}
const obj = new a()   //  a {}
a()                    // 'window'
```

## 对象方法

1. 作为对象方法，this 指向了这个对象；
2. 有变量直接指向了这个方法，this 为 window；

```javascript
const obj = {
  x: 0,
  foo: function () {
    console.log(this)
  }
}
obj.foo()                 // obj
const a = obj.foo
a()                       //window
```

### 特殊情况

如果直方法里面执行函数，this 指向 window；

```javascript
const obj = {
  x: 0,
  foo: function () {
    console.log(this)      // obj
    function foo1() {
      console.log(this)    //window
    }
    foo1()
  }
}
obj.foo()   
```

## 构造函数prototype属性

原型定义方法的 this 指向了实例对象，毕竟是通过对象调用的；

```javascript
function Fn() {
  this.a = 10
  let a = 100
}
Fn.prototype.fn = function () {
  console.log(this.a)            
}
const obj = new Fn()
obj.fn()		// 10
```

## call ,apply, bind

call ,apply, bind 方法中，this 指向传入的对象；

```javascript
const obj = {
  x: 10
}
function fn() {
  console.log(this)
}
fn.call(obj)      	//obj
fn.apply(obj) 		//obj
fn.bind(obj)() 		//obj
```

## DOM事件

Dom 事件中，this 指向绑定事件的对象；

```javascript
document.getElementById('app').addEventListener('click', 	function () {
      console.log(this)           // id为app的这个对象
})
```

## 箭头函数

在方法中定义函数应该是指向 window，但是箭头函数没有自己的 this，所以指向上一层作用域中的 this；

```javascript
obj = {
  a: 10,
  c: function () {
    b = () => {
      console.log(this)           //指向obj
    }
    b()
  }
}
obj.c()
```

```javascript
document.getElementById('app').addEventListener('click', () => {
  	// 改为箭头函数，指向了window,而不是触发对象
    console.log(this)          
 })
```

## 绑定方式

### 隐式绑定

谁调用方法，this指向谁；

### 显示绑定

call,bind,apply方法；

### new 绑定

### 优先级

new>显示绑定>隐式绑定；

