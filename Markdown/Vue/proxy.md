> Proxy 对象用于创建一个对象的代理，从而实现基本操作的拦截和自定义（如属性查找、赋值、枚举、函数调用等）；

1. Proxy 会创建一个新的对象对原对象做代理，外界对原对象的访问，都必须先通过这层代理进行拦截；
2. 拦截的结果是我们只要通过操作新的实例对象就能间接的操作真正的目标对象了；

## 基本用法

> const p = new Proxy（target，handler）;

- target：要使用 Proxy 包装的目标对象，可以是任何类型的对象，包括原生数组，函数，甚至另一个代理；
- handler：一个通常以函数作为属性的对象，各属性中的函数分别定义了在执行各种操作时代理 p 的行为；

```javascript
const man = {
  name: "阿三",
};

const proxy = new Proxy(man, {
  get(target, property, receiver) {
    console.log(`正在访问${property}属性`);
    return target[property];
  },
});

console.log(proxy.name); // 正在访问name属性，阿三
console.log(proxy.age);	// 正在访问age属性，undefined
```

### handler 对象支持的捕获器

> 所有的捕获器都是可选的，如果没有定义某个捕获器，那就会保留源对象的默认行为；

列举常用的捕获器：

1. handler.get（）：属性读取操作的捕获器；
2. handler.set（）：属性设置操作的捕获器；
3. handler.deleteProperty（）：delete 操作符的捕获器；
4. handler.ownKeys（）Object.getOwnPropertyNames 方法和 Object.getOwnPropertySymbols 方法的捕获器；
5. handler.has（）：in 操作符的捕获器；

## Reflect 

> Reflect 是一个内置对象，它提供拦截 Javascript 操作的方法，这些方法与 proxy handlers 的方法；

Reflect 不是一个函数对象，因此它是不可构造的；

### 基本用法

```javascript
const man = {
  name: "阿三",
  city: "Beijing",
};

console.log(Reflect.set(man, "sex", 1)); // true
console.log(Reflect.has(man, "name")); // true
console.log(Reflect.has(man, "age")); // false
console.log(Reflect.ownKeys(man)); // [ 'name', 'city', 'sex' ]
```

### Reflect 对象支持的静态方法

> Reflect 的所有属性和方法都是静态的，该对象提供了与 Proxy handler 对象相关的 13 个方法；

1. Reflect.get(target, propertyKey[, receiver])：获取对象身上某个属性的值，类似于 target[name]；
2. Reflect.set(target, propertyKey, value[, receiver])：将值赋值给属性的函数，返回一个布尔值，如果更新成功，则返回 true；
3. Reflect.deleteProperty(target, propertyKey)：删除 target 对象的指定属性，相当于执行 delete target[name]；
4. Reflect.has(target, propertyKey)：判断一个对象是否存在某个属性，和 in 运算符的功能完全相同；
5. Reflect.ownKeys(target)：返回一个包含所有自身属性（不包含继承属性）的数组；

## 使用场景

### 增强型数组

> 增强后的数组对象，就可以支持负数索引、分片索引等功能；

```javascript
function enhancedArray(arr) {
  return new Proxy(arr, {
    get(target, property, receiver) {
      const range = getRange(property);
      const indices = range ? range : getIndices(property);
      const values = indices.map(function (index) {
        const key = index < 0 ? String(target.length + index) : index;
        return Reflect.get(target, key, receiver);
      });
      return values.length === 1 ? values[0] : values;
    },
  });

  function getRange(str) {
    var [start, end] = str.split(":").map(Number);
    if (typeof end === "undefined") return false;

    let range = [];
    for (let i = start; i < end; i++) {
      range = range.concat(i);
    }
    return range;
  }

  function getIndices(str) {
    return str.split(",").map(Number);
  }
}

const arr = enhancedArray([1, 2, 3, 4, 5]);

console.log(arr[-1]); // 5
console.log(arr[[2, 4]]); // [ 3, 5 ]
console.log(arr[[2, -2, 1]]); // [ 3, 4, 2 ]
console.log(arr["2:4"]); // [ 3, 4]
console.log(arr["-2:3"]); // [ 4, 5, 1, 2, 3 ]
```

###  增强型对象

> 增强后的对象，可以方便地访问普通对象内部的深层属性；

```javascript
const enhancedObject = (target) =>
  new Proxy(target, {
    get(target, property) {
      if (property in target) {
        return target[property];
      } else {
        return searchFor(property, target);
      }
    },
  });

let value = null;
function searchFor(property, target) {
  for (const key of Object.keys(target)) {
    if (typeof target[key] === "object") {
      searchFor(property, target[key]);
    } else if (typeof target[property] !== "undefined") {
      value = target[property];
      break;
    }
  }
  return value;
}

const data = enhancedObject({
  user: {
    name: "阿三",
    settings: {
      theme: "dark",
    },
  },
});

console.log(data.user.settings.theme); // dark
console.log(data.theme); // dark
```

### 创建只读对象

```javascript
function freeze (obj) {
  return new Proxy(obj, {
    set () { return true; },
    deleteProperty () { return false; },
    defineProperty () { return true; },
    setPrototypeOf () { return true; }
  });
}

let frozen = freeze([1, 2, 3]);
frozen[0] = 6;
delete frozen[0];
frozen = Object.defineProperty(frozen, 0, { value: 66 });
console.log(frozen); // [ 1, 2, 3 ]
```

### 拦截方法调用

```javascript
function traceMethodCalls(obj) {
  const handler = {
    get(target, propKey, receiver) {
      const origMethod = target[propKey]; // 获取原始方法
      return function (...args) {
        const result = origMethod.apply(this, args);
        console.log(
          propKey + JSON.stringify(args) + " -> " + JSON.stringify(result)
        );
        return result;
      };
    },
  };
  return new Proxy(obj, handler);
}


const obj = {
  multiply(x, y) {
    return x * y;
  },
};

const tracedObj = traceMethodCalls(obj);
tracedObj.multiply(2, 5); // multiply[2,5] -> 10
```

