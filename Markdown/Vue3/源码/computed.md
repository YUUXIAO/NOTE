例子：

```javascript
// 响应式数据
const data = reactive({ count: 0 });
// 计算属性
const plusOne = computed(() => data.count + 1);
// 依赖收集
effect(() => console.log(plusOne.value));
// 触发上面的effect重新执行
data.count++;
```

## 简化版源码

> computed 其实也是一个 effect；

```javascript
export function computed(getter) {
  let dirty = true;
  let value: T;

  // 利用了effect做依赖收集
  const runner = effect(getter, {
    // 保证初始化的时候不去执行getter
    lazy: true,
    computed: true,
    scheduler: () => {
      // 在触发更新时 只是把dirty置为true,而不去立刻计算值 所以计算属性有lazy的特性
      dirty = true;
    },
  });
  return {
    get value() {
      if (dirty) {
        // 在真正的去获取计算属性的value的时候,依据dirty的值决定去不去重新执行getter获取最新值
        value = runner();
        dirty = false;
      }
      // 这里是关键
      trackChildRun(runner);
      return value;
    },
    set value(newValue: T) {
      setter(newValue);
    },
  };
}
```

### trackChildRun

```javascript
trackChildRun(runner);

function trackChildRun(childRunner: ReactiveEffect) {
  for (let i = 0; i < childRunner.deps.length; i++) {
    const dep = childRunner.deps[i];
    dep.add(activeEffect);
  }
}
```

