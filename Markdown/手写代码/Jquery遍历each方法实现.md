> 实现一个通用遍历方法，可用于遍历对象和数组；

## 语法

语法为：

```
jQuery.each(object, [callback])；
```

回调函数拥有两个参数：第一个为对象的成员或数组的索引，第二个为对应变量或内容；

```javascript
// 遍历数组
$.each( [0,1,2], function(i, n){
    console.log( "Item #" + i + ": " + n );
});

// Item #0: 0
// Item #1: 1

// 遍历对象
$.each({ name: "John", lang: "JS" }, function(i, n) {
    console.log("Name: " + i + ", Value: " + n);
});

// Name: name, Value: John
// Name: lang, Value: JS
```

## 退出循环

> 尽管 ES5 提供了 forEach 方法，但是 forEach 没有办法中止或者跳出 forEach 循环，除了抛出一个异常。但是对于 jQuery 的 each 函数，如果需要退出 each 循环可使回调函数返回 false，其它返回值将被忽略；

```javascript
$.each( [0, 1, 2, 3, 4, 5], function(i, n){
    if (i > 2) return false;
    console.log( "Item #" + i + ": " + n );
});

// Item #0: 0
// Item #1: 1
// Item #2: 2
```

## 简易版本

1. 根据参数的类型进行判断；
2. 如果是数组和类数组，就调用 for 循环；
3. 如果是对象，就使用 for in 循环；

```javascript
function each(obj, callback) {
    if ( isArrayLike(obj) ) {
        for (let i=0; i < obj.length; i++ ) {
            callback(i, obj[i])
        }
    } else {
        for ( let item in obj ) {
            callback(item, obj[item])
        }
    }
    return obj;
}
```

## 中止循环

> 当回调函数返回 false 的时候，我们就中止循环；

```javascript
function each(obj, callback) {
    if ( isArrayLike(obj) ) {
        for (let i=0; i < obj.length; i++ ) {
            if (callback(i, obj[i]) === false) {
              break;
          	}
        }
    } else {
        for ( let item in obj ) {
            if (callback(item, obj[item]) === false) {
              break;
          	}
        }
    }
    return obj;
}
```

## this 指向

在 callback 函数中用到 this，this是指向当前 遍历的元素；

```javascript
function each(obj, callback) {
  if (isArrayLike(obj)) {
    for (let i = 0; i < obj.length; i++) {
      if (callback.call(obj[i], i, obj[i]) === false) {
        break;
      }
    }
  } else {
    for (let item in obj) {
      if (callback.call(obj[item], item, obj[item]) === false) {
        break;
      }
    }
  }
  return obj;
}
```

