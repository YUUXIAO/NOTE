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

## Lock

### 固定版本

> 固定版本就是直接把第三方依赖版本都写死；

- 虽然锁定了 webpack 的版本，但是 webpack 的依赖却没有办法锁定，如果 webpack 的某个依赖法生产不遵循 semver 的 breaking change，我们的应用还是会受到影响；
- 除非我们保证所有的第三方以及第三方的依赖都是写死版本，这即意味着整个社区放弃 semver，这显然是不可能的；

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/254f89758c0a4750aa0b789cdcced031~tplv-k3u1fbpfcp-zoom-1.image)

### yarn lock vs npm lock

> yarn 的 lock 和 npm 的 lock 都支持将项目里的依赖和第三方依赖同时锁定；

- vant 的所有依赖及其依赖的依赖的版本在 lock 文件里都锁定了, 这样另一个用户或者环境，能够凭借 lock 文件复现 node_modules 里各个库的版本；
- 当我们第一次安装创建项目时或者第一次安装某个依赖的时候，此时即使第三方库里含有 lock 文件，但是 npm install|(yarn install) 并不会去读取第三方依赖的 lock，这导致第一次创建项目的时候，用户还是会可能触发 bug，这有可能是该依赖的版本的某个上游依赖发生了 breaking change, 因为不存在全局环境的 lock；

比如项目安装了 vant 的依赖：

```javascript
"dependencies": {
  "vant": "2.11.1",
},
```

其 lock 文件如下：

```javascript
"vant": {
  "version": "2.11.1",
  "resolved": "https://registry.npmjs.org/vant/-/vant-2.11.1.tgz",
  "integrity": "sha512-w98McSiJLQUiud8HV0k1SokJhF9s2lmqIwVm2IH64/HBEhCLaFgRYxDe5oeRIWNSP06K3NK95NTp7aUaWs/X2w==",
  "requires": {
    "@babel/runtime": "7.x",
    "@vant/icons": "1.4.0",
    "@vant/popperjs": "^1.0.0",
    "@vue/babel-helper-vue-jsx-merge-props": "^1.0.0",
    "vue-lazyload": "1.2.3"
  },
  "dependencies": {
    "vue-lazyload": {
      "version": "1.2.3",
      "resolved": "https://registry.npmjs.org/vue-lazyload/-/vue-lazyload-1.2.3.tgz",
      "integrity": "sha512-DC0ZwxanbRhx79tlA3zY5OYJkH8FYp3WBAnAJbrcuoS8eye1P73rcgAZhyxFSPUluJUTelMB+i/+VkNU/qVm7g=="
    }
  }
},
```

### Resolutions

如果安装了一个新的webpack-cli，发现该 cli 的一个上游依赖portfinder的最近一个版本有bug 且这个 bug 不能及时修复；

yarn 提供了一个 Selective dependency resolutions 的机制，使得可以忽略 dependency 的限制，强行将portfinder锁定为某个没有bug的版本；

- npm 本身没有提供 resolution 机制，但可以通过 npm-force-resolution  这个库实现类似机制；

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d2e1b26027c94c3ca66664ac8bb7fe02~tplv-k3u1fbpfcp-zoom-1.image)

### 库里应该提交 lock 文件吗

因为 npm 和 yarn 在 install 的时候并不会读取第三方库里的 lock 文件，那所以在编写库的时候必要提供 lock 文件；

如果库的开发者将当前的编译环境的 lock 提交上来，可以很大程度上避免编译工具的某个上游依赖的 breaking change 所致的 bug；

### determinism

> determinism 指的是在给定 package.json 和 lock 文件下，每次重新 install 都会得到同样的 node_modules 的拓扑结构；

1. yarn 仅保证了同一版本的确定性而无法保证不同版本的确定性；
2. npm 保证了不同版本的确定性；

yarn.lock 保证了所有第三方库和其依赖的版本号是锁定的，但是 yarn.lock 里并没有包含任何的 node_modules 拓扑信息；

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bbd594ea9f91459fbf69926ea0711192~tplv-k3u1fbpfcp-zoom-1.image)

上面的例子，该 lock 文件只保证了 has-flag 的版本和 suppors-colors 的版本，却没有保证 has-flag 是出现在 top level 还是出现在 supports-color 里，如下两种拓扑结构都是合理的：

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4ea27b380f1445749a9e688962b856bb~tplv-k3u1fbpfcp-zoom-1.image)

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3b35849679af42a092513adff70ffcdb~tplv-k3u1fbpfcp-zoom-1.image)

npm 的 lock 信息则包含了拓扑结构信息，下面结构表明 has-flag 和 supports-color 处于同一层级；

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a7723474d4674eb7a3e976f035b7096b~tplv-k3u1fbpfcp-zoom-1.image)

