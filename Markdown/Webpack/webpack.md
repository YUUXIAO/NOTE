> webpack 是一个 Javascript 静态模块打包器，它会递归构建一个依赖关系图，这个关系图包含应用程序所需要的每个模块，然后将这些模块打包成一个或多个 bundle； 

## 核心概念

- Entry：指定webpack开始构建的入口模块，从该模块开始构建并计算出直接或间接依赖的模块或者库；
- Output：webpack在哪里输出所创建的bundles,以及如何命名这些文件；
- Loader：因为 Webpack 只认识 JavaScript，所以 Loader 就成了翻译官，对其他类型的资源进行转译的预处理工作；本质就是一个函数，在该函数中对接收的到的内容进行转换，返回转换后的结果；
- Plugin：基于事件流 框架 Tapable，插件可以扩展 webpack的功能，在 webpack 运行的生命周期会广播出许多事件，Plugin 可以监听这些事件，在合适的时机通过 webpack 提供的 api 改变输出的结果；
- Mode：通过选择development或是production之中的一个，来设置mode参数，可以启用相应webpack配置下的优化；
- module：除了 js 范畴内的es module、commonJs、AMD等，css @import、url(...)、图片、字体等在webpack中都被视为模块；
- chunk：code split 的产物：我们可以对 一些代码打包成一个单独的 chunk，比如某些公共模块、去重、更好的利用缓存；或者按需加载某些功能模块，优化加载时间；

## 为什么选择 webpack

1. 社区生态丰富；
2. 配置灵活和插件化拓展；
3. 官方更新迭代速度快；

## 作用

1. 开发时，启动本地服务；
2. 解决 js 、css 的依赖问题（比如引入顺序问题）；
3. 具备代码编译能力：将 ES6、vue / react、 jsx 语法编译为浏览器可识别的代码；
4. 合并、压缩、优化打包后的体积；
5. css 前缀补齐 / 预处理器；
6. 使用 eslint 校验代码；
7. 单元测试；

## 开发环境性能优化

### 模块热替换

>  只重新打包变更的模块，局部刷新，保留数据状态，（而不是将所有模块重新打包，刷新页面）提升构建速度，使开发更加方便；

1. 开启 dev-server 后默认只要有一个文件变化，会重新构建，刷新浏览器页面；

2. 通过 devServer.hot 启用，其内部依赖 webpack.HotModuleReplacementPlugin 实现，HotModuleReplacementPlugin 会在 hot：true 时自动引入

   - 样式文件：可以直接使用 HMR 功能，因为 style-loader 内部实现了；

   - js 文件：默认不能使用 HMR 功能，需要修改 js 代码，添加支持 HMR 功能的代码：

     ```javascript
     if(module.hot){  // 如果开启了HMR功能
     // 监听xxx.js文件的变化，一旦发生变化，其他模块不会重新打包，会执行回调函数
       module.hot.accept('./xxx.js', function(){ 
          fn()
       })
     }
     ```

   - html文件：没有热替换，也没有热更新，热更新可以通过在入口文件添加 html 文件路径来打开，但通常没有必要；

### 使用 source-map

由于经过 webpack 打包后的代码是经过各种 loaders，plugins 转换过后的一个大的 js 文件，开发过程中无法调试；

source-map 是一种提供源代码到构建后代码映射的技术，报错时通过 source map 可以定位到源代码；

启用方式：

```javascript
module.exports = {
  devtool: 'source-map' 
}
```

选项：

`[inline- | hidden- | eval-] [nosources- ] [cheap- [module- ]]source-map`

1. source-map: 产生 .map 文件，提供错误代码准确信息和源代码的错误位置；
2. inline：内联，将 .map 作为 DataURI 嵌入，不单独生成 .map 文件，构建速度更快；
3. eval：内联，使用 eval 包裹模块代码，指定模块对应文件；
4. cheap：只精确到行，不精确到列 ；
5. module：包含 loader 的 sourcemap；

推荐组合：

- 开发环境：速度快，调试更友好；
  - eval-source-map： ( eval 速度最快，source-map 调试最友好)
- 生产环境：内联会让代码体积变大，所以生产环境不用内联；
  - nosources-source-map ---全部隐藏；
  - hidden-source-map ---只隐藏源代码，会提示构建后代码错误信息；
  - source-map --- 调试友好；

