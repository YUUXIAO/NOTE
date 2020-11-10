## 原理

> Tree-shaking 的作用是把项目中没必要的模块全部抖掉，用于在不同的模块之间消除无用的代码；

Tree-shaking 的消除原理是依赖 ES6 的模块特性；

- 只能作为模块顶层语句出现；
- import 的模块名只能是字符串常量；
- import binding 是 immutable的；

总结：

1. ES6的模块引入是静态分析的，故而可以在编译时正确判断到底加载了什么代码；
2. 分析程序流，判断哪些变量未被使用、引用，进而删除此代码；

## Webpack Tree Shaking

### 清除未使用的模块

> Webpack Tree Shaking 从 ES6 顶层模块开始分析，可以清除未使用的模块；

```javascript
//App.js
import { cube } from './utils.js';
cube(2);

//utils.js
export function square(x) {
  console.log('square');
  return x * x;
}

export function cube(x) {
  console.log('cube');
  return x * x * x;
}
```

结果：

```javascript
function(e, t, r) {
  "use strict";
  r.r(t), console.log("cube")
}
```

### 重构多层调用的模块

> Webpack Tree Shaking 会对多层调用的模块进行重构，提取其中的代码，简化函数的调用模块；

```javascript
//App.js
import { getEntry } from './utils'
console.log(getEntry());

//utils.js
import entry1 from './entry.js'
export function getEntry() {
  return entry1();
}

//entry.js
export default function entry1() {
  return 'entry1'
}
```

结果：

```javascript
//摘录核心代码
function(e, t, r) {
  "use strict";
  r.r(t), console.log("entry1")
}
```

### 不会清除 IIFE

> Webpack Tree Shaking 不会清除 IIFE ( 立即调用函数表达式 )；

```javascript
//App.js
import { cube } from './utils.js';
console.log(cube(2));

//utils.js
var square = function(x) {
  console.log('square');
}();

export function cube(x) {
  console.log('cube');
  return x * x * x;
}
```

结果：

```javascript
function(e, t, n) {
  "use strict";
  n.r(t);
  console.log("square");
  console.log(function(e) {
    return console.log("cube"), e * e * e
  }(2))
}
```

因为 IIFE 比较特殊，它在被翻译时就会执行， Webpack 不做程序流分析，它不知道 IIFE 会做什么特别的事情，所以不会删除这部分代码；

```javascript
var V8Engine = (function () {
  function V8Engine () {}
  V8Engine.prototype.toString = function () { return 'V8' }
  return V8Engine
}())

var V6Engine = (function () {
  function V6Engine () {}
  V6Engine.prototype = V8Engine.prototype // <---- side effect
  V6Engine.prototype.toString = function () { return 'V6' }
  return V6Engine
}())

console.log(new V8Engine().toString())
 
// 输出V6,而并不是V8
```

如果V6这个IIFE里面再搞一些全局变量的声明，那就当然不能删除了；

### 对于 IIFE 的返回函数，如果未使用会清除

> Webpack Tree Shaking 如果发现 IIFE 的返回函数没有地方调用的话，依旧是可以删除的；

```javascript
//App.js
import { cube } from './utils.js';
console.log(cube(2));

//utils.js
var square = function(x) {
  console.log('square');
  return x * x;
}();

function getSquare() {
  console.log('getSquare');
  square();
}

export function cube(x) {
  console.log('cube');
  return x * x * x;
}
```

结果：

```javascript
function(e, t, n) {
  "use strict";
  n.r(t);
  console.log("square");   <= square这个IIFE内部的代码还在
  console.log(function(e) {
    return console.log("cube"), e * e * e  <= square这个IIFEreturn的方法因为getSquare未被调用而被删除
  }(2))
}
```

### 结合第三方包使用

```javascript
//App.js
import { getLast } from './utils.js';
console.log(getLast('abcdefg'));

//utils.js
import _ from 'lodash';   <=这里的引用方式不同，会造成bundle的不同结果

export function getLast(string) {
  console.log('getLast');
  return _.last(string);
}
```

结果：

```javascript
import _ from 'lodash';
    Asset      Size 
bundle.js  70.5 KiB

import { last } from 'lodash';
    Asset      Size
bundle.js  70.5 KiB

import last from 'lodash/last';   <=这种引用方式明显降低了打包后的大小
    Asset      Size
bundle.js  1.14 KiB
```

## Babel带来的问题

### 语法转换(Babel6)

> Webpack 的 Tree Shaking 有能力除去导出但没有使用的代码块，但是结合 Babel 使用之后会出现问题；

```javascript
//App.js
import { Apple } from './components'

const appleModel = new Apple({   <==仅调用了Apple
  model: 'IphoneX'
}).getModel()

console.log(appleModel)

//components.js
export class Person {
  constructor ({ name, age, sex }) {
    this.className = 'Person'
    this.name = name
    this.age = age
    this.sex = sex
  }
  getName () {
    return this.name
  }
}

export class Apple {
  constructor ({ model }) {
    this.className = 'Apple'
    this.model = model
  }
  getModel () {
    return this.model
  }
}
```

结果：

```javascript
function(e, t, n) {
  "use strict";
  n.r(t);
  const r = new class {
    constructor({ model: e }) {
      this.className = "Apple", this.model = e
    }
    getModel() {
      return this.model
    }
  }({ model: "IphoneX" }).getModel();
  console.log(r)
}
//仅有Apple的类，没有Person的类(Tree shaking成功)
//class还是class，并没有经过语法转换(没有经过Babel的处理)
```

如果上面代码加上 Babel（babel-loader）的处理，结果如下：

