cpselvis github



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

   ```javascript
   scripts：{
     test: "node_modules/mocha/bin/_mocha"
   }
   ```

4. 执行测试命令

   ```javascript
   npm run test
   ```

### 测试覆盖率-instanbul

## 持续集成

优点：

1. 快速发现错误；
2. 防止分支大幅偏离主干；

核心措施是：代码集成到主干之前，必须通过自动化测试，只要有一个测试用例失败，就不能集成；

## 发布npm

```javascript
添加用户：npm address
升级版本：
	升级大版本号：npm version major
    升级小版本号：npm version minor
    升级补丁版本号：npm version patch
发布版本：npm publish
```

## git-commit规范和changeLog生成

### 良好的git Commit 规范优势

- 加快 code review 的流程；
- 根据 git commit 的元数据生成 changelog；
- 后续维护者可以知道 feature 生成的原因

### 本地开发阶段增加 precommit 钩子

### 安装 husky

```javascript
npm install husky --save-dev
```

### 通过commitmsg 钩子校验

```javascript
scripts:{
  commitmsg: "validate-commit-msg",
  changelog: "conventinal-changelog -p angular -i CHANGELOG.md -s -r 0"
},
devDependencies: {
  "validate-commit-msg": "^2.11.1",
  "conventinal-changelog-cli": "^1.2.1",
  "husky": "^0.13.1",
}
```

## 语义化版本（semantic-versioning）规范格式

1. 版本通常由三位数组成：X.Y.Z；
2. 版本是严格递增的；
3. 在发布重要版本时，可以先发布 alpha、rc等先行版本；
   - aplha：内部测试版，一般不向外发布，会有很多 bug，测试人员使用；
   - beta：也是测试版，这个阶段的版本会一直加入新功能，在 Alpha版本后发布；
   - rc：（release candlidate）系统平台上就是发行候选版本，不会再加入新的功能了，主要着重于除错；

### 遵守 semver 规范的优势

1. 避免出现循环依赖；
2. 依赖冲突减少；

## 使用内置的stats

stats：构建的统计信息

package.json 中使用 stats

```javascript
scripts: {
  "build:stats": "webpack --env production --json > stats.json"
}
```

## 速度分析：speed-measure-webpack-plugin

```javascript
const SpeedMeasureWebpackPlugin = require('speed-measure-webpack-plugin')

const smp = new SpeedMeasureWebpackPlugin()

const webpackConfig = smp.wrap({
  plugins:[
    new myPlugin(),
    new myOtherPlugin()
  ]
})
```

分析整个打包总耗时；

可以看到每个loader和插件执行耗时；

## 体积分析：webpack-bundle-analyzer

```javascript
const BundleAnalyzerPlugin = require('webpack-bundle-analyzer')

module.exports = {
  plugins: [
    new BundleAnalyzerPlugin()
  ]
}
```

分析依赖的第三方模块文件的大小；

业务里面的组件代码大小；

## 高版本的webpack和node.js

### 使用webpack4 优化的原因

1. v8带来的优化（for of 替代 forEach、Map 和 Set 替代 Object，includes 替代 indexOf）
2. 默认使用更快的 md4 hash算法；
3. webpacks AST 可以直接从 loader 传递给 AST，减少解析时间；
4. 使用字符串方法代替正则表达式；

## 多进程/多实例构建：资源并行解析可选方案

thread-loader（官方推荐）

parallel-webpack

### HappyPack

原理：每次 webpack解析一个模块，HappyPack会将它及它的依赖分配给 worker 线程中；

```javascript
export.plugins = [
  new HappyPack({
    id: 'styles',
    threads: 2,
    loaders: ['style-loader', 'css-loader', 'less-loader']
  })
]
```

### thread-loader

原理：每次 webpack解析一个模块，threade-loader 会将它及它的依赖分配给 worker 线程中；

```javascript
modules.exports = {
  module: {
    rules: [
      {
        test: /.js$/,
        include: path.resolve('src'),
        use:[
          "thread-loader",
          "babel-loader",
          // ...
        ]
      }
    ]
  }
}
```

## 多进程/多实例并行压缩

### parallel-uglify-plugin

```javascript
const ParallelUglifyPlugin = require('parallel-uglify-plugin')

module.exports = {
  new ParallelUglifyPlugin({
    uglifyJs: {
      output: {
        beautify: false,
        comments: false,
      },
      compress: {
        warnings: false,
        drop_console: true,
        collapse_vars: true,
        reduce_vars: true,
      }
    }
  })
}
```

## uglifyjs-webpack-plugin开启parallel参数



```javascript
const ParallelUglifyPlugin = require('uglifyjs-webpack-plugin')

module.exports = {
  plugins: [
    new UglifyJsPlugin({
      uglifyOptions: {
        warning: false,
        parse: {},
        compress: {},
        output: null.
        toplevel: false,
        nameCache: null,
        ie8: false,
      }，
      parallel： true
    })
  ]
}
```

### terser-webpack-plugin开启parallel参数（webpack4）

支持es6语法

```
const TeserPlugin = require('terser-webpack-plugin')

module.exports = {
  optimization: {
    minimizer: [
      new TeserPlugin({
        parallel: 4
      })
    ]
  }
}
```

