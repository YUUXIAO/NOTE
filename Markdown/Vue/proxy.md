## Object.defineProperty

> Object.defineProperty（） 方法会直接在一个对象上定义一个新属性，或者修改一个对象的现有属性，并返回这个对象；

### 基本用法

```javascript
Object.defineProperty(obj, prop, descriptor)
```

descriptor 的属性描述符有两种形式，一种是数据描述符，另一种是存取描述符；

- 数据描述符
  1. configurable：数据是否可删除，可配置；
  2. enumerable：属性是否可枚举；
  3. value：属性值，默认为 undefined；
  4. writable：属性是否可读写；
- 存取描述符：
  1. configurable：数据是否可删除，可配置；
  2. enumerable：属性是否可枚举；
  3. get：一个给属性提供 getter 的方法，如果没有 getter 则为 undefined；
  4. set：一个给属性提供 setter 的方法，如果没有 setter 则为 undefined；
- 数据描述符的 value，writable 和 存取描述符的 get，set 属性不能同时存在，否则会抛出异常；

### 数组类型

Object.defineProperty 的 get 和 set 方法是对对象进行监测并响应变化；

- 已知长度的数组是可以通过索引属性来设置属性的访问器属性的；

- 数组的添加无法进行拦截，因为数组所添加的索引值并没有预先加入数据拦截中；

## Proxy

> Proxy 针对目标对象会创建一个新的实例对象，并将目标对象代理到新的实例对象上；

- Proxy 会创建一个新的对象对原对象做代理，外界对原对象的访问，都必须先通过这层代理进行拦截；
- 拦截的结果是我们只要通过操作新的实例对象就能间接的操作真正的目标对象了；

### 基本用法

> const p = new Proxy（target，handler）;

```javascript
var obj = {}
var nobj = new Proxy(obj, {
    get(target, key, receiver) {
        console.log('获取值')
        return Reflect.get(target, key, receiver)
    },
    set(target, key, value, receiver) {
        console.log('设置值')
        return Reflect.set(target, key, value, receiver)
    }
})

nobj.a = '代理'
console.log(obj)
// 结果
// 设置值
// {a: "代理"}
```

### 数组类型

Proxy 完美的解决了数组的监听检测问题

```javascript
var arr = [1, 2, 3]
let obj = new Proxy(arr, {
    get: function (target, key, receiver) {
        // console.log("获取数组元素" + key);
        return Reflect.get(target, key, receiver);
    },
    set: function (target, key, receiver) {
        console.log('设置数组');
        return Reflect.set(target, key, receiver);
    }
})
// 1. 改变已存在索引的数据
obj[2] = 3
// result: 设置数组

// 2. push,unshift添加数据
obj.push(4)
// result: 设置数组(索引和length属性都会触发setter)

// 3. 直接通过索引添加数组
obj[5] = 5
// result: 设置数组

// 4. 删除数组元素
obj.splice(1, 1)
```

## initProxy

> initProxy 在开发环境下执行，作用是为了开发者在使用了一些不存在、内部不可访问的或者关键字的时候给出警告；

```javascript
Vue.prototype._init = function(options) {
    // 选项合并
    ...
    {
        // 对vm实例进行一层代理
        initProxy(vm);
    }
    ...
}
```

```javascript
// 代理函数
var initProxy = function initProxy (vm) {
    // 判断浏览器是否支持原生的 proxy
    if (hasProxy) {
        var options = vm.$options;
        var handlers = options.render && options.render._withStripped
            ? getHandler
            : hasHandler;
        // 代理vm实例到vm属性_renderProxy
        vm._renderProxy = new Proxy(vm, handlers);
    } else {
        vm._renderProxy = vm;
    }
};
```

- 当浏览器支持 Proxy 时，vm._renderProxy 会代理 vm 实例，并且代理过程也会随着参数的不同呈现不同的效果；
- 当浏览器不支持Proxy 时，直接将 vm 赋值给 vm._renderProxy；

> getHandler 方法主要是针对读取代理对象某个属性时进行的操作，当访问的属性不是 String 类型或者属性值在被代理的对象上不存在时，抛出错误提示，否则就返回属性值；

```javascript
const getHandler = {
  get (target, key) {
    if (typeof key === 'string' && !(key in target)) {
      warnNonPresent(target, key)
    }
    return target[key]
  }
}
```

### 数据过滤

我们可以通过 data 选项去设置实例数据，但是这些数据的命名是不可以随意（例如 js 的关键字）；

Vue 源码内部使用了以 $,_ 作为开头的内部变量，所以以这些开头的变量名也是不被允许的；

这就构成了数据过滤监测的前提；

> hasHandler 方法可以在开发者错误的调用vm属性时，提供提示作用

