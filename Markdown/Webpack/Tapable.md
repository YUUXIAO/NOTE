> webpack 的插件架构主要是基于 Tapable 实现的，Tapable 是 webpack 项目组的一个内部库，主要是抽象了一套插件机制，专注于自定义事件的触发和操作；

## hooks概览

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8a1d41bd086d4cf68afd7c9486979a1b~tplv-k3u1fbpfcp-zoom-1.image)

> 常用的钩子主要包含以下几种，分为同步和异步，异步又分为并发执行和串行执行；

```javascript
const {
  	// 同步串行: 不关心监听函数的返回值
	SyncHook,
  	// 同步串行: 只要监听函数中有一个函数的返回值不为 null，则跳过剩下所有的逻辑
	SyncBailHook,
  	// 同步串行: 上一个监听函数的返回值可以传给下一个监听函数
	SyncWaterfallHook,
  	// 同步循环: 当监听函数被触发的时候，如果该监听函数返回true时则这个监听函数会反复执行，如果返回 undefined 则表示退出循环
	SyncLoopHook,
    // 异步并发: 不关心监听函数的返回值
	AsyncParallelHook,
  	// 异步并发: 只要监听函数的返回值不为 null，就会忽略后面的监听函数执行，直接跳跃到callAsync等触发函数绑定的回调函数，然后执行这个被绑定的回调函数
	AsyncParallelBailHook,
  	// 异步串行: 不关系callback()的参数
	AsyncSeriesHook,
  	// 异步串行: callback()的参数不为null，就会直接执行callAsync等触发函数绑定的回调函数
	AsyncSeriesBailHook,
  	// 异步串行: 上一个监听函数的中的callback(err, data)的第二个参数,可以作为下一个监听函数的参数
	AsyncSeriesWaterfallHook
 } = require("tapable");
```

所有钩子类的构造函数都接收一个可选的参数，这个参数是一个由字符串参数组成的数组：

```
const hook = new SyncHook(["arg1", "arg2", "arg3"]);
```

## 钩子

### sync* 钩子

- 注册在该钩子下面的插件的执行顺序都是顺序执行；
- 只能使用 tap 注册，不能使用 tapPromise 和 tapAsync 注册；

#### SyncHook

```javascript
const { SyncHook } = require("tapable");
let queue = new SyncHook(['name']); //所有的构造函数都接收一个可选的参数，这个参数是一个字符串的数组。

// 订阅
queue.tap('1', function (name, name2) {// tap 的第一个参数是用来标识订阅的函数的
    console.log(name, name2, 1);
    return '1'
});
queue.tap('2', function (name) {
    console.log(name, 2);
});
queue.tap('3', function (name) {
    console.log(name, 3);
});

// 发布
queue.call('webpack', 'webpack-cli');// 发布的时候触发订阅的函数 同时传入参数

// 执行结果:
/* 
webpack undefined 1 // 传入的参数需要和new实例的时候保持一致，否则获取不到多传的参数
webpack 2
webpack 3
*/                       
```

原理

```javascript
class SyncHook_MY{
    constructor(){
        this.hooks = [];
    }

    // 订阅
    tap(name, fn){
        this.hooks.push(fn);
    }

    // 发布
    call(){
        this.hooks.forEach(hook => hook(...arguments));
    }
}
```

#### SyncBailHook

> 只要监听函数中有一个函数的返回值不为 undefined，则跳过剩下所有的逻辑；

```javascript
const {
    SyncBailHook
} = require("tapable");

let queue = new SyncBailHook(['name']); 

queue.tap('1', function (name) {
    console.log(name, 1);
});
queue.tap('2', function (name) {
    console.log(name, 2);
    return 'wrong'
});
queue.tap('3', function (name) {
    console.log(name, 3);
});

queue.call('webpack');

// 执行结果:
/* 
webpack 1
webpack 2
*/

```

原理