## 生产环境性能优化

### 文件指纹

当设置了 http 强缓存，比如有效期为一天：如果不使用 hash，当这个文件改变了，因为文件名没变，所以客户端使用的还是旧的缓存；如果使用了 hash，这时文件名就改变了，就会请求新的资源，而没有更改过的文件继续使用缓存；

1. hash：构建的 hash，每次构建都会改变，不建议使用；
2. chunkhash：和 webpack 打包的 chunk 有关，不同的 entry 会生成不同的 chunkhash 值；
3. contenthash：根据文件内容来定义 hash ，文件内容不变，则 contenthash 不变，推荐在 css 文件上使用；

js 文件的指纹设置：

```javascript
//设置 output 的 filename，使用 [chunkhash]
module.exports = {
  output: {
    filename: '[name][chunkhash:8].js',
    path:__dirname+'/dist'
  }
}
```

css 文件的指纹设置：

使用 MiniCssExtractPlugin 将  css 从 js 中提出来，然后使用  contenthash；

```javascript
plugins: [
  new MiniCssExtractPlugin({
    filename: '[name][contenthash:8].css'
  })
]
```

### Tree shaking

> 摇树优化：一个模块可能有多个方法，只要其中某个方法使用到了，则整个文件都会被打到 bundle 里面去，tree shaking 就是只把用到的方法打入到 bundle ，没有用到的方法会在 uglify 阶段被擦除掉；

1.  webpack 默认支持，在 .babelrc 里设置 module：false 即可；
2.  webpack 会在 mode 为 production  的情况下默认开启 tree shaking；
3.  要求：必须是 es6 语法，cjs 的方式不支持；

#### 原理

DCE：永远不会被用到的代码，比如引入了一个方法但是没调用 或者 if(false){xxx}；

利用 ES6 模块的特点：

1. 只能作为模块顶层的语句出现；
2. import 的模块名只能是字符串常量；
3. import binding 是 immutable 的；

在打包之前静态的分析文件，在uglify阶段删除无用代码；

### Code split

> 将一个大bundle文件拆包，拆包的方案可以在cacheGroups里配置；

- splitChunks

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

1. 会将 node_mudules 文件夹中的模块打包进一个叫 vendors 的 bundle 中；
2. 所有引用超过两次的模块分配到 default bundle 中 ，可以通过 priority 来设置优先级；

### DLL

> 将第三方库和业务基础包单独打成一个文件，只在第一次打，或者需要更新依赖的时候打，此后每次就可以只打自己的源代码，加快了构建速度；

方法：使用 DLLPlugin 进行分包，DllReferencePlugin 对 manifest.json 引用；

分包需要单独的配置文件：

```javascript
// webpack.dll.js
module.exports={
  entry: {
    lib: [
      'lodash',
      'jquery'
    ]
  },
  output: {
    filename: '[name]_[chunkhash].dll.js',
    path: path.join(__dirname, 'build/lib'),
    library: '[name]' // 打包后对外暴露的全局变量名称
  },
  plugins: [
    new webpack.DllPlugin({
      name: '[name]', // manifest.json中的name，要与ouput.library名一致
      path: path.join(__dirname, 'build/lib/manifest.json'),
    })
  ]
}

```

在 package.json 中添加命令对 dll 单独打包：

```javascript
"scripts": {
    "dll": "webpack --config webpack.dll.js"
  },
```

使用 DllReferencePlugin 对 manifest.json 进行引用，告诉 webpack 使用了哪些动态链接库，不用再打包这里面的东西：

```javascript
// webpack.prod.js
new webpack.DllReferencePlugin({
  manifest: require('./build/lib/manifest.json')
}),

```

使用 addAssetHtmlWebpackPlugin 将 dll 资源插到 html 里：

```javascript
// webpack.prod.js
new addAssetHtmlWebpackPlugin([
  {
    filepath: path.resolve(__dirname, './build/lib/*.dll.js'),
    outputPath: 'static', // 将*.dll.js拷贝后的输出路径，相对于html文件
    publicPath: 'static' 
  }
])
```

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a10ffd865c8b437abbffd70a6e606809~tplv-k3u1fbpfcp-zoom-1.image)

### splitChunks 和 dll 的区别

