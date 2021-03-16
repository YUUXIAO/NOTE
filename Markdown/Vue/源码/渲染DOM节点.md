## Virtual DOM

> Virtual Dom 只是用来映射到真实 DOM 的渲染，可以将多个改动合并成一个批量的操作，从而减少 DOM 的重排次数，缩短了生成渲染树和绘制所花的时间；

## Vnode

> Virtual Dom 是用 Vnode 这个构造函数去描述一个 DOM 节点；

### Vnode构造函数

```javascript
var VNode = function VNode (tag,data,children,text,elm,context,componentOptions,asyncFactory) {
    this.tag = tag; // 标签
    this.data = data;  // 数据
    this.children = children; // 子节点
    this.text = text;
    ···
    ···
  };

```

### 创建Vnode注释节点

```javascript
// 创建注释vnode节点
var createEmptyVNode = function (text) {
    if ( text === void 0 ) text = '';

    var node = new VNode();
    node.text = text;
    node.isComment = true; // 标记注释节点
    return node
};
```

### 创建Vnode文本节点

```javascript
// 创建文本vnode节点
function createTextVNode (val) {
    return new VNode(undefined, undefined, undefined, String(val))
}
```

### 克隆vnode

cloneVnode 对 Vnode 的克隆是一层浅拷贝，不会对子节点进行深度克隆；

```javascript
function cloneVNode (vnode) {
  var cloned = new VNode(
    vnode.tag,
    vnode.data,
    vnode.children && vnode.children.slice(),
    vnode.text,
    vnode.elm,
    vnode.context,
    vnode.componentOptions,
    vnode.asyncFactory
  );
  cloned.ns = vnode.ns;
  cloned.isStatic = vnode.isStatic;
  cloned.key = vnode.key;
  cloned.isComment = vnode.isComment;
  cloned.fnContext = vnode.fnContext;
  cloned.fnOptions = vnode.fnOptions;
  cloned.fnScopeId = vnode.fnScopeId;
  cloned.asyncMeta = vnode.asyncMeta;
  cloned.isCloned = true;
  return cloned
}
```

## Virtual DOM的创建

### 挂载流程

1. 调用 Vue 实例上 $mount 方法，这个方法的核心是 mountComponent 函数；
2. 如果传递的是 template 模板，模板会先经过编译器的解析，并最终根据不同的平台生成对应的代码，此时对应的是用 with 语句封装好的 render 函数；
3. 如果传递的是 render 函数，则跳过模板编译过程，直接进入下一阶段；
4. 拿到 render 函数，调用 vm._render 方法将 render 函数转化为 Virtual DOM ,最终通过 vm._update 方法将 Virtual DOM 渲染为真实的 DOM 节点；

```javascript
Vue.prototype.$mount = function(el, hydrating) {
    ···
    return mountComponent(this, el)
}
function mountComponent() {
    ···
    updateComponent = function () {
        vm._update(vm._render(), hydrating);
    };
}
```

### vm._render函数

> vm._render 方法是将 render 函数转化为Virtual DOM 的；

Vue 在代码引入时有一个 renderMixin 过程，它是定义跟渲染有关的函数；

- _render 函数的核心是 ：

  ```
  vnode = render.call(vm._renderProxy, vm.$createElement);
  ```

- vm._renderProxy 函数的本质是为了做数据过滤检测，它也绑定了 render 函数执行的 this 指向；

- vm.$createElement 函数会作为 render 函数的参数传入；

```javascript
// 引入Vue时，执行renderMixin方法，该方法定义了Vue原型上的几个方法，其中一个便是 _render函数
renderMixin();//
function renderMixin() {
    Vue.prototype._render = function() {
        var ref = vm.$options;
        var render = ref.render;
        ···
        try {
            vnode = render.call(vm._renderProxy, vm.$createElement);
        } catch (e) {
            ···
        }
        ···
        return vnode
    }
}
```

### vm.$createElement 方法

初始化 _init 时，有一个 initRender 函数，是用来定义渲染函数方法的：

- vm. _c 方法是 template 内部编译成 render 函数时调用的方法；
- vm. $createElement 是手写 render 函数时调用的方法；

两者的唯一区别是最后一个参数的不同，通过模板生成的 render 方法可以子节点都是 Vnode，而手写的 render 需要一些检验和转换；

