## 问题与思考

- 为什么 setup 里面 defindProps 不能使用 import 的 ts，在3.3更新了，之前是因为 宏编辑器

## Vite的优点

- 基于 ES Modules 实现
- 解决了 webpack-dev-server 冷启动时间过长，HMR反应速度过慢的问题

使用 webpack 打包的原因

- 浏览器环境不支持模块化，webpack支持 esModule、requrire、等模块化方式
- 打包成 bundles 的概念-解决零散的模块文件会产生大量的 HTTP请求 - HTTP2解决（复用链接）



vite 利用浏览器原生支持 ESM 的这个特性，省略了对模块的打包，也不需要生成 bundle（webpack 就花了很多时间在这一步，预处理的时间很长），所以初次启动时间更快，HMR特性友好

Vite 开发模式下，通过启动 connect 服务器，在服务端完成模块的改写（比如单文件的解析编译等）和请求处理，实现真正的按需编译

Vite Server的所有逻辑基本都依赖于中间件，这些中间件，拦截了请求后，做了下面这些事

- 处理ESM 语法，比如将代码中的 import 第三方依赖路径转为浏览器可识别的依赖路径
- 对 .ts 、.vue 等文件进行即时编译
- 对 Sass/Less 等需要编译的模块进行编译
- 和浏览器端建立 Socket 连接（文件wather），实现 HMR

## Vite为什么那么快

- 在开发环境，vite是一个开发服务器，它会根据浏览器的请求来编译源文件

  - 无需提前打包编译，做到真正的按需使用（type=module属性，支持ESM模块引入，所以不支持低版本浏览器），不会加载无相关的文件

- 没有修改的文件返回304状态码（浏览器缓存），所以浏览器不会再请求，直接使用缓存
- 通过 esbuild 来支持 .(t|j)sx？ 文件，打包编译速度更快

## Vite如何做到按需加载

- 处理 index.html 文件的内容

  - 在文件中插入 script 标签，注入环境变量，定义 process：

    ````
     window.process = { env: { NODE_ENV: 'DEV' } }
    ````


- 在 index.html 中会请求 /src/main.js 文件

  - 将 import { createApp } from 'vue' 引入依赖包的代码处理成 import { createApp } from '/@modules/vue' ，把路径换成相对路径

- 请求 node_modules 中的文件

  - 浏览器向开发服务器请求 /@modules/vue，./App.vue，./index.css
  - 后台收到以上文件请求，会去读取项目 node_modules/vue/package.json 中的 module 字段，拿到  "dist/vue.runtime.esm-bundler.js"，接着去请求这个文件

- main.js 文件中 import 了 ./App.vue 文件（相对路径），浏览器向后台请求 /src/App.vue
- 后台处理.vue 的代码，主要是处理 template 的内容

  - 页面引用路径调整成 ${url}?type=template

    ```js
    import { ref, computed } from '/@modules/vue'
    const __script = {
        setup() {
            const count = ref(1)
            function add() {
                count.value++
            }
            const double = computed(() => count.value * 2)
            return { count, double, add }
        },
    }
    import { render as __render } from '/src/App.vue?type=template'
    __script.render = __rend
    export default __script
    ```

  - 经过后台处理的 App.vue 文件的 template：可以理解为把 App.vue 的 template  编译成一个 render  函数

    ```js
    export function render(_ctx, _cache) {
        return (
            _openBlock(),
            _createElementBlock('div', null, [
                _createElementVNode(
                    'h1',
                    null,
                    _toDisplayString(_ctx.count) + ' * 2 = ' + _toDisplayString(_ctx.double),
                    1 /* TEXT */,
                ),
                _createElementVNode('button', { onClick: _ctx.add }, 'click', 8 /* PROPS */, [
                    'onClick',
                ]),
            ])
        )
    }
    ```


- 后台处理 style 的代码

  ```css
  if (url.endsWith('.css')) {
      const p = path.resolve(__dirname, url.slice(1))
      const file = fs.readFileSync(p, 'utf-8')
      const content = `
          const css = '${file.replace(/\n/g, '')}'
          let link = document.createElement('style')
          link.setAttribute('type','text/css')
          document.head.appendChild(link)
          link.innerHTML = css
          export default css
      `
      ctx.type = 'application/javascript'
      ctx.body = content
  }
  ```

- 因为请求/@modules/vue返回的内容中import了/@modules/@vue/runtime-dom，所以浏览器会向后台请求/@modules/@vue/runtime-dom，直到把所有依赖请求加载完毕，然后页面才会渲染





1. vite 天然支持引入 .ts 文件，但仅执行 .ts 文件的转译工作，并不执行任何的类型检查；

   - 可以在构建脚本中运行 tsc --noEmit 或 安装 vue-tsc 然后运行 vue-tsc --noEmit 来对 .vue 文件做类型检查

2. vite 默认的类型定义是写给 Node.js API 的，要将其补充到一个 vite 应用的客户端环境中；

   - 添加一个 d.ts 声明文件；

     ```javascript
     /// <reference types="vite/client" />
     ```

   - 将 vite/client 添加到 tsconfig 中的 compilerOptions.types 下；

     ```javascript
     {
       "compilerOptions": {
         "types": ["vite/client"]
       }
     }
     ```

   - 会提供以下类型定义的补充：

     1. 资源导入 (例如：导入一个 .svg 文件)；
     2. import.meta.env 上 Vite 注入的在 的环境变量的类型定义；
     3. import.meta.hot 上的 HMR API 类型定义





