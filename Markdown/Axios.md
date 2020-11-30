> Axios 是一个基于 Promise 的 HTTP 客户端，同时支持浏览器和 Node.js 环境；

## 特性

1. 支持 Promise API；
2. 能够拦截请求和响应；
3. 能够转换请求和响应数据；
4. 同时支持浏览器和 Node.js 环境：从浏览器中创建 XMLHttpRequest，从 node.js 创建 http 请求；
5. 能够取消请求以及自动转换 JSON 数据；
6. 客户端支持防御 CSRF 攻击；

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

#### CreateInstance 方法

CreateInstance 函数返回了一个函数 instance：

1. instance 是一个函数 Axios.prototype.request 且执行上下文绑定到 context；
2. instance 里面还有 Axios.prototype 上的所有方法，并且这些方法的执行上下文也绑定到 context；
3. instance 里面还有 context 上的方法；

```javascript
// /lib/axios.js
function createInstance(defaultConfig) {
  // 创建一个Axios实例
  var context = new Axios(defaultConfig);

  // 以下代码也可以这样实现：var instance = Axios.prototype.request.bind(context);
  // 这样instance就指向了request方法，且上下文指向context，所以可以直接以 instance(option) 方式调用 
  // Axios.prototype.request 内对第一个参数的数据类型判断，使我们能够以 instance(url, option) 方式调用
  var instance = bind(Axios.prototype.request, context);

  // 把Axios.prototype上的方法扩展到instance对象上，
  // 这样 instance 就有了 get、post、put等方法
  // 并指定上下文为context，这样执行Axios原型链上的方法时，this会指向context
  utils.extend(instance, Axios.prototype, context);

  // 把context对象上的自身属性和方法扩展到instance上
  // 注：因为extend内部使用的forEach方法对对象做for in 遍历时，只遍历对象本身的属性，而不会遍历原型链上的属性
  // 这样，instance 就有了  defaults、interceptors 属性。（这两个属性后面我们会介绍）
  utils.extend(instance, context);

  return instance;
}

// 接收默认配置项作为参数（后面会介绍配置项），创建一个Axios实例，最终会被作为对象导出
var axios = createInstance(defaults);
```

#### defaults

```javascript
// lib/ defaults.js
var utils = require('./utils');
var normalizeHeaderName = require('./helpers/normalizeHeaderName');

var DEFAULT_CONTENT_TYPE = {
  'Content-Type': 'application/x-www-form-urlencoded'
};

function setContentTypeIfUnset(headers, value) {
  if (!utils.isUndefined(headers) && utils.isUndefined(headers['Content-Type'])) {
    headers['Content-Type'] = value;
  }
}
// getDefaultAdapter 方法是来获取请求的方式
function getDefaultAdapter() {
  var adapter;
  // process 是 node 环境的全局变量
  if (typeof process !== 'undefined' && Object.prototype.toString.call(process) === '[object process]') {
    // 如果是 node 环境那么久通过 node http 的请求方法
    adapter = require('./adapters/http');
  } else if (typeof XMLHttpRequest !== 'undefined') {
   // 如果是浏览器啥的 有 XMLHttpRequest 的就用 XMLHttpRequest
    adapter = require('./adapters/xhr');
  }
  return adapter;
}

var defaults = {
    // adapter 就是请求的方法
  adapter: getDefaultAdapter(),
	// 下面一些请求头，转换数据，请求,详情的数据
    // 这也就是为什么我们可以直接拿到请求的数据时一个对象，如果用 ajax 我们拿到的都是 jSON 格式的字符串
    // 然后每次都通过 JSON.stringify（data）来处理结果。
  transformRequest: [function transformRequest(data, headers) {
    normalizeHeaderName(headers, 'Accept');
    normalizeHeaderName(headers, 'Content-Type');
    if (utils.isFormData(data) ||
      utils.isArrayBuffer(data) ||
      utils.isBuffer(data) ||
      utils.isStream(data) ||
      utils.isFile(data) ||
      utils.isBlob(data)
    ) {
      return data;
    }
    if (utils.isArrayBufferView(data)) {
      return data.buffer;
    }
    if (utils.isURLSearchParams(data)) {
      setContentTypeIfUnset(headers, 'application/x-www-form-urlencoded;charset=utf-8');
      return data.toString();
    }
    if (utils.isObject(data)) {
      setContentTypeIfUnset(headers, 'application/json;charset=utf-8');
      return JSON.stringify(data);
    }
    return data;
  }],

  transformResponse: [function transformResponse(data) {
    /*eslint no-param-reassign:0*/
    if (typeof data === 'string') {
      try {
        data = JSON.parse(data);
      } catch (e) { /* Ignore */ }
    }
    return data;
  }],

  /**
   * A timeout in milliseconds to abort a request. If set to 0 (default) a
   * timeout is not created.
   */
  timeout: 0,

  xsrfCookieName: 'XSRF-TOKEN',
  xsrfHeaderName: 'X-XSRF-TOKEN',

  maxContentLength: -1,

  validateStatus: function validateStatus(status) {
    return status >= 200 && status < 300;
  }
};

defaults.headers = {
  common: {
    'Accept': 'application/json, text/plain, */*'
  }
};

utils.forEach(['delete', 'get', 'head'], function forEachMethodNoData(method) {
  defaults.headers[method] = {};
});

utils.forEach(['post', 'put', 'patch'], function forEachMethodWithData(method) {
  defaults.headers[method] = utils.merge(DEFAULT_CONTENT_TYPE);
});

module.exports = defaults;
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

#### InterceptorManager 方法

> InterceptorManager 构造函数是用来实现拦截器的，这个构造函数上有3个方法：use、eject 和 forEach ；

通过 use 方法，知道注册的拦截器都会被保存到 InterceptorManager 对象的 handlers 属性中：

```javascript
// /lib/core/InterceptorManager.js
function InterceptorManager() {
  this.handlers = []; // 存放拦截器方法，数组内每一项都是有两个属性的对象，两个属性分别对应成功和失败后执行的函数。
}

