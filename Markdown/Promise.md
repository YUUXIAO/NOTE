---
typora-root-url: ..
---

## 理解与使用

Promise 对象是一个代理对象（代理一个值），被代理的值在Promise对象创建时可能是未知的。它允许你为异步操作的成功和失败分别绑定相应的处理方法（handlers）。 这让异步方法可以像同步方法那样返回值，但并不是立即返回最终执行结果，而是一个能代表未来出现的结果的promise对象；

#### Promise是什么

1. 抽象表达：Promise 是JS中进行异步编程的新的解决文案；
2. 具体表达： 
   - 从语法上来说：Promise 是一个构造函数；
   - 从功能上来说：Promise 对象用来封装一个异步操作并可以获取其结果；

#### Promise的调用流程

1. Promise 的构造方法接收一个executor()，在 new Promise()时就立刻执行这个executor回调；
2. executor()内部的异步任务被放入宏 / 微任务队列，等待执行；
3. then( )被执行，收集成功/失败回调，放入成功/失败队列；
4. executor() 的异步任务被执行，触发resolve/reject，从成功/失败队列中取出回调依次执行；

#### Promise的状态

1. pending：初始状态，既不是成功也也不是失败状态；
2. fulfilled：意味着操作成功完成；
3. rejected：意味着操作失败；

