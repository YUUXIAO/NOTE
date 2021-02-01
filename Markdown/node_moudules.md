## Dependency Hell

## npm

> node 的解决方式是依赖的 node 加载模块的路径查找算法和 node_moudules 的目录结构来配合解决；

### 从 node_moudules 加载 package

> 核心是递归向上查找 node_modules 里的 package；

如果在 '/home/ry/projects/foo.js'  文件里调用了 require('bar.js')，则 Node.js 会按以下顺序查找：

1. /home/ry/projects/node_modules/bar.js
2. /home/ry/node_modules/bar.js
3. /home/node_modules/bar.js
4. /node_modules/bar.js

该算法有两个核心：

- 优先读取最近的 node_moudules 的依赖；
- 递归向上查找 node_modules 的依赖；

## 目录结构

### nest mode



### flat mode

### doppelgangers

## 版本重复

### 全局types冲突

虽然各个 package 之前的代码不会相互污染，但是他们的 types 仍然可以相互影响，很多的第三方库会修改全局的类型定义，典型的就是 @types/react, 如下是一个常见的错误；

- 错误原因就在于全局的 types 形成了命名冲突，因此假如版本重复可能会导致全局的类型错误，一般的解决方式就是自己控制包含哪些加载的 @types/xxx；

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2cd45f8e68cb44ccb128e0ec53d23d39~tplv-k3u1fbpfcp-zoom-1.image)

### 破坏单例模式

#### require 的缓存机制

> node 会对加载的模块进行缓存，第一次加载某个模块后会将结果缓存下来，后续的 require 调用都返回同一结果；

- node 的 require 的缓存不是基于 module 名，而是基于 resolve 的文件的路径，且大小写敏感；

以 react-loadable 为例，其同时在 browser 和 node 层使用；

browser 里使用：

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b7a3e3cac9ad448494b34656f80006fe~tplv-k3u1fbpfcp-zoom-1.image)

node 层使用：

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4a05b223bf3b45899bc842002e37ec45~tplv-k3u1fbpfcp-zoom-1.image)

