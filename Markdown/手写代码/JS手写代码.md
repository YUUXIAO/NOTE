## 数组

### 数组扁平化

> 把一个多维数组变为一维数组；

```javascript
let array = [1, [2, 3, [4, 5]]];

// ES6的flat()
array.flat(Infinity); // [1, 2, 3, 4, 5]

// 序列化后正则
let str = JSON.stringify(array).replace(/(\[|\])/g, "");
str = "[" + str + "]";
JSON.parse(str); 

// 递归处理
function flat(arr) {
  let result = [];
  for (const item of arr) {
    item instanceof Array
      ? (result = result.concat(flat(item)))
      : result.push(item);
  }
  return result;
}

// reduce 方法
function flat(arr) {
  return arr.reduce((prev, current) => {
    return prev.concat(
      current instanceof Array ? flat(current) : current
    );
  }, []);
}

// 迭代+扩展运算符
while (array.some(Array.isArray)) {
  array = [].concat(...array);
}
```

### 数组去重

```javascript
let array = ["banana", "apple", "orange", "lemon", "apple", "lemon"];

// filter 方法
function removeDuplicates(data) {
  return data.filter((value, index) => data.indexOf(value) === index);
}

// ES6 的 Set
function removeDuplicates(data) {
  return [...new Set(data)];
}

// forEach 方法
function removeDuplicates(data) {
  let unique = [];
  data.forEach(element => {
    if (!unique.includes(element)) {
      unique.push(element);
    }
  });
  return unique;
}

//  reduce 方法
function removeDuplicates(data) {
  return data.reduce((acc, curr) => acc.includes(curr) ? acc : [...acc, curr], []);
}

// Array.from + ES6 Set
function removeDuplicates(data) {
  return Array.from(new Set(arr))
}
```

### 从对象数组中删除重复的对象

```javascript
let users = [
  { id: 1, name: 'susan', age: 25 },
  { id: 2, name: 'cherry', age: 28 },
  { id: 3, name: 'cindy', age: 27 },
  { id: 2, name: 'cherry', age: 28 },
  { id: 1, name: 'susan', age: 25 },
]

function uniqueByKey(data, key) {
  const object = {};
  data = data.reduce((prev, next) => {
    // eslint-disable-next-line no-unused-expressions
    object[next[key]]
      ? ''
      : (object[next[key]] = true && prev.push(next));
    return prev;
  }, []);
  return data;
}

console.log(uniqueByKey(users, "id"));

// [ { id: 1, name: 'susan', age: 25 },
//   { id: 2, name: 'cherry', age: 28 },
//   { id: 3, name: 'cindy', age: 27 } ]

```

### 数组取交集

```javascript
let a = [1, 2, 3];
let b = [2, 4, 5];

// Array.prototype.includes
let intersection = a.filter(v => b.includes(v));

// Array.from
let aSet = new Set(a);
let bSet = new Set(b);
let intersection = Array.from(new Set(a.filter(v => bSet.has(v))));

// Array.prototype.indexOf
let intersection = a.filter((v) => b.indexOf(v) > -1);

```

### 数组取并集

```javascript
let a = [1, 2, 3];
let b = [2, 4, 5];

// Array.prototype.includes
let union = a.concat(b.filter(v => !a.includes(v)));

// Set
let union = Array.from(new Set(a.concat(b)));

// Array.prototype.indexOf
let union = a.concat(b.filter((v) => a.indexOf(v) === -1));

```

### 数组取差集

```javascript
let a = [1, 2, 3];
let b = [2, 4, 5];

// Array.prototype.includes
let difference = a.concat(b).filter(v => !a.includes(v) || !b.includes(v));

// Array.from
let aSet = new Set(a);
let bSet = new Set(b);
let difference = Array.from(new Set(a.concat(b).filter(v => !aSet.has(v) || !bSet.has(v))));

// Array.prototype.indexOf
let difference = a.filter((v) => b.indexOf(v) === -1).concat(b.filter((v) => a.indexOf(v) === -1));

```