```javascript
class SyncBailHook_MY {
    constructor() {
        this.hooks = [];
    }

    // 订阅
    tap(name, fn) {
        this.hooks.push(fn);
    }

    // 发布
    call() {
        for (let i = 0, l = this.hooks.length; i < l; i++) {
            let hook = this.hooks[i];
            let result = hook(...arguments);
            if (result) {
                break;
            }
        }
    }
}
```

#### SyncWaterfallHook

> 上一个监听函数的返回值可以传给下一个监听函数；

```javascript
const {
    SyncWaterfallHook
} = require("tapable");

let queue = new SyncWaterfallHook(['name']);

// 上一个函数的返回值可以传给下一个函数
queue.tap('1', function (name) {
    console.log(name, 1);
    return 1;
});
queue.tap('2', function (data) {
    console.log(data, 2);
    return 2;
});
queue.tap('3', function (data) {
    console.log(data, 3);
});

queue.call('webpack');

// 执行结果:
/* 
webpack 1
1 2
2 3
*/
```

原理

```javascript
class SyncWaterfallHook_MY{
    constructor(){
        this.hooks = [];
    }
    
    // 订阅
    tap(name, fn){
        this.hooks.push(fn);
    }

    // 发布
    call(){
        let result = null;
        for(let i = 0, l = this.hooks.length; i < l; i++) {
            let hook = this.hooks[i];
            result = i == 0 ? hook(...arguments): hook(result); 
        }
    }
}

```

#### SyncLoopHook

> 当监听函数被触发的时候，如果该监听函数返回 true 时则这个监听函数会反复执行，如果返回 undefined 则表示退出循环；

```javascript
const {
    SyncLoopHook
} = require("tapable");

let queue = new SyncLoopHook(['name']); 

let count = 3;
queue.tap('1', function (name) {
    console.log('count: ', count--);
    if (count > 0) {
        return true;
    }
    return;
});

queue.call('webpack');

// 执行结果:
/* 
count:  3
count:  2
count:  1
*/
```

原理

```javascript
class SyncLoopHook_MY {
    constructor() {
        this.hook = null;
    }

    // 订阅
    tap(name, fn) {
        this.hook = fn;
    }

    // 发布
    call() {
        let result;
        do {
            result = this.hook(...arguments);
        } while (result)
    }
}
```

### async* 钩子

#### AsyncParallelHook

> 不关心监听函数的返回值，有三种注册/发布的模式（tap / callAsync，tapAsync / callAsync，tapPromise / promise）;

- usage - tap

```javascript
const {
    AsyncParallelHook
} = require("tapable");

let queue1 = new AsyncParallelHook(['name']);
console.time('cost');
queue1.tap('1', function (name) {
    console.log(name, 1);
});
queue1.tap('2', function (name) {
    console.log(name, 2);
});
queue1.tap('3', function (name) {
    console.log(name, 3);
});
queue1.callAsync('webpack', err => {
    console.timeEnd('cost');
});

// 执行结果
/* 
webpack 1
webpack 2
webpack 3
cost: 4.520ms
*/
```

- usage - tapAsync

```javascript
let queue2 = new AsyncParallelHook(['name']);
console.time('cost1');
queue2.tapAsync('1', function (name, cb) {
    setTimeout(() => {
        console.log(name, 1);
        cb();
    }, 1000);
});
queue2.tapAsync('2', function (name, cb) {
    setTimeout(() => {
        console.log(name, 2);
        cb();
    }, 2000);
});
queue2.tapAsync('3', function (name, cb) {
    setTimeout(() => {
        console.log(name, 3);
        cb();
    }, 3000);
});

queue2.callAsync('webpack', () => {
    console.log('over');
    console.timeEnd('cost1');
});

// 执行结果
/* 
webpack 1
webpack 2
webpack 3
over
time: 3004.411ms
*/
```

- usage - promise

