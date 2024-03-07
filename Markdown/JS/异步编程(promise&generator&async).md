## 传统的定时器

传统的异步编程有：setTimeout( )，setInterval( )等，但是当同步代码较多时，不确定异步定时器的任务能在指定的时间执行； 

定时器只是能保证在xx秒之后把任务推进任务队列，不能保证在指定的时间执行代码

## ES6的异步编程

### Promise

[Promise相关内容请参考这篇：Promise](Promise.md)

```javascript
new promise((resolve,reject)=>{ resolve() }).then()
```

### Generator（生成器）

生成器(Generator) 是ES6中引入的特殊函数，使用 **function* 语法定义**，返回一个迭代器对象；

它是一个可以暂停和恢复执行的函数，允许在函数内部使用 yield 关键字来暂停和恢复函数的执行

1. **暂停和恢复执行：**Generator 函数可以在执行过程中暂停，并在需要时恢复执行。每次调用 Generator 函数的 next() 方法时，函数会从上次暂停的地方继续执行，直到遇到下一个 yield 关键字或函数结束。
2. **双向通信：**Generator 函数可以通过 yield 关键字向外部传递数据，也可以通过 next() 方法传递数据给 Generator 函数。
3. **异步操作：**Generator 函数可以与异步操作结合使用，通过 yield 关键字可以暂停函数的执行，等待异步操作完成后再继续执行。
4. **遍历器对象**：Generator 函数返回的是一个遍历器对象，可以使用 for...of 循环来遍历 Generator 函数内部产生的值

```javascript
// 代码1
const result = function* genFunction() {
  yield "hello!";
}
result.next() // {value: 'hello', done: false}
result.next() // {value: undefined, done: true}


// 代码2
const result2 = function* genFunction2() {
  console.log('开始执行');
  yield '暂停';
  console.log('继续执行');
  return '停止';
}
logger.next(); 
// { value: '暂停', done: false }
logger.next();
// { value: '停止', done: true }

```

生成器对象是实现了迭代器协议，所以可以通过 for...of循环和拓展运算符来进行操作

```javascript
function* abcs() {
  yield 'a';
  yield 'b';
  yield 'c';
}

for (let letter of abcs()) {
  console.log(letter); // a, b, c
}

[...abcs()] // [ "a", "b", "c" ]
```

#### 迭代器(Iterator)

迭代器就是实现了 next（）方法的一种特殊的对象，这个next（）方法返回一个对象，包含了value和done两个属性：value是从对象中取的值，done表示迭代是否结束

```javascript
Iterator.next() // { value: '1', done: false }
Iterator.next() // { value: '2', done: false }
Iterator.next() // { value: undefined, done: true } // 迭代结束
```

迭代器的工作原理如下：

1. 创建一个指针对象，指向当前数据结构的起始位置（感觉迭代器对象本质也是一个指针对象）
2. 第一次调用next（）方法，那就把指针挪向数据结构的第一个成员，返回包含了value和done两个属性的值
3. 后面以此类推
4. 不断调用指针对象的next方法，直到它指向数据结构的结束位置

#### [Symbol.iterator]属性

[Symbol.iterator] 是默认的对象迭代方法，当它内部返回一个迭代器对象时，那这个对象就可以迭代了

比如我们用for...of或者拓展运算符遍历对象时就是迭代操作

一般那些原本就可以迭代的对象、数组它们有自己的默认的[Symbol.Iterator]函数的，如果遇到不能迭代的对象或者不满意迭代的结果，我门也可以尝试自己写：

```javascript
let obj = {
    data: [1, 2, 3, 4, 5, 6],
    [Symbol.iterator]() {
        const self = this;
        let index = 0;
        return {
            next() {
                if (index < self.data.length) {
                    return {
                        value: self.data[index++],
                        done: false
                    }
                } else {
                    return {
                        value: undefined,
                        done: true
                    }
                }
            }
        }
    }
}
```

#### 接收入参

yield 还能接收参数，next（）传入的第一个值会被yield 接收，注意执行第一次next（）传入的值会被忽略，因为这个时候还没有遇到 yield 关键词

```javascript
function* listener() {
  console.log("你说，我在听...");
  while (true) {
    let msg = yield;
    console.log('我听到你说:', msg);
  }
}

let l = listener();
l.next('在吗？'); // 你说，我在听...
l.next('你在吗？'); // 我听到你说: 你在吗？
l.next('芜湖！'); // 我听到你说: 芜湖！

```

#### 递归生成器

在生成器中想要实现递归调用不能直接使用 yield，因为执行生成器函数返回的值是生成器对象（value:xxx, done: false)，在实际应用中我们希望得到的生成后的值

通过 yield* 实现生成器函数的递归，可以通过给对象实现 Symbol.iterator，这样我们就可以用拓展运算符或者 for..of 循环对其进行遍历

```javascript
const test = {
    x: 3,
    * [Symbol.iterator]() {
        for (let i = 1; i <= this.x; i++) yield i
    }
}
console.log(...test)// 1 2 3

```

#### 生成器应用

**自定义序列生成**

生成器可以帮我们更方便地进行序列的生成。例如我们想要开发一个扑克游戏，需要一个包含了所有扑克牌数值的序列，如果一一列举会非常麻烦，用生成器就可以简化我们的工作

```javascript
cards = ({
  suits: ["♣️", "♦️", "♥️", "♠️"],
  court: ["J", "Q", "K", "A"],
  [Symbol.iterator]: function* () {
    for (let suit of this.suits) {
      for (let i = 2; i <= 10; i++) yield suit + i;
      for (let c of this.court) yield suit + c;
    }
  }
})
```

**惰性计算**

因为生成器函数执行时会在yield处停止，所以即便是while(true)这样的死循环也不会导致程序卡死，我们可以写一个不断生成随机数的generateRandomNumbers函数

```javascript
function* generateRandomNumbers(count) {
  for (let i = 0; i < count; i++) {
    yield Math.random()
  }
}
```

**状态机**

利用**生成器可以接受入参的特性，可以通过生成器函数来构建一个状态机**，比如一个取款的操作或者日常一次一抽奖的活动等

```javascript
function* bankAccount() {
  let balance = 0;
  while (balance >= 0) {
    balance += yield balance;
  }
  return '你破产了！';
}

let account = bankAccount();
account.next();    // { value: 0, done: false }
account.next(50);  // { value: 50, done: false }
account.next(-10); // { value: 40, done: false }
account.next(-60); // { value: "你破产了！", done: true }
```



### async

[async相关内容请参考这篇：Promise](Promise.md)