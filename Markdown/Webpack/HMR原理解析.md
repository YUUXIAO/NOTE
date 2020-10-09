```
Hot Module Replacement（简称 HMR）是 webpack，当你对代码进行修改并保存后，webpack 将对代码重新打包，并将新的模块发送到浏览器端，浏览器通过新的模块替换老的模块，这样在不刷新浏览器的前提下就能够对应用进行更新
```

## 为什么需要 HMR

1. live reload 工具并不能够保存应用的状态，当刷新页面后，应用之前状态丢失，而 webapck HMR 则不会刷新浏览器，而是运行时对模块进行热替换，保证了应用状态不会丢失，提升了开发效率；（比如点击按钮显示弹窗）
2. 简化开发流程，不需要手动运行代码打包刷新页面，通过 HMR 工作流自动化完成；
3. HMR 兼容市面上大多前端框架或库，能够监听 React 或者 Vue 组件的变化，实时将最新的组件更新到浏览器端；

## 工作原理图解

![img](https://pic1.zhimg.com/80/v2-f7139f8763b996ebfa28486e160f6378_720w.jpg)

```
绿色的方框是 webpack 代码控制的区域;
蓝色方框是 webpack-dev-server 代码控制的区域;
洋红色的方框是文件系统，文件修改后的变化就发生在这，而青色的方框是应用本身;
```

1. 第一步：在 webpack 的 watch 模式下，文件系统的某一个文件发生修改，webpack 监听到文件变化，根据配置文件对模块重新编译打包，并将打包后的代码通过简单的 javascript 对象保存在内存中；
2. 第二步：是 webpack-dev-server 和 webpack 之间的接口交互，在这一步主要是 dev-server 的中间件 webpack-dev-middleware 和 webpack 之间的交互， webpack-dev-middleware 调用 webpack 暴露的 API 对代码变化进行监控，并且告诉 webpack，将代码打包到内存中；
3. 第三步：是 webpack-dev-server 对文件变化的一个监控，不同于第一步，并不是监控代码变化重新打包，当我们在配置文件中配置了 devServer.watchContentBase 为 true 的时候，Server 会监听这些配置文件夹中静态文件的变化，变化后会通知浏览器端对应用进行 live reload。这儿是浏览器刷新，和 HMR 是两个概念；
4. 第四步：是 webpack-dev-server 代码的工作，该步骤主要是通过 sockjs（webpack-dev-server 的依赖）在浏览器端和服务端之间建立一个 websocket 长连接，将 webpack 编译打包的各个阶段的状态信息告知浏览器端，同时也包括第三步中 Server 监听静态文件变化的信息。浏览器端根据这些 socket 消息进行不同的操作。当然服务端传递的最主要信息还是新模块的 hash 值，后面的步骤根据这一 hash 值来进行模块热替换；
5. 第五步：webpack-dev-server/client 端并不能够请求更新的代码，也不会执行热更模块操作，而把这些工作又交回给了 webpack，webpack/hot/dev-server 的工作就是根据 webpack-dev-server/client 传给它的信息以及 dev-server 的配置决定是刷新浏览器还是进行模块热更新。如果只是刷新浏览器，也就没有后面那些步骤了；
6. 第六步：HotModuleReplacement.runtime 是客户端 HMR 的中枢，它接收到上一步传递给他的新模块的 hash 值，它通过 JsonpMainTemplate.runtime 向 server 端发送 Ajax 请求，服务端返回一个 json，该 json 包含了所有要更新的模块的 hash 值，获取到更新列表后，该模块再次通过 jsonp 请求，获取到最新的模块代码。这就是上图中 7、8、9 步骤；
7. 第十步：是决定 HMR 成功与否的关键步骤，在该步骤中，HotModulePlugin 将会对新旧模块进行对比，决定是否更新模块，在决定更新模块后，检查模块之间的依赖关系，更新模块的同时更新模块间的依赖引用；
8. 第十一步：当 HMR 失败后，回退到 live reload 操作，也就是进行浏览器刷新来获取最新打包代码。