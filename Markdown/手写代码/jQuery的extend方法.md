## 语法

> 合并两个或者更多的对象的内容到第一个对象中；

语法：

1. 第一个参数 target表示要拓展的目标，就称为目标对象；
2. 后面的参数，都传入对象，内容都会复制到目标对象中，就称为待复制对象；
3. 当两个对象出现相同字段的时候，后者会覆盖前者，而不会进行深层次的覆盖；

```javascript
jQuery.extend( target [, object1 ] [, objectN ] )
```

举例：

```javascript
var obj1 = {
    a: 1,
    b: { b1: 1, b2: 2 }
};

var obj2 = {
    b: { b1: 3, b3: 4 },
    c: 3
};

var obj3 = {
    d: 4
}

console.log($.extend(obj1, obj2, obj3));

// 浅拷贝方式
// {
//    a: 1,
//    b: { b1: 3, b3: 4 },
//    c: 3,
//    d: 4
// }

// 深拷贝方式 
// {
//    a: 1,
//    b: { b1: 3, b2: 2, b3: 4 },
//    c: 3,
//    d: 4
// }
```

## 简易版本

```javascript
function extend() {
    const len = arguments.length;
    let target = arguments[0];

    for (let i = 1; i < len; i++) {
      let option = arguments[i];

      if (option !== null) {
        for (let name in option) {
          let copy = option[name];
          if (copy !== undefined) {
            target[name] = copy;
          }
        }
      }
    }

    return target;
  }
```

## 深拷贝

> 函数的第一个参数可以传一个布尔值，如果为 true，我们就会进行深拷贝，false 依然当做浅拷贝；
>
> 语法：jQuery.extend( [deep], target, object1 [, objectN ] )

1. 需要根据第一个参数的类型，确定 target 和要合并的对象的下标起始值；
2. 如果是深拷贝，根据 copy 的类型递归 extend；

```javascript
 function extend() {
   	// 默认不进行深拷贝
    let deep = false;
    let len = arguments.length;
   	// 记录要复制的对象的下标
    let i = 1;
   	// 第一个参数不传布尔值的情况下，target默认是第一个参数
    let target = arguments[0] || {};
    // 如果第一个参数是布尔值，第二个参数是才是target
    if (typeof arguments[0] == "boolean") {
      deep = target;
      target = arguments[i] || {};
      i = 2;
    }
    // 如果target不是对象，我们是无法进行复制的，所以设为{}
    if (typeof target !== "object" && target !== null) {
      target = {};
    }

    for (; i < len; i++) {
      // 获取当前要复制的对象
      let option = arguments[i];
      // 要求不能为空 避免extend(a,,b)这种情况
      if (option !== null) {
        for (let name in option) {
          // 目标对象属性值
          let src = target[name];
          // 要复制的对象的属性值
          let copy = option[name];

          if (deep && copy && typeof copy === "object") {
            target[name] = extend(deep, src, copy);
          } else if (copy !== undefined) {
            target[name] = copy;
          }
        }
      }
    }

    return target;
  }
```