```javascript
function initRender(vm) {
    vm._c = function(a, b, c, d) { return createElement(vm, a, b, c, d, false); }
    vm.$createElement = function (a, b, c, d) { return createElement(vm, a, b, c, d, true); };
}
```

createElement 方法实际上是对 _createElement 方法的封装，在调用 

```javascript
// 没有data
new Vue({
    el: '#app',
    render: function(createElement) {
        return createElement('div', this.message)
    },
    data() {
        return {
            message: 'dom'
        }
    }
})
// 有data
new Vue({
    el: '#app',
    render: function(createElement) {
        return createElement('div', {}, this.message)
    },
    data() {
        return {
            message: 'dom'
        }
    }
})
```

如果第二个参数是变量或者数组，则默认是没有传递 data，因为 data 一般是对象形式存在；

```javascript
function createElement (
  context, // vm 实例
  tag, // 标签
  data, // 节点相关数据，属性
  children, // 子节点
  normalizationType,
  alwaysNormalize // 区分内部编译生成的render还是手写render
) {
  // 对传入参数做处理，如果没有data，则将第三个参数作为第四个参数使用，往上类推。
  if (Array.isArray(data) || isPrimitive(data)) {
    normalizationType = children;
    children = data;
    data = undefined;
  }
  // 根据是alwaysNormalize 区分是内部编译使用的，还是用户手写render使用的
  if (isTrue(alwaysNormalize)) {
    normalizationType = ALWAYS_NORMALIZE;
  }
  return _createElement(context, tag, data, children, normalizationType) // 真正生成Vnode的方法
}
```

### 数据规范检测

_createElement 在创建Vnode 前，会先数据的规范性进行检测，将不合法的数据类型错误提前暴露给用户；

```javascript
function _createElement (context,tag,data,children,normalizationType) {
  // 1. 数据对象不能是定义在Vue data属性中的响应式数据。
  if (isDef(data) && isDef((data).__ob__)) {
    warn(
      "Avoid using observed data object as vnode data: " + (JSON.stringify(data)) + "\n" +
      'Always create fresh vnode data objects in each render!',
      context
    );
    return createEmptyVNode() // 返回注释节点
  }
  if (isDef(data) && isDef(data.is)) {
    tag = data.is;
  }
  if (!tag) {
    // 防止动态组件 :is 属性设置为false时，需要做特殊处理
    return createEmptyVNode()
  }
  // 2. key值只能为string，number这些原始数据类型
  if (isDef(data) && isDef(data.key) && !isPrimitive(data.key)
  ) {
    {
      warn(
        'Avoid using non-primitive value as key, ' +
        'use string/number value instead.',
        context
      );
    }
  }
  ···
}
```

- 用响应式对象做 data 属性；

```javascript
new Vue({
    el: '#app',
    render: function (createElement, context) {
       return createElement('div', this.observeData, this.show)
    },
    data() {
        return {
            show: 'dom',
            observeData: {
                attr: {
                    id: 'test'
                }
            }
        }
    }
})
```

- 当特殊属性key的值为非字符串，非数字类型时；

```javascript
new Vue({
    el: '#app',
    render: function(createElement) {
        return createElement('div', { key: this.lists }, this.lists.map(l => {
           return createElement('span', l.name)
        }))
    },
    data() {
        return {
            lists: [{
              name: '111'
            },
            {
              name: '222'
            }
          ],
        }
    }
})
```

### 子节点children规范化

Virtual DOM tree 是由每个 Vnode 以树形状拼成的虚拟 DOM树，因此需要保证每一个子节点都是 Vnode 类型；

- 模板编译 render 函数，通常 template 模板通过编译生成的 render 函数都是 Vnode 类型，但是函数式组件返回的是一个数组，这时 Vue 的处理是将整个 children 拍平成一堆数组；
- 用户定义的 render 函数：
  1. 当 children 为文本节点时，这时 createTextVNode 创建一个文本节点 Vnode；
  2. 当 children 中有 v-for 时会出现嵌套数组，这时的处理逻辑是遍历 children ，对每个节点判断，如果依旧是数组则继续递归调用，直到类型为基础类型时，调用 createTextVNode 方法转化为 Vnode，
  3. 这样递归，children 也变成了一个类型为 VNode 的数组；

