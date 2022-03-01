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

### 根据不同环境配置引入压缩文件

设置package.json文件的main字段；

/index.js 判断环境引入压缩文件；

## 优化构建命令行日志

### 统计信息 stats

| Preset       | Alternative | Description     |
| ------------ | ----------- | --------------- |
| "error-only" | none        | 只在发生错误时输出       |
| “minimal”    | none        | 只在发生错误或有新的编译时输出 |
| "none"       | false       | 没有输出            |
| “normal”     | true        | 标准输出            |
| “verbose”    | none        | 全部输出            |

### friendlyErrorsWebpackPlugin 插件

## 构建异常和错误处理

- webpack4 之前的版本构建失败不会抛出错误码；
- Node.js 中的 process.exit 规范：
  - 0表示成功完成，回调函数中，err为null；
  - 非0表示执行失败，回调函数中，err 不为 null，err.code 就是传给exit的数字;

### 主动捕获并处理构建错误

compiler 在每次构建结束后会触发 done 这个 hook；

process.exit 主动处理构建报错；

```javascript
plugins:[
  function(){
    this.hooks.done.tap('done',(stats)=>{
      if(stats.compilation.errors && stats.compilation.errors.length && process.argv.indexOf('--watch') == -1){
        // TODO 错误上报
        console.log('build error')
        process.exit(1)
      }
    })
  }
]
```

## 构建配置包设计

- 通用性
  - 业务开发无需关注构建配置
  - 统一团队构建脚本
- 可维护性
  - 构建配置合理的拆分
  - README文档、ChangeLog文档
- 质量
  - 冒烟测试、单元测试、测试覆盖率；
  - 持续集成

### 构建配置管理方案

- 通过多个配置文件管理不同环境的配置，webpack --config 参数进行控制；

  通过webpack-merge组合配置

  ```
  基础配置：webpack.base.js
  开发环境：webpack.dev.js
  生产环境：webpack.prod.js
  ```

- 将构建配置设计成一个库，比如 com-webpack；

- 抽成一个工具进行管理，比如 create-react-app;

- 将所有配置放在一个文件，通过 --env参数控制分支选择；

## 功能模块设计和目录结构 

- 功能模块设计见 xMind;
- 目录结构设计
  - lib放置源码；
  - test放置测试代码；

## 使用ESlint规范构建脚本

使用 eslint-config-airbnb-base；

eslint --fix 可以自动处理空格；

## 冒烟测试-Mocha

预测试,保证基本功能可用；

1. 构建是否成功；

   - 在示例项目里面运行构建，看是否报错；

2. 每次构建完成 build 目录是否有内容输出：

   1. 编写mocha测试用例

   - 是否有 js、css等静态资源文件；
   - 是否有 HTML 文件；

## 单元测试和测试覆盖率

### 单元测试接入

1. 安装 mocha + chai

   ```javascript
   npm i mocha chai -D
   ```

2. 新建 test目录，并增加 xxx.test.js 测试文件

3. 在 package.json 中的 scripts 字段增加 test 命令；

   ```
   scripts：{
     test: "node_modules/mocha/bin/_mocha"
   }
   ```

   ​



