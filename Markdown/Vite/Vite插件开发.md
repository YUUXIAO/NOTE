## 参考文档

[官方文档 ](https://cn.vitejs.dev/guide/api-plugin.html)

https://blog.csdn.net/qq_34621851/article/details/123535975?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_baidulandingword~default-1-123535975-blog-124456738.pc_relevant_3mothn_strategy_and_data_recovery&spm=1001.2101.3001.4242.2&utm_relevant_index=4

https://juejin.cn/post/7258201505336410169

## vite 工作机制

用户向vite请求资源时，vite 内部启了一个开发服务器 vite devServer ，这个服务器内部有个工作机制（相当于一个流水线），用户请求过来必定经过所有插件去处理一下，处理完成之后把转换的结果返回给用户

## ViteDevServer对象

ViteDevServer 主要是 createServer 方法返回的对象，主是包含一些对象实例和启动 devServer 的相关核心方法

[ViteDevServer 的ts定义](https://cn.vitejs.dev/guide/api-javascript.html#vitedevserver)

config.plugins 会返回所有配置的plugins信息，每个对象会说明插件信息和调用的钩子函数

![vite3](F:\Yabby\NOTE\images\vite\vite3.png)

createServer 方法主要处理：

- 配置参数解析：vite 核心配置、http配置、chokidar配置
- 创建 HTTP 和 webscoket server，用来处理热更新
- 启动 chokidar 实现文件监听-> 热更新
- 创建 ModuleGraph 实例，记录模块依赖关系
- 创建插件容器，管理插件的生命周期、执行过程、插件之间传递上下文、
- 定义ViteServer 对象，
- 执行 configureServer 定义函数，创建自定义中间件
- 定义内部中间件

## 处理 index.html 文件（transformIndexHtml）

vite 提供了一个转换 index.html （可以利用这个钩子让相关插件向 HMTL 中加入一些自已的代码/功能）的专用钩子 [`transformIndexHtml`](https://cn.vitejs.dev/guide/api-plugin.html#transformindexhtml)，这个钩子用来接收当前的 HTML 字符串和转换上下文

vite 在这一步加入了 @vite/client 文件，用来客户端客户端创建 WebSocket，接收服务端热更新传递的消息 

这个钩子返回其下类型之一：

- 经过转换的 HTML 字符串
- 注入到现有 HTML 中的标签描述符对象数组（`{ tag, attrs, children }`）。每个标签也可以指定它应该被注入到哪里（默认是在 `<head>` 之前）
- 一个包含 `{ html, tags }` 的对象

```javascript
server.transformIndexHtml = createDevHtmlTransformFn(server); 

function createDevHtmlTransformFn(server) {
    const [preHooks, postHooks] = resolveHtmlTransforms(server.config.plugins);
    return (url, html, originalUrl) => {
        return applyHtmlTransforms(html, [...preHooks, devHtmlHook, ...postHooks], {
            path: url,
            filename: getHtmlFilename(url, server),
            server,
            originalUrl
        });
    };
}

function resolveHtmlTransforms(plugins) {
    const preHooks = [];
    const postHooks = [];
    for (const plugin of plugins) {
        const hook = plugin.transformIndexHtml;
        if (hook) {
            if (typeof hook === 'function') {
                postHooks.push(hook);
            }
            else if (hook.enforce === 'pre') {
                preHooks.push(hook.transform);
            }
            else {
                postHooks.push(hook.transform);
            }
        }
    }
    return [preHooks, postHooks];
}
```

默认情况下，这个钩子会在HTML 转换后应用，但是开发者也可以通过传入 order = 'pre'｜‘post’ 来定义钩子函数在处理 HMTL 前应用 或者是在所有未定义 order 的钩子函数被应用后应用

```javascript
async function applyHtmlTransforms(html, hooks, ctx) {
    const headTags = [];
    const headPrependTags = [];
    const bodyTags = [];
    const bodyPrependTags = [];
    for (const hook of hooks) {
        const res = await hook(html, ctx);
        if (!res) {
            continue;
        }
        if (typeof res === 'string') {
            html = res; // 针对直接返回html字符串的情况
        }
        else {
            let tags;
            if (Array.isArray(res)) {
                tags = res; // 返回对象数组（｛tag，attrs, children｝）
            }
            else {
                // 包含｛html，tags｝的对象 
                html = res.html || html;
                tags = res.tags;
            }
            // 处理标签插入HTML位置
            for (const tag of tags) {
                if (tag.injectTo === 'body') {
                    bodyTags.push(tag);
                }
                else if (tag.injectTo === 'body-prepend') {
                    bodyPrependTags.push(tag);
                }
                else if (tag.injectTo === 'head') {
                    headTags.push(tag);
                }
                else {
                    headPrependTags.push(tag);
                }
            }
        }
    }
    // inject tags
    if (headPrependTags.length) {
        html = injectToHead(html, headPrependTags, true);
    }
    if (headTags.length) {
        html = injectToHead(html, headTags);
    }
    if (bodyPrependTags.length) {
        html = injectToBody(html, bodyPrependTags, true);
    }
    if (bodyTags.length) {
        html = injectToBody(html, bodyTags);
    }
    return html;
}
```

## transformMiddleware中间件

transformMiddleware 中间件主要处理大部分 js、vue、css 等文件资源的加载、解析和转换，针对了一些特定的请求做相关处理：

- isJSRequest：js、ts、jsx、tsx、mjs、vue、svelte等后缀的文件
- isImportRequest：以?或&i开头的import
- isCSSRequest：css、less、sass、scss、styl、stylus、pcss、postcss后缀的文件
- isHTMLProxy：?html-proxy&index=*.js文件

```javascript
 if (isJSRequest(url) ||
                isImportRequest(url) ||
                isCSSRequest(url) ||
                isHTMLProxy(url)) {
                // strip ?import
                url = removeImportQuery(url);
                // Strip valid id prefix. This is prepended to resolved Ids that are
                // not valid browser import specifiers by the importAnalysis plugin.
                url = unwrapId(url);
                // for CSS, we need to differentiate between normal CSS requests and
                // imports
                if (isCSSRequest(url) &&
                    !isDirectRequest(url) &&
                    ((_c = req.headers.accept) === null || _c === void 0 ? void 0 : _c.includes('text/css'))) {
                    url = injectQuery(url, 'direct');
                }
                // check if we can return 304 early
                const ifNoneMatch = req.headers['if-none-match'];
                if (ifNoneMatch &&
                    ((_e = (_d = (await moduleGraph.getModuleByUrl(url, false))) === null || _d === void 0 ? void 0 : _d.transformResult) === null || _e === void 0 ? void 0 : _e.etag) === ifNoneMatch) {
                    isDebug$1 && debugCache(`[304] ${prettifyUrl(url, root)}`);
                    res.statusCode = 304;
                    return res.end();
                }
                // resolve, load and transform using the plugin container
                const result = await transformRequest(url, server, {
                    html: (_f = req.headers.accept) === null || _f === void 0 ? void 0 : _f.includes('text/html')
                });
                if (result) {
                    const type = isDirectCSSRequest(url) ? 'css' : 'js';
                    const isDep = DEP_VERSION_RE.test(url) || isOptimizedDepUrl(url);
                    return send(req, res, result.code, type, {
                        etag: result.etag,
                        // allow browser to cache npm deps!
                        cacheControl: isDep ? 'max-age=31536000,immutable' : 'no-cache',
                        headers: server.config.server.headers,
                        map: result.map
                    });
                }
            }
```

上面也会设置对应响应头信息，利用浏览器缓存机制尽可能缓存模块文件

- 模块文件用304 - ETag＋If-Non-Match 协商缓存（容易变更）
- 依赖模块通过 设置 Cache-Control = max-age=31536000,immutable 强缓存，因为它们基本不需要改动再次缓存

### transformRequest

```
function transformRequest(url, server, options = {}) {
    const cacheKey = (options.ssr ? 'ssr:' : options.html ? 'html:' : '') + url;
    // This module may get invalidated while we are processing it. For example
    // when a full page reload is needed after the re-processing of pre-bundled
    // dependencies when a missing dep is discovered. We save the current time
    // to compare it to the last invalidation performed to know if we should
    // cache the result of the transformation or we should discard it as stale.
    //
    // A module can be invalidated due to:
    // 1. A full reload because of pre-bundling newly discovered deps
    // 2. A full reload after a config change
    // 3. The file that generated the module changed
    // 4. Invalidation for a virtual module
    //
    // For 1 and 2, a new request for this module will be issued after
    // the invalidation as part of the browser reloading the page. For 3 and 4
    // there may not be a new request right away because of HMR handling.
    // In all cases, the next time this module is requested, it should be
    // re-processed.
    //
    // We save the timestamp when we start processing and compare it with the
    // last time this module is invalidated
    const timestamp = Date.now();
    const pending = server._pendingRequests.get(cacheKey);
    if (pending) {
        return server.moduleGraph
            .getModuleByUrl(removeTimestampQuery(url), options.ssr)
            .then((module) => {
            if (!module || pending.timestamp > module.lastInvalidationTimestamp) {
                // The pending request is still valid, we can safely reuse its result
                return pending.request;
            }
            else {
                // Request 1 for module A     (pending.timestamp)
                // Invalidate module A        (module.lastInvalidationTimestamp)
                // Request 2 for module A     (timestamp)
                // First request has been invalidated, abort it to clear the cache,
                // then perform a new doTransform.
                pending.abort();
                return transformRequest(url, server, options);
            }
        });
    }
    const request = doTransform(url, server, options, timestamp);
    // Avoid clearing the cache of future requests if aborted
    let cleared = false;
    const clearCache = () => {
        if (!cleared) {
            server._pendingRequests.delete(cacheKey);
            cleared = true;
        }
    };
    // Cache the request and clear it once processing is done
    server._pendingRequests.set(cacheKey, {
        request,
        timestamp,
        abort: clearCache
    });
    request.then(clearCache, clearCache);
    return request;
}
```



### doTransform

doTransform 方法主要是返回模块处理后的 code 、map和 etag 对象

主要流程：

1. 去掉 url 上的时间戳，再根据 url 通过 ModuleGraph 获取 ModuleNode

2. 判断该 ModuleNode 判断是否经过处理，如果是直接返回 cached

3. 获取文件（code 和 map）：

   1. 如果是插件就调用插件的 load 方法，`pluginContainer.load(id, { ssr })`
   2. 非插件：读取文件的内容 

4. 对文件进行监听

   ```javascript
   const mod = await moduleGraph.ensureEntryFromUrl(url, ssr);
   ensureWatchedFile(watcher, mod.file, root);
   ```

5. 对代码进行转换，也会注入一些公共代码：

   ```javascript
      const transformResult = await pluginContainer.transform(code, id, {
           inMap: map,
           ssr
       });
    if (map && mod.file) {
           map = (typeof map === 'string' ? JSON.parse(map) : map);
           if (map.mappings && !map.sourcesContent) {
               await injectSourcesContent(map, mod.file, logger);
           }
       }
   ```

   ​

## 插件钩子

### 通用钩子

在开发中，Vite 开发服务器会创建一个插件容器来调用 [Rollup 构建钩子](https://rollupjs.org/plugin-development/#build-hooks)，与 Rollup 如出一辙

在**服务器启动时**被调用：

- options：在收集 rollup 配置前，Vite 服务器启动时调用，可以和 rollup 配置进行合并


- buildStart：在 rollup 构建中，vite 服务启动时调用，在这里可以访问 rollup 的配置

在**每个传入模块请求时**被调用：

- transform：代码转译，这个函数的功能类似于 webpack 的 loader功能
- resolveId：在解析模块时调用，可以返回一个特殊的 resolveId 来指定某个 import 语句来加载特定的模块
- load：在解析模块时调用，可以返回代码块来指定某个 `import` 语句加载特定的模块

在**服务器关闭时**被调用：

- buildEnd：在 `vite` 本地服务关闭前，`rollup` 输出文件到目录前调用
- closeBundle：在 `vite` 本地服务关闭前，`rollup` 输出文件到目录前调用

### Vite 独有钩子

在解析 Vite 配置前调用：config

- config：在解析 Vite 配置后调用，可以在这里读取 Vite 的配置
- configResolved：在解析 Vite 配置后调用，可以读取 Vite 的配置，进行一些操作
- configureServer：是用于配置开发服务器的钩子，最常见的是在内部 connect 应用程序中添加自定义中间件
- handleHotUpdate：在执行自定义 HMR（模块热替换）更新处理
- transformIndexHtml：转换 `index.html` 的专用钩子。

## 插件顺序

插件可以额外指定一个 enforce 属性来调整它的应用顺序

- Alias
- 【执行带有 enforce：’pre‘ 的用户插件】：这个时机可以直接解析到原模板文件，如果需要处理原模模版文件应该配置这个
- Vite 核心插件
- 【执行没有配置 enforce属性的用户插件】
- Vite 构建用的插件
- 【执行没有配置 enforce：’post‘属性的用户插件】
- Vite 后置构建插件（最小化，manifest，报告）

## 创建插件入口文件

入口文件的内容主要是要有 name、enforce、transform 三个属性

- name：插件名称
- enforce：该插件在 plugin-vue 插件之前执行，这样就可以直接解析到原模板文件
- transform：代码转译，这个函数的功能类似于 webpack 的 loader功能

```js
export default function myVitePlugin(): Plugin {
  return {
    // 插件名称
    name: 'vite:markdown',
    // 该插件在 plugin-vue 插件之前执行，这样就可以直接解析到原模板文件
    enforce: 'pre',
    // 代码转译，这个函数的功能类似于 `webpack` 的 `loader`
    transform(code, id, opt) {}
  }
}

module.exports = myVitePlugin
myVitePlugin['default'] = myVitePlugin
```