```javascript
let queue3 = new AsyncParallelHook(['name']);
console.time('cost3');
queue3.tapPromise('1', function (name, cb) {
   return new Promise(function (resolve, reject) {
       setTimeout(() => {
           console.log(name, 1);
           resolve();
       }, 1000);
   });
});

queue3.tapPromise('1', function (name, cb) {
   return new Promise(function (resolve, reject) {
       setTimeout(() => {
           console.log(name, 2);
           resolve();
       }, 2000);
   });
});

queue3.tapPromise('1', function (name, cb) {
   return new Promise(function (resolve, reject) {
       setTimeout(() => {
           console.log(name, 3);
           resolve();
       }, 3000);
   });
});

queue3.promise('webpack')
   .then(() => {
       console.log('over');
       console.timeEnd('cost3');
   }, () => {
       console.log('error');
       console.timeEnd('cost3');
   });
/* 
webpack 1
webpack 2
webpack 3
over
cost3: 3007.925ms
*/
```

#### AsyncParallelBailHook

> 只要监听函数的返回值不为  null，就会忽略后面的监听函数执行，直接跳跃到 callAsync 等触发函数绑定的回调函数，然后执行这个被绑定的回调函数；

- usage - tap

```javascript
let queue1 = new AsyncParallelBailHook(['name']);
console.time('cost');
queue1.tap('1', function (name) {
    console.log(name, 1);
});
queue1.tap('2', function (name) {
    console.log(name, 2);
    return 'wrong'
});
queue1.tap('3', function (name) {
    console.log(name, 3);
});
queue1.callAsync('webpack', err => {
    console.timeEnd('cost');
});
// 执行结果:
/* 
webpack 1
webpack 2
cost: 4.975ms
 */
```

- usage - tapAsync

```javascript
let queue2 = new AsyncParallelBailHook(['name']);
console.time('cost1');
queue2.tapAsync('1', function (name, cb) {
    setTimeout(() => {
        console.log(name, 1);
        cb();
    }, 1000);
});
queue2.tapAsync('2', function (name, cb) {
    setTimeout(() => {
        console.log(name, 2);
        return 'wrong';// 最后的回调就不会调用了
        cb();
    }, 2000);
});
queue2.tapAsync('3', function (name, cb) {
    setTimeout(() => {
        console.log(name, 3);
        cb();
    }, 3000);
});

queue2.callAsync('webpack', () => {
    console.log('over');
    console.timeEnd('cost1');
});

// 执行结果:
/* 
webpack 1
webpack 2
webpack 3
*/
```

- usage - promise

```javascript
let queue3 = new AsyncParallelBailHook(['name']);
console.time('cost3');
queue3.tapPromise('1', function (name, cb) {
    return new Promise(function (resolve, reject) {
        setTimeout(() => {
            console.log(name, 1);
            resolve();
        }, 1000);
    });
});

queue3.tapPromise('2', function (name, cb) {
    return new Promise(function (resolve, reject) {
        setTimeout(() => {
            console.log(name, 2);
            reject('wrong');// reject()的参数是一个不为null的参数时，最后的回调就不会再调用了
        }, 2000);
    });
});

queue3.tapPromise('3', function (name, cb) {
    return new Promise(function (resolve, reject) {
        setTimeout(() => {
            console.log(name, 3);
            resolve();
        }, 3000);
    });
});

queue3.promise('webpack')
    .then(() => {
        console.log('over');
        console.timeEnd('cost3');
    }, () => {
        console.log('error');
        console.timeEnd('cost3');
    });

// 执行结果:
/* 
webpack 1
webpack 2
error
cost3: 2009.970ms
webpack 3
*/
```

#### AsyncSeriesHook

> 不关心 callback() 的参数；

- usage - tap

```javascript
const {
    AsyncSeriesHook
} = require("tapable");

// tap
let queue1 = new AsyncSeriesHook(['name']);
console.time('cost1');
queue1.tap('1', function (name) {
    console.log(1);
    return "Wrong";
});
queue1.tap('2', function (name) {
    console.log(2);
});
queue1.tap('3', function (name) {
    console.log(3);
});
queue1.callAsync('zfpx', err => {
    console.log(err);
    console.timeEnd('cost1');
});
// 执行结果
/* 
1
2
3
undefined
cost1: 3.933ms
*/
```

- usage - tapAsync

