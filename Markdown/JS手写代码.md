## 实现 new 方法

```javascript
/*
* 1. 创建一个对象
* 2. 链接到原型
* 3. 绑定this值 
* 4. 返回新对象 
*/

function createNew() {
    // 1.创建一个空对象
    let obj = {}  
    // 获取构造函数
    let constructor = [].shift.call(arguments)  
    
    // 设置原型，使__proto__指向构造函数的原型，这样，新对像就可以访问到构造函数原型上的属性与方法
    obj.__proto__ = constructor.prototype  

  	// 改变this指向，这样，新对象就可以访问构造函数的属性和方法
    let result = constructor.apply(obj, arguments)  

    // 如果返回值是一个对象就返回该对象，否则返回构造函数的一个实例对象
    // 构造函数有返回且返回的是个指定对象情况
    return typeof result === 'object' ? result : obj  
}

function People(name,age) {
    this.name = name
    this.age = age
}

let peo = createNew(People,'Bob',22)
console.log(peo.name)
console.log(peo.age)
```



## 实现 Promise

```javascript
/*
* 1. 自动执行函数
* 2. 三个状态
* 3. then 
*/

class Promise{
  contructor(fn){
    this.state = 'pending';
    this.value = undefined;
    this.reason = undefined;
    
    let resolve = value => {
      if(this.state === 'pending'){
        this.state = 'fulfilled';
        this.value = value;
      }
    }
    
    let reject = value =>{
      if(this.state === 'pending'){
        this.state = 'rejected';
        this.reason = value
      }
    }
    
    // 自动执行函数
    try {
      fn(resolve, reject)
    } catch(e) {
      reject(e)
  	}
  }
  then(onFulfilled, onRejected) {
      switch (this.state) {
          case: 'fufulfilled:
            onFulfilled(this.value)
          	break
          case: 'rejected:
            onRejected(this.value)
          	break
          default:
      }
  }
}
```



## 实现一个 call 函数

```javascript
/*
* 1. 将函数设为对象的属性
* 2. 执行&删除这个函数
* 3. 指定this到函数并传入给定参数执行函数
* 4. 如果不传入参数，默认指向window
*/

Function.prototype.call = function (context){
  if(type0f this !== 'function'){
    throw new TypeError('not function');
  }
  
  context = context || window;
  context.fn = this;
  
  let args = [...arguments].slice(1);
  let result = context.fn(args);
  
  delete context.fn;
  
  return result
}
```



## 实现一个 apply 函数

```javascript
Function.prototype.myapply = function (context) {
  if (typeof this !== 'function') {
    throw new TypeError('not funciton')
  }
  
  context = context || window
  context.fn = this
  
  let result
  if (arguments[1]) {
    result = context.fn(...arguments[1])
  } else {
    result = context.fn()
  }
  
  delete context.fn
  
  return result
}
```



## 实现一个 bind 函数

> 会创建一个新函数。当这个新函数被调用时，bind() 的第一个参数将作为它运行时的 this，之后的一序列参数将会在传递的实参前传入作为它的参数。

```javascript
Function.prototype.bind2 = function(content) {
    if(typeof this != "function") {
        throw Error("not a function")
    }
    // 若没问参数类型则从这开始写
    let fn = this;
    let args = [...arguments].slice(1);
    
    let resFn = function() {
        return fn.apply(this instanceof resFn ? this : content,args.concat(...arguments) )
    }
    function tmp() {}
    tmp.prototype = this.prototype;
    resFn.prototype = new tmp();
    
    return resFn;
}
```

## 浅拷贝、深拷贝的实现

## 实现一个节流函数

> 可以理解为事件在一个管道中传输，加上这个节流阀以后，事件的流速就会减慢。实际上这个函数的作用就是如此，它可以将一个函数的调用频率限制在一定阈值内，例如 1s，那么 1s 内这个函数一定不会被调用两次

```javascript
function throttle(fn, wait) {
  let prev = new Date();
  return function() { 
      const now = new Date();
      if (now - prev > wait) {
          fn.apply(this, arguments);
          prev = new Date();
      }
  }
}

```



## 实现一个防抖函数

> 当一次事件发生后，事件处理器要等一定阈值的时间，如果这段时间过去后 再也没有 事件发生，就处理最后一次发生的事件。假设还差 `0.01` 秒就到达指定时间，这时又来了一个事件，那么之前的等待作废，需要重新再等待指定时间。
>
> 典型例子：限制 鼠标连击 触发。

```javascript
function debounce(fn, delay) {
  // 持久化一个定时器 timer
  let timer = null;
  // 闭包函数可以访问 timer
  return function() {
    // 通过 'this' 和 'arguments'
    // 获得函数的作用域和参数
    let context = this;
    let args = arguments;
    // 如果事件被触发，清除 timer 并重新开始计时
    clearTimeout(timer);
    timer = setTimeout(function() {
      fn.apply(context, args);
    }, delay);
  }
}
```

## instanceof 的原理

## 柯里化函数的实现

## Object.create 的基本实现原理

## 实现一个基本的 Event Bus

## 实现一个双向数据绑定

## 实现一个简单路由

## 实现懒加载

## rem 基本设置

## 手写实现 AJAX