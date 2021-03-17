Vue 中 watch 有两种使用方式：

```javascript
// watch选项
var vm = new Vue({
  el: '#app',
  data() {
    return {
      num: 12
    }
  },
  watch: {
    num() {}
  }
})
vm.num = 111

// $watch api方式
vm.$watch('num', function() {}, {
  deep: ,
  immediate: ,
})
```

## 收集依赖

Vue 初始化数据时会执行 initWatch，initWatch 的核心是 createWatch；

```javascript
function initWatch (vm, watch) {
  for (var key in watch) {
    var handler = watch[key];
    // handler可以是数组的形式，执行多个回调
    if (Array.isArray(handler)) {
      for (var i = 0; i < handler.length; i++) {
        createWatcher(vm, key, handler[i]);
      }
    } else {
      createWatcher(vm, key, handler);
    }
  }
}

function createWatcher (vm,expOrFn,handler,options) {
  // 针对watch是对象的形式，此时回调回选项中的handler
  if (isPlainObject(handler)) {
    options = handler;
    handler = handler.handler;
  }
  if (typeof handler === 'string') {
    handler = vm[handler];
  }
  return vm.$watch(expOrFn, handler, options)
}
```

### $watch

无论是选项的形式还是 api 的形式，最终都会调用实例的 $watch 方法，其中 expOrFn 是监听的字符串，handler 是监听的回调函数，options 是相关配置；

1. $watch 的核心是创建一个 user watcher，options.user 是当前用户定义 watcher 的标志；
2. 如果有 immediate 属性，立即执行回调函数；

```javascript
Vue.prototype.$watch = function (expOrFn,cb,options) {
    var vm = this;
    if (isPlainObject(cb)) {
      return createWatcher(vm, expOrFn, cb, options)
    }
    options = options || {};
    options.user = true;
    var watcher = new Watcher(vm, expOrFn, cb, options);
    // 当watch有immediate选项时，立即执行cb方法，即不需要等待属性变化，立刻执行回调。
    if (options.immediate) {
      try {
        cb.call(vm, watcher.value);
      } catch (error) {
        handleError(error, vm, ("callback for immediate watcher \"" + (watcher.expression) + "\""));
      }
    }
    return function unwatchFn () {
      watcher.teardown();
    }
  };
}
```

实例化 watcher 时会执行一次 getter 求值，这时 user watcher 会作为依赖被数据收集；

```javascript
var Watcher = function Watcher() {
  ···
  this.value = this.lazy
      ? undefined
      : this.get();
}

Watcher.prototype.get = function get() {
  ···
  try {
    // getter回调函数，触发依赖收集
    value = this.getter.call(vm, vm);
  } 
}
```

## 派发更新

数据发生改变时， setter 拦截对依赖进行更新，而此前 user watcher 已经被当成依赖收集了，这个时候依赖的更新就是回调函数的执行；