```javascript
let queue2 = new AsyncSeriesHook(['name']);
console.time('cost2');
queue2.tapAsync('1', function (name, cb) {
    setTimeout(() => {
        console.log(name, 1);
        cb();
    }, 1000);
});
queue2.tapAsync('2', function (name, cb) {
    setTimeout(() => {
        console.log(name, 2);
        cb();
    }, 2000);
});
queue2.tapAsync('3', function (name, cb) {
    setTimeout(() => {
        console.log(name, 3);
        cb();
    }, 3000);
});

queue2.callAsync('webpack', (err) => {
    console.log(err);
    console.log('over');
    console.timeEnd('cost2');
}); 
// 执行结果
/* 
webpack 1
webpack 2
webpack 3
undefined
over
cost2: 6019.621ms
*/
```

- usage - promise

```javascript
let queue3 = new AsyncSeriesHook(['name']);
console.time('cost3');
queue3.tapPromise('1',function(name){
   return new Promise(function(resolve){
       setTimeout(function(){
           console.log(name, 1);
           resolve();
       },1000)
   });
});
queue3.tapPromise('2',function(name,callback){
    return new Promise(function(resolve){
        setTimeout(function(){
            console.log(name, 2);
            resolve();
        },2000)
    });
});
queue3.tapPromise('3',function(name,callback){
    return new Promise(function(resolve){
        setTimeout(function(){
            console.log(name, 3);
            resolve();
        },3000)
    });
});
queue3.promise('webapck').then(err=>{
    console.log(err);
    console.timeEnd('cost3');
});

// 执行结果
/* 
webapck 1
webapck 2
webapck 3
undefined
cost3: 6021.817ms
*/
```

原理

```javascript
class AsyncSeriesHook_MY {
    constructor() {
        this.hooks = [];
    }

    tapAsync(name, fn) {
        this.hooks.push(fn);
    }

    callAsync() {
        var slef = this;
        var args = Array.from(arguments);
        let done = args.pop();
        let idx = 0;

        function next(err) {
            // 如果next的参数有值，就直接跳跃到 执行callAsync的回调函数
            if (err) return done(err);
            let fn = slef.hooks[idx++];
            fn ? fn(...args, next) : done();
        }
        next();
    }
}
```

#### AsyncSeriesBailHook

> callback() 的参数不为 null，就会直接执行 callAsync 等触发函数绑定的回调函数；

- usage - tap

```javascript
const {
    AsyncSeriesBailHook
} = require("tapable");

// tap
let queue1 = new AsyncSeriesBailHook(['name']);
console.time('cost1');
queue1.tap('1', function (name) {
    console.log(1);
    return "Wrong";
});
queue1.tap('2', function (name) {
    console.log(2);
});
queue1.tap('3', function (name) {
    console.log(3);
});
queue1.callAsync('webpack', err => {
    console.log(err);
    console.timeEnd('cost1');
});

// 执行结果:
/* 
1
null
cost1: 3.979ms
*/
```

- usage - tapAsync

```javascript
let queue2 = new AsyncSeriesBailHook(['name']);
console.time('cost2');
queue2.tapAsync('1', function (name, callback) {
    setTimeout(function () {
        console.log(name, 1);
        callback();
    }, 1000)
});
queue2.tapAsync('2', function (name, callback) {
    setTimeout(function () {
        console.log(name, 2);
        callback('wrong');
    }, 2000)
});
queue2.tapAsync('3', function (name, callback) {
    setTimeout(function () {
        console.log(name, 3);
        callback();
    }, 3000)
});
queue2.callAsync('webpack', err => {
    console.log(err);
    console.log('over');
    console.timeEnd('cost2');
});
// 执行结果

/* 
webpack 1
webpack 2
wrong
over
cost2: 3014.616ms
*/
```

- usage - promise

