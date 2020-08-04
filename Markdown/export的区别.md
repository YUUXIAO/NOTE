## CommonJS规范

- Node 应用由模块组成，采用 CommonJS 模块规范。
- 每个文件就是一个模块，有自己的作用域。在一个文件里面定义的变量、函数、类，都是私有的，对其它文件不可见。
- CommonJS规范规定，每个模块内部，module变量代表当前模块：这个变量是一个对象，它的exports属性是对外的接口。加载某个模块，其实是加载该模块的module.exports属性。

```javascript
//  下面代码通过module.exports输出变量x和函数addX。

var x = 5;
var addX = function (value) {
  return value + x;
};
module.exports.x = x;
module.exports.addX = addX;
```

CommonJS模块的特点如下：

> 1. 所有代码都运行在模块作用域，不会污染全局作用域；
> 2.  模块可以多次加载，但是只会在第一次加载时运行一次，然后运行结果就被缓存了，以后再加载，就直接读取缓存结果
> 3. 不过这种缓存方式是经过文件路径定位的，即使两个完全相同的文件，但是位于不同的路径下，会在缓存中维持两份。 
> 4. 要想让模块再次运行，必须清除缓存；
> 5. 模块加载的顺序，按照其在代码中出现的顺序；

### module对象

> Node内部提供一个Module构建函数；
>
> 所有模块都是Module的实例；

每个模块内部，都有一个module对象，代表当前模块。它有以下属性：

```
module.id 模块的识别符，通常是带有绝对路径的模块文件名。
module.filename 模块的文件名，带有绝对路径。
module.loaded 返回一个布尔值，表示模块是否已经完成加载。
module.parent 返回一个对象，表示调用该模块的模块。
module.children 返回一个数组，表示该模块要用到的其他模块。
module.exports 表示模块对外输出的值。
```

### module.exports属性

> module.exports属性表示当前模块对外输出的接口，其他文件加载该模块，实际上就是读取module.exports变量；

```javascript
var EventEmitter = require('events').EventEmitter;
module.exports = new EventEmitter();

setTimeout(function() {
  module.exports.emit('ready');
}, 1000);
```

```javascript
// 上面模块会在加载后1秒后，发出ready事件。其他文件监听该事件，可以写成下面这样:

var a = require('./a');
a.on('ready', function() {
  console.log('module a is ready');
});
```

### exports变量

> Node为每个模块提供一个exports变量，指向module.exports；

```javascript
// 这等同在每个模块头部，有一行这样的命令:

var exports = module.exports;
```

这样在对外输出模块接口时，可以向exports对象添加方法；

```javascript
exports.area = function (r) {
  return Math.PI * r * r;
};

exports.circumference = function (r) {
  return 2 * Math.PI * r;
};
```

```javascript
// 不能直接将exports变量指向一个值，因为这样等于切断了exports与module.exports的联系
// 下面这样的写法是无效的，因为exports不再指向module.exports了

exports = function(x) {console.log(x)};

// 下面代码中，hello函数是无法对外输出的，因为module.exports被重新赋值了。

exports.hello = function() {
  return 'hello';
};

module.exports = 'Hello world';
```

这意味着，如果一个模块的对外接口，就是一个单一的值，不能使用exports输出，只能使用module.exports输出：

```javascript
module.exports = function (x){ console.log(x);};
```

### require( )函数

> Node使用CommonJS模块规范，内置的require命令用于加载模块文件。
>
> require 命令的基本功能是，读入并执行一个 JavaScript 文件，然后返回该模块的 exports 对象，如果没有发现指定模块，会报错；

1. require导入是在运行时，理论上可以在任意地方调用require导入模块；
2. require( )的路径可以是表达式：require（’/app' + '/index'）;
3. require 导出的是 module.exports 对象；
4. exports 是对 module.exports 的一个引用，当 module.exports 指向变化，exports 导出就会出问题；
5. require返回对应module.exports对象的浅拷贝；
   - 如果是module.exports里的基本类型的值，会得到该值的副本；
   - 如果是module.exports里的对象类型的值，会得到该值的引用；

代码示例：

```javascript
// example.js
var invisible = function () {
  console.log("invisible");
}

exports.message = "hi";
exports.say = function () {
  console.log(message);
}

// 运行下面的命令，可以输出exports对象
var example = require('./example.js');

// { 
//   message: "hi",
//   say: [Function]
// }
```

如果模块输出的是一个函数，那就不能定义在exports对象上面，而要定义在module.exports变量上面；

```javascript
// 下面代码中，require命令调用自身，等于是执行module.exports，因此会输出 hello world

module.exports = function () {
  console.log("hello world")
}

require('./example2.js')()
```