```javascript
function _createElement() {
  ···
  if (normalizationType === ALWAYS_NORMALIZE) {
    // 用户定义render函数
    children = normalizeChildren(children);
  } else if (normalizationType === SIMPLE_NORMALIZE) {
    // 模板编译生成的的render函数
    children = simpleNormalizeChildren(children);
  }
}

// 处理编译生成的render 函数
function simpleNormalizeChildren (children) {
  for (var i = 0; i < children.length; i++) {
      // 子节点为数组时，进行开平操作，压成一维数组。
      if (Array.isArray(children[i])) {
      return Array.prototype.concat.apply([], children)
      }
  }
  return children
}

// 处理用户定义的render函数
function normalizeChildren (children) {
  // 递归调用，直到子节点是基础类型，则调用创建文本节点Vnode
  return isPrimitive(children)
    ? [createTextVNode(children)]
    : Array.isArray(children)
      ? normalizeArrayChildren(children)
      : undefined
}

// 判断是否基础类型
function isPrimitive (value) {
  return (
    typeof value === 'string' ||
    typeof value === 'number' ||
    typeof value === 'symbol' ||
    typeof value === 'boolean'
  )
}
```

### 举例

用一个实际的例子，描述 render 函数到 Virtual DOM 的分析；

- template 模板形式

```javascript
var vm = new Vue({
  el: '#app',
  template: '<div><span>virtual dom</span></div>'
})
```

- 模板编译生成 render 函数

```javascript
(function() {
  with(this){
    return _c('div',[_c('span',[_v("virual dom")])])
  }
})
```

- Virtual DOM tree 的结果（省略版）

```javascript
{
  tag: 'div',
  children: [{
    tag: 'span',
    children: [{
      tag: undefined,
      text: 'virtual dom'
    }]
  }]
}
```

## 虚拟Vnode映射成真实DOM

updateComponent 的最后一个过程，虚拟 DOM 树在生成 Virtual DOM 后，会调用 Vue 原型上 _update 方法：将虚拟 DOM 映射成真实的 DOM ；

_update 的调用时机有两个：一个是发生在初次渲染阶段；另一个是发生数据更新阶段；

```javascript
updateComponent = function () {
  // render生成虚拟DOM，update渲染真实DOM
  vm._update(vm._render(), hydrating);
};
```

### vm._update函数

vm._update 方法定义在 lifecycleMixin 中；

```javascript
lifecycleMixin()
function lifecycleMixin() {
  Vue.prototype._update = function (vnode, hydrating) {
    var vm = this;
    var prevEl = vm.$el;
    var prevVnode = vm._vnode; // prevVnode为旧vnode节点
    // 通过是否有旧节点判断是初次渲染还是数据更新
    if (!prevVnode) {
        // 初次渲染
        vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false)
    } else {
        // 数据更新
        vm.$el = vm.__patch__(prevVnode, vnode);
    }
}
```

### _ patch_方法

_update 的核心是 _patch _ 方法，如果是服务端渲染，由于没有 DOM，_patch _ 方法是一个空函数；在浏览器环境下，_patch _  是 patch 函数的引用；

```javascript
// 浏览器端才有DOM，服务端没有dom，所以patch为一个空函数
Vue.prototype.__patch__ = inBrowser ? patch : noop;
```

patch 方法是 createPatchFunction 方法的返回值，createPatchFunction  方法传递一个对象做为参数，对象拥有两个属性：nodeOps 和 modules ；

1. nodeOps 封装了一系列操作原生 DOM 对象的方法；
2. modules 定义了模块的钩子函数；

- createPatchFunction 函数内部会调用封装好的 DOM api，根据 Virtual DOM 结果去生成真实的节点，其中如果遇到组件 Vnode，会递归调用子组件的挂载过程；

```javascript
var patch = createPatchFunction({ nodeOps: nodeOps, modules: modules });

// 将操作dom对象的方法合集做冻结操作
var nodeOps = /*#__PURE__*/Object.freeze({
  createElement: createElement$1,
  createElementNS: createElementNS,
  createTextNode: createTextNode,
  createComment: createComment,
  insertBefore: insertBefore,
  removeChild: removeChild,
  appendChild: appendChild,
  parentNode: parentNode,
  nextSibling: nextSibling,
  tagName: tagName,
  setTextContent: setTextContent,
  setStyleScope: setStyleScope
});

// 定义了模块的钩子函数
var platformModules = [
  attrs,
  klass,
  events,
  domProps,
  style,
  transition
];

var modules = platformModules.concat(baseModules);
```

