> 包实际上是一个存档文件，即一个目录直接打包为 .zip 或 tar.gz 格式的文件，安装解压后还原为目录；

完全符合 CommonJS 规范的包目录结构应该包含如下：

1. package.json：包描述文件；
2. bin: 用于存放可执行二进制文件的目录；
3. lib: 用于存放 Javascript 代码的目录；
4. doc: 用于存放文档的目录；
5. test: 用于存放单元测试用例的代码；

## package.json文件

[xmind属性字段分类](https://mmbiz.qpic.cn/mmbiz_png/EO58xpw5UMO5o6m7MzbCAbXRYJGekcC98XV28Oia6K9DUwHN2sAp1jdBDa0UFFFl6COoONvIf9xOh0oG1sicnUnQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

假设我们现有项目包结构如下：

```json
// package.json
{
  "name": "project",
  "dependencies": {
    "packageA": "^1.0.0"
  },
  "devDependencies": {
    "packageB": "^1.0.0"
  }
}
```

![](image/dependcy.png)

当我们在 project 项目下执行 npm install 时，packageA、 packageB 、packageA1都会添加到 node_modules 文件夹中，packageA2 则不会下载，在node_modules 文件夹中找不到该依赖

## 依赖配置

总结放最上面吧，其实package.json 的依赖配置都是node 设计的一个配置规范，

如果你项目只是自己在开发或者工作环境不改变，写不写或者依赖放的位置在哪都无所谓，npm install 都会安装这些依赖

如果你是在开发一个 node 包去提供给别人引用，那可能就要注意这些依赖放哪个位置，因为在 npm install 时，只会安装配置在 dependencies 下的依赖到 node_modules 里，否则运行打包时在 node_modules 找不到对应的依赖会报错

模块可否被打包，取决于项目里是否被引入了该模块！不是看依赖放在dependencies  或者是  devDependencies下

根据项目依赖包的不同用途，可以将他们配置在下面的五个属性：

### dependencies 与 devDependencies

dependencies  pm install xxx --save，将依赖添加到 dependencies 下，表示为项目运行且上线也依然需要的依赖，比如 vue、elementUI 等业务依赖

npm install xxx --save-dev，将依赖添加到 devDependencies下，表示为项目在开发阶段中需要的依赖，上线时无需打包此依赖，比如 webpack、typescript、prettier、eslint、babel 等做为测试、打包、编译等功能作用的开发依赖

可以理解为如果是自已团队的项目里，将依赖安装到 dependencies 或 devDependencies 下，其实没有太大的区别，执行 npm install 时都会下载包到 node_modules 文件夹下

如果只在自己的项目里，也不想安装自己的 devDependencies 下的依赖，可以使用 npm install --production 命令，

如果是开发一个 node 包或者别的项目需要依赖你的包，那么将依赖添加到 dependencies 或 devDependencies 可能需要好好留意一下

### peerDependencies

假设 packageA 依赖中声明了 peerDependencies 属性下有 packageC，可以理解为使用到 packageA 的项目需要同时安装 packageC，否则程序就可能会有异常，目的是插件所依赖的包是宿主环境统一安装的 npm 包，解决了插件和宿主环境依赖版本不一致的情况

如果在 npm3.x 下执行 npm install，控制台就会告警  UNMET PEER DEPENDENCY packageC（只是打印警告提示）

如果在 npm1.x 或 npm2.x 下执行 npm install，不会报错而是自动会把 packageA 依赖的 peerDependencies 属性下的包安装上（强制安装）

比如安装 [ant-design@3.x](mailto:ant-design@3.x)，它只是提供一套 react 组件的 UI 库，所以它是要求在 运行环境要安装指定版本的 react 依赖

```javascript
//  ant-design@3.x 的 package.json 部分配置
{
  'name': 'ant-design',
  'version': '3.1'
  "peerDependencies": {
    "react": ">=16.0.0",
    "react-dom": ">=16.0.0"
  }
}

// 此时项目中 node_modules下依赖结构为
-- ant-design
-- react
-- react-dom

// 项目中使用
import * as React from 'react';
import * as ReactDOM from 'react-dom';
```

在项目中执行 npm install 时 ，可以在 node_modules 下找到 react 和 react-dom 依赖

此时在项目中引入的 react 或 react-dom，其实是引用的运行项目自己提供的依赖包，不是用的 node_modules 下 ant-design 依赖下的 react 或 react-dom

如果在 npm3.x 下，peerDependencies 属性下的依赖是不会强制安装的，所以 如果需要引入 react-dom，是需要开发者在项目的 package.json 手动安装该属性下的依赖，如果你项目中的一个依赖包更新了它的  peerDependencies ，开发者也需要自己更新项目中 package.json 该依赖的版本

![](image/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_16759138135947.png)

### bundledDependencies

假设你的项目 package.json 文件如下：

```json
{
  "name": "packageA",
  "dependencies": {
    "packageA1": "xxx",
    "packageA2": "xxx",
    "packageA3": "xxx"
  },
  "bundleDependencies": [
    "packageA1",
    "packageA2"
  ]
}
```

那么当你的项目中使用 npm3.x 安装依赖 packageA 时，项目的 node_modules 会是下面这样

````
-- node_modules
----packageA
------packageA1
------packageA2
----packageA3
````

这个属性的作用就是在项目安装了依赖 packageA 时，将该依赖下 bundleDependencies 所声明的依赖安装在自身的 node_modules下，其他的依赖否则会按照 npm3.x 的“扁平”处理

### optionalDependencies

如果你项目中的一个依赖了一个包 packageA，这个 packageA 没有安装，你仍然想让程序正常执行，这个时候就可以配置 optionalDependencies  属性

如果同一个依赖packageA同时在 dependencies 和 optionalDependencies 中声明，option 还会*覆盖* dependency 的声明。

如果 package-optional 这个包是可选的，在代码中需要做兼容处理：

```javascript
try {
    var pkgOpt = require('packageOptional');
} catch (e) {
    pkgOpt = null;
}
```

## NPM

### 嵌套结构

npm3.x之前的版本，处理依赖的方式是以一个递归的形式处理：

严格按照 package.json 结构以及子依赖包的 package.json 的结构将依赖安装到他们各自的 node_modules 中

优点：node_modules 的结构和 package.json 结构一一对应，实现了多版本兼容，层级结构明显，做并且保证了每次安装目录结构都是相同的，

缺点：如果出现不同层级的包引用了同一个依赖的情况下，会造成相同模块冗余，不能复用，目录结构嵌套较深的问题

### 扁平解构

npm3.x之后的版本，将早期的嵌套结构改为扁平结构，就是把所有依赖以及其子依赖包都统一“拍平”处理，

安装模块时，不管其是直接依赖还是子依赖的依赖，**优先**将其安装在 node_modules 根目录

当安装到相同模块时，先判断已安装的模块版本是否符合新模块的版本范围，如果符合则跳过，不符合则在当前模块的 node_modules 下安装该模块。

在项目代码中引用了模块，模块查找流程：

当前模块路径 --> 当前模块node_modules --> 上级模块node_modules --> ... --> 全局路径中的 node_modules

比如：项目中依赖的包有 A、B、C，A下依赖了 Aa@^1.0.1，B下依赖了Bb@^1.0.1，C不依赖其他模块,此时在执行 npm install 时得到的目录结构：

````
app
A、Aa@^1.0.2、B、Bb@^1.0.1、C
````

如果此时项目又引入了 Aa@^1.0.1，判断得到新模块的版本范围符合  Aa@^1.0.2，则跳过安装

如果此时B引入了 Aa@^1.0.3，判断得到新模块的版本范围不符合  Aa@^1.0.2，此时在执行 npm install 时得到的目录结构：

````
app
A、Aa@^1.0.2、B、Bb@^1.0.1、C
-、-、Aa@^1.0.3、-
````

此时会发现，扁平结构的设计并不能完全解决老版本的模块冗余问题，还带来了一个新的问题

在执行  npm install 的时候，按照 package.json 里依赖的顺序依次解析，如果项目依赖A和依赖B同时引用了同一模块C的不同版本，模块C在node_modules 的依赖结构取决于依赖A和依赖B在 package.json 的顺序

在 package.json通常只会锁定大版本，而且在某些依赖包小版本更新后，同样可能造成依赖结构的改动，依赖结构的不确定性可能会给程序带来不可预知的问题

### Lock文件（json格式）

npm 5.x 版本新增了 package-lock.json 文件，而安装方式还是 npm 3.x 的扁平化的方式，Lock 文件解决了 npm3.x install 时依赖不稳定的情况

lock.json 的作用是锁定依赖结构，即只要你目录下有 package-lock.json 文件，那么你每次执行 npm install 后生成的 node_modules 目录结构一定是完全相同的。

```javascript
"name": "xxx",
"version": "0.1.0",
"lockfileVersion": 1,
"requires": true,
 // dependencies 是一个对象，和node_modules中的包结构一一对应，key为包名称，值为包的一些描述信息
"dependencies":｛
  "element-ui": {
    "version": "2.14.1", 
    "resolved": "https://registry.npmjs.org/element-ui/-/element-ui-2.14.1.tgz", 
    "integrity": "sha512-Uje0J12dBaXdyvt/EtuDA8diFbYTdO7uI4QCfl7zmEJmE1WxgCSVKhlRRoL8MDonO8pyNVhB4n0AFAR14g56nw==",
    "requires": {
      "async-validator": "~1.8.1",
      "babel-helper-vue-jsx-merge-props": "^2.0.0",
      "deepmerge": "^1.2.0",
      "normalize-wheel": "^1.0.1",
      "resize-observer-polyfill": "^1.5.0",
      "throttle-debounce": "^1.0.1"
    },
    "dependencies":{
      "is-extendable": {
          "version": "1.0.1",
          "resolved": "xx",
          "integrity": "xx",
          "dev": true,
          "requires": {}
        }
    }
  }
｝
```

- `version`：包版本 —— 这个包当前安装在 `node_modules` 中的版本
- `resolved`：包具体的安装来源
- `integrity`：包 `hash` 值，基于 `Subresource Integrity` 来验证已安装的软件包是否被改动过、是否已失效
- `requires`：对应子依赖的依赖，与子依赖的 `package.json` 中 `dependencies`的依赖项相同。
- `dependencies`：结构和外层的 `dependencies` 结构相同，存储安装在子依赖 `node_modules` 中的依赖包。

  并不是所有的子依赖都有 `dependencies` 属性，只有子依赖的依赖和当前已安装在根目录的  `node_modules` 中的依赖冲突之后，才会有这个属性


优点：lock文件已经记录缓存子依赖包包的具体版本和下载链接，不需要再去远程仓库进行查询，可以直接进入文件完整性校验环节，减少了大量网络请求，所以项目中使用 lock 文件可以显著加速依赖安装时间

在团队开发项目时，建议上传lock文件到远端，这样能保证所有项目开发者以及 CI 环节可以在执行 npm install 时安装的依赖版本都是一致的

如果是在开发一个 npm包 时，你的 npm包 是需要被其他仓库依赖的，由于上面我们讲到的扁平安装机制，如果你锁定了依赖包版本，你的依赖包就不能和其他依赖包共享同一 semver 范围内的依赖包，这样会造成不必要的冗余。所以我们不应该把package-lock.json 文件发布出去（ npm 默认也不会把 package-lock.json文件发布出去）

### 缓存

在执行 npm install 或 npm update命令下载依赖后，除了将依赖包安装在node_modules 目录下外，还会在本地的缓存目录缓存一份

通过 npm config get cache 命令可以查询到：在 Linux 或 Mac 默认是用户主目录下的 .npm/_cacache 目录。  
在这个目录下又存在两个目录：content-v2、index-v5，content-v2 目录用于存储 tar包的缓存，而index-v5目录用于存储tar包的 hash。  
npm 在执行安装时，可以根据 package-lock.json 中存储的 integrity、version、name 生成一个唯一的 key 对应到 index-v5 目录下的缓存记录，从而找到 tar包的 hash，然后根据 hash 再去找缓存的 tar包直接使用。

以上的缓存策略是从 npm v5 版本开始的，在 npm v5 版本之前，每个缓存的模块在 ~/.npm 文件夹中以模块名的形式直接存储，储存结构是{cache}/{name}/{version}。

### 文件完整性

在下载依赖包之前，我们一般就能拿到 npm 对该依赖包计算的 hash 值

用户下载依赖包到本地后，需要确定在下载过程中没有出现错误，所以在下载完成之后需要在本地在计算一次文件的 hash 值，如果两个 hash 值是相同的，则确保下载的依赖是完整的，如果不同，则进行重新下载。

### NPM 钩子命令

> package.json 文件中的 scripts 字段的提出就是让包在安装或卸载等过程中提供钩子的作用；

````
"scripts":{
  "preinstall":"preinstall.js",
  "install":"install.js",
  "preuninstall":"uninstall.js",
}
````

### 发布包

1. 编写模块；

   ```javascript
   // hello.js
   exports.sayHello = function(){
     return "hello, world"
   }
   ```

2. 初始化包描述文件；

   - 配置 package.json 文件；

3. 注册包仓库账户；

   - 注册账号的命令是 npm adduser，这是一个提问式的交互过程；

4. 上传包；

   - 上传包的命令是 npm publish < folder >，在 package.json 文件所在的目录下执行命令；

5. 管理包权限；

   - 如果需要多人进行发布，可以使用 npm owner 命令帮助管理包的所有者；

   ```javascript
   npm owner ls <package_name>
   npm owner add <user> <package_name>
   npm owner rm <user> <package_name>
   ```

6. 分析包；

   - 使用 npm ls 命令分析出当前路径下通过模块路径找到的所有的包，并生成依赖树；


### npm ci

npm ci 可以理解为是和 npm install 差不多的功能，用来下载安装依赖包到 node_modules 里，一般用在持续集成的环境中

与 npm install 的区别：

- npm ci 要求项目中有 package-lock.json 或npm-shrinkwrap.json 文件，如果没有找到该文件会报错推出执行，并且只根据 package-lock.json 来安装依赖
- npm ci 执行时会先删除项目中的 node_modules 文件夹，再一次性安装所有项目依赖，它相比 npm install 是不能安装单个依赖包的
- 如果检测到 package.json 和 package-lock.json 中的依赖项不匹配的话，将退出并报错，而不是更新两个文件中的版本号

## Yarn

yarn 也是采用的是 npm v3.x 的扁平结构来管理依赖，安装依赖后默认会生成一个 yarn.lock 文件

package-lock.json 使用的是 json 格式，yarn.lock 使用的是一种自定义格式

```json
"@babel/runtime@7.x":
  "integrity" "sha512-38Y8f7YUhce/K7RMwTp7m0uCumpv9hZkitCbBClqQIow1qSbCvGkcegKOXpEWCQLfWmevgRiWokZ1GkpfhbZug=="
  "resolved" "https://registry.npmmirror.com/@babel/runtime/-/runtime-7.18.3.tgz"
  "version" "7.18.3"
  dependencies:
    "regenerator-runtime" "^0.13.4"

"@commitlint/cli@^12.1.1":
  "integrity" "sha1-dANw5VeooX9BUFKCHN1Sduywq5g="
  "resolved" "https://registry.nlark.com/@commitlint/cli/download/@commitlint/cli-12.1.1.tgz"
  "version" "12.1.1"
  dependencies:
    "@commitlint/format" "^12.1.1"
    "@commitlint/lint" "^12.1.1"
    "@commitlint/load" "^12.1.1"
    "@commitlint/read" "^12.1.1"
    "@commitlint/types" "^12.1.1"
    "get-stdin" "8.0.0"
    "lodash" "^4.17.19"
    "resolve-from" "5.0.0"
    "resolve-global" "1.0.0"
    "yargs" "^16.2.0"
```

注意看 yarn.lock 中子依赖的版本号并不是和npm一样是固定的，意味着单独又一个 yarn.lock确定不了 node_modules 目录结构，还需要和 package.json 文件进行配合。而 package-lock.json 只需要一个文件即可确定。

yarn 的缓策略看起来和 npm v5 之前的很像，每个缓存的模块被存放在独立的文件夹，文件夹名称包含了模块名称、版本号等信息。

![img](C:%5CUsers%5Cslb0930%5CAppData%5CLocal%5CTemp%5C%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_16755044198818.png)

使用命令 yarn cache dir可以查看缓存数据的目录。  
yarn 默认使用 prefer-online 模式，即优先使用网络数据，如果网络数据请求失败，再去请求缓存数据。