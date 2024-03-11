## 插件入口文件

入口文件的内容主要是要有 name、enforce、transform 三个属性

- name：插件名称
- enforce：该插件在 plugin-vue 插件之前执行，这样就可以直接解析到原模板文件
- transform：代码转译，这个函数的功能类似于 webpack 的 loader功能

```javascript
export default function myVitePlugin(): Plugin {
  return {
    // 插件名称
    name: 'vite:xxx',
    // 该插件在 plugin-vue 插件之前执行，这样就可以直接解析到原模板文件
    enforce: 'pre',
    // 代码转译，这个函数的功能类似于 `webpack` 的 `loader`
    transform(code, id, opt) {}
  }
}

module.exports = myVitePlugin
myVitePlugin['default'] = myVitePlugin
```

## 插件钩子

### 通用钩子（和rollup共享）

在开发中，Vite 开发服务器会创建一个插件容器来调用 [Rollup 构建钩子](https://rollupjs.org/plugin-development/#build-hooks)，与 Rollup 如出一辙

在**服务器启动时**被调用：

- **options：**在收集 rollup 配置前，Vite 服务器启动时调用，可以和 rollup 配置进行合并
- **buildStart：**在 rollup 构建中，vite 服务启动时调用，在这里可以访问 rollup 的配置

在**每个传入模块请求时**被调用：

- **transform：**代码转译，这个函数的功能类似于 webpack 的 loader功能
- **resolveId：**在解析模块时调用，可以返回一个特殊的 resolveId 来指定某个 import 语句来加载特定的模块
- **load：**在解析模块时调用，可以返回代码块来指定某个 `import` 语句加载特定的模块

在**服务器关闭时**被调用：

- **buildEnd：**在 `vite` 本地服务关闭前，`rollup` 输出文件到目录前调用
- **closeBundle：**在 `vite` 本地服务关闭前，`rollup` 输出文件到目录前调用

### Vite 独有钩子

在解析 Vite 配置前调用：

#### transformIndexHtml

用来转换 index.html 的专用钩子[transformIndexHtml](https://cn.vitejs.dev/guide/api-plugin.html#transformindexhtml)，这个钩子用来接收当前的 HTML 字符串和转换上下文，**可以利用这个钩子让相关插件向 HMTL 中加入一些自已的代码/功能**

这个钩子返回其下类型之一：

- **经过转换的 HTML 字符串：**默认情况下，这个钩子会在HTML 转换后应用，但是开发者也可以通过传入 order = 'pre'｜‘post’ 来定义钩子函数在处理 HMTL 前应用 或者是在所有未定义 order 的钩子函数被应用后应用
- **注入到现有 HTML 中的标签描述符对象数组**（`{ tag, attrs, children }`）：每个标签也可以指定它应该被注入到哪里（默认是在 `<head>` 之前）
- 一个包含 `{ html, tags }` 的对象

#### **config**

在解析 Vite 配置后调用，可以在这里读取 Vite 的配置

#### **configResolved**

在解析 Vite 配置后调用，可以读取 Vite 的配置，进行一些操作

#### **configureServer**

是用于配置开发服务器的钩子，最常见的是在内部 connect 应用程序中添加自定义中间件

#### **handleHotUpdate**

在执行自定义 HMR（模块热替换）更新处理

## 插件顺序

插件可以额外指定一个 enforce 属性来调整它的应用顺序

- Alias
- 【执行带有 enforce：’pre‘ 的用户插件】：这个时机可以直接解析到原模板文件，如果需要处理原模模版文件应该配置这个
- Vite 核心插件
- 【执行没有配置 enforce属性的用户插件】
- Vite 构建用的插件
- 【执行没有配置 enforce：’post‘属性的用户插件】
- Vite 后置构建插件（最小化，manifest，报告）



