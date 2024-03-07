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

### push & pop

```javascript
// push 方法
Array.prototype.push2 = function(...rest){
  this.splice(this.length, 0, ...rest)
  return this.length;
}

// pop 方法
Array.prototype.pop2 = function(){
  return this.splice(this.length - 1, 1)[0];
}
```

## new方法

> new 运算符创建一个用户定义的对象类型的实例或具有构造函数的内置对象类型之一；

```javascript
/*
* 1. 创建了一个全新的对象
* 2. 它会被执行[[Prototype]]（也就是__proto__）链接
* 3. 它使this指向新创建的对象 
* 4. 通过new创建的每个对象将最终被[[Prototype]]链接到这个函数的prototype对象上
* 5. 如果函数没有返回对象类型Object(包含Functoin, Array, Date, RegExg, Error)，那 么new表达式中的函数调用将返回该对象引用
*/

function createNew() {
    let obj = {}  
    let constructor = [].shift.call(arguments)  
    obj.__proto__ = constructor.prototype  
    let result = constructor.apply(obj, arguments)  
    return typeof result === 'object' ? result : obj  
}
```

## call函数

> call 方法接收的是多个参数；

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
  let result = context.fn(...args);
  
  delete context.fn;
  return result
}
```

## apply函数

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

> bind() 的第一个参数将作为它运行时的 this，之后的一序列参数将会在传递的实参前传入作为它的参数，返回一个新函数；

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
function deepCopy(target,map=new Map()){
  // 基础类型判断，直接返回
  if (typeof target !== 'object' || target === null) {
    return target;
  }
  // 引用类型
  var result = target.constructor == Array ? [] : {};
  // 处理循环引用 
  if(map.get(target)) return map.get(target);
  for(let i in target){
    result[i] = typeof target[i] == "object" ? deepCopy(target[i],map) : target[i];
  }
  map.set(target, result);
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

## 节流

> 规定在一个单位时间内，只能触发一次函数，如果这个单位时间内触发多次函数，只有一次生效；

1. 鼠标点击事件，比如mousedown只触发一次；
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

## 防抖

> 在事件被触发n秒后再执行回调，如果n秒内又被触发，则重新计时；

1. search搜索，用户不断输入值时，用防抖来节约Ajax请求,也就是输入框事件。
2. window触发resize时，不断的调整浏览器窗口大小会不断的触发这个事件，用防抖来让其只触发一次

```javascript
// 每次事件触发都会删除原有定时器，建立新的定时器，只认最后一次，从最后一次开始计时；
function debounce(fn, delay) {
  let timer = null;
  return function() {
    let context = this;
    let args = arguments;
    if(timer) clearTimeout(timer)
    timer = setTimeout(function() {
      fn.apply(context, args);
    }, delay);
  }
}

```

## 斐波那契数列

> 斐波那契数列从第三项开始，每一项都等于前两项之和；

例如：0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89, 144 …

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
```

## 栈结构

```javascript
const Stack = (() => {
  const wm = new WeakMap()
  class Stack {
    constructor() {
      wm.set(this, [])
      this.top = 0
    }

    push(...nums) {
      let list = wm.get(this)
      nums.forEach(item => {
        list[this.top++] = item
      })
    }

    pop() {
      let list = wm.get(this)
      let last = list[--this.top]
      list.length = this.top
      return last
    }

    peek() {
      let list = wm.get(this)
      return list[this.top - 1]
    }

    clear() {
      let list = wm.get(this)
      list.length = 0
    }

    size() {
      return this.top
    }

    output() {
      return wm.get(this)
    }

    isEmpty() {
      return wm.get(this).length === 0
    }
  }
  return Stack
})()

let s = new Stack()

s.push(1, 2, 3, 4, 5)
console.log(s.output()) // [ 1, 2, 3, 4, 5 ]

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

```javascript
Object._create = (o) => {
    let Fn = function() {}; // 临时的构造函数
    Fn.prototype = o;
    return new Fn;
}

let obj1 = {id: 1};
let obj2 = Object._create(obj1);
console.log(obj2.__proto__ === obj1); // true
console.log(obj2.id); // 1

// 原生的Object.create
let obj3 = Object.create(obj1);
console.log(obj3.__proto__ === obj1); // true
console.log(obj3.id); // 1
```

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

1. 拿到所有图片 Dom；
2. 判断当前图片是否到了可视区范围内；
3. 到了可视区的高度以后，就将img的data-src属性设置给src；
4. 绑定window的scroll事件；

#### 第一种方式

> scrollTop + clientHeight 和offsetTop的比较 ，如果前者大于后者说明图片进入可视区域；

```javascript
function lazyload() {
  const imgs = document.getElementsByTagName('img');
  const len = imgs.length;
  // 视口的高度
  const viewHeight = document.documentElement.clientHeight;
  // 滚动条高度
  const scrollHeight = document.documentElement.scrollTop || document.body.scrollTop;
  for (let i = 0; i < len; i++) {
    const offsetHeight = imgs[i].offsetTop;
    if (offsetHeight < viewHeight + scrollHeight) {
      const src = imgs[i].dataset.src;
      imgs[i].src = src;
    }
  }
}

