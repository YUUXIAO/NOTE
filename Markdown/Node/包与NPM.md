CommonJS 的包规范的定义是由包结构和包描述文件两个部分组成；

1. 包结构文件用于组织包中的各种文件；
2. 包描述文件用于描述包的相关信息；

https://blog.csdn.net/azl397985856/article/details/103982369?spm=1001.2101.3001.6650.10&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EOPENSEARCH%7ERate-10-103982369-blog-126813607.pc_relevant_3mothn_strategy_and_data_recovery&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EOPENSEARCH%7ERate-10-103982369-blog-126813607.pc_relevant_3mothn_strategy_and_data_recovery&utm_relevant_index=11

https://segmentfault.com/q/1010000009864039 【dependencies、devDependencies 的区别】

https://segmentfault.com/a/1190000008398819 【`dependency`，`devDependency`】

**https://zhuanlan.zhihu.com/p/128625669 【npm 进化详解】**

https://blog.csdn.net/liuyan19891230/article/details/103856130

## package.json文件

xmind属性字段分类

https://mmbiz.qpic.cn/mmbiz_png/EO58xpw5UMO5o6m7MzbCAbXRYJGekcC98XV28Oia6K9DUwHN2sAp1jdBDa0UFFFl6COoONvIf9xOh0oG1sicnUnQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1

### 依赖配置

根据项目依赖包的不同用途，可以将他们配置在下面的五个属性：

#### dependencies

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

缺点：如果出现不同层级的包引用了同一个模块或包的情况下，会造成冗余，不能复用

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

```json
"name": "xxx",
"version": "0.1.0",
"lockfileVersion": 1,
"requires": true,
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
    }
  }
｝
```



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

