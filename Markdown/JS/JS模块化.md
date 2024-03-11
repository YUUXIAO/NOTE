## 进化过程

1. 全局 function 模式：将不同的功能封装成不同的全局函数；

   - 污染全局全名空间，容易引起全名冲突或数据的不安全，模块成员之间看不出直接关系；

2. namespace 模式：简单对象封装

   - 作用：减少了全局变量，解决全名冲突；
   - 问题：数据不安全(外部可以直接修改模块内部的数据)；

   ```javascript
   let myModule = {
     data: 'www.baidu.com',
     foo() {
       console.log(`foo() ${this.data}`)
     },
     bar() {
       console.log(`bar() ${this.data}`)
     }
   }
   myModule.data = 'other data' //能直接修改模块内部的数据
   myModule.foo() // foo() other data
   ```

3. IIFE 模式：匿名函数自调用（闭包）

   - 数据是私有的，外部只能通过暴露的方法操作；
   - 将数据和行为封装到一个函数内部，通过给window添加属性来向外暴露接口；
   - 问题：如果当前模块依赖另一个模块怎么办？

   ```javascript
   // module.js文件
   (function(window) {
     let data = 'www.baidu.com'
     //操作数据的函数
     function foo() {
       //用于暴露有函数
       console.log(`foo() ${data}`)
     }
     function bar() {
       //用于暴露有函数
       console.log(`bar() ${data}`)
       otherFun() //内部调用
     }
     function otherFun() {
       //内部私有的函数
       console.log('otherFun()')
     }
     //暴露行为
     window.myModule = { foo, bar } //ES6写法
   })(window)
   
   // index.html文件
   myModule.foo()
   myModule.bar()
   console.log(myModule.data) //undefined 不能访问模块内部数据
   myModule.data = 'xxxx' //不是修改的模块内部的data
   myModule.foo() //没有改变
   ```


## 优势

1. 解决变量污染问题，每个文件都是独立的作用域，不存在变量污染；
2. 解决代码维护问题，一个文件里代码非常清晰；
3. 解决文件依赖问题，一个文件里可以清楚的看到依赖了那些其它文件；
4. 更好的分离，按需加载；
5. 更高的复用性；

## CommonJS

CommonJS规范加载模块是同步的，只有加载完成，才能执行后面的操作；

### 导出

> 使用 module.exports 导出变量及函数，可以导出任意类型的值，也可以省略 module 关键字，直接写 exports 导出也可以；

```javascript
// 导出一个对象
module.exports = {
    name: "蛙人",
    age: 24,
    sex: "male"
}

// 导出任意值
module.exports.name = "蛙人"
module.exports.sex = null
module.exports.age = undefined

exports.name = "蛙人"
exports.sex = "male"
```

### 引入

> 使用 require（xxx） 语法导入，如果想要单个的值，通过解构对象来获取；

- 如果是第三方模块，xxx为模块名；
- 如果是自定义模块，xxx为模块文件路径；

特点：

1. 所有代码都运行在模块作用域，不会污染全局作用域；
2. 模块可以多次加载，但是只会在第一次加载时运行一次，然后运行结果就被缓存了，以后再加载，就直接读取缓存结果；
3. 这种缓存方式是经过文件路径定位的，即使两个完全相同的文件，但是位于不同的路径下，会在缓存中维持两份；
4. 模块加载的顺序，按照其在代码中出现的顺序；

CommonJS 加载的是一个对象，该对象在脚本运行完才会生成，即在输入时是先加载整个模块，生成一个对象，然后再从这个对象上面读取方法，这种加载称为“运行时加载”；

```javascript
// CommonJS模块
let { stat, exists, readFile } = require('fs');

// 等同于
let _fs = require('fs');
let stat = _fs.stat;
let exists = _fs.exists;
let readfile = _fs.readfile;

// 上面代码的实质是整体加载fs模块（即加载fs的所有方法），生成一个对象，然后再从这个对象上面读取 3 个方法。因为只有运行时才能得到这个对象，导致完全没办法在编译时做“静态优化”;
```

### module

每个模块内部，都有一个module对象，代表当前模块，它有以下属性：

```javascript
module.id 模块的识别符，通常是带有绝对路径的模块文件名
module.filename 模块的文件名，带有绝对路径
module.loaded 返回一个布尔值，表示模块是否已经完成加载
module.parent 返回一个对象，表示调用该模块的模块
module.children 返回一个数组，表示该模块要用到的其他模块
module.exports 表示模块对外输出的值
```

### module.exports

> module.exports 属性表示当前模块对外输出的接口，其他文件加载该模块，实际上就是读取module.exports变量；

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

### exports

> Node 为每个模块提供一个 exports 变量，指向 module.exports；

exports 是对 module.exports 的一个引用，当 module.exports 指向变化，exports 导出就会出问题；

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

不能直接将 exports 变量指向一个值，因为这样等于切断了 exports 与module.exports 的联系；