![img](https://mdn.mozillademos.org/files/8633/promises.png)

#### Prmoise 关键问题

##### promise异常传透

- 当使用promise的then链式调用时, 可以在最后指定失败的回调；
- 前面任何操作出了异常, 都会传到最后失败的回调中处理；
- 如果要中断 Promise 链：在回调函数中返回一个pendding状态的promise对象；

##### 为什么要使用promise？

-  指定回调函数的方式更加灵活: 
  1. 旧的: 必须在启动异步任务前指定；
  2. promise: 启动异步任务、返回promie对象 、 给promise对象绑定回调函数(甚至可以在异步任务结束后指定)，无论何时查询，都能得到这个状态；
-  支持链式调用, 可以解决回调地狱问题：
  1.  回调地狱的缺点：不便于阅读 / 不便于异常处理；
  2.  解决方案：promise链式调用；
  3.  终极解决方案：async/await；

##### 如何改变promise的状态?

- resolve(value): 如果当前是pendding就会变为resolved；
- reject(reason): 如果当前是pendding就会变为rejected；
- 抛出异常: 如果当前是pendding就会变为rejected；

##### 如何串连多个操作任务的？

- promise的then()返回一个新的promise, 可以开成then()的链式调用；
- 通过then的链式调用串连多个同步/异步任务；

##### 改变promise状态和指定回调函数谁先谁后？

- 都有可能, 正常情况下是先指定回调再改变状态, 但也可以先改状态再指定回调；
- 先改状态再指定回调：
  1. 在执行器中直接调用resolve() / reject()；
  2. 延迟更长时间才调用then()；
- 什么时候才能得到数据：
  1. 如果先指定的回调, 那当状态发生改变时, 回调函数就会调用, 得到数据；
  2. 如果先改变的状态, 那当指定回调时, 回调函数就会调用, 得到数据；

##### promise.then()返回的新promise的结果状态由什么决定？

- 简单表达: 由then()指定的回调函数执行的结果决定；
- 详细表达:
  1. 如果抛出异常, 新promise变为rejected, reason为抛出的异常；
  2. 如果返回的是非promise的任意值, 新promise变为resolved, value为返回的值；
  3. 如果返回的是另一个新promise, 此promise的结果就会成为新promise的结果；

##### promise、async和await在事件循环中的执行过程

![promise1](/images/promise1.jpg)

![promise2](/images/promise2.jpg)

##### promise的缺点

1. promise一旦新建就会立即执行，无法中途取消；
2. 如果不设置回调函数，promise内部的错误就无法反映到外部；
3. 当处于pending状态时，无法得知当前处于哪一个状态，是刚刚开始还是刚刚结束；


#### 主要API方法

##### Promise (excutor) 

```
1. excutor函数: 同步执行  (resolve, reject) => {}；
2. resolve函数: 内部定义成功时我们调用的函数 value => {}；
3. reject函数: 内部定义失败时我们调用的函数 reason => {}；
4. 说明: excutor会在Promise内部立即同步回调,异步操作在执行器中执行；
```

##### Promise.prototype.then（onResolved，onRejected）

```
1. onResolved函数: 成功的回调函数  (value) => {}；
2. onRejected函数: 失败的回调函数 (reason) => {}；
3. 说明: 指定用于得到成功value的成功回调和用于得到失败reason的失败回调，返回一个新的promise对象
```

##### Promise.prototype.catch（onRejected）

```
1. onRejected函数: 失败的回调函数 (reason) => {}；
2. 说明: then()的语法糖, 相当于: then(undefined, onRejected)
```

##### Promise.resolve（reason）

```
1. reason: 成功的数据或promise对象；
2. 说明: 返回一个状态由给定value决定的Promise对象：
  1. 如果该值是thenable(即，带有then方法的对象)，返回的Promise对象的最终状态由then方法执行决定；
  2. 否则的话(该value为空，基本类型或者不带then方法的对象),返回的Promise对象状态为fulfilled，并且将该value传递给对应的then方法；

```

##### Promise.reject（reason）

```
1. reason: 失败的原因；
2. 说明: 返回一个失败的promise对象；
```

##### Promise.all（iterable）

```
1. iterable: 包含n个promise的数组;
2. 说明: 返回一个新的promise, 只有所有的promise都成功才成功, 只要有一个失败了就直接失败;
3. 新的promise对象在触发成功状态以后，会把一个包含iterable里所有promise返回值的数组作为成功回调的返回值，顺序跟iterable的顺序保持一致；
```

##### Promise.race（iterable）

```
1.  iterable: 包含n个promise的数组；
2. 说明: 返回一个新的promise, 第一个完成的promise的结果状态就是最终的结果状态；
```

##### Promise.any（iterable）

- 接收一个Promise对象的集合，当其中的一个promise 成功，就返回那个成功的promise的值；

##### Promise.allSettled（iterable）

- 等到所有promises都完成（每个promise返回成功或失败）；
- 返回一个promise，该promise在所有promise完成后完成，并带有一个对象数组，每个对象对应每个promise的结果；

##### Promise.prototype.finally（onFinally）

- 添加一个事件处理回调于当前promise对象，并且在原promise对象解析完毕后，返回一个新的promise对象。回调会在当前promise运行完毕后被调用，无论当前promise的状态是完成(fulfilled)还是失败(rejected)；

## Async 与 Await

1. async 函数：
   - 返回值为 promise 对象；
   - promise 对象的结果由async函数执行的返回值决定；
2. await 表达式：
   - await 右侧的表达式一般为promise 对象，但也可以是其它的值 ；
   - 如果表达式是promise对象，await 返回的是promise 成功的值；
   - 如果表达式是其它的值，直接将此值作为await 的返回值 ；
3. 注意：
   - await必须写在async函数中, 但async函数中可以没有await；
   - 如果await的promise失败了, 就会抛出异常, 需要通过try...catch来捕获处理；

## JS异步之宏队列与微队列

```
js的运行机制是同步任务都在主线程上运行，形成一个栈，而异步任务就会放在任务队列中，等主线程的任务全部执行完之后，就会去任务队列里面看看是否存在任务需要执行；

其中在任务队列中还分为宏任务和微任务，需注意的是微任务是优先于宏任务的，每次都会先执行微任务，所以在每次执行一次宏任务之后，都会去看看有没有微任务需要执行，如果有，那么就是先执行微任务；

总结以上的一段话就是：同步任务在主线程中执行，异步任务放在任务队列中等待执行，一旦主线程的栈为空，则去任务队列中查看是否有微任务，有，则执行微任务，没有，则执行宏任务。
```

![宏队列与微队列](http://vipkshttp1.wiz.cn/ks/share/resources/49c30824-dcdf-4bd0-af2a-708f490b44a1/92b8cbfb-a474-4859-943b-6048e9dc66f6/index_files/60b9ff398449db2dcfef9197e2187ae6.png)

1. 宏列队: 用来保存待执行的宏任务(回调)，比如: 定时器回调 / DOM事件回调 / ajax回调；
2. 微列队: 用来保存待执行的微任务(回调)， 比如: promise的回调 / MutationObserver的回调；
3. JS执行时会区别这2个队列：
   - JS引擎首先必须先执行所有的初始化同步任务代码；
   - 每次准备取出第一个宏任务执行前, 都要将所有的微任务一个一个取出来执行；



## 手写Promise

```javascript
/* 
自定义Promise函数模块: IIFE
*/
(function (window) {
  const PENDING = "pending";
  const RESOLVED = "resolved";
  const REJECTED = "rejected";

  /* 
    Promise构造函数
    excutor: 执行器函数(同步执行)
  */
  function Promise(excutor) {
    // 将当前promise对象保存起来
    const self = this;
    // 给promise对象指定status属性, 初始值为pending
    self.status = PENDING;
    // 给promise对象指定一个用于存储结果数据的属性
    self.data = undefined;
    // 每个元素的结构: { onResolved() {}, onRejected() {}}
    self.callbacks = [];

    function resolve(value) {
      // 如果当前状态不是pending, 直接结束
      if (self.status !== PENDING) return;

      // 将状态改为resolved
      self.status = RESOLVED;
      // 保存value数据
      self.data = value;
      // 如果有待执行callback函数, 立即异步执行回调函数onResolved
      if (self.callbacks.length > 0) {
        setTimeout(() => {
          // 放入队列中执行所有成功的回调
          self.callbacks.forEach((calbacksObj) => {
            calbacksObj.onResolved(value);
          });
        });
      }
    }

    function reject(reason) {
      // 如果当前状态不是pending, 直接结束
      if (self.status !== PENDING) return;

      // 将状态改为rejected
      self.status = REJECTED;
      // 保存value数据
      self.data = reason;
      // 如果有待执行callback函数, 立即异步执行回调函数onRejected
      if (self.callbacks.length > 0) {
        setTimeout(() => {
          // 放入队列中执行所有成功的回调
          self.callbacks.forEach((calbacksObj) => {
            calbacksObj.onRejected(reason);
          });
        });
      }
    }

    // 立即同步执行excutor
    try {
      excutor(resolve, reject);
    } catch (error) {
      // 如果执行器抛出异常, promise对象变为rejected状态
      reject(error);
    }
  }

  /* 
    Promise原型对象的then()
    指定成功和失败的回调函数
    返回一个新的promise对象
    返回promise的结果由onResolved/onRejected执行结果决定
  */
  Promise.prototype.then = function (onResolved, onRejected) {
    const self = this;
    // 指定回调函数的默认值(必须是函数)
    onResolved =
      typeof onResolved === "function" ? onResolved : (value) => value;
    onRejected =
      typeof onRejected === "function"
        ? onRejected
        : (reason) => {
            throw reason;
          };

    // 返回一个新的promise
    return new Promise((resolve, reject) => {
      /* 
        执行指定的回调函数
        根据执行的结果改变return的promise的状态/数据
      */
      function handle(callback) {
        // 返回promise的结果由onResolved / onRejected执行结果决定;
        // 1. 抛出异常, 返回promise的结果为失败, reason为异常
        // 2. 返回的是promise, 返回promise的结果就是这个结果
        // 3. 返回的不是promise, 返回promise为成功, value就是返回值
        try {
          const result = callback(self.data);
          if (result instanceof Promise) {
            // 返回的是promise, 返回promise的结果就是这个结果
            result.then(
              (value) => resolve(vlaue),
              (reason) => reject(reason)
            );
            // result.then(resolve, reject);
          } else {
            // 返回的不是promise, 返回promise为成功, value就是返回值
            resolve(result);
          }
        } catch (error) {
          // 抛出异常, 返回promise的结果为失败, reason为异常
          reject(error);
        }
      }

      // 当前promise的状态是resolved
      if (self.status === RESOLVED) {
        // 立即异步执行成功的回调函数
        setTimeout(() => {
          handle(onResolved);
        });
      } else if (self.status === REJECTED) {
        // 当前promise的状态是rejected
        // 立即异步执行失败的回调函数
        setTimeout(() => {
          handle(onRejected);
        });
      } else {
        // 当前promise的状态是pending
        // 将成功和失败的回调函数保存callbacks容器中缓存起来
        self.callbacks.push({
          onResolved(value) {
            handle(onResolved);
          },
          onRejected(reason) {
            handle(onRejected);
          },
        });
      }
    });
  };

  /* 
    Promise原型对象的catch()
    指定失败的回调函数
    返回一个新的promise对象
  */
  Promise.prototype.catch = function (onRejected) {
    return this.then(undefined, onRejected);
  };

  /* 
    Promise函数对象的resolve方法
    返回一个指定结果的成功的promise
  */
  Promise.resolve = function (value) {
    // 返回一个成功/失败的promise
    return new Promise((resolve, reject) => {
      // value是promise
      if (value instanceof Promise) {
        // 使用value的结果作为promise的结果
        value.then(resolve, reject);
      } else {
        // value不是promise  => promise变为成功, 数据是value
        resolve(value);
      }
    });
  };

  /* 
    Promise函数对象的reject方法
    返回一个指定reason的失败的promise
  */
  Promise.reject = function (reason) {
    // 返回一个失败的promise
    return new Promise((resolve, reject) => {
      reject(reason);
    });
  };

  /* 
    Promise函数对象的all方法
    返回一个promise, 只有当所有proimse都成功时才成功, 否则只要有一个失败的就失败
  */
  Promise.all = function (promises) {
    // 用来保存所有成功value的数组
    const values = new Array(promises.length);
    // 用来保存成功promise的数量
    let resolvedCount = 0;
    // 返回一个新的promise
    return new Promise((resolve, reject) => {
      // 遍历promises获取每个promise的结果
      promises.forEach((p, index) => {
        Promise.resolve(p).then(
          (value) => {
            resolvedCount++; // 成功的数量加1
            // p成功, 将成功的vlaue保存vlaues
            values[index] = value;
            // 如果全部成功了, 将return的promise改变成功
            if (resolvedCount === promises.length) {
              resolve(values);
            }
          },
          (reason) => {
            // 只要一个失败了, return的promise就失败
            reject(reason);
          }
        );
      });
    });
  };

  /* 
    Promise函数对象的race方法
    返回一个promise, 其结果由第一个完成的promise决定
  */
  Promise.race = function (promises) {
    // 返回一个promise
    return new Promise((resolve, reject) => {
      // 遍历promises获取每个promise的结果
      promises.forEach((p, index) => {
        Promise.resolve(p).then(
          (value) => {
            // 一旦有成功了, 将return变为成功
            resolve(value);
          },
          (reason) => {
            // 一旦有失败了, 将return变为失败
            reject(reason);
          }
        );
      });
    });
  };

  /* 
    返回一个promise对象, 它在指定的时间后才确定结果
  */
  Promise.resolveDelay = function (value, time) {
    // 返回一个成功/失败的promise
    return new Promise((resolve, reject) => {
      setTimeout(() => {
        // value是promise
        if (value instanceof Promise) {
          // 使用value的结果作为promise的结果
          value.then(resolve, reject);
        } else {
          // value不是promise  => promise变为成功, 数据是value
          resolve(value);
        }
      }, time);
    });
  };

  /* 
    返回一个promise对象, 它在指定的时间后才失败
  */
  Promise.rejectDelay = function (reason, time) {
    // 返回一个失败的promise
    return new Promise((resolve, reject) => {
      setTimeout(() => {
        reject(reason);
      }, time);
    });
  };

  // 向外暴露Promise函数
  window.Promise = Promise;
})(window);

```