## AMD规范与CommonJS规范

> ```
> CommonJS规范加载模块是同步的，也就是说，只有加载完成才能执行后面的操作；
> AMD规范则是非同步加载模块，允许指定回调函数；
> ```

由于Node.js主要用于服务器编程，模块文件一般都已经存在于本地硬盘，所以加载起来比较快，不用考虑非同步加载的方式，所以CommonJS规范比较适用。

但是，如果是浏览器环境，要从服务器端加载模块，这时就必须采用非同步模式，因此浏览器端一般采用AMD规范。

AMD规范使用define方法定义模块：

```javascript
define(id?, dependencies?, factory)

-- id:字符串，模块名称(可选)
-- dependencies: 是我们要载入的依赖模块(可选)，使用相对路径。,注意是数组格式
-- factory: 工厂方法，返回一个模块函数
```

```javascript
define(['package/lib'], function(lib){
  function foo(){
    lib.log('hello world!');
  }

  return {
    foo: foo
  };
});
```

AMD规范允许输出的模块兼容 CommonJS 规范，这时define方法需要写成下面这样：

```javascript
define(function (require, exports, module){
  var someModule = require("someModule");
  var anotherModule = require("anotherModule");

  someModule.doTehAwesome();
  anotherModule.doMoarAwesome();

  exports.asplode = function (){
    someModule.doTehAwesome();
    anotherModule.doMoarAwesome();
  };
});
```

## requireJs

 特点：

```
1. 运行时加载
2. 拷贝到本页面
3. 全部引入
```

### 用法

require.js的诞生，就是为了解决这两个问题： 

```
1. 实现js文件的异步加载，避免网页失去响应；
2. 管理模块之间的依赖性，便于代码的编写和维护；
```



```javascript
// index.html
<script src="https://cdn.bootcss.com/require.js/2.3.6/require.min.js" data-main="js/main"></script>

// main.js
// RequireJS全局配置文件
requirejs.config({
    //  设置项目路径，项目会以baseUrl作为相对路径去查找模块文件
    baseUrl:"./js",
    //  预加载JS文件的配置项，默认可不用添加.js后缀
    paths:{
        //RequireJS默认假定所有的依赖资源都是JS脚本，因此无需再module ID上再加上js后缀。
        jquery:"https://cdn.bootcss.com/require.js/2.3.6/require.min.js",
        boostrap: "bootstrap"
    }
});

// 引用配置好的模块
requirejs(['jquery', 'bootstrap'],function($, undefined){

});

// 引用自定义模块
require(['js/test2.js'], function(math) {
   console.log(math);
 });

// test2.js 自定义模块
define(function() {
  var add = (x, y) => {
    return x + y;
  };
  return {
    add: add
  }
});
// 如果有依赖define(['jquery'], function($) {return {}});// 自定义命名define('math/add', ['jquery'], function($) {return {add: XXX}});
```

## AMD

> AMD是 ”Asynchronous Module Definition” 的缩写，意思就是”异步模块定义”。
>
> 它采用异步方式加载模块，模块的加载不影响它后面语句的运行；所有依赖这个模块的语句，都定义在一个回调函数中，等到加载完成之后，这个回调函数才会运行。

第二行math.add(2, 3)，在第一行require(‘math’)之后运行，因此必须等math.js加载完成。也就是说，如果加载时间很长，整个应用就会停在那里等。
这对服务器端不是一个问题，因为所有的模块都存放在本地硬盘，可以同步加载完成，等待时间就是硬盘的读取时间。但是，对于浏览器，这却是一个大问题，因为模块都放在服务器端，等待时间取决于网速的快慢，可能要等很长时间，浏览器处于”假死”状态。
因此，浏览器端的模块，不能采用”同步加载”，只能采用”异步加载”；这就是AMD规范诞生的背景。

```javascript
//require([module], callback);
require(['math'], function (math) {
	math.add(2, 3);
});
```

## import

> 静态的 import 语句用于导入由另一个模块导出的绑定。
>
> 无论是否声明了 strict mode，导入的模块都运行在严格模式下。
>
> 可对比 require 的特点，发现 import 完胜 require ,推荐用 import 取代 require

### 特点

1. 编译时加载
2. 只引用定义 
3. 按需加载

### 用法

> import有两种模块导入方式：命名式导入（名称导入）和默认导入（定义式导入），以及 import( )

```javascript
import defaultMember from "module-name";
import * as name from "module-name";
import { member } from "module-name";
import { member as alias } from "module-name";
import { member1 , member2 } from "module-name";
import { member1 , member2 as alias2 , [...] } from "module-name";
import defaultMember, { member [ , [...] ] } from "module-name";
import defaultMember, * as name from "module-name";
import "module-name";
```