## CSS

> 导入的 .css 文件会把内容插入到 style 标签中，也带有 HRM 支持；

### @import 内联和重命名

1. Vite 通过 postcss-import 预配置支持了 CSS  @import 内联；
2. Vite 的路径别名也遵从 CSS @import；

### PostCSS

如果项目包含有效的 PostCSS 配置 ，例如 postcss.config.js，它将会自动应用于所有已导入的 CSS；

### CSS Modules

> 任何以 .module.css 为后缀名的  CSS 文件都被认为是一个 CSS  modules 文件，导入这样的文件会返回一个相应的模块对象；

```javascript
// example.module.css
.red {
  color: red;
}

// demo.js
import classes from './example.module.css'
document.getElementById('foo').className = classes.red

// 当css.modules.localsConvention 设置了 camelCase 格式变量名转换（例如 localsConvention: 'camelCaseOnly'），可以使用按名导入
import { applyColor } from './example.module.css'
document.getElementById('foo').className = applyColor
```

### CSS 预处理器

Vite 也同时提供了对 .scss, .sass, .less, .styl 和 .stylus 文件的内置支持，所以不用为它们安装特定的 Vite 插件，但必须安装相应的预处理器依赖；

1. 如果是用的是单文件组件，可以通过 < style lang="sass" >（或其他与处理器）自动开启；
2. Vite 为 Sass 和 Less 引进了 @import 解析，保证别名也能使用；
3. 由于和 Stylus API 冲突，@import 别名和 URL 变基不支持 Stylus；

## 静态资源

导入一个静态资源会返回解析后的 URL；

添加一些特殊的查询参数可以更改资源被引入的方式：

```javascript
// 显式加载资源为一个 URL
import assetAsURL from './asset.js?url'

// 以字符串形式加载资源
import assetAsString from './shader.glsl?raw'

// 加载为 Web Worker
import Worker from './worker.js?worker'

// 在构建时 Web Worker 内联为 base64 字符串
import InlineWorker from './worker.js?worker&inline'
```

## JSON

> JSON 文件支持直接导入和具名导入；

```javascript
// 导入整个对象
import json from './example.json'
// 对一个根字段使用具名导入 —— 有效帮助 treeshaking！
import { field } from './example.json'
```

## Glob 导入

> Vite 支持使用特殊的 import.meta.glob 函数从文件系统导入多个模块；

```javascript
// 懒加载，通过动态导入实现，并在构建时分离为独立的 chunk
const modules = import.meta.glob('./dir/*.js')
// 直接引入所有模块
const modules = import.meta.globEager('./dir/*.js')

// vite 生成的代码【glob】
const modules = {
  './dir/foo.js': () => import('./dir/foo.js'),
  './dir/bar.js': () => import('./dir/bar.js')
}

for (const path in modules) {
  modules[path]().then((mod) => {
    console.log(path, mod)
  })
}

// vite 生成的代码【globEager】
import * as __glob__0_0 from './dir/foo.js'
import * as __glob__0_1 from './dir/bar.js'
const modules = {
  './dir/foo.js': __glob__0_0,
  './dir/bar.js': __glob__0_1
}
```

## 构建优化

> 下面所罗列的功能会自动应用为构建过程的一部分，没有必要在配置中显式地声明，除非想禁用它们；

### CSS 代码分割

1. Vite 会自动地将一个异步 chunk 模块中使用到的 CSS 代码抽取出来并为其生成一个单独的文件；
2. 这个文件将在异步 chunk 加载完成时自动通过 link 标签载入；
3. 该异步 chunk 会保证只在 CSS 加载完成后执行；
4. 如果倾向将所有的 CSS 抽取到一个文件中，可以通过设置 build.cssCodeSplit 为 false 禁用；

### 预加载指令生成

Vite 会为入口 chunk 和它们在打包出的 HTML 中的直接引入自动生成 < link rel="modulepreload" > 指令；

### 异步 Chunk 加载优化

1. 在无优化的情境下，当异步 chunkA  被导入时，浏览器将必须请求和解析A，然后再请求和解析 ChunkB ,这会导致额外的网络往返；
2. Vite 将使用一个预加载步骤自动重写代码，来分割动态导入调用，因而当 ChunkA 被请求时，ChunkB 也将同时被获取到；
3. Vite 的优化会跟踪所有的直接导入，无论导入的深度如何，都能够完全消除不必要的往返；

## HMR实现原理

- 通过 watcher 监听文件改动
- 通过 server 端编译资源，并推送新模块内容给浏览器
- 浏览器收到新的模块内容，执行框架层面的 rerend/reload

## 插件

1. 要使用一个插件，先将它添加到项目的 devDependencies；
2. 在 vite.config.js 配置文件中的 plugins 数组中引入它；
3. 如果需要强制执行某些插件的顺序，使用 enforce 修饰符来强制插件的位置；
4. 如果插件在服务或构建期间按需使用，使用 apply 属性指明它们仅在 ‘build’ 或 ‘serve’ 模式时调用；

```javascript
// vite.config.js
import legacy from '@vitejs/plugin-legacy'

export default {
  plugins: [
    legacy({
      targets: ['defaults', 'not IE 11']
    }),
    {
      ...image(),
      enforce: 'pre' // 强制插件顺序
    }，
    {
      ...typescript2(),
      apply: 'build' // 按需引入 
    }
  ]
}
```

