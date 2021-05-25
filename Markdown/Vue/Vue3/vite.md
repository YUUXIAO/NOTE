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



1. <script setup>
   - 减少组件声明和导出 
   - import 直接导入组件，无需声明
   - defineProps 声明 props
   - defineEmit 声明 emit
   - 获取上下文： useContext()，挂载了 attrs、slots、emit、expose
   - ctx.expose 向外暴露
2. Mock插件： vite-plugin-mock
3. vue-router@4.x和vuex@4.x
4. 样式管理 
5. 引入 element3
   - plugins 插件形式
   - export default function (app)
   - app.use(element3)
6. 基础布局