import( )返回一个 Promise 对象；

> **export **语句用于从模块中导出函数、对象或原始值；
>
> **export**有两种模块导出方式：命名式导出（名称导出）和默认导出（定义式导出）;
>
> 命名式导出每个模块可以多个，而默认导出每个模块仅一个；

```javascript
export { name1, name2, …, nameN };
export { variable1 as name1, variable2 as name2, …, nameN };
export let name1, name2, …, nameN; 
export let name1 = …, name2 = …, …, nameN; 
 
export default expression;
export default function (…) { … } 
export default function name1(…) { … } 
export { name1 as default, … };
 
export * from …;
export { name1, name2, …, nameN } from …;
export { import1 as name1, import2 as name2, …, nameN } from …;

```

```javascript
// 报错
if (x === 2) {
  import MyModual from './myModual';
}

// 引擎处理import语句是在编译时，这时不会去分析或执行if语句，所以import语句放在if代码块之中毫无意    义，因此会报句法错误，而不是执行时错误。没办法像require样根据条件动态加载。
// 于是提案引入import()函数，编译时分析if语句,完成动态加载。

if(x === 2){
  import('myModual').then((MyModual)=>{
    new MyModual();
  })
}
```



### 缓存策略

> Node.js 会自动缓存经过 require 引入的文件，使得下次再引入不需要经过文件系统而是直接从缓存中读取。
>
> 不过这种缓存方式是经过文件路径定位的，即使两个完全相同的文件，但是位于不同的路径下，会在缓存中维持两份。 
>
> 可以通过 require.cache 获取目前在缓存中的所有文件

## 常见问题

#### require和import的区别

1.  import是ES6标准中的模块化解决方案（因为浏览器支持情况不同，项目中本质是使用node中的babel将es6转码为es5再执行，import会被转码为require）

    require是node中遵循CommonJS规范的模块化解决方案

2.  ES6模块是编译时输出接口，CommonJS模块是运行时加载

3.  ES6模块是动态引用：即导入和导出的值都指向同一个内存地址，所以导入的值会随着导出值变化。

   CommonJs的模块是对象：导出时是值拷贝，就算导出的值变化了，导入的值也不会变化，如果想要更新值就要重新导入

4. import语句导入同一个模块如果加载多次只执行一次，require语句导入次数和实际执行次数相同

5. import必须用在当前模块的顶层，如果在局部作用域内，会报错，es6这样的设计可以提高编译器效率，但没法实现运行时加载

   require可以用在代码的任何地方

6. require支持动态引入，也就是require(${path}/xx.js)，import目前不支持，但是已有提案

7. commonjs导出的值会复制一份，require引入的是复制之后的值（引用类型只复制引用），es module导出的值是同一份（不包括export default），不管是基础类型还是应用类型。

8. 写法上有差别，import可以使用import * 引入全部的export，也可以使用import aaa, { bbb}的方式分别引入default和非default的export，相比require更灵活。

9. 由于有了 babel 将还未被宿主环境（各浏览器、Node.js）直接支持的 ES6 Module 编译为 ES5 的 CommonJS —— 也就是 require/exports 这种写法 ；这也就是说 require/exports 是必要且必须的。因为 import/export 最终都是编译为 require/exports 来执行的；

   这也就是为什么说 require/exports 是必要且必须的。因为 import/export 最终都是编译为 require/exports 来执行的；

10. . ES6模块之中，顶层的 this 关键字返回 undefined，而不是指向 window。

    也就是说，在模块顶层使用 this 关键字，是无意义的：利用顶层的 this 等于 undefined 这个语法点，可以侦测当前代码是否在 ES6 模块之中。

    ```
    const isNotModuleScript = this !== undefined
    ```

    require的模块中this指向window,this === window.

#### 重复引入问题

因为 Node.js 默认先从缓存中加载模块，一个模块被加载一次之后，就会在缓存中维持一个副本，如果遇到重复加载的模块会直接提取缓存中的副本，也就是说在任何时候每个模块都只在缓存中有一个实例

####  加载模块的时候是同步的

1. 一个作为公共依赖的模块，当然想一次加载出来，同步更好
2. 模块的个数往往是有限的，而且 Node.js 在 require 的时候会自动缓存已经加载的模块，再加上访问的都是本地文件，产生的IO开销几乎可以忽略。


1. ​

#### require和import会不会循环引用

答案是不会，因为模块执行后会把导出的值缓存，下次再require或者import不会再次执行。这样也就不会循环引用了。

