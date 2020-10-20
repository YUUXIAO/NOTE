> Babel 是代码转换器，实现 Babel 代码转换功能的核心就是 Babel 插件（Plugin）;
>
> 原始代码 --> [ Babel Plugin ] --> 转换后的代码；

## Babel Plugin

Babel插件一般尽可能拆成小的力度，开发者可以按需引进：比如对ES6转ES5的功能，Babel官方拆成了20+个插件，比如开发者想要体验ES6的箭头函数特性，那他只需要引入transform-es2015-arrow-functions插件就可以，而不是加载ES6全家桶；

比如将 ES6 代码转成 ES5：

```javascript
// 箭头函数
[1,2,3].map(n => n + 1);

// 模板字面量
let nick = '程序猿小卡';
let desc = `hello ${nick}`;
```

安装依赖：

```javascript
npm install --save-dev babel-cli 
npm install --save-dev babel-plugin-transform-es2015-arrow-functions
npm install --save-dev babel-plugin-babel-plugin-transform-es2015-template-literals
```

执行转换：

```javascript
`npm bin`/babel --plugins babel-plugin-transform-es2015-arrow-functions,babel-plugin-transform-es2015-template-literals index.js
```

## Babel Preset

> Present 可以称为是一些功能的 Plugins 的合集，比如 babel-preser-es1025 就包含了所有跟 ES6  转换有关的插件；

以上个例子为例：

安装依赖：

```javascript
npm install --save-dev babel-cli 
npm install --save-dev babel-preset-es2015
```

执行转换：

```javascript
`npm bin`/babel --presets babel-preset-es2015 index.js
```

preset可以分为下面几种：

1. 按照官方内容：env，react，flow，minify；
2. stag-x；

## Babel-preset-lates（已弃用 ）

最初为了让开发者能够尽早用上新的JS特性，babel 团队开发了 babel-preset-latest , 它是多个 preset 的集合（es 2015+）,并且随着 ECMA 规范的更新增加它的内容；

- 特点：包含了所有年度预设（babel-preset-es2015、babel-preset-es2016、babel-preset-es2017），无需用户单独指定某个预设；
- 缺点：加载的插件越来越多，编译速度会越来越慢；随着用户浏览器的升级，ECMA规范的支持逐步完善，编译至低版本规范的必要性在减少，多余的转换不单降低执行效率，还浪费带宽；

## Babel-preset-env

babel-preset-env 可以根据开发者的配置，按需加载插件，配置项大致包括：

- 需要支持的平台：比如 node、浏览器等；
- 需要支持的平台的版本：比如支持 node@6.1等；

默认配置的情况下，它跟babel-preset-latest是等同的，会加载从es2015开始的所有preset：

```javascript
//默认设置
{
  "presets": ["env"]
}
```

示例代码：

```javascript
// index.js
async function foo () {}
```

#### 针对node版本的配置

因为 node@8.9.3 已经支持了 async/await ，所以在以上版本我们不需要转码来兼容代码，修改 .babelrc，加上配置参数 target ,它表示我们需要支持哪些平台 + 哪些版本；

```javascript
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

#### 针对浏览器版本的配置

如果我们需要支持 IE8+、chrome62+，那么可以这样配置：

```javascript
{
  "presets": [
    ["env", {
      "targets": {
        "browsers": [ "ie >= 8", "chrome >= 62" ]
      }      
    }]
  ]
}
```

#### 实现原理

1. 检测浏览器对JS特性的支持程度，比如通过通过 compat-table 这样的外部数据；
2. 将 JS特性 跟 特定的babel插件 建立映射；
3. stage-x 的插件不包括在内；
4. 根据开发者的配置项，确定至少需要包含哪些插件。比如声明了需要支持 IE8+、chrome62+，那么，所有IE8+需要的插件都会被包含进去；

## stage-x（实验阶段 presets）

stage-0 至 stage-3代表了es标准支持的不同阶段：

- 0 阶段是最初级阶段，可以理解为才刚刚开始讨论标准，基本没有什么浏览器支持 es 新标准；
- 3 阶段表示成熟阶段，意味着主流浏览器的新版本都支持了大部分新标准，基础的es 新标准特性不需要降级编译为 es5，浏览器可原生支持；

##### stage-3

- transform-async-to-generator  支持async/await
- transform-exponentiation-operator 支持幂运算符语法糖

##### stage-2

- 包括stage-3的所有插件；
- syntax-trailing-function-commas 支持尾逗号函数
- transform-object-reset-spread 支持对象的解构赋值

##### stage-1

- 包括stage-2的所有插件；
- transform-class-constructor-call 支持class的构造函数
- transform-class-properties 支持class的static属性
- transform-decorators  支持es7的装饰者模式即@符号引入的方法 
- transform-export-extensions 支持export方法

##### stage-0

- 包括stage-1的所有插件；
- transform-do-expressions 支持在jsx中书写if/else
- transform-function-bind 支持::操作符来切换上下文，类似于es5的bind

```javascript
"presets": [
    [
        "env",
        {
          "modules": false //  将ES6模块语法转换为另一种模块类型,"amd" | "umd" | "systemjs" | "commonjs" | false
        }
    ],

    // 不带配置项，直接列出名字
    "stage-2"
]
```

## Plugin与Preset执行顺序

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