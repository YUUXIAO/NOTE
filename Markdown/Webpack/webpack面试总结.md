## webpack特点

1. 对 CommonJS、AMD、ES6 的语法做了兼容；
2. 对 js、css 、图片等资源都支持打包；
3. 串联模块加载器以及插件机制，让其具有更好的灵活性和拓展性；
4. 可以将代码切割成不同的 chunk，实现按需加载，降低了初始化时间；
5. 支持 sourcemap，易于调试；
6. 具有强大的 plugin 接口，大多是内部插件，使用起来灵活；

## 解析代码路径

> webpack 依赖 enhanced-resolve 来解析代码模块的路径；

模块解析规则分三种：

1. 解析相对路径
   - 查找相对当前模块的路径下是否有对应文件或文件夹，是文件则直接加载；
   - 如果是文件夹则找到对应文件夹下是否有 package.json 文件；
   - 有的话就按照文件中的 main 字段的文件名来查找文件；
   - 没有 package.json 或 main，则查找 index.js 文件；
2. 解析绝对路径
   - 直接查找对应路径的文件；
3. 解析模块名
   - 查找当前文件目录，父级直至根目录下的 node_modules 文件夹，看是否有对应名称的模块；

## source map是什么

> source map 是将编译、打包、压缩后的代码映射回源码的过程；

打包压缩后的代码不具备良好的可读性，想要调试源码就需要 source map，出错的时候，浏览器控制台将直接显示原始代码出错的位置；

避免在生产中使用 inline- 和 eval-，因为它们会增加 bundle 体积大小，并降低整体性能；

map 文件只要不打开开发者工具，浏览器是不会加载的；

生产环境一般有三种处理方案：

1. source-map：map 文件包含完整的原始代码，但是打包会很慢，打包后的 js 最后一行是 map 文件地址的注释。通过 nginx 设置将 .map 文件只对白名单开放；
2. hideen-source-map：与 sourceMap 相同，也生成 map 文件，但是打包后的 js 最后没有 map 文件地址的引用。浏览器不会主动去请求 map 文件，一般用于网站错误分析，需要让错误分析工具按名称匹配到 map 文件。 或者借助第三方错误监控平台 Sentry 使用；
3. nosources-source-map：只会显示具体行数以及查看源码的错误栈。安全性比 sourcemap 高；

## 模块打包原理

1. webpack 根据 webpack.config.js 中的入口文件，在入口文件中识别模块依赖，不论这里的依赖是根据 CommonJS、ES6 Module 规范写的，webpack 会自动进行分析，并通过转换、编译代码、打包成最终的文件；
2. 最终文件中的模块实现是基于 webpack 自己实现的 webpack_require（ES5 代码），所以打包后的文件可以跑在浏览器上；
3. 从 webpack2 开始，内置了对 ES6、CommonJS、AMD 模块化语句的支持，webpack 会对各种模块进行语法分析，并做转换编译；
4. 针对异步模块，webpack 实现模块的异步加载类似 jsonp 的流程；
   - 遇到异步模块时，使用 _ webpack_require_.e 函数把异步代码加载进来，该函数会在 html 的 head 标签中动态增加 script 标签，src 指向指定的异步模块存放的文件；
   - 加载的异步模块文件会执行 webpackJsonpCallback 函数，把异步模块加载到主文件中；
   - 后续可以像同步模块一样,直接使用 __ webpack_require__("./src/async.js") 加载异步模块；

## 文件监听原理 

在发现源码发生变化时，自动重新构建出新的输出文件；

轮询判断文件的最后编辑时间是否变化，初次构建时把文件的修改时间储存起来，下次有修改时会和上次修改时间比对，发现不一致的时候，并不会立刻告诉监听者，而是先缓存起来，等 aggregateTimeout 后，把变化列表一起构建，并生成到 bundle 文件夹；

```javascript
module.export = {
  // 默认 false，也就是不开启
  watch: true,
  watchOptions: {
    // 默认为空，不监听的文件夹或者文件，支持正则匹配
    ignore: /node_modules/,
    // 监听到变化发生后会等 300ms 再去执行，默认 300ms
    aggregateTimeout: 300,
    // 判断文件是否发生变化是通过不停询问系统指定文件有没有变化实现的，默认每秒询问 1000 次
    poll: 1000,
  },
};
```

## 代码分割

> 代码分割的本质是在源代码直接上线和打包成唯一脚本 main.bundle.js 这两种极端方案之间的一种更适合实际场景的中间状态；

1. 源代码直接上线：

   - http 请求过多，性能开销大；
2. 打包成唯一脚本：

   - 服务器压力小，但是页面留白时间长，用户体验不好；
   - 大体积文件会增加编译的时间，影响开发效率；
   - 多页应用，独立访问单个页面时，需要下载大量不相干的资源；


代码分割的意义：

- 复用的代码抽离到公共模块中，解决代码冗余；

- 公共模块再按照使用的页面多少进一步拆分，用来减少文件体积，可以优化首屏加载速度；

拆分原则：

1. 业务代码和第三方库分离打包，实现代码分割；
2. 业务代码中的公共业务模块提取打包到一个模块；
3. 首屏相关模块单独打包；


### splitChunks

> 将一个大bundle文件拆包，拆包的方案可以在cacheGroups里配置；

```javascript
// splitChunks默认配置
optimization: {
    splitChunks: {
      chunks: 'all',  // 无论同步引入还是异步引入
      cacheGroups: {
        vendors: {
          test: /[\\/]node_modules[\\/]/,  // 匹配node_modules目录下的文件
          priority: -10   // 优先级配置项
        },
        default: {
          minChunks: 2,  // 至少引用了2次
          priority: -20,   // 优先级配置项
          reuseExistingChunk: true
        }
      }
    }
  }
```

在默认设置中：

1. 将 node_mudules 文件夹中的模块打包进叫 vendors 的 bundle 中；
2. 所有引用超过两次的模块分配到 default bundle 中 ，可以通过 priority 来设置优先级；