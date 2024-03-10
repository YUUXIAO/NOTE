> Babel 是一个JS编译器，主要用于将 ES6+ 版本代码向后兼容JS语法，以便可以运行到旧版本浏览器或其它环境中；

## Babel的作用

- **语法转换**（ES6转ES5，转 JSX 语法）
- 通过 **Polyfill** 方式在目标环境中不支持的ES6+的API：比如旧版浏览器不支持的 Promise、Array.prototype.includes等；
- Babel 支持 **Source map**，可以调试编译后的代码

### 处理步骤

#### 解析（parse）

将代码解析成抽象语法树（AST），先词法分析再语法分析，最终转换成 AST，使用 @babel/parser 解析代码，对不同词法添加不同type。

每个JS引擎都有自己的 AST 解析器，Babel 是通过 BabyIon 实现的；

- **词法分析：**词法分析阶段把字符串形式的代码转换为令牌（tokens）流，令牌类似于 AST 中节点；
- **语法分析：**语法分析阶段会把一个令牌转换成 AST 的形式，同时这个阶段会把令牌中的信息转换成 AST 的表述结构；

#### 转换（transform）

在这个阶段，Babel 接收得到 AST 并通过 babel-traverse 对其进行深度优先遍历，在此过程中对节点进行添加、更新及移除操作；

这部分也是 Babel 插件介入工作的部分；

#### 生成（generator）

在这个阶段，将经过转换的 AST 通过 babel-generator 再转换成 js 代码，同时还会创建源码映射（source maps），过程就是深度优先遍历整个 AST ，然后构建可以表示转换后代码的字符串；

### 模块

#### [@babel/core](https://babeljs.io/docs/babel-core)

Babel的核心模块，它是Babel实现编译的核心，一般我们使用 Babel这个包是必不可少的

```javascript
var babel = require("@babel/core");

/*
 * @param {string} code 要转译的代码字符串
 * @param {object} options 可选，配置项
 * @return {object} 
*/
babel.transform(code, options, function(err, result) {
  result; // => { code, map, ast }
  // 返回一个对象(主要包括三个部分)：
  // {
    // generated code, //生成码
    // sources map, //源映射
    // AST  //即abstract syntax tree，抽象语法树
  // }
});

```

#### [@babel/cli](https://babeljs.io/docs/babel-cli)

Babel自带的一个内置的 CLI 命令工具， 主要用于文件的输入输出；

```javascript
// 安装
npm install --save-dev @babel/core @babel/cli
// 使用
npx babel xxx.js
```

#### @babel/node

babel-node是随babel-cli一起安装的，只要安装了babel-cli就会自带babel-node；

在命令行输入babel-node会启动一个REPL（Read-Eval-Print-Loop），这是一个支持ES6的js执行环境；

#### @babel/register

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

#### [@babel/polyfill](https://babeljs.io/docs/options#passperpreset)

> babel-polyfill 主要是用已经存在的语法和 api 实现一些浏览器还没有实现的 api，对浏览器的一些缺陷做一些修补；

- 这个包由core-js（版本为2.x.x）与regenerator-runtime两个包组成
- 这个包在**Babel 7.4.0以后就废弃**了，在之后的半们，如果想让一些不支持ES6+ API的旧版本浏览器支持这些API，应该直接安装[core-js@3.x.x](mailto:core-js@3.x.x)的包

  ```javascript
  import "core-js/stable";
  ```


### 项目使用

### 配置文件

babel 所有的操作基本都会来读取.babelrc配置文件，除了一些在回调函数中设置 options 参数的，如果没有这个配置文件，会从 package.json 文件的babel 属性中读取配置；

### plugins(插件)

> Babel 插件一般分为两种：语法插件和转换插件

**语法插件：**允许 babel 解析（parse）特定类型的语法（注意这里是语法，不包含新出的 api），可以在 AST 转换时使用，以支持解析新语法

```javascript
import * as babel from "@babel/core";
const code = babel.transformFromAstSync(ast, {
    // 支持箭头函数
    plugins: ["@babel/plugin-transform-es2015-arrow-functions"],
    babelrc: false
}).code;
```

Babel 插件一般**尽可能拆成小的粒度**，开发者可以按需引进：比如对ES6转ES5的功能，Babel官方拆成了20+个插件，比如开发者想要体验ES6的箭头函数特性，那他只需要引入transform-es2015-arrow-functions插件就可以，安装依赖：

```javascript
npm install --save-dev babel-cli 
npm install --save-dev babel-plugin-transform-es2015-arrow-functions
```

执行转换：