// 可以使用节流优化一下
window.addEventListener('scroll', lazyload);
```

#### 第二种方式

> 使用 element.getBoundingClientRect() API 直接得到 top 值；

```javascript
function lazyload() {
  const imgs = document.getElementsByTagName('img');
  const len = imgs.length;
  // 视口的高度
  const viewHeight = document.documentElement.clientHeight;
  // 滚动条高度
  const scrollHeight = document.documentElement.scrollTop || document.body.scrollTop;
  for (let i = 0; i < len; i++) {
    if (imgs[i].getBoundingClientRect().top < viewHeight + scrollHeight) {
      const src = imgs[i].dataset.src;
      imgs[i].src = src;
    }
  }
}

// 可以使用节流优化一下
window.addEventListener('scroll', lazyload);
```

## 二分查找

```javascript
const bsearch = (array,target)=>{
  let l = 0;
  let r= array.length-1;

  let mid;
  while(l<=r){
    mid = Math.ceil((l+r)/2)
    if(array[mid] === target) return mid;
    if(array[mid] >target){
      r = mid-1
    }else{
      l = mid
    }
  }
  return -1
}
```

## 实现 AJAX

```javascript
function ajax() {
  return new Promise((resolve, reject) => {
    const xhr = XMLHttpRequest ? new XMLHttpRequest() : new ActiveXObject('Mscrosoft.XMLHttp');
    xhr.open('get', url, false);
    xhr.setRequestHeader('Accept', 'application/json');
    xhr.onreadystatechange = function () {
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

## 事件处理程序

IE 添加和删除事件处理程序的写法有点小区别；

```javascript
var EventUtil = {
    addHandler: function (el, type, handler) {
        if (el.addEventListener) {
            el.addEventListener(type, handler, false);
        } else {
            el.attachEvent('on' + type, handler);
        }
    },
    removeHandler: function (el, type, handler) {
        if (el.removeEventListener) {
            el.removeEventListerner(type, handler, false);
        } else {
            el.detachEvent('on' + type, handler);
        }
    }
};
```

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

## VirtualDom转换为DOM

```javascript
function render(vnode, container) {
  container.appendChild(_render(vnode));
}

function _render(vnode) {
  // 如果是数字类型转化为字符串
  if (typeof vnode === 'number') {
    vnode = String(vnode);
  }
  // 判断文本节点
  if (typeof vnode === 'string') {
    return document.createTextNode(vnode);
  }
  // 普通DOM
  const dom = document.createElement(vnode.tag);
  if (vnode.attrs) {
    // 遍历属性
    Object.keys(vnode.attrs).forEach(key => {
      const value = vnode.attrs[key];
      dom.setAttribute(key, value);
    })
  }
  // 子数组进行递归操作
  vnode.children.forEach(child => render(child, dom));
  return dom;
}
```

## 渲染几万条数据不卡页面

1. 合理使用 createDocumentFragment 和 requestAnimationFrame；
2. 将操作切分为一小段一小段执行；

```javascript
setTimeout(() => {
  // 插入十万条数据
  const total = 100000;
  // 一次插入的数据
  const once = 20;
  // 插入数据需要的次数
  const loopCount = Math.ceil(total / once);
  let countOfRender = 0;
  const ul = document.querySelector('ul');
  // 添加数据的方法
  function add() {
    const fragment = document.createDocumentFragment();
    for(let i = 0; i < once; i++) {
      const li = document.createElement('li');
      li.innerText = Math.floor(Math.random() * total);
      fragment.appendChild(li);
    }
    ul.appendChild(fragment);
    countOfRender += 1;
    loop();
  }
  function loop() {
    if(countOfRender < loopCount) {
      window.requestAnimationFrame(add);
    }
  }
  loop();
}, 0)

```

## 计时器实现任务调度

```javascript
class Cron {
  constructor() {
    this.map = new Map()
    this.timer = null
    this.interval = 60 * 60 * 1000
    this._initCron()
    this._runCron()
  }
  
  // 添加任务
  addSub (time, fn) {
    if (this.map.has(time)) {
      this.map.get(time).push(fn)
    }
  }
  
  // 执行任务
  _run (time) {
    if (this.map.has(time)) {
      this.map.get(time).forEach(fn => fn())
    }
  }
  
  // 初始化任务器
  _initCron() {
    for (let i = 0; i <= 23; i++) {
      this.map.set(i, [])
    }
  }
  
  // 重置
  resetCron () {
    for (let i = 0; i <= 23; i++) {
      this.map.set(0, [])
    }
  }
  
  // 每小时执行任务
  _runCron () {
    this.timer = setInterval(() => {
      const hour = (new Date()).getHours()
      this._run(hour)
    }, this.interval)
  }
  
  // 停止任务器
  stopCron () {
    clearInterval(this.timer)
  }
  runCron () {
    this._runCron()
  }
}

const cron = new Cron()
```