## new方法

> new 运算符创建一个用户定义的对象类型的实例或具有构造函数的内置对象类型之一；

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
    // 取出第一个参数，就是要传入的构造函数。因为shift会修改原数组，所以arguments会除去第一个参数
    let constructor = [].shift.call(arguments)  
    // 将 obj 的原型指向构造函数，这样 obj 就可以访问到构造函数原型中的属性
    obj.__proto__ = constructor.prototype  

  	// 使用 apply 改变构造函数 this 的指向到新建的对象，这样 obj 就可以访问到构造函数中的属性
    let result = constructor.apply(obj, arguments)  

    // 如果返回值是一个对象就返回该对象，否则返回构造函数的一个实例对象
    return typeof result === 'object' ? result : obj  
}

function People(name,age) {
    this.name = name
    this.age = age
}

let peo = createNew(People,'Bob',22)
console.log(peo.name)
```

##  Promise（简易版）

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

> call 是函数对象的原型方法，它的作用是绑定 this 参数，并执行函数；
>
> ```
> function.call(thisArg, arg1, arg2, ...)；
> ```

```javascript
/*
* 1. 将函数设为对象的属性
* 2. 执行&删除这个函数
* 3. 指定this到函数并传入给定参数执行函数
* 4. 如果不传入参数，默认指向window
*/

