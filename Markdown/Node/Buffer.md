## Buffer 结构

> Buffer 是一个像 Array 的对象，但它主要用于操作字节；

### 模块结构

1. Buffer 是一个典型的 javascript 与 C++ 结合的模块，它将性能相关用 C++ 实现，非性能相关用 javascript 实现；
2. Buffer 所占用的内存不是通过V8 分配的，是堆外内存；
3. 由于 Buffer 太过常见，Node 在进程启动时就已经加载了它，并将其放在全局对象（global）上，所以在使用 Buffer 时，无需通过 require() 即可使用；

### Buffer 对象

> Buffer对象类似于数组，它的元素为 16 进制的两位数，即0到255的数值；

```javascript
var str = "深入浅出node.js"
var buf = new Buffer(str, 'utf-8')
console.log(buf)
// <Buffer e6 b7 bq e5 85 a5 e6 b5 85 e5 87 ba 6e 6f 64 65 2e 6a 73>
```

Buffer 可以访问length 属性得到长度，也可以通过下标访问元素；

```javascript
var buf = new Buffer(100)
console.log(buf.length)  // 100
console.log(buf[10])  // 得到 0-255 的随机值 

buf[10] = 10
console.log(buf[10])  // 10
```

如果给元素赋值不是0-255的整数时：

1. 赋值小于 0，就将该值逐次加256，直到得到一个0-255之间的整数 ；
2. 赋值大于255，就逐次减 256，直到得到一个0-255之间的整数 ；
3. 如果是小数，就舍弃小数部分，只保留整数部分；

```javascript
buf[20] = -100
console.log(buf[20])  // 156

buf[21] = 300
console.log(buf[21])  // 44

buf[22] = 3.1415
console.log(buf[22])  // 3
```

### Buffer内存分配