> Axios 是一个基于 Promise 的 HTTP 客户端，同时支持浏览器和 Node.js 环境；

## 特性

1. 支持 Promise API；
2. 能够拦截请求和响应；
3. 能够转换请求和响应数据；
4. 客户端支持防御 CSRF 攻击；
5. 同时支持浏览器和 Node.js 环境；
6. 能够取消请求以及自动转换 JSON 数据；

## HTTP 拦截器

拦截器的作用是在请求发送前统一执行某些操作，比如在请求头中添加 token 字段；

```javascript
// 添加请求拦截器 —— 处理请求配置对象
axios.interceptors.request.use(function (config) {
  config.headers.token = 'added by interceptor';
  return config;
});

// 添加响应拦截器 —— 处理响应对象
axios.interceptors.response.use(function (data) {
  data.data = data.data + ' - modified by interceptor';
  return data;
});

axios({
  url: '/hello',
  method: 'get',
}).then(res =>{
  console.log('axios res.data: ', res.data)
})
```

### 任务注册

#### Axios 对象的定义

axios 实例是通过 CreateInstance 方法创建的，该方法最终返回的是 Axios.prototype.request 函数对象；

```javascript
// lib/axios.js
function createInstance(defaultConfig) {
  var context = new Axios(defaultConfig);
  var instance = bind(Axios.prototype.request, context);

  // Copy axios.prototype to instance
  utils.extend(instance, Axios.prototype, context);
  // Copy context to instance
  utils.extend(instance, context);
  return instance;
}

// Create the default instance to be exported
var axios = createInstance(defaults);
```

#### Axios 的构造函数

interceptors.request 和 interceptors.response 对象都是 InterceptorManager 类的实例：

```javascript
// lib/core/Axios.js
function Axios(instanceConfig) {
  this.defaults = instanceConfig;
  this.interceptors = {
    request: new InterceptorManager(),
    response: new InterceptorManager()
  };
}
```

#### InterceptorManager 构造函数

通过 use 方法，知道注册的拦截器都会被保存到 InterceptorManager 对象的 handlers 属性中：

```javascript
// lib/core/InterceptorManager.js
function InterceptorManager() {
  this.handlers = [];
}

InterceptorManager.prototype.use = function use(fulfilled, rejected) {
  this.handlers.push({
    fulfilled: fulfilled,
    rejected: rejected
  });
  // 返回当前的索引，用于移除已注册的拦截器
  return this.handlers.length - 1;
};
```

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/98a62da97d49487bb294cb55715e6ff9~tplv-k3u1fbpfcp-zoom-1.image)

### 任务编排

#### Axios.prototype.request（）

```javascript
// lib/core/Axios.js
Axios.prototype.request = function request(config) {
  config = mergeConfig(this.defaults, config);

  // 省略部分代码
  var chain = [dispatchRequest, undefined];
  var promise = Promise.resolve(config);

  // 任务编排
  this.interceptors.request.forEach(function unshiftRequestInterceptors(interceptor) {
    chain.unshift(interceptor.fulfilled, interceptor.rejected);
  });

  this.interceptors.response.forEach(function pushResponseInterceptors(interceptor) {
    chain.push(interceptor.fulfilled, interceptor.rejected);
  });

  // 任务调度
  while (chain.length) {
    promise = promise.then(chain.shift(), chain.shift());
  }

  return promise;
};
```

任务编排前和任务编排后的对比图：

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9b660c577cf2498e95b995f4bb804cd0~tplv-k3u1fbpfcp-zoom-1.image)

### 任务调度

任务编排完成后要发起 HTTP 请求，需要按编排后的顺序执行任务调度：

```javascript
 // lib/core/Axios.js
Axios.prototype.request = function request(config) {
  // 省略部分代码
  var promise = Promise.resolve(config);
  while (chain.length) {
    promise = promise.then(chain.shift(), chain.shift());
  }
}
```

因为 chain 是数组，所以通过 while 语句可以不断地取出设置的任务，然后组装成 Promise 调用链从而实现任务调度，对应的处理流程如下图所示：

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/12cbfa5ce9aa4983b80a039eb5e5d83b~tplv-k3u1fbpfcp-zoom-1.image)

