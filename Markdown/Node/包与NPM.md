CommonJS 的包规范的定义是由包结构和包描述文件两个部分组成；

1. 包结构文件用于组织包中的各种文件；
2. 包描述文件用于描述包的相关信息；

https://segmentfault.com/q/1010000009864039 【dependencies、devDependencies 的区别】

https://segmentfault.com/a/1190000008398819 【`dependency`，`devDependency`】

https://zhuanlan.zhihu.com/p/128625669 【npm Install 详解】

https://blog.csdn.net/liuyan19891230/article/details/103856130

## 包结构

> 包实际上是一个存档文件，即一个目录直接打包为 .zip 或 tar.gz 格式的文件，安装解压后还原为目录；

完全符合 CommonJS 规范的包目录结构应该包含如下：

1. package.json：包描述文件；
2. bin: 用于存放可执行二进制文件的目录；
3. lib: 用于存放 Javascript 代码的目录；
4. doc: 用于存放文档的目录；
5. test: 用于存放单元测试用例的代码；

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

