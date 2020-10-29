## webpack 文件监听

> 文件监听是在发现源码发生改变时，自动重新构建新的输出文件；
>
> 缺陷：每次需要手动刷新浏览器；

webpack 开启监听模式，有两种方法：

1. 启动 webpack 命令时，加上 --watch 参数；
2. 在配置 webpack.config.js 中 设置 watch: true；

### 文件监听原理

轮询判断文件的最后编辑时间是否在变化，某个文件发生了变化，并不会立即告诉监听者，而是先缓存起来，等 aggregateTimeout；

```javascript
module.exports = {
	// 默认为 false， 也就是不开启
	watch: true,
	// 只有开启监听模式的时候，watchOptions 才有意义
	watchOptions: {
		// 默认为空，不监听的文件或文件夹，支持正则匹配
		ignored: /node_modules/,
		// 监听到变化后会等 300ms 再去执行，默认 300ms
		aggregateTimeout: 300,
		// 判断文件是否发生变化是通过不停询问系统指定文件有没有变化实现的，默认每秒问 1000 次
		poll: 1000
	}
}
```

## webpack 热更新及原理解析

> Hot Module Replacement，简称 HMR，无需完全刷新整个页面的同时，更新模块；

![img](https://img2020.cnblogs.com/blog/1735070/202009/1735070-20200910094229881-465690773.png)

1. Webpack Compiler：将 JS 编译成 Bundle；

2. HMR Server：将热更新的文件输出给 HMR Runtime；

3. Bundle Server：提供文件在浏览器的访问；

   > 比如说编译好的 bundle.js，其实在浏览器里面正常访问是以文件目录的形式来访问的，然后使用 BundleServer 可以让我们以类似于服务器的方式来访问，比如说 localhost:8080

4. HMR Runtime：会被注入到浏览器，更新文件的变化；

   > 开发阶段，打包阶段，会注入到浏览器中的 bundle.js 里面，这样浏览器端的 bundle.js 会和服务器建立一个链接，通常是 websocket；这样就能动态更新了；

5. bundle.js：构建输出的文件；

### webpack的编译构建过程

1. 在项目启动后，进行构建打包，控制台会输出构建过程同时会生成 一个 hash 值；
2. 每次修改代码保存后，会生成 一个 新的 hash 值、新的 json 文件、新的 js 文件；
   - hash：代表每次编译的标志，本次输出的 hash 会被作为下次热更新的标志；
   - json 文件：返回结果中，h 代表本次新生成的 hash，用于下次文件热更新的标志； c 代表当前要热更新的文件对应的模块；
   - js 文件：就是本次修改的代码，重新打包编译后的；

### 热更新原理

####  webpack-dev-server启动本地服务

```javascript
// node_modules/webpack-dev-server/bin/webpack-dev-server.js

// 生成webpack编译主引擎 compiler
let compiler = webpack(config);

// 启动本地服务
let server = new Server(compiler, options, log);
server.listen(options.port, options.host, (err) => {
  if (err) {throw err};
});

```

本地服务代码：

```javascript
// node_modules/webpack-dev-server/lib/Server.js
class Server {
    constructor() {
        this.setupApp();
        this.createServer();
    }
    
    setupApp() {
        // 依赖了express
    	this.app = new express();
    }
    
    createServer() {
        this.listeningApp = http.createServer(this.app);
    }
    listen(port, hostname, fn) {
        return this.listeningApp.listen(port, hostname, (err) => {
            // 启动express服务后，启动websocket服务
            this.createSocketServer();
        }
    }                                   
}
```

这一小节代码主要做了三件事：

1. 启动 webpack ,生成 compiler 实例；
2. 使用 express 框架启动本地 server , 让浏览器可以请求本地的静态资源；
3. 本地 server 启动之后，再去启动 websocket 服务，通过 websocket，可以建立本地服务和浏览器的双向通信。这样就可以实现当本地文件发生变化，立马告知浏览器可以热更新代码；

#### 修改webpack.config.js的entry配置

启动本地服务前，调用了 updateCompiler（this.compiler）方法，这个方法有两个关键方法：

1. 获取 webscoket 客户端代码路径；
2. 根据配置获取 webpack 热更新代码路径；

```javascript
// 获取websocket客户端代码
const clientEntry = `${require.resolve(
    '../../client/'
)}?${domain}${sockHost}${sockPath}${sockPort}`;

// 根据配置获取热更新代码
let hotEntry;
if (options.hotOnly) {
    hotEntry = require.resolve('webpack/hot/only-dev-server');
} else if (options.hot) {
    hotEntry = require.resolve('webpack/hot/dev-server');
}
```

修改后的 webpack 入口配置如下：

```javascript
// 修改后的entry入口
{ entry:
    { index: 
        [
            // 上面获取的clientEntry
            'xxx/node_modules/webpack-dev-server/client/index.js?http://localhost:8080',
            // 上面获取的hotEntry
            'xxx/node_modules/webpack/hot/dev-server.js',
            // 开发配置的入口
            './src/index.js'
    	],
    },
}      

```

#### webpack-dev-server/client/index.js

这个文件是用于 scoket 的，因为 webscoket 是双向通信，在 webpack-dev-server 初始化的过程中，启动的是本地服务端的 webscoket；所以我们需要把 webscoket 客户端通信代码写到我们的代码中；

#### webpack/hot/dev-server.js

这个文件是检查更新逻辑的；

### 监听webpack编译结束

修改好入口配置后，又调用了 setupHooks 方法，这个方法是用来注册监听事件的，监听每次 webpack 编译完成；

```javascript
// node_modules/webpack-dev-server/lib/Server.js
// 绑定监听事件
setupHooks() {
    const {done} = compiler.hooks;
    // 监听webpack的done钩子，tapable提供的监听方法
    done.tap('webpack-dev-server', (stats) => {
        this._sendStats(this.sockets, this.getStats(stats));
        this._stats = stats;
    });
};
```

当监听到一次 webpack 编译结束，就会调用 _sendStats 方法通过 webscoket 给浏览器发送通知， ok 和 hash 事件，这样浏览器就可以拿到最新的 hash 值了，做检查更新逻辑；

```javascript
// 通过websoket给客户端发消息
_sendStats() {
    this.sockWrite(sockets, 'hash', stats.hash);
    this.sockWrite(sockets, 'ok');
}
```

### webpack监听文件变化

每次修改代码会触发编译，主要是通过 setupDevMiddleware 方法实现的，这个方法主要执行了 webpack-dev-middleware 库；

> webpack-dev-server 只负责启动服务和前置准备工作，所有文件相关的操作都抽离到了 webpack-dev-middleware 库了，主要是本地文件的编译和输出以及监听，职责的划分更为清晰；

```javascript
// node_modules/webpack-dev-middleware/index.js
compiler.watch(options.watchOptions, (err) => {
    if (err) { /*错误处理*/ }
});

// 通过“memory-fs”库将打包后的文件写入内存
setFs(context, compiler); 
```

1. 调用 compiler.watch 方法，这个方法只要做了2件事：
   - 对本地文件代码进行编译打包，也就是 webpack 的一系列编译流程；
   - 编译结束后，开启对本地文件的监听，当文件发生变化，重新编译，编译完成后继续监听；
   - 监听本地文件的变化主要是通过文件的生成时间是否有变化；
2. 执行 setFs 方法，这个方法主要就是将编译后的文件打包到内存，因为访问内存中的代码比访问文件系统中的文件更快，而且也减少了代码写入文件的开销，这一切都归功于memory-fs；

### 浏览器接收到热更新的通知

当监听到一次 webpack 编译结束，_sendStats 方法就通过 webscoket 给浏览器发送通知，检查是否需要热更新；

在 entry 配置中增加的入口文件，也就是 webscoket 客户端代码：

```javascript
'xxx/node_modules/webpack-dev-server/client/index.js?http://localhost:8080'
```

这个代码会被打包到 bundle.js 中，运行到浏览器中：

```javascript
// webpack-dev-server/client/index.js
var socket = require('./socket');
var onSocketMessage = {
    hash: function hash(_hash) {
        // 更新currentHash值
        status.currentHash = _hash;
    },
    ok: function ok() {
        sendMessage('Ok');
        // 进行更新检查等操作
        reloadApp(options, status);
    },
};
// 连接服务地址socketUrl，?http://localhost:8080，本地服务地址
socket(socketUrl, onSocketMessage);

function reloadApp() {
	if (hot) {
        log.info('[WDS] App hot update...');
        
        // hotEmitter其实就是EventEmitter的实例
        var hotEmitter = require('webpack/hot/emitter');
        hotEmitter.emit('webpackHotUpdate', currentHash);
    } 
}
```

1. scoket 方法建立了 webscoket 和服务端的连接，并注册了2个监听事件：
   - hash 事件：更新最新一次打包后的 hash 值；
   - ok 事件：进行热更新检查；
2. 热更新检查是调用 reloadApp 方法，利用 node.js的 EventEmitter，发出webpackHotUpdate 消息；
3. webscoket 仅仅有于客户端和服务端进行通信，而真正的检查交给了 webpack；

在 entry 配置中增加的第三个入口文件：

```javascript
'xxx/node_modules/webpack/hot/dev-server.js'
```

这个文件的代码同样会被打包到 bundle.js 中，运行在浏览器中：

```javascript
// node_modules/webpack/hot/dev-server.js
var check = function check() {
    module.hot.check(true)
        .then(function(updatedModules) {
            // 容错，直接刷新页面
            if (!updatedModules) {
                window.location.reload();
                return;
            }
            
            // 热更新结束，打印信息
            if (upToDate()) {
                log("info", "[HMR] App is up to date.");
            }
    })
        .catch(function(err) {
            window.location.reload();
        });
};

var hotEmitter = require("./emitter");
hotEmitter.on("webpackHotUpdate", function(currentHash) {
    lastHash = currentHash;
    check();
});

```

这里 webpack 监听到了 webpackHotUpdate 事件，并获取最新了最新的 hash 值，然后进行检查更新了，检查更新呢调用的是 module.hot.check 方法；

### HotModuleReplacementPlugin

对比 配置热更新和不配置时 bundle.js 的区别：

1. 发现 moudle 新增了一个属性为 hot，再看 hotCreateModule 方法，找到 module.hot.check 是哪里冒出来的；
2. 经过对比打包后的文件，__ webpack_require __中的 moudle 以及代码行数的不同，可以发现 HotModuleReplacementPlugin 也是塞了很多代码到 bundle.js 中呀。因为检查更新是在浏览器中操作，这些代码必须在运行时的环境；

### moudle.hot.check 开始热更新

moudle.hot.check 方法主要做了：

1. 利用上一次保存的 hash 值，调用 hotDownloadManifest 发送 xxx/hash.hot-update.json 中的 ajax 请求；
2. 请求结果获取热更新模块，以及下次热更新的 hash 标识，并进入热更新准备联阶段；

```javascript
hotAvailableFilesMap = update.c; 	// 需要更新的文件
hotUpdateNewHash = update.h; 		// 更新下次热更新hash值
hotSetStatus("prepare"); 			// 进入热更新准备状态
```

3. 调用 hotDownloadUpdateChunk 发送  xxx/hash.hot-update.js 请求，通过 JSONP方式，因为 JSONP 获取的代码可以直接执行：

```javascript
function hotDownloadUpdateChunk(chunkId) {
    var script = document.createElement("script");
    script.charset = "utf-8";
    script.src = __webpack_require__.p + "" + chunkId + "." + hotCurrentHash + ".hot-update.js";
    if (null) script.crossOrigin = null;
    document.head.appendChild(script);
 }

```

xxx/hash.hot-update.js 返回新编译后的代码是在一个 webpackHotUpdate 函数体内部的，也就是要立即执行 webpackHotUpdate 这个方法；

![img](https://user-gold-cdn.xitu.io/2019/12/1/16ec04316d6ac5e3?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

```javascript
window["webpackHotUpdate"] = function (chunkId, moreModules) {
    hotAddUpdateChunk(chunkId, moreModules);
} ;
```

- hotAddUpdateChunk 方法会把更新的模块 moreModules 赋值给全局变量 hotUpdate；
- hotUpdateDownloaded 方法会调用 hotApply 进行代码替换；

```javascript
function hotAddUpdateChunk(chunkId, moreModules) {
    // 更新的模块moreModules赋值给全局全量hotUpdate
    for (var moduleId in moreModules) {
        if (Object.prototype.hasOwnProperty.call(moreModules, moduleId)) {
	    hotUpdate[moduleId] = moreModules[moduleId];
        }
    }
    // 调用hotApply进行模块的替换
    hotUpdateDownloaded();
}
```

### hotApply 热更新模块替换

#### 删除过期的模块，就是需要替换的模块

通过 hotUpdate 可以找到旧模块：

```javascript
var queue = outdatedModules.slice();
while (queue.length > 0) {
    moduleId = queue.pop();
    // 从缓存中删除过期的模块
    module = installedModules[moduleId];
    // 删除过期的依赖
    delete outdatedDependencies[moduleId];
    
    // 存储了被删掉的模块id，便于更新代码
    outdatedSelfAcceptedModules.push({
        module: moduleId
    });
}
```

#### 将新的模块添加到 modules 中

```javascript
appliedUpdate[moduleId] = hotUpdate[moduleId];
for (moduleId in appliedUpdate) {
    if (Object.prototype.hasOwnProperty.call(appliedUpdate, moduleId)) {
        modules[moduleId] = appliedUpdate[moduleId];
    }
}
```

#### 通过__ webpack_require __执行相关模块的代码

```javascript
for (i = 0; i < outdatedSelfAcceptedModules.length; i++) {
    var item = outdatedSelfAcceptedModules[i];
    moduleId = item.module;
    try {
        // 执行最新的代码
        __webpack_require__(moduleId);
    } catch (err) {
        // ...容错处理
    }
}
```

![img](https://user-gold-cdn.xitu.io/2019/12/1/16ec13499800dfce?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

### 工作原理图解

![img](https://pic1.zhimg.com/80/v2-f7139f8763b996ebfa28486e160f6378_720w.jpg)

```
绿色的方框是 webpack 代码控制的区域;
蓝色方框是 webpack-dev-server 代码控制的区域;
洋红色的方框是文件系统，文件修改后的变化就发生在这，而青色的方框是应用本身;
Manifest： 它是webpack维护的一份用于管理构建过程中所有模块及关联关系的数据表，包含了各个模块之间的依赖关系、模块内容等详细信息，是webpack解析和加载模块的重要依据；
```

1. 第一步：在 webpack 的 watch 模式下，文件系统的某一个文件发生修改，webpack 监听到文件变化，根据配置文件对模块重新编译打包，并将打包后的代码保存在内存中；
2. 第二步：是 webpack-dev-server 和 webpack 之间的接口交互，在这一步主要是 dev-server 的中间件 webpack-dev-middleware 和 webpack 之间的交互， webpack-dev-middleware 调用 webpack 暴露的 API 对代码变化进行监控，并且告诉 webpack，将代码打包到内存中；
3. 第三步：是 webpack-dev-server 对文件变化的一个监控，不同于第一步，并不是监控代码变化重新打包，当我们在配置文件中配置了 devServer.watchContentBase 为 true 的时候，Server 会监听这些配置文件夹中静态文件的变化，变化后会通知浏览器端对应用进行 live reload。这儿是浏览器刷新，和 HMR 是两个概念；
4. 第四步：是 webpack-dev-server 代码的工作，该步骤主要是通过 sockjs（webpack-dev-server 的依赖）在浏览器端和服务端之间建立一个 websocket 长连接，将 webpack 编译打包的各个阶段的状态信息告知浏览器端，同时也包括第三步中 Server 监听静态文件变化的信息。浏览器端根据这些 socket 消息进行不同的操作。当然服务端传递的最主要信息还是新模块的 hash 值，后面的步骤根据这一 hash 值来进行模块热替换；
5. 第五步：webpack-dev-server/client 端并不能够请求更新的代码，也不会执行热更模块操作，而把这些工作又交回给了 webpack，webpack/hot/dev-server 的工作就是根据 webpack-dev-server/client 传给它的信息以及 dev-server 的配置决定是刷新浏览器还是进行模块热更新。如果只是刷新浏览器，也就没有后面那些步骤了；
6. 第六步：HotModuleReplacement.runtime 是客户端 HMR 的中枢，它接收到上一步传递给他的新模块的 hash 值，它通过 JsonpMainTemplate.runtime 向 server 端发送 Ajax 请求，服务端返回一个 json，该 json 包含了所有要更新的模块的 hash 值，获取到更新列表后，该模块再次通过 jsonp 请求，获取到最新的模块代码。这就是上图中 7、8、9 步骤；
7. 第十步：是决定 HMR 成功与否的关键步骤，在该步骤中，HotModulePlugin 将会对新旧模块进行对比，决定是否更新模块，在决定更新模块后，检查模块之间的依赖关系，更新模块的同时更新模块间的依赖引用；
8. 第十一步：当 HMR 失败后，回退到 live reload 操作，也就是进行浏览器刷新来获取最新打包代码；

### 源码说明

1. webpack 对文件系统进行 watch 打包到内存中：

   webpack-dev-middleware 调用 webpack 的 api 对文件系统 watch ，当文件改变后，webpack重新对文件进行打包编译，然后保存到内存中：

   ```javascript
   // webpack-dev-middleware/lib/Shared.js
   if(!options.lazy) {
       var watching = compiler.watch(options.watchOptions, share.handleCompilerCallback);
       context.watching = watching;
   }
   ```

   - webpack 将 bundle.js 文件打包到了内存中，不生成文件的原因在于访问内存中的代码比访问文件系统中的文件更快，也减少了代码写入文件的开销；
   - memory-fs 是 webpack-dev-middleware 的一个依赖库，webpack-dev-middleware 将 webpack 原本的 outputFileSystem 替换成了MemoryFileSystem 实例，这样代码就将输出到内存中;

   ```javascript
   // webpack-dev-middleware/lib/Shared.js
   var isMemoryFs = !compiler.compilers && compiler.outputFileSystem instanceof MemoryFileSystem;
   if(isMemoryFs) {
       fs = compiler.outputFileSystem;
   } else {
       fs = compiler.outputFileSystem = new MemoryFileSystem();
   }
   ```

2. devServer 通知浏览器端文件发生改变：

   sockjs 是服务端和浏览器端之间的桥梁，在启动 devServer 的时候，sockjs 在服务端和浏览器端建立了一个 webSocket 长连接，以便将 webpack 编译和打包的各个阶段状态告知浏览器，最关键的步骤还是 webpack-dev-server 调用 webpack api 监听 compile的 done 事件，当compile 完成后，webpack-dev-server通过 _sendStatus 方法将编译打包后的新模块 hash 值发送到浏览器端；

   ```javascript
   // webpack-dev-server/lib/Server.js
   compiler.plugin('done', (stats) => {
     // stats.hash 是最新打包文件的 hash 值
     this._sendStats(this.sockets, stats.toJson(clientStats));
     this._stats = stats;
   });
   ...
   Server.prototype._sendStats = function (sockets, stats, force) {
     if (!force && stats &&
     (!stats.errors || stats.errors.length === 0) && stats.assets &&
     stats.assets.every(asset => !asset.emitted)
     ) { return this.sockWrite(sockets, 'still-ok'); }
     // 调用 sockWrite 方法将 hash 值通过 websocket 发送到浏览器端
     this.sockWrite(sockets, 'hash', stats.hash);
     if (stats.errors.length > 0) { this.sockWrite(sockets, 'errors', stats.errors); } 
     else if (stats.warnings.length > 0) { this.sockWrite(sockets, 'warnings', stats.warnings); }      else { this.sockWrite(sockets, 'ok'); }
   };
   ```

3. webpack-dev-server/client 接收到服务端消息做出响应：

   - webpack-dev-server 修改了webpack 配置中的 entry 属性，在里面添加了 webpack-dev-client 的代码，这样在最后的 bundle.js 文件中就会有接收 websocket 消息的代码；
   - webpack-dev-server/client 当接收到 type 为 hash 消息后会将 hash 值暂存起来，当接收到 type 为 ok 的消息后对应用执行 reload 操作，hash 消息是在 ok 消息之前；
   - 在 reload 操作中，webpack-dev-server/client 会根据 hot 配置决定是刷新浏览器还是对代码进行热更新（HMR）：
     1. 首先将 hash 值暂存到 currentHash 变量，当接收到 ok 消息后，对 App 进行 reload；
     2. 如果配置了模块热更新，就调用 webpack/hot/emitter 将最新 hash 值发送给 webpack，将控制权交给 webpack 客户端代码；
     3. 如果没有配置模块热更新，就直接调用 location.reload 方法刷新页面；

   ```javascript
   // webpack-dev-server/client/index.js
   hash: function msgHash(hash) {
       currentHash = hash;
   },
   ok: function msgOk() {
       // ...
       reloadApp();
   },
   // ...
   function reloadApp() {
     // ...
     if (hot) {
       log.info('[WDS] App hot update...');
       const hotEmitter = require('webpack/hot/emitter');
       hotEmitter.emit('webpackHotUpdate', currentHash);
       // ...
     } else {
       log.info('[WDS] App updated. Reloading...');
       self.location.reload();
     }
   }
   ```

4. webpack 接收到最新 hash 值验证并请求模块代码：

   - 首先是 webpack/hot/dev-server 监听第三步 webpack-dev-server/client 发送的 webpackHotUpdate 消息，调用 webpack/lib/HotModuleReplacement.runtime 中的 check 方法，检测是否有新的更新；
   - 在 check 过程中会利用 webpack/lib/JsonpMainTemplate.runtime 中的两个方法 hotDownloadUpdateChunk 和 hotDownloadManifest ;
   - hotDownloadManifest 方法是调用 AJAX 向服务端请求是否有更新的文件，如果有将发更新的文件列表返回浏览器端，返回的是最新的 hash 值；
   - hotDownloadUpdateChunk 是通过 jsonp 请求最新的模块代码，然后将代码返回给 HMR runtime，HMR runtime 会根据返回的新模块代码做进一步处理，可能是刷新页面，也可能是对模块进行热更新；

5. HotModuleReplacement.runtime 对模块进行热更新：

   - 模块热更新都是发生在HMR runtime 中的 hotApply 方法中；
   - 如果在热更新过程中出现错误，热更新将回退到刷新浏览器；
   - 第一阶段先找出 outdatedModules 和 outdatedDependencies；
   - 第二个阶段从缓存中删除过期的模块和依赖；
   - 第三个阶段是将新的模块添加到 modules 中，当下次调用 __ webpack_require __ (webpack 重写的 require 方法)方法的时候，就获取到了新的模块代码了；

   ```javascript
   // webpack/lib/HotModuleReplacement.runtime
   function hotApply() {
       // ...
       var idx;
       var queue = outdatedModules.slice();
       while(queue.length > 0) {
           moduleId = queue.pop();
           module = installedModules[moduleId];
           // ...
           // remove module from cache
           delete installedModules[moduleId];
           // when disposing there is no need to call dispose handler
           delete outdatedDependencies[moduleId];
           // remove "parents" references from all children
           for(j = 0; j < module.children.length; j++) {
               var child = installedModules[module.children[j]];
               if(!child) continue;
               idx = child.parents.indexOf(moduleId);
               if(idx >= 0) {
                   child.parents.splice(idx, 1);
               }
           }
       }
       // ...
       // insert new code
       for(moduleId in appliedUpdate) {
           if(Object.prototype.hasOwnProperty.call(appliedUpdate, moduleId)) {
               modules[moduleId] = appliedUpdate[moduleId];
           }
       }
       // ...
   }
   ```

6. 业务代码需要做些什么

   - 当用新的模块代码替换老的模块后，但是我们的业务代码并不能知道代码已经发生变化，在文件修改后，我们要在 index.js 文件中调用 HMR 的 accept 方法，添加模块更新后的处理函数：

   ```javascript
   // index.js
   if(module.hot) {
       module.hot.accept('./hello.js', function() {
           div.innerHTML = hello()
       })
   }
   ```

### 热更新流程总结

1. Webpack编译期，为需要热更新的 entry 注入热更新代码(EventSource通信)；
2. 页面首次打开后，服务端与客户端通过 EventSource 建立通信渠道，把下一次的 hash 返回前端；
3. 客户端获取到hash，这个hash将作为下一次请求服务端 hot-update.js 和 hot-update.json的hash；
4. 修改页面代码后，Webpack 监听到文件修改后，开始编译，编译完成后，发送 build 消息给客户端；
5. 客户端获取到hash，成功后客户端构造hot-update.js script链接，然后插入主文档；
6. hot-update.js 插入成功后，执行hotAPI 的 createRecord 和 reload方法，获取到 Vue 组件的 render方法，重新 render 组件， 继而实现 UI 无刷新更新；