### 拦截属性调用

```javascript
function tracePropAccess(obj, propKeys) {
  const propKeySet = new Set(propKeys);
  return new Proxy(obj, {
    get(target, propKey, receiver) {
      if (propKeySet.has(propKey)) {
        console.log("GET " + propKey);
      }
      return Reflect.get(target, propKey, receiver);
    },
    set(target, propKey, value, receiver) {
      if (propKeySet.has(propKey)) {
        console.log("SET " + propKey + "=" + value);
      }
      return Reflect.set(target, propKey, value, receiver);
    },
  });
}

const man = {
  name: "semlinker",
};
const tracedMan = tracePropAccess(man, ["name"]);

console.log(tracedMan.name); // GET name; semlinker
console.log(tracedMan.age); // undefined
tracedMan.name = "kakuqo"; // SET name=kakuqo
```

### 隐藏属性

```javascript
const hideProperty = (target, prefix = "_") =>
  new Proxy(target, {
    has: (obj, prop) => !prop.startsWith(prefix) && prop in obj,
    ownKeys: (obj) =>
      Reflect.ownKeys(obj).filter(
        (prop) => typeof prop !== "string" || !prop.startsWith(prefix)
      ),
    get: (obj, prop, rec) => (prop in rec ? obj[prop] : undefined),
  });

const man = hideProperty({
  name: "阿三",
  _pwd: "www.baidu.com",
});

console.log(man._pwd); // undefined
console.log('_pwd' in man); // false
console.log(Object.keys(man)); // [ 'name' ]
```

### 验证属性值

```javascript
const validatedUser = (target) =>
  new Proxy(target, {
    set(target, property, value) {
      switch (property) {
        case "email":
          const regex = /^(([^<>()\[\]\\.,;:\s@"]+(\.[^<>()\[\]\\.,;:\s@"]+)*)|(".+"))@((\[[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\])|(([a-zA-Z\-0-9]+\.)+[a-zA-Z]{2,}))$/;
          if (!regex.test(value)) {
            console.error("The user must have a valid email");
            return false;
          }
          break;
        case "age":
          if (value < 20 || value > 80) {
            console.error("A user's age must be between 20 and 80");
            return false;
          }
          break;
      }

      return Reflect.set(...arguments);
    },
  });

let user = {
  email: "",
  age: 0,
};

user = validatedUser(user);
user.email = "阿三"; // The user must have a valid email
user.age = 100; // A user's age must be between 20 and 80
```

## 优势

1. 可直接监听数组类型的数据变化；
2. 监听的目标是对象本身，不需要像 Object.defineProperty 一样遍历每个属性，有一定的性能提升；
3. 可拦截 apply、ownkeys、has 等13种方法，Object.defineProperty 不行；
4. 可以直接实现对象属性的新增/删除；


## 相关问题

### this指向

```javascript
const target = {
  foo() {
    return {
      thisIsTarget: this === target,
      thisIsProxy: this === proxy,
    };
  },
};

const handler = {};
const proxy = new Proxy(target, handler);

console.log(target.foo()); 
// { thisIsTarget: true, thisIsProxy: false }

console.log(proxy.foo());
// { thisIsTarget: false, thisIsProxy: true }
```

### get捕获器receiver参数

get 捕获器用于拦截对象的读取属性操作，该捕获器含有三个参数：

- target：目标对象；
- property：被读取的属性名；
- receiver：指向当前的 Proxy 对象或者继承于当前 Proxy 的对象；

```javascript
const proxy = new Proxy({},
  {
    get: function (target, property, receiver) {
      return receiver;
    },
  }
);

console.log(proxy.getReceiver === proxy); // true
var inherits = Object.create(proxy);
console.log(inherits.getReceiver === inherits); // true
```

可以通过 Reflect 对象提供的 get 方法动态设置 receiver 对象的值；

```javascript
console.log(Reflect.get(proxy, "getReceiver", "阿三"));
```

### 包装内置构造函数的实例

当使用 Proxy 包装内置构造函数实例的时候，可能会出现一些问题；

```javascript
const target = new Date();
const handler = {};
const proxy = new Proxy(target, handler);

proxy.getDate(); // TypeError: this is not a Date object.
```

因为有些原生对象的内部属性，只有通过正确的 this 才能拿到，所以 Proxy 无法代理这些原生对象的属性；

可以为 getDate 方法绑定正确的 this；

```javascript
const target = new Date();
const handler = {
  get(target, property, receiver) {
    if (property === "getDate") {
      return target.getDate.bind(target);
    }
    return Reflect.get(target, property, receiver);
  },
};

const proxy = new Proxy(target, handler);
console.log(proxy.getDate());
```

### 创建可撤销的代理对象

> 通过 Proxy.revocable() 方法可以用来创建一个可撤销的代理对象，返回一个对象，结构是 {"proxy": proxy, "revoke": revoke}；

当 Proxy 对象被撤销之后，就不可以对已撤销的 proxy 对象执行任何操作；

```javascript
Proxy.revocable(target, handler);
```

- target：将用 Proxy 封装的目标对象；
- handler：一个对象，其属性是一批可选的函数，这些函数定义了对应的操作被执行时代理的行为；

```javascript
const target = {}; 
const handler = {};
const { proxy, revoke } = Proxy.revocable(target, handler);

proxy.name = "阿三";
console.log(proxy.name); // 阿三

revoke();
console.log(proxy.name); 
// Uncaught TypeError: Cannot perform 'get' on a proxy that has been revoked at <anonymous>
```

