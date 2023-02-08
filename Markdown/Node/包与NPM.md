CommonJS 的包规范的定义是由包结构和包描述文件两个部分组成；

1. 包结构文件用于组织包中的各种文件；
2. 包描述文件用于描述包的相关信息；

https://blog.csdn.net/azl397985856/article/details/103982369?spm=1001.2101.3001.6650.10&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EOPENSEARCH%7ERate-10-103982369-blog-126813607.pc_relevant_3mothn_strategy_and_data_recovery&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EOPENSEARCH%7ERate-10-103982369-blog-126813607.pc_relevant_3mothn_strategy_and_data_recovery&utm_relevant_index=11

https://segmentfault.com/q/1010000009864039 【dependencies、devDependencies 的区别】

**~~https://segmentfault.com/a/1190000008398819 【`dependency`，`devDependency`】~~**

**<u>~~https://zhuanlan.zhihu.com/p/128625669 【npm 进化详解】~~</u>**

**~~https://blog.csdn.net/liuyan19891230/article/details/103856130【npm install 原理分析】~~**

## package.json文件

xmind属性字段分类

https://mmbiz.qpic.cn/mmbiz_png/EO58xpw5UMO5o6m7MzbCAbXRYJGekcC98XV28Oia6K9DUwHN2sAp1jdBDa0UFFFl6COoONvIf9xOh0oG1sicnUnQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1

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

![dependcy](F:\Yabby\NOTE\Markdown\Node\image\dependcy.png)

当我们在 project 项目下执行 npm install 时，



## 依赖配置

根据项目依赖包的不同用途，可以将他们配置在下面的五个属性：

#### dependencies 与 devDependencies

> 该字段声明的是项目的生产环境所必须的依赖包，安装依赖时使用--save参数，可以将新安装的npm包写入dependencies属性。



#### devDependencies



#### peerDependencies

#### bundledDependencies

#### optionalDependencies 

## 包结构

> 包实际上是一个存档文件，即一个目录直接打包为 .zip 或 tar.gz 格式的文件，安装解压后还原为目录；

完全符合 CommonJS 规范的包目录结构应该包含如下：

1. package.json：包描述文件；
2. bin: 用于存放可执行二进制文件的目录；
3. lib: 用于存放 Javascript 代码的目录；
4. doc: 用于存放文档的目录；
5. test: 用于存放单元测试用例的代码；

## NPM

### 嵌套结构

npm3.x之前的版本，处理依赖的方式是以一个递归的形式处理：

严格按照 package.json 结构以及子依赖包的 package.json 的结构将依赖安装到他们各自的 node_modules 中

优点：node_modules 的结构和 package.json 结构一一对应，层级结构明显，并且保证了每次安装目录结构都是相同的

缺点：如果出现不同层级的包引用了同一个依赖的情况下，会造成冗余，不能复用

### 扁平解构

npm3.x之前的版本，将早期的嵌套结构改为扁平结构，就是把所有依赖以及其子依赖包都统一“拍平”处理，

安装模块时，不管其是直接依赖还是子依赖的依赖，**优先**将其安装在 node_modules 根目录

当安装到相同模块时，先判断已安装的模块版本是否符合新模块的版本范围，如果符合则跳过，不符合则在当前模块的 node_modules 下安装该模块。

在项目代码中引用了模块，模块查找流程：

当前模块路径 --> 当前模块node_modules --> 上级模块node_modules --> ... --> 全局路径中的 node_modules

比如：项目中依赖的包有 A、B、C，A下依赖了 Aa@^1.0.1，B下依赖了Bb@^1.0.1，C不依赖其他模块,此时在执行 npm install 时得到的目录结构：

```
app
A、Aa@^1.0.2、B、Bb@^1.0.1、C
```

如果此时项目又引入了 Aa@^1.0.1，判断得到新模块的版本范围符合  Aa@^1.0.2，则跳过安装

如果此时B引入了 Aa@^1.0.3，判断得到新模块的版本范围不符合  Aa@^1.0.2，此时在执行 npm install 时得到的目录结构：

```
app
A、Aa@^1.0.2、B、Bb@^1.0.1、C
-、-、Aa@^1.0.3、-
```

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

![img](C:\Users\slb0930\AppData\Local\Temp\企业微信截图_16755044198818.png)

使用命令 yarn cache dir可以查看缓存数据的目录。
yarn 默认使用 prefer-online 模式，即优先使用网络数据，如果网络数据请求失败，再去请求缓存数据。

## 包描述文件与NPM

> 包描述文件用于表达非代码相关的信息，它是一个 JSON 格式的文件,位于包的根目录下；

CommonJS  为 packag.json 文件定义了一些必需的字段：

1. name：包名，需要由小写字母和数字组成，可以包含 .\ - 和 _ ，但不允许出现空格；
2. description：包简介；
3. version：版本号；
4. keywords：关键词数组，NPM 主要用来做分类搜索；
5. maintainers：包维护者列表，每个维护者由 name、email 和 web 三个属性组成；
6. contributors：贡献者列表；
7. bugs：一个可以反馈 bug 的网页地址或邮件地址；
8. licenses：当前包所使用的许可证列表；
9. responsitories：托管源代码的位置列表；
10. dependencies：使用当前包所需要依赖的包列表；
11. devDependencies：一些模块只在开发时需要依赖；
12. main：模块在引入方法 require() 在引入包时，会优先检查这个字段，并将其作为包中其余模块的入口，如果不存在这个字段，require() 方法会查找包目录下的 index.js、index.node、index.json 文件作为包文件入口；


## 常用功能

借助 NPM，Node 与第三方模块之间形成了一个很好的生态系统；

### 安装依赖包

1. 执行语句 npm install xxx；
2. NPM 会在当前目录下创建 node_modules 目录；
3. 在 node_moudles 目录下创建 xxx 目录，接着将包解压到这个目录下；
4. 安装好依赖包后，直接在代码中调用 require('xxx') 引入该包；
5. require（）方法在做路径分析时会通过模块路径查找到 xxx 所在的位置；

#### 全局模式安装

> 通过命令 npm install xxx -g 进行全局模式安装；

-g 是将一个包安装为全局可用的可执行命令；

通过全局模式安装的所有模块包都被安装进了一个统一的目录下，这个目录可以通过如下 方式推断出来:

```javascript
path.resolve(process.execPath,'..','..','lib','node_modules')
```

#### 从本地安装 

> 对于一些没有发布到 NPM 上的包或者因为网络原因无法直接安装的包，可以通过将包下载到本地，然后本地安装；

本地安装只需为 NPM 指明 package.json 文件所在的位置即可，它可以是：

1. 一个包含 package.json 的存档文件；
2. 一个 URL 地址；
3. 一个目录下有 package.json 文件的目录位置；

```javascript
npm install <tarball file>
npm install <tarball url>
npm install <folder>
```

### NPM 钩子命令

> package.json 文件中的 scripts 字段的提出就是让包在安装或卸载等过程中提供钩子的作用；

```
"scripts":{
  "preinstall":"preinstall.js",
  "install":"install.js",
  "preuninstall":"uninstall.js",
}
```

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