```javascript
var hasHandler = {
    has: function has (target, key) {
        var has = key in target;
        // isAllowed用来判断模板上出现的变量是否合法。
        var isAllowed = allowedGlobals(key) ||
            (typeof key === 'string' && key.charAt(0) === '_' && !(key in target.$data));
        // _和$开头的变量不允许出现在定义的数据中，因为他是vue内部保留属性的开头
        // warnReservedPrefix: 警告不能以$ _开头的变量
        // warnNonPresent: 警告模板出现的变量在vue实例中未定义
        if (!has && !isAllowed) {
            if (key in target.$data) { warnReservedPrefix(target, key); }
            else { warnNonPresent(target, key); }
        }
        return has || !isAllowed
    }
};

// 模板中允许出现的非vue实例定义的变量
var allowedGlobals = makeMap(
    'Infinity,undefined,NaN,isFinite,isNaN,' +
    'parseFloat,parseInt,decodeURI,decodeURIComponent,encodeURI,encodeURIComponent,' +
    'Math,Number,Date,Array,Object,Boolean,String,RegExp,Map,Set,JSON,Intl,' +
    'require' // for Webpack/Browserify
);

// 模板使用未定义的变量
var warnNonPresent = function (target, key) {
    warn(
    "Property or method \"" + key + "\" is not defined on the instance but " +
    'referenced during render. Make sure that this property is reactive, ' +
    'either in the data option, or for class-based components, by ' +
    'initializing the property. ' +
    'See: https://vuejs.org/v2/guide/reactivity.html#Declaring-Reactive-Properties.',
    target
    );
};

// 使用$,_开头的变量
var warnReservedPrefix = function (target, key) {
    warn(
    "Property \"" + key + "\" must be accessed with \"$data." + key + "\" because " +
    'properties starting with "$" or "_" are not proxied in the Vue instance to ' +
    'prevent conflicts with Vue internals' +
    'See: https://vuejs.org/v2/api/#data',
    target
    );
};
```

- allowedGlobals 定义了 javascript 保留的关键字，这些关键字是不允许作为用户变量存在的；
- 对以 $ _开头的变量做过滤；
- 对没有在 data 中定义的变量做过滤；

#### 不支持proxy的情况

只有在浏览器支持 Proxy 的情况下才会做 initProxy 设置代理，在不支持的情况下，数据过滤就失效了，此时非法的数据定义也不能正常运行；

```
// 模板使用未定义的变量
var warnNonPresent = function (target, key) {
    warn(
    "Property or method \"" + key + "\" is not defined on the instance but " +
    'referenced during render. Make sure that this property is reactive, ' +
    'either in the data option, or for class-based components, by ' +
    'initializing the property. ' +
    'See: https://vuejs.org/v2/guide/reactivity.html#Declaring-Reactive-Properties.',
    target
    );
};

// 使用$,_开头的变量
var warnReservedPrefix = function (target, key) {
    warn(
    "Property \"" + key + "\" must be accessed with \"$data." + key + "\" because " +
    'properties starting with "$" or "_" are not proxied in the Vue instance to ' +
    'prevent conflicts with Vue internals' +
    'See: https://vuejs.org/v2/api/#data',
    target
    );
};
```

1. 支持 proxy 浏览器的结果

![img](https://user-gold-cdn.xitu.io/2019/11/4/16e34e5762eaa7ab?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

2. 不支持 proxy 浏览器的结果

![img](https://user-gold-cdn.xitu.io/2019/11/4/16e34e668c5ecf7a?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

- 在没有经过代理的情况下，使用 $,_ 开头的变量变成了 js 语言层面的错误，表示该变量没有被声明；
- 在初始化数据阶段，Vue 已经为数据进行了一层筛选的代理；

```javascript
function initData(vm) {
  vm._data = typeof data === 'function' ? getData(data, vm) : data || {}
  if (!isReserved(key)) {
      // 数据代理，用户可直接通过vm实例返回data数据
      proxy(vm, "_data", key);
  }
}

function isReserved (str) {
  var c = (str + '').charCodeAt(0);
  // 首字符是$, _的字符串
  return c === 0x24 || c === 0x5F
}
```

- isReserved 会过滤以 $,_ 开头的变量，proxy 会为实例数据的访问做代理；
- 当访问 this.message 时，实际上访问的是 this._data.message；

proxy 方法的实现是基于 Object.defineProperty 来实现的；

```javascript
function proxy (target, sourceKey, key) {
    sharedPropertyDefinition.get = function proxyGetter () {
        // 当访问this[key]时，会代理访问this._data[key]的值
        return this[sourceKey][key]
    };
    sharedPropertyDefinition.set = function proxySetter (val) {
        this[sourceKey][key] = val;
    };
    Object.defineProperty(target, key, sharedPropertyDefinition);
}
```

