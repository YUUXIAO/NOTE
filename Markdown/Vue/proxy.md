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

### 优势

1. 可直接监听数组类型的数据变化；
2. 监听的目标是对象本身，不需要像 Object.defineProperty 一样遍历每个属性，有一定的性能提升；
3. 可拦截 apply、ownkeys、has 等13种方法，Object.defineProperty 不行；
4. 可以直接实现对象属性的新增/删除；





