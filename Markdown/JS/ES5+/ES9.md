## 对象扩展操作符

```javascript
var obj1 = { foo: 'bar', x: 42 };
var obj2 = { foo: 'baz', y: 13 };

var clonedObj = { ...obj1 };
// 克隆后的对象: { foo: "bar", x: 42 }

var mergedObj = { ...obj1, ...obj2 };
// 合并后的对象: { foo: "baz", x: 42, y: 13 }

```

## Promise.prototype.finally

finally() 方法会返回一个 Promise，当 Promise 状态变更，不管是变成 rejected 或者 fulfilled，最终都会执行 finally() 的回调；

```javascript
fetch(url)
  .then((res) => {
    console.log(res)
  })
  .catch((error) => { 
    console.log(error)
  })
  .finally(() => { 
    console.log('结束')
})

```