```javascript
// 下面这样的写法是无效的，因为exports不再指向module.exports了
exports = function(x) {console.log(x)};

// hello函数是无法对外输出的，因为module.exports被重新赋值了；
exports.hello = function() {
  return 'hello';
};

module.exports = 'Hello world';
```

如果一个模块的对外接口，就是一个单一的值，不能使用exports输出，只能使用 module.exports 输出：

```javascript
module.exports = function (x){ console.log(x);};
```

### require( )

> require 命令的基本功能是，读入并执行一个 JavaScript 文件，然后返回该模块的 exports 对象，如果没有发现指定模块，会报错；

1. require 导入是在运行时，可以在任意地方调用 require 导入模块；
2. require( ) 的路径可以是表达式：require（’/app' + '/index'）;
3. require 导出的是 module.exports 对象指向的内存块内容；
4. require返回对应 module.exports 对象的浅拷贝；

   - 如果 module.exports 里的基本类型的值，会得到该值的副本；
   - 如果 module.exports 里的对象类型的值，会得到该值的引用；

5. Node.js 会自动缓存经过 require 引入的文件，使得下次再引入不需要经过文件系统而是直接从缓存中读取；不过这种缓存方式是经过文件路径定位的；
6. 可以通过 require.cache 获取目前在缓存中的所有文件；

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
// require命令调用自身，等于是执行module.exports，因此会输出 hello world

module.exports = function () {
  console.log("hello world")
}

require('./example2.js')()
```

### 加载机制

> CommonJS 模块的加载机制是，输入的是被输出的值的拷贝，一旦输出一个值，模块内部的变化就影响不到这个值；

```javascript
// lib.js
var counter = 3;
function incCounter() {
  counter++;
}
module.exports = {
  counter: counter,
  incCounter: incCounter,
};

// main.js
var counter = require('./lib').counter;
var incCounter = require('./lib').incCounter;

console.log(counter);  // 3
incCounter();
console.log(counter); // 3

// 上面代码说明，counter输出以后，lib.js模块内部的变化就影响不到counter了。
// 这是因为counter是一个原始类型的值，会被缓存。除非写成一个函数，才能得到内部变动后的值。
```

## AMD

> AMD 是 ”Asynchronous Module Definition” 的缩写，就是”异步模块定义”，AMD 是非同步加载模块，允许指定回调函数，浏览器端一般采用AMD规范；

AMD规范使用 define 方法定义模块：

```javascript
define(id?, dependencies?, factory)

-- id:字符串，模块名称(可选)
-- dependencies: 是我们要载入的依赖模块(可选)，使用相对路径,是数组格式
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

AMD 规范允许输出的模块兼容 CommonJS 规范，这时 define 方法需要写成下面这样：

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

### RequireJs

> RequrieJS 就是 AMD 现在用的最广泛，最流行的实现；

RequireJS 是一个工具库，主要用于客户端的模块管理，遵守AMD规范；

基本思想是，通过 define 方法，将代码定义为模块；

通过require方法，实现代码的模块加载；

```javascript
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

// 定义没有依赖的模块
define(function(){
   return 模块
})

// 定义有依赖的模块
define(['module1', 'module2'], function(m1, m2){
   return 模块
})

// 引入使用模块
require(['module1', 'module2'], function(m1, m2){
   //使用m1/m2
})
```

## CMD

> CMD 专门用于浏览器端，模块的加载是异步的，模块使用时才会加载执行；

### Sea.js

```javascript
//定义没有依赖的模块
define(function(require, exports, module){
  exports.xxx = value
  module.exports = value
})

//定义有依赖的模块
define(function(require, exports, module){
  //引入依赖模块(同步)
  var module2 = require('./module2')
    //引入依赖模块(异步)
    require.async('./module3', function (m3) {
    })
  //暴露模块
  exports.xxx = value
})

//引入使用模块
define(function (require) {
  var m1 = require('./module1')
  var m4 = require('./module4')
  m1.show()
  m4.show()
})
```

### CMD&AMD

两者都是异步加载，只是执行时机不一样：

1. AMD 依赖前置，提前执行：js 可以方便知道依赖模块是谁，立即加载；
2. CMD就近依赖，延迟执行：要使用把模块应为字符串解析一遍才知道依赖了哪些模块（牺牲性能带来开发的遍历性，实际上解析模块用的时间短到可以忽略）；

## UMD

> UMD 是 AMD 和 CommonJS 的糅合；

1. AMD模块以浏览器第一的原则发展，异步加载模块；
2. CommonJS 模块以服务器第一原则发展，选择同步加载，它的模块无需包装；
3. UMD 先判断是否支持 Node.js 的模块（exports）是否存在，存在则使用 Node.js 模块模式；
4. 再判断是否支持AMD（define是否存在），存在则使用 AMD 方式加载模块；

