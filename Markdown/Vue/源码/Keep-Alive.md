1. keep-alive 是源码内部定义的组件选项配置，它会先注册为全局组件供开发者全局使用，其中 render 函数定义了它的渲染过程；
2. 和普通组件一致，当父在创建真实节点的过程中，遇到 keep-alive 的组件会进行组件的初始化和实例化；
3. 实例化会执行挂载 $mount 的过程，这一步会执行 keep-alive 选项中的 render 函数；
4. render 函数在初始渲染时，会将渲染的子 Vnode 进行缓存，同时对应的子真实节点也会被缓存起来；

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
    mounted() {
        console.log('child1 mounted')
    },
    activated() {
        console.log('child1 activated')
    },
    deactivated() {
        console.log('child1 deactivated')
    },
    destoryed() {
        console.log('child1 destoryed')
    }
}
var child2 = {
    template: '<div>child2</div>',
    mounted() {
        console.log('child2 mounted')
    },
    activated() {
        console.log('child2 activated')
    },
    deactivated() {
        console.log('child2 deactivated')
    },
    destoryed() {
        console.log('child2 destoryed')
    }
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

> 内置的 keep-alive 组件，让子组件在第一次渲染时将 vnode 和真实的 elm 进行了缓存；

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

- keep-alive 选项和普通组件的选项基本类似，不同的是 keep-alive 组件没有用 template 而是使用 render 函数；
- keep-alive 本质上是存缓存和拿缓存的过程，并没有实际的节点渲染，所以使用 render 处理；

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

### 缓存 vnode

keep-alive 在执行组件实例化之后会进行组件的挂载；

#### getFirstComponentChild 函数

> getFirstComponentChild 用于获取 keep-alive 下插槽的内容，也就是需要渲染的子组件；

```javascript
function getFirstComponentChild (children) {
  if (Array.isArray(children)) {
    for (var i = 0; i < children.length; i++) {
      var c = children[i];
      // 组件实例存在，则返回，理论上返回第一个组件vnode
      if (isDef(c) && (isDef(c.componentOptions) || isAsyncPlaceholder(c))) {
        return c
      }
    }
  }
}
```

#### 判断组件满足缓存的匹配条件

> 如果组件不满足缓存的要求，则直接返回组件的 vnode，不做任何处理，此时组件会进入正常的挂载环节；

在 keep-alive 组件的使用过程中，Vue 源码允许使用 exclude、include 来定义匹配组件，include  规定发只有名称匹配的组件才被缓存，exclude 规定了任何名称匹配的组件都不会被缓存；

还可以使用 max 来限制可以缓存多少匹配实例；

- 匹配的规则允许使用数组、字符串、正则的形式；

```javascript
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

// matches
function matches (pattern, name) {
  // 允许使用数组['child1', 'child2']
  if (Array.isArray(pattern)) {
      return pattern.indexOf(name) > -1
  } else if (typeof pattern === 'string') {
      // 允许使用字符串 child1,child2
      return pattern.split(',').indexOf(name) > -1
  } else if (isRegExp(pattern)) {
      // 允许使用正则 /^child{1,2}$/g
      return pattern.test(name)
  }
  /* istanbul ignore next */
  return false
}
```

#### 缓存 vnode

由于是第一次执行 render 函数，选项中的 cache 和 keys 数据都没有值，其中 cache 是一个空对象，它用来缓存 ｛name：vnode｝枚举，keys 用来缓存组件名；

在第一次渲染 keep-alive 时，会将需要渲染的子组件 vnode 进行缓存；

```javascript
cache[key] = vnode;
keys.push(key);
```

#### 打标记

将已经缓存的 vnode 打上标记，并将子组件的 Vnode 返回；

```javascript
vnode.data.keepAlive = true;
```

### 真实节点的保存

createComponent 会先执行 keep-alive 组件的初始化流程，也包括了子组件的挂载，通过 componentInstance 拿到了 keep-alive 组件的实例，接下来就是将真实的 dom 保存在 vnode 中；

```javascript
function createComponent(vnode, insertedVnodeQueue) {
  ···
  if (isDef(vnode.componentInstance)) {
      // 其中一个作用是保留真实dom到vnode中
      initComponent(vnode, insertedVnodeQueue);
      // 将真实节点添加到父节点中
      insert(parentElm, vnode.elm, refElm);
      if (isTrue(isReactivated)) {
          reactivateComponent(vnode, insertedVnodeQueue, parentElm, refElm);
      }
      return true
  }
}

function initComponent() {
  ···
  // vnode保留真实节点
  vnode.elm = vnode.componentInstance.$el;
  ···
}
```

- keep-alive 需要一个 max 来限制缓存组件的数量：原因是 keep-alive 缓存的组件数据除了包括 vnode 这一描述对象外，还保留着真实的 dom 节点，真实节点对象是庞大的，所以大量保存缓存组件是耗费性能的，因此需要严格控制缓存组件数量；

## 再次渲染

### 重新渲染组件

当数据发生改变时，收集过的依赖会进行派发更新操作；

1. 父组件中负责实例挂载的过程作为依赖会被执行，即执行父组件的 vm._ update(vm._render(), hydrating)；
2. _ render 的 _ update 分别代表两个过程，_ render 函数会根据数据的变化为组件生成新的 Vnode 节点，而 _ update 会为新的 Vnode 生成真实的节点；
3. 在生成真实节点的过程中，会利用 vitrual dom 的 diff 算法对前后 vnode 节点进行对比，使之尽可能少的更改真实节点；

patch 是新旧 Vnode 节点对比的过程，而 patchVnode 是其核心：

```javascript
function patchVnode (oldVnode,vnode,insertedVnodeQueue,ownerArray,index,removeOnly) {
  ···
  // 新vnode  执行prepatch钩子
  if (isDef(data) && isDef(i = data.hook) && isDef(i = i.prepatch)) {
      i(oldVnode, vnode);
  }
  ···
}
```

执行 prepatch 钩子时会拿到新旧组件的实例并执行 upadteChildComponent 函数，而 upadteChildComponent 会针对新的组件实例对旧实例进行状态的更新，包括 props，listeners 等，最终会调用 vue 全局提供的 vm.$forceUpdate( ) 方法进行实例的重新渲染；

```javascript
var componentVNodeHooks = {
    // 之前分析的init钩子 
    init: function() {},
    prepatch: function prepatch (oldVnode, vnode) {
      // 新组件实例
      var options = vnode.componentOptions;
      // 旧组件实例
      var child = vnode.componentInstance = oldVnode.componentInstance;
      updateChildComponent(
        child,
        options.propsData, // updated props
        options.listeners, // updated listeners
        vnode, // new parent vnode
        options.children // new children
      );
    },
}

function updateChildComponent() {
    // 更新旧的状态，不分析这个过程
    ···
    // 迫使实例重新渲染。
    vm.$forceUpdate();
}
```

$forceUpdate() 是源码对外暴露的一个 api，强制使 Vue 实例重新渲染，本质上是执行实例所收集的依赖；

```javascript
Vue.prototype.$forceUpdate = function () {
  var vm = this;
  if (vm._watcher) {
    vm._watcher.update();
  }
};
```

### 重用缓存组件

由于 vm.$forceUpdate 会强制 keep-alive 组件重新渲染，因此 keep-alive 组件会再一次执行 render 过程，由于第一次对 vnode 的缓存，keep-alive 在实例的 cache 对象中找到了缓存的组件；

- cache 对象中存储了再次使用的 vnode 对象，所以直接通过 cache[key] 取出缓存的组件实例并赋值给 vnode 的 compoentInstance 属性；

```javascript
// keepalive组件选项
var keepAlive = {
	.....
    render: function render () {
      	.....
        if (cache[key]) {
          // 直接取出缓存组件
          vnode.componentInstance = cache[key].componentInstance;
          // keys命中的组件名移到数组末端
          remove(keys, key);
          keys.push(key);
        } else {
          // 初次渲染时，将vnode缓存
         ....
        }
        vnode.data.keepAlive = true;
      }
      return vnode || (slot && slot[0])
    }
}
```

### 真实节点的替换

_ update 过程会调用 createComponent 递归创建子组件 vnode；

- 此时的 vnode 是缓存取出的子组件 vnode，由于在第一次渲染时对组件进行了标记 vnode.data.keepAlive = true，所以 isReactivated 的值为 true；

```javascript
function createComponent (vnode, insertedVnodeQueue, parentElm, refElm) {
  // vnode为缓存的vnode
  var i = vnode.data;
  if (isDef(i)) {
    // 此时isReactivated为true
    var isReactivated = isDef(vnode.componentInstance) && i.keepAlive;
    if (isDef(i = i.hook) && isDef(i = i.init)) {
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

i.init 依旧会执行子组件的初始化过程，但是由于有缓存所以执行过程也不完全相同；

- 由于有 keepAlive 的标志，所以子组件不再走挂载流程，只是执行 prepatch 钩子对组件状态进行更新，并且很好的利用了缓存 vnode 之前保留的真实节点进行节点的替换；

```javascript
var componentVNodeHooks = {
  init: function init (vnode, hydrating) {
    if (
      vnode.componentInstance &&
      !vnode.componentInstance._isDestroyed &&
      vnode.data.keepAlive
    ) {
      // 当有keepAlive标志时，执行prepatch钩子
      var mountedNode = vnode; // work around flow
      componentVNodeHooks.prepatch(mountedNode, mountedNode);
    } else {
      var child = vnode.componentInstance = createComponentInstanceForVnode(
        vnode,
        activeInstance
      );
      child.$mount(hydrating ? vnode.elm : undefined, hydrating);
    }
  },
}
```

## 生命周期

从 child1 切换到 child2，再切回 child1 时；

- child1 不会再执行 mounted 钩子，只执行 activated 钩子；
- child2 不会执行 destoryed 钩子，只会执行 deactivated 钩子； 
- child2 的 deactivated 钩子比 child1 的 activated 钩子提前执行；

### deactivated

> patch 过程会对新旧节点的改变进行对比，从而尽可能范围小的去操作真实节点，当完成 diff 算法并对节点进行操作完毕时，接下来的步骤是对旧组件执行销毁移除操作；

当 child1 切换到 child2 时，child1 会执行 deactivated 钩子而不是 destoryed 钩子；

```javascript
function patch(···) {
  // 分析过的patchVnode过程
  // 销毁旧节点
  if (isDef(parentElm)) {
    removeVnodes(parentElm, [oldVnode], 0, 0);
  } else if (isDef(oldVnode.tag)) {
    invokeDestroyHook(oldVnode);
  }
}

function removeVnodes (parentElm, vnodes, startIdx, endIdx) {
  // startIdx,endIdx都为0
  for (; startIdx <= endIdx; ++startIdx) {
    // ch 会拿到需要销毁的组件
    var ch = vnodes[startIdx];
    if (isDef(ch)) {
      if (isDef(ch.tag)) {
        // 真实节点的移除操作
        removeAndInvokeRemoveHook(ch);
        invokeDestroyHook(ch);
      } else { // Text node
        removeNode(ch.elm);
      }
    }
  }
}
```

- removeAndInvokeRemoveHook 会对旧的节点进行移除操作，会将真实节点从父元素中删除；
- invokeDestroyHook 是执行销毁组件钩子的核心，如果组件下存在子组件，会递归调用 invokeDestroyHook 执行销毁操作，销毁过程会执行组件内部的 destory 钩子；

```javascript
function invokeDestroyHook (vnode) {
  var i, j;
  var data = vnode.data;
  if (isDef(data)) {
    if (isDef(i = data.hook) && isDef(i = i.destroy)) { i(vnode); }
    // 执行组件内部destroy钩子
    for (i = 0; i < cbs.destroy.length; ++i) { cbs.destroy[i](vnode); }
  }
  // 如果组件存在子组件，则遍历子组件去递归调用invokeDestoryHook执行钩子
  if (isDef(i = vnode.children)) {
    for (j = 0; j < vnode.children.length; ++j) {
      invokeDestroyHook(vnode.children[j]);
    }
  }
}
```

destory 钩子的逻辑：

- 当组件是 keep-alive 缓存过的组件，即已经用 keepAlive 标记过，则不会执行实例的销毁；
- $destory 过程会做一系列的组件销毁操作，其中 beforeDestory，destory 钩子也是在此过程中调用；

```javascript
var componentVNodeHooks = {
  destroy: function destroy (vnode) {
    // 组件实例
    var componentInstance = vnode.componentInstance;
    // 如果实例还未被销毁
    if (!componentInstance._isDestroyed) {
      // 不是keep-alive组件则执行销毁操作
      if (!vnode.data.keepAlive) {
        componentInstance.$destroy();
      } else {
        // 如果是已经缓存的组件
        deactivateChildComponent(componentInstance, true /* direct */);
      }
    }
  }
}
```

deactivateChildComponent 的处理过程；

- _directInactive 是用来标记这个被打上停用标签的组件是否是最顶层的组件；
- _inactive 是停用的标志，同样子组件也需要递归调用 deactivateChildComponent ，打上停用的标记；
- 最终会执行用户定义的 deactivated 钩子；

```javascript
function deactivateChildComponent (vm, direct) {
  if (direct) {
    vm._directInactive = true;
    if (isInInactiveTree(vm)) {
      return
    }
  }
  if (!vm._inactive) {
    // 已经被停用
    vm._inactive = true;
    // 对子组件同样会执行停用处理
    for (var i = 0; i < vm.$children.length; i++) {
      deactivateChildComponent(vm.$children[i]);
    }
    // 最终调用deactivated钩子
    callHook(vm, 'deactivated');
  }
}
```

### activated

同样是 patch 过程，在对旧节点移除并执行销毁或者停用的钩子后，对新的节点也会执行相应的钩子，这也是停用的钩子比启用的钩子先执行的原因；

```javascript
function patch(···) {
  // 销毁旧节点
  {
    if (isDef(parentElm)) {
      removeVnodes(parentElm, [oldVnode], 0, 0);
    } else if (isDef(oldVnode.tag)) {
      invokeDestroyHook(oldVnode);
    }
  }
  // 执行组件内部的insert钩子
  invokeInsertHook(vnode, insertedVnodeQueue, isInitialPatch);
}

function invokeInsertHook (vnode, queue, initial) {
  // 当节点已经被插入时，会延迟执行insert钩子
  if (isTrue(initial) && isDef(vnode.parent)) {
    vnode.parent.data.pendingInsert = queue;
  } else {
    for (var i = 0; i < queue.length; ++i) {
      queue[i].data.hook.insert(queue[i]);
    }
  }
}
```

组件内部的 insert 钩子逻辑：

- 当第一次实例化组件时，由于实例的 _ isMounted 不存在，所以会调用 mounted 钩子；
- 当再次切为 child1 时，由于 child1 只是停用而没有被销毁，不会再调用 mounted 钩子，此时会执行 queueActivatedComponent activateChildComponent 函数对组件进行状态处理；

```javascript
// 组件内部自带钩子
var componentVNodeHooks = {
  insert: function insert (vnode) {
    var context = vnode.context;
    var componentInstance = vnode.componentInstance;
    // 实例已经被挂载
    if (!componentInstance._isMounted) {
      componentInstance._isMounted = true;
      callHook(componentInstance, 'mounted');
    }
    if (vnode.data.keepAlive) {
      if (context._isMounted) {
        queueActivatedComponent(componentInstance);
      } else {
        activateChildComponent(componentInstance, true /* direct */);
      }
    }
  },
}
```

activateChildComponent 将 _inactive 标记为已启用，并且对子组件递归调用 activateChildComponent 做状态处理；

```javascript
function activateChildComponent (vm, direct) {
  if (direct) {
    vm._directInactive = false;
    if (isInInactiveTree(vm)) {
      return
    }
  } else if (vm._directInactive) {
    return
  }
  if (vm._inactive || vm._inactive === null) {
    vm._inactive = false;
    for (var i = 0; i < vm.$children.length; i++) {
      activateChildComponent(vm.$children[i]);
    }
    callHook(vm, 'activated');
  }
}
```

## 缓存优化

### FIFO

先进先出策略，通过记录数据使用的时间，当缓存大小即将溢出时，优先清除离当前时间最远的数据；

### LRU

最近最少使用，LRU 策略遵循的是，如果数据最近被访问过，那么将来被访问的几率会更高，如果以一个数组云记录数据，当有一数据被访问时，该数据会被移动到数组的末尾，表明最近被使用过，当缓存溢出时，会删除数组的头部数据，即将最不频繁使用的数据移除；

### LFU

计数最少策略，用次数云标记数据使用频率，次数最少的会在缓存溢出时被淘汰；

keep-alive 在处理缓存时的利用了 LRU 的缓存策略；

```javascript
var keepAlive = {
  render: function() {
    ···
    if (cache[key]) {
      vnode.componentInstance = cache[key].componentInstance;
      remove(keys, key);
      keys.push(key);
    } else {
      cache[key] = vnode;
      keys.push(key);
      if (this.max && keys.length > parseInt(this.max)) {
        pruneCacheEntry(cache, keys[0], keys, this._vnode);
      }
    }
  }
}

function remove (arr, item) {
  if (arr.length) {
    var index = arr.indexOf(item);
    if (index > -1) {
      return arr.splice(index, 1)
    }
  }
}
```

删除缓存时会把 keys[0] 代表的组件删除，因为最近被访问到的元素会位于数组的尾部，所以头部的数据是最远访问的，会优先删除头部的元素；

并且会再次调用 remove 方法，将 keys 的首个元素删除；

```javascript
function pruneCacheEntry (cache,key,keys,current) {
  var cached$$1 = cache[key];
  // 销毁实例
  if (cached$$1 && (!current || cached$$1.tag !== current.tag)) {
    cached$$1.componentInstance.$destroy();
  }
  cache[key] = null;
  remove(keys, key);
}
```

## 抽象组件

> Vue 提供的内置组件都有一个描述组件类型的选项 ｛astract：true｝，它表明了该组件是抽象组件；

1. 抽象组件没有真实的节点，它在组件渲染阶段不会去解析渲染成真实的 dom 节点，而是作为中间数据过渡层处理，在 keep-alive 是对组件缓存的处理；
2. 在组件初始化的时候父子组件会显式的建立一层关系，这层关系奠定了父子组件之间通信的基础；

```javascript
Vue.prototype._init = function() {
    ···
    var vm = this;
    initLifecycle(vm)
}

function initLifecycle (vm) {
  var options = vm.$options;

  var parent = options.parent;
  if (parent && !options.abstract) {
      // 如果有abstract属性，一直往上层寻找，直到不是抽象组件
    while (parent.$options.abstract && parent.$parent) {
      parent = parent.$parent;
    }
    parent.$children.push(vm);
  }
  ···
}
```

在子组件注册阶段会父实例挂载到自身选项上的 parent 属性上，在 initLifecycle 过程中，会反向拿到 parent 上的父组件 vnode，并为其 $children 属性添加子组件 vnode，如果在反向查找父组件的过程中，父组件拥有 abstract 属性，即可判定该组件为抽象组件，此时利用 parent 的链条向上寻找，直到组件不是抽象组件为止；

initLifecycle 的处理，让每个组件都能找到上层的父组件以及下层的子组件，使得组件之前形成一个关系树；