Function.prototype.myCall = function (context){
  if(typeof this !== 'function'){
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

> apply 方法接收的是一个包含多个参数的数组；

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

> 会创建一个新函数：当这个新函数被调用时，bind() 的第一个参数将作为它运行时的 this，之后的一序列参数将会在传递的实参前传入作为它的参数。

```javascript
Function.prototype.bind2 = function(content) {
    if(typeof this != "function") {
        throw Error("not a function")
    }
   
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

## 深拷贝

```javascript
function deepCopy(target){
  // 对于传入参数处理
  if (typeof target !== 'object' || target === null) {
    return target;
  }
  //判断是否是简单数据类型，
  var result = target.constructor == Array ? [] : {};
  
  for(let i in target){
    result[i] = typeof target[i] == "object" ? deepCopy(target[i]) : target[i];
  }
  return result;
}
```

## instanceOf

> instanceOf 用于检测构造函数的 prototype 属性是否出现在某个实例对象的原型链上；

```javascript
function instanceOf(left,right) {
    let proto = left.__proto__;	// 取left的隐式原型
    let prototype = right.prototype；// 取right的显示原型
    while(true) {
        if(proto === null) return false
        if(proto === prototype) return true
        proto = proto.__proto__;
    }
}
```

## JSONP

> script 标签不遵循同源协议，可以用来进行跨域请求，兼容性好但仅限于GET请求；

```javascript
const jsonp = ({ url, params, callbackName }) => {
  const generateUrl = () => {
    let dataSrc = '';
    for (let key in params) {
      if (Object.prototype.hasOwnProperty.call(params, key)) {
        dataSrc += `${key}=${params[key]}&`;
      }
    }
    dataSrc += `callback=${callbackName}`;
    return `${url}?${dataSrc}`;
  }
  return new Promise((resolve, reject) => {
    const scriptEle = document.createElement('script');
    scriptEle.src = generateUrl();
    document.body.appendChild(scriptEle);
    window[callbackName] = data => {
      resolve(data);
      document.removeChild(scriptEle);
    }
  })
}
```

## AJAX

```javascript
const getJSON = function(url) {
  return new Promise((resolve, reject) => {
    const xhr = XMLHttpRequest ? new XMLHttpRequest() : new ActiveXObject('Mscrosoft.XMLHttp');
    xhr.open('GET', url, false);
    xhr.setRequestHeader('Accept', 'application/json');
    xhr.onreadystatechange = function() {
      if (xhr.readyState !== 4) return;
      if (xhr.status === 200 || xhr.status === 304) {
        resolve(xhr.responseText);
      } else {
        reject(new Error(xhr.responseText));
      }
    }
    xhr.send();
  })
}
```

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

> 规定在一个单位时间内，只能触发一次函数。如果这个单位时间内触发多次函数，只有一次生效；

**应用场景：**

1. 鼠标点击事件，比如mousedown只触发一次
2. 监听滚动事件，比如是否滑到底部自动加载更多；

```javascript
function throttle(fn, delay) {
  let timer = null;
  return function () {
    let context = this;
    let args = arguments;

    if (!timer) {
      timer = setTimeout(() => {
        fn.apply(context, args);
        timer = null;
      }, delay);
    }
  };
}
```

## 防抖函数

> 在事件被触发n秒后再执行回调，如果n秒内又被触发，则重新计时；

每次事件触发都会删除原有定时器，建立新的定时器，只认最后一次，从最后一次开始计时；

应用场景：

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

## 高阶函数实现AOP（面向切面编程）

```javascript
Function.prototype.before = function (beforefn) {
    let _self = this; // 缓存原函数的引用
    return function () { // 代理函数
        beforefn.apply(this, arguments); // 执行前置函数
        return _self.apply(this, arguments); // 执行原函数
    }
}

Function.prototype.after = function (afterfn) {
    let _self = this;
    return function () {
        let set = _self.apply(this, arguments);
        afterfn.apply(this, arguments);
        return set;
    }
}

let func = () => console.log('func');
func = func.before(() => {
    console.log('===before===');
}).after(() => {
    console.log('===after===');
});

func();

// 输出结果：
===before===
func
===after===   
```

## 斐波那契数列

> 斐波那契数列从第三项开始，每一项都等于前两项之和。指的是这样一个数列：0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89, 144 …

```javascript
// 递归
function fib(n) {
  if (n === 1 || n === 2) return n - 1;
  return fib(n - 1) + fib(n - 2)
}
console.log(fib(10)); // 34

// 非递归
function fib(n) {
  let a = 0;
  let b = 1;
  let c = a + b;
  for (let i = 3; i < n; i++) {
    a = b;
    b = c;
    c = a + b;
  }
  return c;
}
console.log(fib(10)); // 34
```

## 迭代器next

> Iterator 函数返回一个对象，它实现了遗留的迭代协议，并且迭代了一个对象的可枚举属性；

只要对应的数据结构有Symbol.iterator属性，就可以完成遍历操作；

```javascript
function createIterator(items) {
    var i = 0;
    return {
        next: function() {
            var done = (i >= items.length);
            var value = !done ? items[i++] : undefined;
            return {
                done: done,
                value: value
            };
        }
    };
}
var iterator = createIterator([1, 2, 3]);
console.log(iterator.next()); // "{ value: 1, done: false }"
console.log(iterator.next()); // "{ value: 2, done: false }"
console.log(iterator.next()); // "{ value: 3, done: false }"
console.log(iterator.next()); // "{ value: undefined, done: true }"
```

## Object.create

## Object.assign

> Object.assign() 方法用于将所有可枚举属性的值从一个或多个源对象复制到目标对象；它将返回目标对象，这个操作是浅拷贝；

```javascript
Object.defineProperty(Object, 'assign', {
  value: function(target, ...args) {
    if (target == null) {
      return new TypeError('Cannot convert undefined or null to object');
    }
    
    // 目标对象需要统一是引用数据类型，若不是会自动转换
    const to = Object(target);

    for (let i = 0; i < args.length; i++) {
      // 每一个源对象
      const nextSource = args[i];
      if (nextSource !== null) {
        // 使用for...in和hasOwnProperty双重判断，确保只拿到本身的属性、方法（不包含继承的）
        for (const nextKey in nextSource) {
          if (Object.prototype.hasOwnProperty.call(nextSource, nextKey)) {
            to[nextKey] = nextSource[nextKey];
          }
        }
      }
    }
    return to;
  },
  // 不可枚举
  enumerable: false,
  writable: true,
  configurable: true,
})
```



## 实现一个基本的 Event Bus

## 实现一个双向数据绑定

## 实现一个简单路由

## 发布-订阅模式

```javascript
class EventHub{
  cache = {};
  on(eventName, fn){
    this.cache[eventName] = this.cache[eventName] || [];
    this.cache[eventName].push(fn);
  }
  emit(eventName, data){
    (this.cache[eventName]||[]).forEach(fn=>fn&&fn(data));
  }
  off(eventName, fn){
    const index = indexOf(this.cache[eventName], fn);
    if(index==-1){
        return;
    }
    this.cache[eventName].splice(index, 1);
  }
}
function indexOf(arr, item){
  if(arr===undefined) return -1;
  let index = -1;
  for(let i=0;i<arr.length;i++){
    if(arr[i]===item){
      index = i;
      break;
    }
  }
  return index;
}
```

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

## 实现 AJAX

## 字符串转二进制

```javascript
function charToBinary(text) {
  let code = "";
  for (let i of text) {
    // 字符编码,toString()方法可以传入2 ~ 36 之间的整数表示基数，若省略该参数，则使用基数 10；
    let number = i.charCodeAt().toString(2);
    // 1 bytes = 8bit，将 number 不足8位的0补上
    for (let a = 0; a <= 8 - number.length; a++) {
       number = 0 + number;
    }
    code += number;
  }
  return code;
}
```

## 实现一个 sleep 函数

> 比如 sleep(1000) 意味着等待1000毫秒，可从 Promise、Generator、Async/Await 等角度实现；

```javascript
//Promise
const sleep = time => {
  return new Promise(resolve => setTimeout(resolve,time))
}
sleep(1000).then(()=>{
  console.log(1)
})

//Generator
function* sleepGenerator(time) {
  yield new Promise(function(resolve,reject){
    setTimeout(resolve,time);
  })
}
sleepGenerator(1000).next().value.then(()=>{console.log(1)})

//async
function sleep(time) {
  return new Promise(resolve => setTimeout(resolve,time))
}
async function output() {
  let out = await sleep(1000);
  console.log(1);
  return out;
}
output();

//ES5
function sleep(callback,time) {
  if(typeof callback === 'function')
    setTimeout(callback,time)
}

function output(){
  console.log(1);
}
sleep(output,1000);

```

## 模拟一个 localStorage

```javascript
'use strict'
const valuesMap = new Map()

class LocalStorage {
  getItem (key) {
    const stringKey = String(key)
    if (valuesMap.has(key)) {
      return String(valuesMap.get(stringKey))
    }
    return null
  }

  setItem (key, val) {
    valuesMap.set(String(key), String(val))
  }

  removeItem (key) {
    valuesMap.delete(key)
  }

  clear () {
    valuesMap.clear()
  }

  key (i) {
    if (arguments.length === 0) {
      throw new TypeError("Failed to execute 'key' on 'Storage': 1 argument required, but only 0 present.") // this is a TypeError implemented on Chrome, Firefox throws Not enough arguments to Storage.key.
    }
    let arr = Array.from(valuesMap.keys())
    return arr[i]
  }

  get length () {
    return valuesMap.size
  }
}
const instance = new LocalStorage()

global.localStorage = new Proxy(instance, {
  set: function (obj, prop, value) {
    if (LocalStorage.prototype.hasOwnProperty(prop)) {
      instance[prop] = value
    } else {
      instance.setItem(prop, value)
    }
    return true
  },
  get: function (target, name) {
    if (LocalStorage.prototype.hasOwnProperty(name)) {
      return instance[name]
    }
    if (valuesMap.has(name)) {
      return instance.getItem(name)
    }
  }
})

```