// 往拦截器里添加拦截方法
InterceptorManager.prototype.use = function use(fulfilled, rejected) {
  this.handlers.push({
    fulfilled: fulfilled,
    rejected: rejected
  });
  return this.handlers.length - 1;
};

// 用来注销指定的拦截器
InterceptorManager.prototype.eject = function eject(id) {
  if (this.handlers[id]) {
    this.handlers[id] = null;
  }
};

// 遍历this.handlers，并将this.handlers里的每一项作为参数传给fn执行
InterceptorManager.prototype.forEach = function forEach(fn) {
  utils.forEach(this.handlers, function forEachHandler(h) {
    if (h !== null) {
      fn(h);
    }
  });
};
```

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/98a62da97d49487bb294cb55715e6ff9~tplv-k3u1fbpfcp-zoom-1.image)

### 任务编排

#### Axios.prototype.request 方法

> 各种  axios 的调用方式最终都是通过 request 方法发请求的；

```javascript
// lib/core/Axios.js
Axios.prototype.request = function request(config) {
  // Allow for axios('example/url'[, config]) a la fetch API
  if (typeof config === 'string') {
    config = arguments[1] || {};
    config.url = arguments[0];
  } else {
    config = config || {};
  }
    // 合并配置
  config = mergeConfig(this.defaults, config);
    // 请求方式，没有默认为 get
  config.method = config.method ? config.method.toLowerCase() : 'get';
    
    // 重点 这个就是拦截器的中间件
  var chain = [dispatchRequest, undefined];
    // 生成一个 promise 对象
  var promise = Promise.resolve(config);

    // 将请求前方法置入 chain 数组的前面 一次置入两个 成功的，失败的
  this.interceptors.request.forEach(function unshiftRequestInterceptors(interceptor) {
    chain.unshift(interceptor.fulfilled, interceptor.rejected);
  });
	// 将请求后的方法置入 chain 数组的后面 一次置入两个 成功的，失败的
  this.interceptors.response.forEach(function pushResponseInterceptors(interceptor) {
    chain.push(interceptor.fulfilled, interceptor.rejected);
  });

   // 通过 shift 方法把第一个元素从其中删除，并返回第一个元素。
  while (chain.length) {
    promise = promise.then(chain.shift(), chain.shift());
  }

  return promise;
};
```

1. 数组的 shift() 方法用于把数组的第一个元素从其中删除，并返回第一个元素的值；
2. 每次执行 while 循环，从 chain 数组里按序两项，分别作为 promise.then 方法的第一个和第二个参数；

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
  // 类似于
  // promise.then(请求拦截器的成功方法, 请求拦截器的失败方法)
  //  		.then(dispatchRequest, undefined)
  //		 .then(响应拦截器的成功方法, 响应拦截器的失败方法)
}
```

因为 chain 是数组，所以通过 while 语句可以不断地取出设置的任务，然后组装成 Promise 调用链从而实现任务调度，对应的处理流程如下图所示：

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/12cbfa5ce9aa4983b80a039eb5e5d83b~tplv-k3u1fbpfcp-zoom-1.image)

## 数据转换器

### 请求转换器

> transformRequest 是指在请求前对数据进行转换；

### 响应转换器

> transformResponse 是指在请求响应后的响应体进行数据转换；



## HTTP 适配器

> http请求适配器主要指两种：XHR、http；
>
>  XHR的核心是浏览器端的XMLHttpRequest对象， http核心是node的http/https.request方法；

为了支持不同的环境，Axios 引入了适配器，在拦截器的部分，有一个 dispatchRequest 方法用于发送 HTTP 请求：

```javascript
// lib/core/dispatchRequest.js
// 请求取消时候的方法
function throwIfCancellationRequested(config) {
  if (config.cancelToken) {
    config.cancelToken.throwIfRequested();
  }
}

module.exports = function dispatchRequest(config) {
  throwIfCancellationRequested(config);
  // 请求没有取消 执行下面的请求
  if (config.baseURL && !isAbsoluteURL(config.url)) {
    config.url = combineURLs(config.baseURL, config.url);
  }
  config.headers = config.headers || {};
  // 转换数据
  config.data = transformData(
    config.data,
    config.headers,
    config.transformRequest
  );
    // 合并配置
  config.headers = utils.merge(
    config.headers.common || {},
    config.headers[config.method] || {},
    config.headers || {}
  );
  utils.forEach(
    ['delete', 'get', 'head', 'post', 'put', 'patch', 'common'],
    function cleanHeaderConfig(method) {
      delete config.headers[method];
    }
  );
    // 这里是重点， 获取请求的方式，下面会将到
  var adapter = config.adapter || defaults.adapter;
  return adapter(config).then(function onAdapterResolution(response) {
    throwIfCancellationRequested(config);
	// 难道了请求的数据， 转换 data
    response.data = transformData(
      response.data,
      response.headers,
      config.transformResponse
    );
    return response;
  }, function onAdapterRejection(reason) {
      // 失败处理
    if (!isCancel(reason)) {
      throwIfCancellationRequested(config);

      // Transform response data
      if (reason && reason.response) {
        reason.response.data = transformData(
          reason.response.data,
          reason.response.headers,
          config.transformResponse
        );
      }
    }

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