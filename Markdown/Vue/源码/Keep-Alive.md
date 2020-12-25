在不同组件间进行切换时，经常要求保持组件的状态以避免重复渲染组件造成的性能损耗；

## 基本用法

> keep-alive 的使用只需要在动态组件的最外层添加标签即可；

```javascript
<div id="app">
    <button @click="changeTabs('child1')">child1</button>
    <button @click="changeTabs('child2')">child2</button>
    <keep-alive>
        <component :is="chooseTabs">
        </component>
    </keep-alive>
</div>
var child1 = {
    template: '<div><button @click="add">add</button><p>{{num}}</p></div>',
    data() {
        return {
            num: 1
        }
    },
    methods: {
        add() {
            this.num++
        }
    },
}
var child2 = {
    template: '<div>child2</div>'
}
var vm = new Vue({
    el: '#app',
    components: {
        child1,
        child2,
    },
    data() {
        return {
            chooseTabs: 'child1',
        }
    },
    methods: {
        changeTabs(tab) {
            this.chooseTabs = tab;
        }
    }
})
```

## 从模板编译到生成 vnode

> 不管是内置的还是用户自定义组件，本质上组件在模板编译成 render 函数的处理方式是一致的，最终 keep-alive 的 render 函数的结果：

```javascript
with(this){···_c('keep-alive',{attrs:{"include":"child2"}},[_c(chooseTabs,{tag:"component"})],1)}
```

有了 render 函数，接下来从子开始到父会执行生成 Vnode 对象的过程；

_c('keep-alive'···) 的处理，由于 keep-alive 是组件，所以会执行 createComponent 生成组件 Vnode，这个环节和创建普通组件的区别在于：keep-alive 会剔除多余的属性内容，由于 keep-alive 除了 slot 属性之外，其它属性在组件内部没有意义；

```javascript
// 创建子组件Vnode过程
function createComponent(Ctordata,context,children,tag) {
    // abstract是内置组件(抽象组件)的标志
    if (isTrue(Ctor.options.abstract)) {
     // 只保留slot属性，其他标签属性都被移除，在vnode对象上不再存在
        var slot = data.slot;
        data = {};
        if (slot) {
            data.slot = slot;
        }
    }
}
```

## 初次渲染

> keep-alive 不会重复渲染相同的组件，只会利用初次渲染保留的缓存去更新节点；

### 流程

- 和普通组件相同，Vue 会拿到生成的 Vnode 对象执行真实节点创建的过程，patch 执行阶段会调用 createElm 创建真实 dom；
- 在创建节点途中，keep-alive 的 vnode 对象会被认定是一个组件 Vnode，因此针对组件 Vnode 会执行 createComponent 函数，它会对 keep-alive 组件进行初始化和实例化；

```javascript
function createComponent (vnode, insertedVnodeQueue, parentElm, refElm) {
  var i = vnode.data;
  if (isDef(i)) {
    // isReactivated用来判断组件是否缓存。
    var isReactivated = isDef(vnode.componentInstance) && i.keepAlive;
    if (isDef(i = i.hook) && isDef(i = i.init)) {
        // 执行组件初始化的内部钩子 init
      i(vnode, false /* hydrating */);
    }
    if (isDef(vnode.componentInstance)) {
      // 其中一个作用是保留真实dom到vnode中
      initComponent(vnode, insertedVnodeQueue);
      insert(parentElm, vnode.elm, refElm);
      if (isTrue(isReactivated)) {
        reactivateComponent(vnode, insertedVnodeQueue, parentElm, refElm);
      }
      return true
    }
  }
}
```

keep-alive 组件会先调用内部钩子 init 方法进行初始化操作；

```javascript
// 组件内部钩子
var componentVNodeHooks = {
    init: function init (vnode, hydrating) {
      if (
        vnode.componentInstance &&
        !vnode.componentInstance._isDestroyed &&
        vnode.data.keepAlive
      ) {
        // kept-alive components, treat as a patch
        var mountedNode = vnode; // work around flow
        componentVNodeHooks.prepatch(mountedNode, mountedNode);
      } else {
        // 将组件实例赋值给vnode的componentInstance属性
        var child = vnode.componentInstance = createComponentInstanceForVnode(
          vnode,
          activeInstance
        );
        // 执行组件实例的$mount方法进行实例挂载
        child.$mount(hydrating ? vnode.elm : undefined, hydrating);
      }
    },
    prepatch： function() {}
}
```

createComponentInstanceForVnode 方法进行组件实例化并将组件实例赋值给 vnode 的 componentInstance 属性；

```javascript
function createComponentInstanceForVnode (vnode, parent) {
  var options = {
    _isComponent: true,
    _parentVnode: vnode,
    parent: parent
  };
  // 内联模板的处理，忽略这部分代码
  ···
  // 执行vue子组件实例化
  return new vnode.componentOptions.Ctor(options)
}
```

### 内置组件选项

```javascript
// keepalive组件选项
var KeepAlive = {
  name: 'keep-alive',
  // 抽象组件的标志
  abstract: true,
  // keep-alive允许使用的props
  props: {
    include: patternTypes,
    exclude: patternTypes,
    max: [String, Number]
  },

  created: function created () {
    // 缓存组件vnode
    this.cache = Object.create(null);
    // 缓存组件名
    this.keys = [];
  },

  destroyed: function destroyed () {
    for (var key in this.cache) {
      pruneCacheEntry(this.cache, key, this.keys);
    }
  },

  mounted: function mounted () {
    var this$1 = this;
    // 动态include和exclude
    // 对include exclue的监听
    this.$watch('include', function (val) {
      pruneCache(this$1, function (name) { return matches(val, name); });
    });
    this.$watch('exclude', function (val) {
      pruneCache(this$1, function (name) { return !matches(val, name); });
    });
  },
  // keep-alive的渲染函数
  render: function render () {
    // 拿到keep-alive下插槽的值
    var slot = this.$slots.default;
    // 第一个vnode节点
    var vnode = getFirstComponentChild(slot);
    // 拿到第一个组件实例
    var componentOptions = vnode && vnode.componentOptions;
    // keep-alive的第一个子组件实例存在
    if (componentOptions) {
      // check pattern
      //拿到第一个vnode节点的name
      var name = getComponentName(componentOptions);
      var ref = this;
      var include = ref.include;
      var exclude = ref.exclude;
      // 通过判断子组件是否满足缓存匹配
      if (
        // not included
        (include && (!name || !matches(include, name))) ||
        // excluded
        (exclude && name && matches(exclude, name))
      ) {
        return vnode
      }

      var ref$1 = this;
      var cache = ref$1.cache;
      var keys = ref$1.keys;
      var key = vnode.key == null
        ? componentOptions.Ctor.cid + (componentOptions.tag ? ("::" + (componentOptions.tag)) : '')
        : vnode.key;
        // 再次命中缓存
      if (cache[key]) {
        vnode.componentInstance = cache[key].componentInstance;
        // make current key freshest
        remove(keys, key);
        keys.push(key);
      } else {
      // 初次渲染时，将vnode缓存
        cache[key] = vnode;
        keys.push(key);
        // prune oldest entry
        if (this.max && keys.length > parseInt(this.max)) {
          pruneCacheEntry(cache, keys[0], keys, this._vnode);
        }
      }
      // 为缓存组件打上标志
      vnode.data.keepAlive = true;
    }
    // 将渲染的vnode返回
    return vnode || (slot && slot[0])
  }
};
```