```javascript
(function (window, factory) {
    if (typeof exports === 'object') {

        module.exports = factory();
    } else if (typeof define === 'function' && define.amd) {

        define(factory);
    } else {

        window.eventUtil = factory();
    }
})(this, function () {
    //module ...
});
```

## ES6

> ES6 模块的设计思想是尽量的静态化，使得编译时就能确定模块的依赖关系，以及输入和输出的变量；

CommonJS 和 AMD 模块，都只能在运行时确定这些东西。比如，CommonJS 模块就是对象，输入时必须查找对象属性；

ES6 Module默认目前还没有被浏览器支持，需要使用 babel；

### import

> 静态的 import 语句用于导入由另一个模块导出的绑定，返回一个 Promise 对象；

无论是否声明了 strict mode，导入的模块都运行在严格模式下。

1. import 在编译时确定导入；
2. 路径只能是字符串；
3. import 会被提升到文件的最顶部；
4. 导入的变量是只读的；
5. import 导入的是值引用，而不是值拷贝；

   - 模块内部值发生变化，会对应影响到引用的地方；
   - import 导入与导出需要有一一映射关系，类似于解构赋值；


```javascript
// 报错
if (x === 2) {
  import MyModual from './myModual';
}
// 引擎处理import语句是在编译时，这时不会去分析或执行if语句，所以import语句放在if代码块之中毫无意    义，因此会报句法错误，而不是执行时错误。没办法像require样根据条件动态加载。
// ES2021 于是引入import()函数，编译时分析if语句,完成动态加载；

if(x === 2){
  import('myModual').then((MyModual)=>{
    new MyModual();
  })
}
```

#### 导入方式

两种方式导入：命名式导入（名称导入）和默认导入（定义式导入）；

```javascript
import defaultMember from "module-name"; // 默认导入
import * as name from "module-name";
import { member } from "module-name";
import { member as alias } from "module-name";
import { member1 , member2 } from "module-name";
import { member1 , member2 as alias2 , [...] } from "module-name";
import defaultMember, { member [ , [...] ] } from "module-name";
import defaultMember, * as name from "module-name";
import "module-name";
```

### export

> export 语句用于从模块中导出函数、对象或原始值；

#### 导出方式

两种方式导出：命名式导出（名称导出）和默认导出（定义式导出）;

- 命名式导出每个模块可以多个，而默认导出每个模块仅一个；

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

## ES6&CommonJS

1. CommonJS 模块输出的是一个值的拷贝，ES6 模块输出的是值的引用：

   - CommonJS 模一旦输出一个值，模块内部的变化就影响不到这个值;
   - ES6 模块是动态引用，不会缓存值，模块里面的变量绑定其所在的模块：当 JS 引擎遇到 ES6 模块加载命令import，就会生成一个只读引用，等到脚本真正执行时，再根据这个只读引用，到被加载的那个模块里面去取值。；

2. CommonJS 模块无论加载多少次，都只会在第一次加载时运行一次，以后返回的都是第一次运行结果的缓存，除非手动清除系统缓存；
3. CommonJs 可以动态加载语句，代码发生在运行时，ES6 是静态，只能声明在该文件的最顶部，代码发生在编译时；

## require&import

1. import 是ES6标准中的模块化解决方案，require 是 node 中遵循CommonJS 规范的模块化解决方案；
2. import 语句导入同一个模块如果加载多次只执行一次，require 语句导入次数和实际执行次数相同；
3. import 必须用在当前模块的顶层，如果在局部作用域内，会报错，es6这样的设计可以提高编译器效率，但没法实现运行时加载；require 可以用在代码的任何地方；
4. require 支持动态引入，也就是require(${path}/xx.js)，import 目前不支持，但是已有提案；
5. import 可以使用 import * 引入全部的export，也可以使用 import aaa, { bbb} 的方式分别引入 default 和非 default 的export，相比 require 更灵活；
6. ES6模块之中，顶层的 this 关键字返回 undefined，而不是指向 window；

- 利用顶层的 this 等于 undefined 这个语法点，可以侦测当前代码是否在 ES6 模块之中；

````
const isNotModuleScript = this !== undefined
````

- require的模块中 this 指向 window；

## 总结

1. CommonJS 规范主要用于服务端编程，加载模块是同步的，同步意味着阻塞加载，浏览器资源是异步加载的，因此有了AMD、CMD解决方案；
2. AMD 规范在浏览器环境中异步加载模块，可以并行加载多个模块，但AMD规范开发成本高，代码的阅读和书写比较困难，模块定义方式的语义不顺畅；
3. CMD 规范与 AMD 规范很相似，都用于浏览器编程，依赖就近，延迟执行，可以很容易在 Node.js 中运行，但依赖SPM打包，模块的加载逻辑偏重；
4. ES6 在语言标准的层面上，实现了模块功能，实现简单，可以取代 CommonJS 和 AMD 规范，成为浏览器和服务器通用的模块解决方案；