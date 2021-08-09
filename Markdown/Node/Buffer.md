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

> Buffer 对象的内存分配不是在 V8 的堆内存中，而是在Node 的 C++层面实现内存的申请的；

为了更高效地使用申请来的内存，Node 采用了 slab 分配机制，这是一种动态的内存管理机制，slab 具有以下3种状态：

- full：完全分配状态；
- partial：部分分配状态；
- empty：没有被分配状态；

Node 以8kb为界限来区分 Buffer 是大对象还是小对象，这个 8KB 的值是每个 slab 的大小值，在 javascript 层面，以它作为单元进行内存的分配；

#### 分配小Buffer对象

> 如果指定Buffer 的大小小于 8KB，Node会按照小对象的方式进行分配；

1. 分配过程中主要使用一个局部变量 pool 作为中间处理对象，处于分配状态的 slab 单元都指向它；

```javascript
// 分配一个全新的 slab 单元的操作，它会将新申请的SlowBuffer对象指向它；
var pool;
function  allocPool(){
  pool = new SlowBuffer(Buffer.poolsize)
  pool.used = 0
}
```

2. 构造小 Buffer 对象时将会去检查  pool 对象，如果 pool 没有被创建，将会创建一个新的 slab 单元指向它，此时 slab 的状态为 empty；

```javascript
new Buffer（1024）

if(!pool || pool.length - pool.used < this.length) allocPool();
```

3. 同时当前 Buffer 对象的 parent 属性指向该 slab，并记录下是从这个 slab 的哪个位置开始使用，slab 自身也记录了被使用的多少字节，此时 slab 的状态为 partial；

```javascript
this.parent = pool;
this.offset = pool.used;
if(pool.used & 7) pool.used = (pool.used + 8 )& ~7
```

4. 再次创建一个 Buffer 对象时，构造过程会判断这个 slab 的剩余空间是否足够，如果足够使用剩余空间并更新 slab 的分配状态；
5. 如果 slab 的剩余空间不够，会构造新的 slab，原 slab 中剩余的空间会造成浪费； 
6. 由于同一个 slab 可能分配给多个 Buffer 对象使用，只有这些小 Buffer 对象在作用域释放并都可以回收时，slab 的8KB空间才会被回收；

#### 分配大Buffer对象

如果指定Buffer 的大小大于 8KB，将会直接分配一个 SlowBuffer 对象作为 slab 单元，这个 slab 单元将会被这个大 Buffer 对象独占；

## Buffer 的转换

Buffer 可以与字符串之间进行转换，目前支持的字符串编码类型有这几种：

- ASCII
- UTF-8
- UTF-16LE/UCS-2
- Base64
- Binary
- Hex

### 字符串转Buffer

> 字符串转 Buffer 对象主要是通过构造函数来完成的；

1. encoding 参数不传递时，默认按 UTF-8 编码进行转码和存储；

```javascript
new Buffer(str, [encoding])
```

2. 一个 Buffer 对象可以存储不同编码类型的字符串转码的值，调用 write() 方法；

```javascript
buf.write(string,[offset],[length],[encoding])
```

### Buffer转字符串

> Buffer 对象的 toString（） 可以将 Buffer 对象转换为字符串；

如果Buffer对象由多种编码写入，需要在局部指定不同的编码，才能转换成正常的编码；

```javascript
buf.toString([],[start],[end])
```

### Buffer不支持的编码类型

Buffer 提供了一个 isEncoding（）函数来判断编码是否支持转换，如果支持转换返回值为 true，否则为 false。

```javascript
Buffer.isEncoding(encoding)
```

## Buffer的拼接

Buffer 在使用场景中，通常是以一段一段的方式传输；

```javascript
var fs = require('fs')

var rs = fs.createReadStream('test.md')
var data = ''
rs.on('data',function(chunk){
  data += chunk
  // 相等于 data = data.toString() + chunk.toString()
})
rs.on('end',function(){
  console.log(data)
})
```

### 乱码是如何产生的

1. 文件可读流在读取时会逐个读取 Buffer；

2. 如果在文件可读流的每次读取的 Buffer 的长度限制为 11；

   ```javascript
   var rs = fs.createReadStream('test.md',{highWateMark:11})
   ```

3. buf.toString() 方法默认以 UTF-8 为编码，中文字在 UTF-8 下占3个字节；

4. 对于任意长度的 Buffer 而言，宽字节字符串都有可能存在被截断的情况；

### setEncoding()与string_decoder()

可读流有一个可设置编码的方法 setEncoding()，该方法的作用是让 data 事件中传递的不是一个 Buffer 对象，而是编码之后的字符串；

```javascript
var rs = fs.createReadStream('test.md',{highWateMark:11})
rs.setEncoding('utf-8')
```

1. 在调用 setEncoding（）时，可读流对象在内部设置了一个 decoder 对象；
2. 每次 data 事件都通过该 decoder 对象进行 Buffer 到字符串的解码，然后传递给调用者；
3. 最终乱码问题的解决是因为 decoder 的神奇之处：
   - decoder对象来自于 string_decoder 模块StringDecoder 的实例对象；
   - StringDecoder 在得到 编码后，知道宽字节字符串在 UTF-8 编码下是以 3个字节的方式存储的；
   - 所以在第一次 write() 之后，只输出前 9个字节转码形成的字符，后面的字符被保留在了 StringDecoder  实例内部；
   - 第二次 write() 时，会将这个剩余字节和后续字节组合在一起，再次用3的整数倍进行转码；
4. string_decoder 模块目前只能处理 UTF-8、Base64 和 UTF-16LE/UCS-2 这3种编码













