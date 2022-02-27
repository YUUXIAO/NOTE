## 代码分割和动态import

webpack有一个功能是将代码库分割成 chunks(语块)，当代码运行到需要它们的时候再加载；

适用的场景：

1. 抽离相同代码到一个共享块；
2. 脚本懒加载，使得初始下载的代码更小；

### 懒加载JS脚本的方式 

1. CommonJS：require.ensure();
2. ES6：动态import (目前没有原生支持，需要Babel转换)，通过jasonp的形式；

```javascript
// 安装babel插件
npm install @babel/plugin-syntax-dynamic-import --save-dev

// webpack配置
{
  plugins:['@babel/plugin-syntax-dynamic-import'],
  ...
}
```

## 与ESlint结合

1. 不重复造轮子，基于 eslint:recommend 配置进行改进；
2. 能够帮助发现代码错误的规则，全部开启（变量未定义、重复key等）；
3. 帮助保持团队代码风格统一，而不是限制开发体验；

### 执行落地

1. 和CI/CD系统集成；

- 在commit代码后，build之前，加入lint pipline阶段，可以避免git提交绕过precommit钩子；


- 本地开发阶段安装husky,增加precommit 钩子，增加npm script，通过 lint-staged 增量检查修改的文件；

2. 和webpack集成（构建）；

- 使用eslint-loader,构建时检查 JS 规范；

## webpack打包库和组件

webpack除了可以用来打包应用，也可以用来打包js库；

比如实现一个大整数加法库的打包：

1. 需要打包压缩版和非压缩版；

2. 支持 AMD/CJS/ESM 模块引入；

   ```javascript
   // 支持 ES module
   import * as largeNumber from 'large-number'
   // 支持 CJS
   const largeNumber = require('large-number')
   // 支持 AMD
   require(['large-number'],function(largeNumber){
     ...
   })
   // 直接通过script 引入 
   ```

   ​

### 库的目录结构和打包要求

![打包库](C:\File\CODE\NOTE\Markdown\Webpack\image\打包库.png)

### 如何将库暴露出去

```javascript
module.exports = {
  mode: 'production',
  entry: {
    "large-number":＂./src/index.js＂,
    "large-number.min":＂./src/index.js＂
  },
  output: {
    filename: '[name].js',
    library: "largeNumber", // 指定库的全局变量
    libraryExport: "default",
    libraryTarget: "umd", // 支持库引入的方式
  }，
  // 只对.min文件压缩
  optimization: {
	minimize: true,
    mnimizer: [
      new TerserPlugin({
		include: /\.min\.js$/
      })
    ]
  }
}
```

### 1

设置package.json文件的main字段；

/index.js 判断环境引入压缩文件；