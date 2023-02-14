## 参考文档

[官方文档 ](https://cn.vitejs.dev/guide/api-plugin.html)

## 插件钩子

### 通用钩子

在开发中，Vite 开发服务器会创建一个插件容器来调用 [Rollup 构建钩子](https://rollupjs.org/plugin-development/#build-hooks)，与 Rollup 如出一辙

在服务器启动时被调用：[`options`](https://rollupjs.org/plugin-development/#options) 、[`buildStart`](https://rollupjs.org/plugin-development/#buildstart)

在每个传入模块请求时被调用：[`resolveId`](https://rollupjs.org/plugin-development/#resolveid) 、[`load`](https://rollupjs.org/plugin-development/#load) 、[`transform`](https://rollupjs.org/plugin-development/#transform)

在服务器关闭时被调用：[`buildEnd`](https://rollupjs.org/plugin-development/#buildend) 、[`closeBundle`](https://rollupjs.org/plugin-development/#closebundle)

### Vite 独有钩子

在解析 Vite 配置前调用：config