```javascript
let queue3 = new AsyncSeriesBailHook(['name']);
console.time('cost3');
queue3.tapPromise('1', function (name) {
    return new Promise(function (resolve, reject) {
        setTimeout(function () {
            console.log(name, 1);
            resolve();
        }, 1000)
    });
});
queue3.tapPromise('2', function (name, callback) {
    return new Promise(function (resolve, reject) {
        setTimeout(function () {
            console.log(name, 2);
            reject();
        }, 2000)
    });
});
queue3.tapPromise('3', function (name, callback) {
    return new Promise(function (resolve) {
        setTimeout(function () {
            console.log(name, 3);
            resolve();
        }, 3000)
    });
});
queue3.promise('webpack').then(err => {
    console.log(err);
    console.log('over');
    console.timeEnd('cost3');
}, err => {
    console.log(err);
    console.log('error');
    console.timeEnd('cost3');
});
// 执行结果：
/* 
webpack 1
webpack 2
undefined
error
cost3: 3017.608ms
*/
```

#### AsyncSeriesWaterfallHook

> 上一个监听函数的中的 callback(err, data) 的第二个参数,可以作为下一个监听函数的参数；

- usage - tap

```javascript
const {
    AsyncSeriesWaterfallHook
} = require("tapable");

// tap
let queue1 = new AsyncSeriesWaterfallHook(['name']);
console.time('cost1');
queue1.tap('1', function (name) {
    console.log(name, 1);
    return 'lily'
});
queue1.tap('2', function (data) {
    console.log(2, data);
    return 'Tom';
});
queue1.tap('3', function (data) {
    console.log(3, data);
});
queue1.callAsync('webpack', err => {
    console.log(err);
    console.log('over');
    console.timeEnd('cost1');
});

// 执行结果:
/* 
webpack 1
2 'lily'
3 'Tom'
null
over
cost1: 5.525ms
*/
```

- usage - tapAsync

```javascript
let queue2 = new AsyncSeriesWaterfallHook(['name']);
console.time('cost2');
queue2.tapAsync('1', function (name, callback) {
    setTimeout(function () {
        console.log('1: ', name);
        callback(null, 2);
    }, 1000)
});
queue2.tapAsync('2', function (data, callback) {
    setTimeout(function () {
        console.log('2: ', data);
        callback(null, 3);
    }, 2000)
});
queue2.tapAsync('3', function (data, callback) {
    setTimeout(function () {
        console.log('3: ', data);
        callback(null, 3);
    }, 3000)
});
queue2.callAsync('webpack', err => {
    console.log(err);
    console.log('over');
    console.timeEnd('cost2');
});
// 执行结果：
/* 
1:  webpack
2:  2
3:  3
null
over
cost2: 6016.889ms
*/
```

- usage - promise

```javascript
let queue3 = new AsyncSeriesWaterfallHook(['name']);
console.time('cost3');
queue3.tapPromise('1', function (name) {
    return new Promise(function (resolve, reject) {
        setTimeout(function () {
            console.log('1:', name);
            resolve('1');
        }, 1000)
    });
});
queue3.tapPromise('2', function (data, callback) {
    return new Promise(function (resolve) {
        setTimeout(function () {
            console.log('2:', data);
            resolve('2');
        }, 2000)
    });
});
queue3.tapPromise('3', function (data, callback) {
    return new Promise(function (resolve) {
        setTimeout(function () {
            console.log('3:', data);
            resolve('over');
        }, 3000)
    });
});
queue3.promise('webpack').then(err => {
    console.log(err);
    console.timeEnd('cost3');
}, err => {
    console.log(err);
    console.timeEnd('cost3');
});
// 执行结果：
/* 
1: webpack
2: 1
3: 2
over
cost3: 6016.703ms
*/
```

原理

```javascript
class AsyncSeriesWaterfallHook_MY {
    constructor() {
        this.hooks = [];
    }

    tapAsync(name, fn) {
        this.hooks.push(fn);
    }

    callAsync() {
        let self = this;
        var args = Array.from(arguments);

        let done = args.pop();
        console.log(args);
        let idx = 0;
        let result = null;

        function next(err, data) {
            if (idx >= self.hooks.length) return done();
            if (err) {
                return done(err);
            }
            let fn = self.hooks[idx++];
            if (idx == 1) {

                fn(...args, next);
            } else {
                fn(data, next);
            }
        }
        next();
    }
}
```

