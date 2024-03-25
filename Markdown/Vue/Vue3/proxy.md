> Proxy 对象用于创建一个对象的代理，从而实现基本操作的拦截和自定义（如属性查找、赋值、枚举、函数调用等）；

1. Proxy 会创建一个新的对象对原对象做代理，外界对原对象的访问，都必须先通过这层代理进行拦截；
2. 拦截的结果是我们只要通过操作新的实例对象就能间接的操作真正的目标对象了；

## 基本用法

> const p = new Proxy（target，handler）;

- target：要使用 Proxy 包装的目标对象，可以是任何类型的对象，包括原生数组，函数，甚至另一个代理；
- handler：一个以函数作为属性的对象，各属性中的函数分别定义了在执行各种操作时代理 p 的行为；

```javascript
const man = {
  name: "yabby",
};

const proxy = new Proxy(man, {
  get(target, property, receiver) {
    console.log(`正在访问${property}属性`);
    return target[property];
  },
  set(){}, // 设置操作
  deleteProperty(){}, // delete操作
  ownKeys(){}, // Object.getOwnPropertyNames 方法和 Object.getOwnPropertySymbols 方法；
  has(){}, // in 操作
});

console.log(proxy.name); // 正在访问name属性，阿三
console.log(proxy.age);	// 正在访问age属性，undefined
```

### Reflect

Reflect 是一个内置对象，它**提供拦截JS操作的方法**，这些方法与 proxy handlers 的方法相同；

```javascript
const man = {
  name: "yabby",
  city: "shenzhen",
};

console.log(Reflect.set(man, "sex", 1)); // true
console.log(Reflect.has(man, "name")); // true
console.log(Reflect.has(man, "age")); // false
console.log(Reflect.ownKeys(man)); // [ 'name', 'city', 'sex' ]
```

Reflect 的所有属性和方法都是**静态的**，该对象提供了与 Proxy handler 对象相关的 13 个方法；

1. Reflect.get(target, propertyKey[, receiver])：获取对象身上某个属性的值，类似于 target[name]；
2. Reflect.set(target, propertyKey, value[, receiver])：将值赋值给属性的函数，返回一个布尔值，如果更新成功，则返回 true；
3. Reflect.deleteProperty(target, propertyKey)：删除 target 对象的指定属性，相当于执行 delete target[name]；
4. Reflect.has(target, propertyKey)：判断一个对象是否存在某个属性，和 in 运算符的功能完全相同；
5. Reflect.ownKeys(target)：返回一个包含所有自身属性（不包含继承属性）的数组；

## 原理

Proxy 响应式的原理其实就是在第二个参数 handler 中，拦截各种取值、赋值操作，依托 track 和 trigger 两个函数进行依赖收集和派发更新；

### track

track 用来在**读取对象属性时收集依赖**；

1. 全局会存一个 targetMap，用来建立数据-->依赖的映射，是一个 weakMap 数据结构；
2. targetMap 通过 target 可以获取到 depsMap，它用来存放这个数据对应的所有响应式依赖；
3. depsMap 的**键是响应式对象的key，值时deps(Set 数据结构)**，存放着对应 key 的更新函数(vue里面也叫**副作用函数**）；

```javascript
// target：原始对象
// type：是本次收集的类型，也就是收集依赖时用来标志是什么类型的操作，比如 get
// key：是指本次访问的是数据中的哪个 key
function track(target: object, type: TrackOpTypes, key: unknown) {
  const depsMap = targetMap.get(target);
  let dep = new Set() // 收集依赖时 通过 key 建立一个 set
  targetMap.set(ITERATE_KEY, dep)
  // effect可以理解为更新函数存放在dep里，后序如果target修改了对应的属性是会一次执行dep的函数
  dep.add(effect) 
}
```

### trigger

trigger 用来在**更新对象时触发依赖**；

```javascript
export function trigger(
  target: object,
  type: TriggerOpTypes,
  key?: unknown,
) {
  // 简化来说就是通过 key 找到所有更新函数 依次执行
  const dep = targetMap.get(target)
  dep.get(key).forEach(effect => effect())
}
```

## 使用场景

### 增强型数组/增强型对象

比如通过代理数组或对象的get方法，拓展原本的功能：数组支持负索引、对象可以跨级访问到所有子对象的值的等，这个看个人需求，就不写例子了

### 创建只读对象

就是只默认对象的get方法，其他操作都return

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

const person = hideProperty({
  name: "yabby",
  _pwd: "xxx",
});

console.log(person._pwd); // undefined
console.log('_pwd' in person); // false
console.log(Object.keys(person)); // [ 'name' ]
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
user.email = "yabby"; // The user must have a valid email
user.age = 100; // A user's age must be between 20 and 80
```

## 对比Object.defineProperty

1. **使用对象的不同：**

   - Object.defineProperty 方法需要**根据具体的 key** 对数据进行拦截；
   - Proxy 监听的目标是对象本身，不需要像defineProperty 一样遍历每个属性，有一定的性能提升；

2. Proxy 可直接监听数组类型的数据变化，defineProperty 对于数组元素的修改不敏感
3. Proxy 的handles有 apply、ownkeys、has 等13种方法，可以拦截更多的操作符，defineProperty 不行；
4. Proxy 可以直接实现对象属性的新增/删除；
5. Vue3 对于响应式数据，不再像 Vue2 中那样递归对所有的子数据进行响应式定义了，初始化只会代理第一层对象属性，在后面取子属性的时候再去利用 reactive 进一步定义响应式，这对于大量数据的初始化场景来说说性能好不少；

```javascript
function get(target: object, key: string | symbol, receiver: object) {
  const res = Reflect.get(target, key, receiver)
  // 惰性定义
  return isObject(res)
    ? reactive(res)
    : res
}
```

## 其他

### this指向

通过代理对象操作数据的时候，this是Proxy

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

可以通过 Reflect 对象提供的 get 方法动态设置 receiver 对象的值，这种一般出现在proto原型上的对象读写，所以可以用Reflect.get(target, attr, origin)来设置

```javascript
console.log(Reflect.get(proxy, "getReceiver", "yabby"));
```

### 包装内置构造函数的实例

当使用 Proxy 包装内置构造函数实例的时候，可能会出现一些问题；

因为有些原生对象的内部属性，只有通过正确的 this 才能拿到，所以 Proxy 无法代理这些原生对象的属性；

```javascript
const target = new Date();
const handler = {};
const proxy = new Proxy(target, handler);

proxy.getDate(); // TypeError: this is not a Date object.
```

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

通过 Proxy.revocable() 方法可以用来创建一个可撤销的代理对象，返回一个对象，结构是 {"proxy": proxy, "revoke": revoke}；

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

proxy.name = "yabby";
console.log(proxy.name); // yabby

revoke();
console.log(proxy.name); 
// Uncaught TypeError: Cannot perform 'get' on a proxy that has been revoked at <anonymous>
```