```javascript
`npm bin`/babel --plugins babel-plugin-transform-es2015-arrow-functions index.js
```

### presets(预设)

> Present 可以称为是一些功能的 Plugins 的合集，比如 babel-preser-es1025 就包含了所有跟 ES6  转换有关的插件；

```javascript
npm install --save-dev babel-cli 
npm install --save-dev babel-preset-es2015
```

最后一个参数useBuiltIns，有两点必须要注意：

- 如果 useBuiltIns 为 true，项目中必须引入babel-polyfill。
- babel-polyfill 只能被引入一次，多次引入会造成全局作用域的冲突；

```javascript
{
  "presets": [
    ["env", {
      // 指定要转译到哪个环境
      "targets": { 
          "browsers": ["last 2 versions", "safari >= 7"],
          "node": "6.10", //"current"  使用当前版本的node
      },
       // 是否将ES6的模块化语法转译成其他类型
      "modules": 'commonjs',
      // 是否进行debug操作，会在控制台打印出所有插件中的log，已经插件的版本
      "debug": false,
      // 强制开启某些模块，默认为[]
      "include": ["transform-es2015-arrow-functions"],
      // 禁用某些模块，默认为[]
      "exclude": ["transform-es2015-for-of"],
      // 配置我们设置 core-js 的垫平方式的
      // usage：不需要手动import，它会自动import当前targets缺失的polyfill
      // false：它表示不要在每个文件中自动添加polyfill，也不会根据targets判断缺不缺失
      // entry：手动import所有或者某块polyfill
      "useBuiltIns": false
    }]
  ]
}
```

#### @babel/preset-env（预设+环境）

@babel/preset-env 主要是下面两个功能：

- 它只编译ES6+语法；
- 它并不提供polyfill，但是可以通过**配置**代码运行的**目标环境**，使`ES6+`的新特性可以在我们想要的**目标环境**中顺利运行；
- 在**不进行任何配置**的情况下，@babel/preset-env 所包含的插件将**支持所有最新的JS特性**（不包含 stage 阶段），将其转换成 ES5 代码

配置项大致包括：

- **需要支持的平台：**比如 node、浏览器等，如果不是项目需要适用所有平台，建议指定目标环境，这样可以保证编译代码最小
- **需要支持的平台的版本：**比如支持 node@6.1等；

```javascript
// 默认设置，跟babel-preset-latest是等同的，会加载从es2015开始的所有preset
{
  "presets": ["env"]，
}
// 针对浏览器版本，比如需要支持 IE8+、chrome62+
{
  "presets": [
    ["env", {
      "targets": {
        "browsers": [ "ie >= 8", "chrome >= 62" ]
      }      
    }]
  ]
}

// 针对node版本的配置，node@8.9.3 已经支持了async/await ，所以在以上版本我们不需要转码来兼容代码
{
  "presets": [
    ["env", {
      "targets": {
        "node": "8.9.3"
      }      
    }]
  ]
}
```

#### modules选项

modules选项主要是用来启用ES模块语法向另一种模块类型的转换，可取的值："amd" | "umd" | "systemjs" | "commonjs" | "cjs" | "auto" | false

- 当我们设置为 false的时候，Babel编译产生的一些辅助函数的引入方式会变成ES6的模式引入，这样的的好处就是如果使用 Webpack 打包工具是，可以对代码进行静态分析，很好地 tree shaking 代码
- 默认的时候是'commonjs'，这样编译后的文件是通过 require引入的

#### 实现原理

1. **检测浏览器对JS特性的支持程度**，比如通过通过 compat-table 这样的外部数据；
2. 将 JS特性跟**特定的babel插件建立映射**；
3. stage-x 的插件不包括在内；
4. 根据**开发者的配置项，确定至少需要包含哪些插件**：比如声明了需要支持 IE8+、chrome62+，那么，所有IE8+需要的插件都会被包含进去；

### Plugin与Preset执行顺序

可以同时使用多个 Plugin 和 Preset ，此时它们的执行顺序为：

1. 先执行完所有 Plugin，再执行 Preset；
2. 多个 Plugin，按照声明次序顺序进行；
3. 多个 Preset，按照声明次序逆序进行；

比如 .babelrc 配置如下，那么执行的顺序为：

1. Plugin：transform-react-jsx、transform-async-to-generator
2. Preset：es2016、es2015

```javascript
{
  "plugins": [ 
    "transform-react-jsx",
    "transform-async-to-generator"
  ],
  "presets": [ 
    "es2015",
    "es2016"    
  ]
}
```

## 参考

[掘金这位大佬写的很好](https://juejin.cn/post/7197666704435920957#heading-15)

