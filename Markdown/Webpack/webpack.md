> webpack 是一个 Javascript 静态模块打包器，它会递归构建一个依赖关系图，这个关系图包含应用程序所需要的每个模块，然后将这些模块打包成一个或多个 bundle； 

## 核心概念

### Entry

指定webpack开始构建的入口模块，从该模块开始构建并计算出直接或间接依赖的模块或者库；

### Output

webpack在哪里输出所创建的bundles,以及如何命名这些文件；

### Loaders

由于webpack只能处理javascript，所以我们需要对一些非js文件处理成webpack能够处理的模块，比如sass文件；

### Plugins

插件的应用范围为从打包优化压缩到重新定义环境中的变量，插件功能很强大，可以用于处理各式各样的任务；

webpack插件是一个具有apply属性的JavaScript对象，apply属性会被webpack compiler调用，并且compiler对象可在整个生命周期访问；

### chunk

code split 的产物。我们可以对 一些代码打包成一个单独的 chunk，比如某些公共模块、去重、更好的利用缓存；或者按需加载某些功能模块，优化加载时间；

在 webpack3 及以前使用 CommonsChunkPlugin 将一些公共代码分割成一个 chunk，实现单独加载；

在 webpack4 中 CommonsChunkPlugin 被废弃，使用 SplitChunksPlugin；

### 模式

通过选择development或是production之中的一个，来设置mode参数，可以启用相应webpack配置下的优化；

## 模块化带来的问题

1. 无法保证所有的浏览器都冶容模块化标准；
2. 依赖的模块太多、划分太细会导致一次发送多个请求向服务端请求模块资源造成效率低下；
3. 不仅仅 js 需要模块化，其它的 CSS 、HTML、图片等资源也需要模块化；
4. 因为模块比整个程序有更小的接触面，使得校验、测试、调试更加方便；

## 作用

1. 开发时，启动本地服务；
2. 解决 js 、css 的依赖问题（比如引入顺序问题）；
3. 具备代码编译能力：将 ES6、vue / react、 jsx 语法编译为浏览器可识别的代码；
4. 合并、压缩、优化打包后的体积；
5. css 前缀补齐 / 预处理器；
6. 使用 eslint 校验代码；
7. 单元测试；

## 配置组成部分

```javascript
module.exports = {
  entry: '',   // 指定入口文件
  output: '',  // 指定输出目录和输出文件名
  mode: '',    // 环境
  module: {
    rules: [   // loader配置
      {test: '', use: ''}
    ]
  },
  plugins: [   // 插件配置
    new xxxPlugin()
  ]
}
```

## Loader

> webpack 原生只支持 js、json 两种模块类型，所以需要 loader 把其它类型的文件转化为有效的模块，并可以添加到依赖图中；

loader 本身是一个函数，接受源文件作为参数，返回转换的结果：

1. 解析es6：babel-loader（配合 .babelrc 使用）；
2. 解析vue：vue-loader；
3. 解析css：
   - css-loader：用于加载 .css 文件，并转换为 commonjs 对象；
   - style-loader：将样式通过 < style > 标签插入到 head 中；
4. 解析less：less-loader（将 less 转换为 css）;
5. 解析图片和字体：
   - file-loader：用于处理文件（图片、字体）；
   - url-loader：也可以处理图片和字体，和 file-loader 功能类似，但它还可以设置较小资源自动转 base64（内部使用了 file-loader），使用 options：｛limit：xxx｝;

## Plugin

> 插件用于 bundle 文件的优化、资源管理和环境变量的注入，它作用于整个构建过程；

1. clean-webpack-plugin：打包前自动清理 dist 目录；

2. html-webpack-plugin：自动生成 html，并将打包后的 js 插入进去；

3. mini-css-extract-plugin：把css 提取成单独的文件，与 style-loader 功能互斥，不同同时使用；

4. 关于代码压缩：

   - JS 文件的压缩：默认开启了内置的 terser-webpack-plugin ，webpack 在打包时会自动压缩 js 代码；

   - CSS 文件的压缩：使用 optimize-css -assets-webpack-plugin ，同时使用 cssnano（处理器）；

     ```javascript
      plugins: [
        new OptimizeCssAssetsPlugin({
          assetNameRegExp: /\.css$/g,
          cssProcessor: require('cssnano')
        })
      ]
     ```

   - HTML 文件的压缩：修改 html-webpack-plugin ，设置压缩参数；

     ```javascript
     plugins: [
       new HtmlWebpackPlugin({
         template: path.join(__dirname, 'src/index.html'),
         filename: 'index.html',
         chunks: ['main', 'other'], //要包含哪些chunk
         inject: true, //将chunks自动注入html
         minify: { // 压缩相关
           html5: true,
           collapseWhitespace: true, //压缩空白字符
           preserveLineBreaks: false,
           minifyCSS: true,
           minifyJS: true,
           removeComments: true
         }
       })
     ]
     ```

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

## 构建流程分析

webpack 的运行流程是一个串行的过程，从启动到结束会依次执行以下流程：

1. 从配置文件和 Shell 语句中读取与合并参数，并初始化需要使用的插件和配置插件等执行环境所需要的参数；
2. 初始化完成后会调用 Compiler 的 run 来真正启动 webpack 编译构建过程；
3. webpack 的构建流程包括 compile、make、build 、seal、emit 阶段，执行完这些阶段就完成了构建过程；

### 初始化

#### entry-options 启动

从配置文件和 Shell 语句中读取与合并参数，得出最终的参数；

#### run 实例化

compiler：用上一步得到的参数初始化 Compiler 对象，加载所有配置的插件，执行对象的 run 方法开始执行编译

### 编译构建

#### entry 确定入口

根据配置中的 entry 找出所有的入口文件；

#### make 编译模块

从入口文件出发，调用所有配置的 Loader 对模块进行编译，再找出该模块依赖的模块，再递归本步骤直到所有入口依赖文件都经过了本步骤的处理；

#### build module 完成模块编译

经过上面一步使用 Loader 翻译完所模块后，得到了每个模块被翻译后的最终内容以及它们之间的依赖；

#### seal 输出资源

根据入口和模块之间的依赖关系，组装成一个个包含多个模块的 Chunk，再把每个 Chunk 转换成一个单独的文件加入到输出列表，这步是可以修改输出内容的最后机会；

#### emit 输出完成

在确定好输出内容后，根据配置确定输出的路径和文件名，把文件内容写入到文件系统；



![img](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/71b263000fa94db792cf1e98d67a578a~tplv-k3u1fbpfcp-zoom-1.image)