1. splitChunks 是在构建时拆包，dll 是提前构建好基础库，打包的时候就不需要打基础库了，时间上 dll 比 splitChunks 快一点；
2. dll 需要多配置一个 webpack.dll.config.js ， 而且一旦 dll 中的依赖有更新，得走两遍打包，比 splitChunks 麻烦一些 ;
3. 推荐使用 splitChunks 去提取页面间的公共 js 文件，DllPlugin 用于基础包（框架包、业务包）的分离；

### 多进程打包

> 使用 thread-loader 开启多进程打包，加快打包速度；

- 启动进程需要大概  600ms ，进程间通信也有花销，项目小的话开启多进程得不偿失，所以只有当项目比较大，打包耗时较长的时候才适合使用多进程。

```javascript
module: {
  rules: [
    {
      test: /.js$/, 
      use: [
        {
          loader: 'thread-loader',
          options: {
            workers: 2 //开启两个进程
          }
        },
        {
          loader: 'babel-loader',
          options: {
            presets: ['@babel/preset-env'],
            cacheDirectory: true 
          }     
        }
      ]
    },
  ]
}
```

### module、chunk、bundle的区别

1. module：模块，源代码中的一个文件就是一个模块；
2. chunk：一个入口文件所依赖的一大块就是一个 chunk，可以理解为一个 entry 对应一个 chunk；
3. bundle：打包后的资源，一般来说一个 chunk 就对应一个 bundle，但也可以通过一些插件进行拆包，把一个大 chunk 拆分为多个 bundle，比如 MiniCssExtractPlugin；

## 文件指纹

### 优点

1. 用作版本管理，发布项目时，只需要发布修改过的文件；
2. 对于没有修改过的文件，用户在访问的时候依旧可以使用浏览器缓存，无需二次加载，加速页面访问；

### 生成文件指纹

1. Hash：和整个项目的构建有关，只要项目文件有修改，整个项目的构建的 hash 值也会修改；
2. Chunkhash：和 webpack 打包的 chunk 有关，不同的 entry 会生成不同的 chunkhash 值，chunkhash 是使用 md5 加密，hash 的长度值为 32位，[chunkhash:8]表示取hash值的前8位；
3. Contenthash：根据文件内容来定义 hash ，文件内容不变就不变；

#### JS文件指纹设置

使用 output 的 filename，使用 [ chunkhash ]；

```javascript
 output: {
    path: path.join(__dirname, 'dist'),
    filename: '[name]_[chunkhash:8].js'
  },
```

#### CSS 指纹设置

需要使用 MiniCSSExtractPlugin 将 CSS 提取成独立的文件，然后使用 [ Contenthash ] 设置指纹；

MiniCSSExtractPlugin 和 style-loader 功能是互斥的；

```javascript
const MiniCssExtractPlugin = require('mini-css-extract-plugin')
  plugins: [
    // 把css提取成单独的文件
    new MiniCssExtractPlugin({
      filename: '[name]_[contenthash:8].css'
    })
  ]
```

#### 图片和字体的指纹设置

设置 file-loader 的 name , 使用 [ hash ]；

```javascript
{
  test: /.(png|jpg|gif|jpeg)$/,
  use: [
    {
      loader: 'file-loader',
      options: {
        name: '[name]_[hash:8].[ext]'
      }
    }
  ]
},
```

## 代码压缩

### JS 文件的压缩

> UglifyJS Webpack Plugin 插件用来缩小（压缩优化）js文件，至少需要 Node v6.9.0和Webpack v4.0.0版本；

1. webpack 4 之前的版本是通过 webpack.optimize.CommonsChunkPlugin 来压缩js；
2. webpack 4 版本之后被移除了，使用 config.optimization.splitChunks 来代替；

```javascript
//webpack.config.js
const UglifyJsPlugin = require('uglifyjs-webpack-plugin');
module.exports = {
  optimization: {
    minimizer: [
      new UglifyJsPlugin({
      test: /\.js(\?.*)?$/i,  //测试匹配文件,
      include: /\/includes/, //包含哪些文件
      excluce: /\/excludes/, //不包含哪些文件
      //允许过滤哪些块应该被uglified（默认情况下，所有块都是uglified）。 
      //返回true以uglify块，否则返回false。
      chunkFilter: (chunk) => {
          // `vendor` 模块不压缩
          if (chunk.name === 'vendor') {
            return false;
          }
          return true;
        }
      }),
      cache: false,   //是否启用文件缓存，默认缓存在node_modules/.cache/uglifyjs-webpack-plugin.目录
      parallel: true,  //使用多进程并行运行来提高构建速度
    ],
  },
};
```

