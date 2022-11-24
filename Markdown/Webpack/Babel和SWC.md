> Babel 是一个 JavaScript 编译器，主要用于将 ECMAScript 2015+ 版本代码向后兼容 Javascript 语法，以便可以运行到旧版本浏览器或其它环境中；

## Babel的作用 

- 语法转换（ES6转ES5，转 JSX 语法）
- 通过 Polyfill 方式在目标环境中添加缺失的特性（通过引入第三方polyfill模块，比如 core-js）
- Babel 支持 Source map，可以调试编译后的代码

### 处理步骤

#### 解析（parse）

将代码解析成抽象语法树（AST），先词法分析再语法分析，最终转换成 AST，使用 @babel/parser 解析代码，对不同词法添加不同type。

每个 js 引擎都有自己的 AST 解析器，Babel 是通过 BabyIon 实现的；

- 词法分析：词法分析阶段把字符串形式的代码转换为令牌（tokens）流，令牌类似于 AST 中节点；
- 语法分析：语法分析阶段会把一个令牌转换成 AST 的形式，同时这个阶段会把令牌中的信息转换成 AST 的表述结构；

#### 转换（transform）

在这个阶段，Babel 接收得到 AST 并通过 babel-traverse 对其进行深度优先遍历，在此过程中对节点进行添加、更新及移除操作；

这部分也是 Babel 插件介入工作的部分；

#### 生成（generator）

在这个阶段，将经过转换的 AST 通过 babel-generator 再转换成 js 代码，同时还会创建源码映射（source maps），过程就是深度优先遍历整个 AST ，然后构建可以表示转换后代码的字符串；

### 模块

#### babel-core

> babel的核心模块，包括一些核心api例如：transform；

```javascript
/*
 * @param {string} code 要转译的代码字符串
 * @param {object} options 可选，配置项
 * @return {object} 
*/
babel.transform(code: string, options?: Object)
    
//返回一个对象(主要包括三个部分)：
{
    generated code, //生成码
    sources map, //源映射
    AST  //即abstract syntax tree，抽象语法树
}
```

#### babel-cli

> Babel 的 CLI 是一种在命令行下使用 Babel 编译文件的简单方法，主要用于文件的输入输出；

```javascript
// 全局安装
npm install -g babel-cli

//编译后的文件输出在终端
babel test.js

//编译后的文件输出在test-out.js文件中
babel test.js -o test-out.js
```

#### babel-node

babel-node是随babel-cli一起安装的，只要安装了babel-cli就会自带babel-node；

 在命令行输入babel-node会启动一个REPL（Read-Eval-Print-Loop），这是一个支持ES6的js执行环境；

#### babel-register

> babel-register 是一个 babel 的注册器，它在底层改写了 node 的 require 方法；

引入 babel-register 之后所有 require 并以 .es6，.es，.jsx 和 .js 为后缀的模块都会经过 babel 的编译；

```javascript
//test.js
const name = 'test';
module.exports = () => {
    const json = {name};
    return json;
};
//main.js
require('babel-register');
var test = require('./test.js');  //test.js中的es6语法将被转译成es5

console.log(test.toString()); //通过toString方法，看看控制台输出的函数是否被转译
/*
  function () {
    var json = { name: name };
    return json;
  }
*/
```

#### babel-polyfill

> babel-polyfill 主要是用已经存在的语法和 api 实现一些浏览器还没有实现的 api，对浏览器的一些缺陷做一些修补；

### 项目使用

#### .babelrc

babel 所有的操作基本都会来读取这个配置文件，除了一些在回调函数中设置 options 参数的，如果没有这个配置文件，会从 package.json 文件的babel 属性中读取配置；

#### plugins

babel中的插件，通过配置不同的插件才能告诉babel，我们的代码中有哪些是需要转译的；

#### presets

> 预设就是一系列插件的集合；

好像修图一样，把上次修图的一些参数保存为一个预设，下次能直接使用；

最后一个参数useBuiltIns，有两点必须要注意：

- 如果 useBuiltIns 为 true，项目中必须引入babel-polyfill。
- babel-polyfill 只能被引入一次，多次引入会造成全局作用域的冲突；

```javascript
// cnpm install -D babel-preset -env
{
  "presets": [
    ["env", {
      // 指定要转译到哪个环境
      "targets": { 
          "browsers": ["last 2 versions", "safari >= 7"],
          "node": "6.10", //"current"  使用当前版本的node
      },
       // 是否将ES6的模块化语法转译成其他类型
       // 参数："amd" | "umd" | "systemjs" | "commonjs" | false，默认为'commonjs'
      "modules": 'commonjs',
      // 是否进行debug操作，会在控制台打印出所有插件中的log，已经插件的版本
      "debug": false,
      // 强制开启某些模块，默认为[]
      "include": ["transform-es2015-arrow-functions"],
      // 禁用某些模块，默认为[]
      "exclude": ["transform-es2015-for-of"],
      // 是否自动引入polyfill，开启此选项必须保证已经安装了babel-polyfill
      // 参数：Boolean，默认为false.
      "useBuiltIns": false
    }]
  ]
}
```

## SWC

> SWC（`Speedy Web Compiler`）是用 Rust 编写的超快 TypeScript / JavaScript 编译器。是一个社区驱动的项目。

SWC 的编译旨在支持所有 ECMAScript 功能。SWC CLI 旨在替代 Babel。可以说是更快的babel。

### SWC相比于Babel的优势

Babel是JavaScript写的，JavaScript就是有点慢。而swc也提供了对 webpack 良好支持，所以使用webpack + swc搭配没有任何问题。