比如a引入了b，b引入了a，如果a再次执行那么会再引入b，那就循环起来了，但实际上会做缓存，再次引入不会再执行。可以通过require.cache来查看缓存的模块，key为require.resolve(path)的结果。

## exports 与 module.exports 区别

#### js文件启动时

在一个 node 执行一个文件时，会给这个文件内生成一个 exports 和 module 对象， 而module又有一个 exports 属性。他们之间的关系如下图，都指向一块{}内存区域。

```javascript
exports = module.exports = {};
```

![img](https://user-gold-cdn.xitu.io/2019/8/16/16c98da07872f7f3?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

#### require()加载模块

> require导出的内容是module.exports的指向的内存块内容，并不是exports的
>
> 简而言之，区分他们之间的区别就是 exports 只是 module.exports的引用，辅助后者添加内容用的
>
> 用内存指向的方式更好理解。

```javascript
//koala.js
let a = '程序员';

console.log(module.exports); //能打印出结果为：{}
console.log(exports); //能打印出结果为：{}

exports.a = '程序员成长'; //这里把 module.exports 的内容给改成 {a : '程序员成长'}

exports = '指向其他内存区'; //这里把exports的指向指走

//test.js

const a = require('/koala');
console.log(a) // 打印为 {a : '程序员成长'}

```

我们经常看到这样的写法：

```javascript
exports = module.exports = somethings
```

上面的代码等价于:

```javascript
module.exports = somethings
exports = module.exports
```

原理很简单，即 module.exports 指向新的对象时，exports 断开了与 module.exports 的引用，那么通过 exports = module.exports 让 exports 重新指向 module.exports 。

## 部分导出和部分导入

> 当资源体积较大时使用部分导出，可以减少使用者可以部分导入来减少资源体积

#### 部分导出

```javascript
export function helloWorld(){
    conselo.log("Hello World");
}
export function test(){
    conselo.log("this's test function");
}

// 另一种写法，这种方法比较不推荐，因为看起来会比较乱

var helloWorld=function(){
    conselo.log("Hello World");
}
var test=function(){
    conselo.log("this's test function");
}

export helloWorld
export test
```

#### 部分导入

```javascript
// 只导入utils.js中的helloWorld方法
import {helloWorld} from "./utils.js" 

//  执行utils.js中的helloWorld方法
helloWorld(); 
```

#### 部分导出—全部导入

```javascript
//  导入全部的资源，utils为别名，在调用时使用
import * as utils from "./utils.js" 

//执行utils.js中的helloWorld方法
utils.helloWorld(); 
```

## 全部导出和全部导入

> 如果使用全部导出，那么使用者在导入时则必须全部导入，推荐在写方法库时使用部分导出，从而将全部导入或者部分导入的权力留给使用者。
>
> 一个js文件中可以有多个 export ，但只能有一个 export default

#### 全部导出

```javascript
var helloWorld=function(){
    conselo.log("Hello World");
}
var test=function(){
    conselo.log("this's test function");
}
export default{
    helloWorld,
    test
} 
```

#### 全部导入

```javascript
import utils from "./utils.js"

utils.helloWorld();
```

## 注意

模块中通过 export 导出的(属性或者方法)可以修改，但是通过 export default 导出的不可以修改【只能在模块内部修改，不能在导入模块的地方修改】

1. ES6中模块通过对 export 和 export default 暴露出来的属性或者方式不是普通的赋值或者引用，它们是对模块内部定义的标志符类似指针的绑定 
2. 对于一个导出的属性或者方法，在什么地方导出或者导入都不重要，重要的是访问这个绑定的时候的当前值 
3. export 是绑定到标识符，改变标志符的值，然后访问这个绑定，得到的是新值；
4. export default 绑定的是标志符指向的值，如果修改标志符指向另一个值，这个绑定的值不会发生变化
5. 如果想修改默认导出的值，可以使用 export {e1 as default} 这种方法
6. export default 和 export 导出的都是可以修改的：区别在于 export default 如果导出的是对象，则只能修改该对象下的属性，如果导出的是基本类型，则不能修改。export 不论导出的是对象还是基本类型，都是可以修改的。

示例：

```javascript
//  model.js
let e1='export 1';
let e2='export 2';
export {e2};
export default e1;

e1='export 1 modified';
setTimeout(()=>{
    e2='export 2 modified';
},1000);

//  index.js
import e1, {e2}from "./model";

console.log(e1);		// export 1
console.log(e2);		// export 2
setTimeout(()=>{
    console.log('later',e2)		// later export 2 modified
},5000);
```

```javascript
//  model.js修改
export {e1 as default}

//  index.js执行结果
export 1 modified
export 2
later export 2 modified
```