### CSS 文件的压缩

使用 optimize-css-assets-webpack-plugin 插件，同时使用 cssnano；

```javascript
plugins:[
  new OptimizeCSSAssetsPlugin({
    assetsNameRegExp: /\.css$/g,
    cssProcessor: require('cssnano')
  })
]
```

### html 文件的压缩

使用 html-webpack-plugin 插件；

```javascript
plugins:[
  new HtmlWebpackPlugin({
    template: path.join(_dirname,'./src/search.html'),
    filename: 'search.html',
    chunks:	[search],
    inject: true，
    minify: {
      html5：true,
      collapseWhitespace: true,
      preserveLineBreaks: true,
      minifyCss: true,
      minifyJS：true,
      removeComments: false
    }
  })
]
```



## 多页面打包通用配置

多页面打包需要多个入口文件，多个 HtmlWebpackPlugin 产生多个 html；

```javascript
// 核心方法
const setMPA = () => {
  const entry = {};
  const htmlWebpackPlugins = [];

  const entryFiles = glob.sync(path.join(__dirname, './src/*/index.js'));
  entryFiles.forEach(entryFile => {
    const match = entryFile.match(/src\/(.*)\/index\.js/);
    const pageName = match && match[1];

    entry[pageName] = entryFile;
    htmlWebpackPlugins.push(
      new HtmlWebpackPlugin({ 
        template: path.join(__dirname, `src/${pageName}/index.html`),
        filename: `${pageName}.html`,
        chunks: [pageName], //要包含哪些chunk
        inject: true, //将chunks自动注入html
        minify: {
          html5: true,
          collapseWhitespace: true,
          preserveLineBreaks: false,
          minifyCSS: true,
          minifyJS: true,
          removeComments: false
        }
      }),
    )
  })

  return {
    entry,
    htmlWebpackPlugins
  }
}
```

## webpack 打包精简后的代码示例

1. webpack 将所有模块（可以简单理解为文件）包裹于一个函数中，并传入默认参数，将所有模块放入一个数组中，取名 modules，并通过数组下标来作为 moduleId；
2. 将 modules 传入一个自执行函数中，自执行函数中包含一个 installedModules 已经加载过的模块和一个模块加载函数，最后加载入口模块并返回；
3. _ webpack_require _ 模块加载，先判断 installedModules 是否已经加载，加载过了就直接返回 exports 数据，没有加载过该模块就通过 modules[moduleId].call(module.exports, module, module.exports, __ webpack_require __) 执行模块并且将 module.exports 给返回；

换个说法：

1. 经过 webpack 打包出来的是一个匿名闭包函数；
2. modules 是一个数组，每一项是一个模块初始化函数；
3. __ webpack_require __ 用来加载模块，返回 module.exports；
4. 通过 WEBPACK_REQUIRE_METHOD(0) 启动程序；

```javascript
// dist/index.xxxx.js
(function(modules) {
  // 已经加载过的模块
  var installedModules = {};

  // 模块加载函数
  function __webpack_require__(moduleId) {
    if(installedModules[moduleId]) {
      return installedModules[moduleId].exports;
    }
    var module = installedModules[moduleId] = {
      i: moduleId,
      l: false,
      exports: {}
    };
    modules[moduleId].call(module.exports, module, module.exports, __webpack_require__);
    module.l = true;
    return module.exports;
  }
  __webpack_require__(0);
})([
/* 0 module */
(function(module, exports, __webpack_require__) {
  ...
}),
/* 1 module */
(function(module, exports, __webpack_require__) {
  ...
}),
/* n module */
(function(module, exports, __webpack_require__) {
  ...
})]);


```

## 深究问题

1. HMR的原理？
2. Tree shaking原理，为什么需要es module的写法？
3. webpack5的Module Federation有哪些优势，在与http2.0的结合上有哪些有趣的事情，在微前端上的应用？
4. 为什么说rollup比webpack更适合打包组件库？