```javascript
function(e, n, t) {
  "use strict";
  Object.defineProperty(n, "__esModule", { value: !0 });
  var r = function() {
    function e(e, n) {
      for(var t = 0; t < n.length; t++) {
        var r = n[t];
        r.enumerable = r.enumerable || !1, r.configurable = !0, "value" in r && (r.writable = !0), Object.defineProperty(e, r.key, r)
      }
    }
    return function(n, t, r) {
      return t && e(n.prototype, t), r && e(n, r), n
    }
  }();
  function o(e, n) {
    if(!(e instanceof n)) throw new TypeError("Cannot call a class as a function")
  }
  n.Person = function() {
    function e(n) {
      var t = n.name, r = n.age, u = n.sex;
      o(this, e), this.className = "Person", this.name = t, this.age = r, this.sex = u
    }
    return r(e, [{
      key: "getName", value: function() {
        return this.name
      }
    }]), e
  }(), n.Apple = function() {
    function e(n) {
      var t = n.model;
      o(this, e), this.className = "Apple", this.model = t
    }
    return r(e, [{
      key: "getModel", value: function() {
        return this.model
      }
    }]), e
  }()
}

//这次不仅Apple类在，Person类也存在(Tree shaking失败了)
//class已经被babel处理转换了
```

看看Babel到底干了什么：

```javascript
'use strict';
Object.defineProperty(exports, "__esModule", {
  value: true
});

//_createClass本质上也是一个IIFE
var _createClass = function() {
  function defineProperties(target, props) {
    for(var i = 0; i < props.length; i++) {
      var descriptor = props[i];
      descriptor.enumerable = descriptor.enumerable || false;
      descriptor.configurable = true;
      if("value" in descriptor) descriptor.writable = true;
      Object.defineProperty(target, descriptor.key, descriptor);
    }
  }
  return function(Constructor, protoProps, staticProps) {
    if(protoProps) defineProperties(Constructor.prototype, protoProps);
    if(staticProps) defineProperties(Constructor, staticProps);
    return Constructor;
  };
}();

function _classCallCheck(instance, Constructor) {
  if(!(instance instanceof Constructor)) {
    throw new TypeError("Cannot call a class as a function");
  }
}

//Person本质上也是一个IIFE
var Person = exports.Person = function () {
  function Person(_ref) {
    var name = _ref.name,
        age = _ref.age,
        sex = _ref.sex;
    _classCallCheck(this, Person);
    this.className = 'Person';
    this.name = name;
    this.age = age;
    this.sex = sex;
  }
  _createClass(Person, [{    <==这里调用了另一个IIFE
    key: 'getName',
    value: function getName() {
      return this.name;
    }
  }]);
  return Person;
}();
```

因为 Webpack Tree shaking是不处理IIFE的，所以这里即使没有调用Person类在bundle中也存在了Person类的代码；

我们可以设定使用 loose: true 来使得Babel在转化时使用宽松的模式，但是这样也仅仅只能去除 _createClass，Person本身依旧存在；

```javascript
//webpack.config.js
{
  loader: 'babel-loader',
  options: {
    presets: [["env", { loose: true }]]
  }
}
```

结果：

```javascript
function(e, t, n) {
  "use strict";
  function r(e, t) {
    if(!(e instanceof t)) throw new TypeError("Cannot call a class as a function")
  }
  t.__esModule = !0;
  t.Person = function() {
    function e(t) {
      var n = t.name, o = t.age, u = t.sex;
      r(this, e), this.className = "Person", this.name = n, this.age = o, this.sex = u
    }
    return e.prototype.getName = function() {
      return this.name
    }, e
  }(), t.Apple = function() {
    function e(t) {
      var n = t.model;
      r(this, e), this.className = "Apple", this.model = n
    }
    return e.prototype.getModel = function() {
      return this.model
    }, e
  }()
}
```

### 模块转换(Babel6/7)

Babel在处理时默认将所有的模块转换成为了 exports 结合 require 的形式，而Webpack是基于ES6的模块才能做到最大程度的Tree shaking的，所以我们在使用Babel时，应该将Babel的这一行为关闭，方式如下：

```javascript
//babel.rc
presets: [["env", 
  { module: false }
]]
```

但是如果我们都在一个App中，这个module的关闭是没有意义的，因为如果关闭了，那么打包出来的bundle是没有办法在浏览器里面运行的(不支持import)；

所以这里我们应该在App依赖的某个功能库打包时去设置：比如：像lodash/lodash-es , redux，react-redux，styled-component这类库都同时存在ES5和ES6的版本；

```javascript
- redux
  - dist
  - es
  - lib
  - src
  ...
```

同时在packages.json中设置入口配置，就可以让Webpack优先读取ES6的文件：

```javascript
//package.json
"main": "lib/redux.js",
"unpkg": "dist/redux.js",
"module": "es/redux.js",
"typings": "./index.d.ts",
```

## Side Effect

Webpack 4.x 新增了一个 sideEffects 特性，通过给 package.json 加入 sideEffects：false 声明该模块是否包含 sideEffects（副作用），从而可以为 Tree Shaking 提供更大的优化空间；

> 如果我们引入的包/模块被标记为 sideEffects：false，那么不管它是否真的有副作用，只要它没有被引用到，整个包/模块都会被完整的移除；

## 总结

如果想利用好Webpack的Tree shaking， 建议：

1. 对第三方的库：
   - 团队维护的：视情况加上 sideEffects 标记，同时更改 Babel 配置来导出 ES6模块；
   - 第三方的：尽量使用提供 ES 模块的版本；
2. 工具：
   - 升级 webpack 到 4.x；
   - 升级 Babel 到 7.x；