## HTTP 适配器

Axios 同时支持浏览器和 Node.js 环境，对于浏览器环境，可以通过 XMLHttpRequest 或 Fetch API 来发送 HTTP 请求；对于Node 环境，可以通过 Node.js 内置的 http 或 https 模块来发送请求；

为了支持不同的环境，Axios 引入了适配器，在拦截器的部分，有一个 dispatchRequest 方法用于发送 HTTP 请求：

```javascript
// lib/core/dispatchRequest.js
module.exports = function dispatchRequest(config) {
  // 省略部分代码
  var adapter = config.adapter || defaults.adapter;
  
  return adapter(config).then(function onAdapterResolution(response) {
    // 省略部分代码
    return response;
  }, function onAdapterRejection(reason) {
    // 省略部分代码
    return Promise.reject(reason);
  });
};
```

- Axios 支持自定义适配器，也提供默认的适配器；
- 默认的适配器会包含浏览器和 Node.js 环境的适配代码；

```javascript
// lib/defaults.js
var defaults = {
  adapter: getDefaultAdapter(),
  xsrfCookieName: 'XSRF-TOKEN',
  xsrfHeaderName: 'X-XSRF-TOKEN',
  //...
}

function getDefaultAdapter() {
  var adapter;
  if (typeof XMLHttpRequest !== 'undefined') {
    // For browsers use XHR adapter
    adapter = require('./adapters/xhr');
  } else if (typeof process !== 'undefined' && 
    Object.prototype.toString.call(process) === '[object process]') {
    // For node use HTTP adapter
    adapter = require('./adapters/http');
  }
  return adapter;
}
```

###  自定义适配器

```javascript
var settle = require('./../core/settle');
module.exports = function myAdapter(config) {
  // 当前时机点：
  //  - config配置对象已经与默认的请求配置合并
  //  - 请求转换器已经运行
  //  - 请求拦截器已经运行
  
  // 使用提供的config配置对象发起请求
  // 根据响应对象处理Promise的状态
  return new Promise(function(resolve, reject) {
    var response = {
      data: responseData,
      status: request.status,
      statusText: request.statusText,
      headers: responseHeaders,
      config: config,
      request: request
    };

    settle(resolve, reject, response);

    // 此后:
    //  - 响应转换器将会运行
    //  - 响应拦截器将会运行
  });
}
```

##  Axios CSRF 防御

> Axios 内部是使用 双重 Cookie 防御 的方案来防御 CSRF 攻击；

Axios 提供了 xsrfCookieName 和 xsrfHeaderName 两个属性来设置 CSRF  和 Cookie 名称和 HTTP 请求名称：

```javascript
// lib/defaults.js
var defaults = {
  adapter: getDefaultAdapter(),

  // 省略部分代码
  xsrfCookieName: 'XSRF-TOKEN',
  xsrfHeaderName: 'X-XSRF-TOKEN',
};
```

Axios 使用不同的适配器来发送 HTTP 请求，这里我们以浏览器平台为例看一下Axios 如何防御 CSRF 攻击：

```javascript
// lib/adapters/xhr.js
module.exports = function xhrAdapter(config) {
  return new Promise(function dispatchXhrRequest(resolve, reject) {
    var requestHeaders = config.headers;
    
    var request = new XMLHttpRequest();
    // 省略部分代码
    
    // 添加xsrf头部
    if (utils.isStandardBrowserEnv()) {
      var xsrfValue = (config.withCredentials || isURLSameOrigin(fullPath)) && config.xsrfCookieName ?
        cookies.read(config.xsrfCookieName) :
        undefined;

      if (xsrfValue) {
        requestHeaders[config.xsrfHeaderName] = xsrfValue;
      }
    }

    request.send(requestData);
  });
};
```

### 双重 Cookie 防御

双重 Cookie 防御就是将 token 设置在 Cookie 中，在提交请求时提交 Cookie ，并通过请求头或请求体带上  Cookie 中已设置的 token ，服务端收到请求后再进行对比校验；