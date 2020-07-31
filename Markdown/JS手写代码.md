

<https://juejin.im/post/6844903809206976520>

<https://www.cnblogs.com/chenwenhao/p/11294541.html>

## new 方法

```javascript
/*
* 1. 它创建了一个全新的对象
* 2. 它会被执行[[Prototype]]（也就是__proto__）链接
* 3. 它使this指向新创建的对象 
* 4. 通过new创建的每个对象将最终被[[Prototype]]链接到这个函数的prototype对象上
* 5. 如果函数没有返回对象类型Object(包含Functoin, Array, Date, RegExg, Error)，那 么new表达式中的函数调用将返回该对象引用
*/

function createNew() {
    // 1.创建一个空对象
    let obj = {}  
    // 获取构造函数
    let constructor = [].shift.call(arguments)  
    
    // 设置原型，使__proto__指向构造函数的原型，这样，新对象就可以访问到构造函数原型上的属性与方法
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

##  Promise

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



## call 函数

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



## apply 函数

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



## bind 函数

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

## JSON.stringify函数

- Boolean | Number| String 类型会自动转换成对应的原始值
- undefined 、任意函数以及 symbol，会被忽略（出现在非数组对象的属性值中时），或者被转换成 null（出现在数组中时）
- 不可枚举的属性会被忽略
- 如果一个对象的属性值通过某种间接的方式指回该对象本身，即循环引用，属性也会被忽略

```javascript
function jsonStringify(obj) {
    let type = typeof obj;
    if (type !== "object") {
        if (/string|undefined|function/.test(type)) {
            obj = '"' + obj + '"';
        }
        return String(obj);
    } else {
        let json = []
        let arr = Array.isArray(obj)
        for (let k in obj) {
            let v = obj[k];
            let type = typeof v;
            if (/string|undefined|function/.test(type)) {
                v = '"' + v + '"';
            } else if (type === "object") {
                v = jsonStringify(v);
            }
            json.push((arr ? "" : '"' + k + '":') + String(v));
        }
        return (arr ? "[" : "{") + String(json) + (arr ? "]" : "}")
    }
}
jsonStringify({x : 5}) // "{"x":5}"
jsonStringify([1, "false", false]) // "[1,"false",false]"
jsonStringify({b: undefined}) // "{"b":"undefined"}"
```



## 节流函数

> 节流的意思是让函数有节制地执行，而不是毫无节制的触发一次就执行一次。什么叫有节制呢？就是在一段时间内，只执行一次。
>
> 规定在一个单位时间内，只能触发一次函数。如果这个单位时间内触发多次函数，只有一次生效

**应用场景：**

1. 鼠标点击事件，比如mousedown只触发一次
2. 监听滚动事件，比如是否滑到底部自动加载更多

```javascript
function throttle(fn, delay) {
    let flag = true,
        timer = null;
    return function (...args) {
        let context = this;
        if (!flag) return;
        flag = false;
        clearTimeout(timer)
        timer = setTimeout(() => {
            fn.apply(context, args);
            flag = true;
        }, delay)
    }
}
```



## 防抖函数

> 在事件被触发n秒后再执行回调，如果在这n秒内又被触发，则重新计时。
>
> 每次事件触发都会删除原有定时器，建立新的定时器。通俗意思就是反复触发函数，只认最后一次，从最后一次开始计时。

**应用场景：**

1. search搜索，用户不断输入值时，用防抖来节约Ajax请求,也就是输入框事件。
2. window触发resize时，不断的调整浏览器窗口大小会不断的触发这个事件，用防抖来让其只触发一次

```javascript
function debounce(fn, delay) {
  // 持久化一个定时器 timer
  let timer = null;
  // 闭包函数可以访问 timer
  return function() {
    // 通过 'this' 和 'arguments'， 获得函数的作用域和参数
    let context = this;
    let args = arguments;
    // 如果事件被触发，清除 timer 并重新开始计时
    if(timer) clearTimeout(timer)
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

## 图片懒加载

```javascript
页可见区域宽： document.body.clientWidth;
网页可见区域高： document.body.clientHeight;
网页可见区域宽： document.body.offsetWidth (包括边线的宽);
网页可见区域高： document.body.offsetHeight (包括边线的宽);
网页正文全文宽： document.body.scrollWidth;
网页正文全文高： document.body.scrollHeight;
网页被卷去的高： document.body.scrollTop;
网页被卷去的左： document.body.scrollLeft;
网页正文部分上： window.screenTop;
网页正文部分左： window.screenLeft;
屏幕分辨率的高： window.screen.height;
屏幕分辨率的宽： window.screen.width;
屏幕可用工作区高度： window.screen.availHeight;
```

#### 原理思路

1. 拿到所有图片 Dom
2. 判断当前图片是否到了可视区范围内
3. 到了可视区的高度以后，就将img的data-src属性设置给src
4. 绑定window的scroll事件

#### 第一种方式

思路：scrollTop + clientHeight 和offsetTop的比较 ，如果前者大于后者说明图片进入可视区域

```javascript
let Img = document.getElementsByTagName("img"),
            len = Img.length,
            count = 0; 
function lazyLoad () {
    let viewH = document.body.clientHeight, //可见区域高度
        scrollTop = document.body.scrollTop; //滚动条距离顶部高度
    for(let i = count; i < len; i++) {
        if(Img[i].offsetTop < scrollTop + viewH ){
            if(Img[i].getAttribute('src') === 'default.png'){
                Img[i].src = Img[i].getAttribute('data-src')
                count++;
            }
        }
    }
}
window.addEventListener('scroll', throttle(lazyLoad,1000))；

lazyLoad();  

```

#### 第二种方式

思路：使用 `element.getBoundingClientRect()` API 直接得到 top 值

```javascript
let Img = document.getElementsByTagName("img"),
            len = Img.length,
            count = 0; 
function lazyLoad () {
  let viewH = document.body.clientHeight, //可见区域高度
      scrollTop = document.body.scrollTop; //滚动条距离顶部高度
  for(let i = count; i < len; i++) {
      if(Img[i].getBoundingClientRect().top < scrollTop + viewH ){
          if(Img[i].getAttribute('src') === 'default.png'){
              Img[i].src = Img[i].getAttribute('data-src')
              count++;
          }
      }
  }
}
window.addEventListener('scroll', throttle(lazyLoad,1000))

lazyLoad();  // 首次加载 
```

## rem 基本设置

```

```



## 手写实现 AJAX