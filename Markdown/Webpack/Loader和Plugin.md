## Loader

> loader就像一个翻译官，将源文件经过转换后生成目标文件并交由下一流程处理；

本质上 loader 即是一个函数，接收参数并对其进行处理，而后返回处理结果（须为buffer或string）；

### 使用方法

```javascript
module.exports = {
  module: {
    rules: [
      { test: /\.css$/, use: 'css-loader' },
      { test: /\.ts$/, use: 'ts-loader' }
    ]
  }
};
```

### 实现

 实现一个简单的loader，功能是去除换行符；

```javascript
/** lib/loader/loader1.js */

const loaderUtils = require("loader-utils");

/** 过滤console.log和换行符 */
module.exports = function (source) {
  
  // 获取loader配置项
  const options = loaderUtils.getOptions(this);
  
  const result = source.replace(/\n/g, "")；
  
  return result;
}
```

### 编写异步 loader

```javascript
/** 异步loader */
module.exports = function (source) {
  let count = 1;  
  // 1.调用this.async() 告诉webpack这是一个异步loader，需要等待 asyncCallback 回调之后再进行下一个loader处理  
  // 2.this.async 返回异步回调，调用表示异步loader处理结束  
  const asyncCallback = this.async();
  
  const timer = setInterval(() => {
    console.log(`时间已经过去${count++}秒`);
  }, 1000);
  
  // 异步操作
  setTimeout(() => {
    clearInterval(timer);
    asyncCallback(null, source);
  }, 3200);
};
```

## Plugin

> 在webpack编译整个生命周期的特定节点执行特定功能；

### 实现要点

1. 一个命名 JS 函数 或者 JS 类；
2. 在 prototype 上定义一个 apply 方法（供 webpack 调用，并且在调用时注入 compiler 对象）；
3. 在 apply 函数中需要有通过 compiler 对象挂载的 webpack 事件钩子（钩子函数中能拿到当前编译的 Compilation）;
4. 处理 webpack 内部实例的特定数据；
5. 功能完成后调用 webpack 提供的回调；

### 基本模型

```javascript
// 1、Plugin名称
const MY_PLUGIN_NAME="MyBasicPlugin";

class MyBasicPlugin {

  // 2、在构造函数中获取插件配置项
  constructor(options){
    this.options = options
  }

  // 3、在原型对象上定义一个apply函数供webpack调用
  apply(compiler){
    // 4、注册webpack事件监听函数
    compiler.hooks.emit.tapAsync(
      MY_PLUGIN_NAME,
      (compilation, asyncCallback) =>{
        // 5、操作或者改变compilation内部数据
        console.log(compilation);      
        console.log("当前阶段 ======> 编译完成，即将输出到output目录");
        // 6、如果是异步钩子，结束后需要执行异步回调
        asyncCallback();
      }
    );
  }
}

// 7、模块导出
module.exports = MyBasicPlugin;
```

实现一个在dist目录自动生成 README 文件的 plugin；

```javascript
// 实现一个plugin，功能是在dist目录自动生成README文件：
const { compilation } = require("webpack");

// 1、Plugin名称 
const MY_PLUGIN_NAME = 'MyReadMePlugin';

class MyReadMePlugin {

  constructor(options){
     // 2、在构造函数中获取插件配置项
    this.options = options ||{}
  }

  // 3、在原型对象上定义一个apply函数供webpack调用
  apply(compiler){
    // 4、注册webpack事件监听函数
    compiler.hooks.emit.tapAsync(
      MY_PLUGIN_NAME,
      (compilation,asyncCallback) =>{
        compilation.assets['README.md'] = {
          // 文件内容
          source:()=>{
            return  '默认标题';
          },
          // 文件大小
          size: ()=> 30
        }

        // 5、操作Or改变compilation内部数据
        console.log(compilation);  
        console.log("当前阶段 ======> 编译完成，即将输出到output目录");
        // 6、如果是异步钩子，结束后需要执行异步回调
        asyncCallback();
      }
    )
  }
}
// 7、模块导出
module.exports = MyReadMePlugin;
```

