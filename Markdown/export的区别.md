## require

> Node.js 遵循 CommonJS 规范，使用 require 关键字来加载模块

### 缓存策略

> Node.js 会自动缓存经过 require 引入的文件，使得下次再引入不需要经过文件系统而是直接从缓存中读取。
>
> 不过这种缓存方式是经过文件路径定位的，即使两个完全相同的文件，但是位于不同的路径下，会在缓存中维持两份。 
>
> 可以通过 require.cache 获取目前在缓存中的所有文件

#### 重复引入问题

因为 Node.js 默认先从缓存中加载模块，一个模块被加载一次之后，就会在缓存中维持一个副本，如果遇到重复加载的模块会直接提取缓存中的副本，也就是说在任何时候每个模块都只在缓存中有一个实例

####  加载模块的时候是同步的

1. 一个作为公共依赖的模块，当然想一次加载出来，同步更好
2. 模块的个数往往是有限的，而且 Node.js 在 require 的时候会自动缓存已经加载的模块，再加上访问的都是本地文件，产生的IO开销几乎可以忽略。



#### require和import的区别

1. require是 commonjs 的规范，在node中实现的api，import是es的语法，由编译器处理。所以import可以做模块依赖的静态分析，配合webpack、rollup等可以做treeshaking。
2. commonjs导出的值会复制一份，require引入的是复制之后的值（引用类型只复制引用），es module导出的值是同一份（不包括export default），不管是基础类型还是应用类型。
3. 写法上有差别，import可以使用import * 引入全部的export，也可以使用import aaa, { bbb}的方式分别引入default和非default的export，相比require更灵活。

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

