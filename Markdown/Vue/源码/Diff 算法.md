1. 先同级比较再比较子节点；
2. 判断一方有子节点和一方没有子节点的情况：如果新节点有子节点旧节点没有，就用新节点替换旧节点；否则就把旧节点删除；
3. 都有子节点的情况：先通过判断新旧节点的 key、tag、isComment、data 同时定义或不定义以及标签类型为 input 时 type 是否相同来确定两个节点是否为相同的节点，如果不相同就将新节点替换旧节点；
4. 如果是相同节点就进入 patchVnode 阶段：这个阶段采用双指针的算法，同时从新旧节点的两端进行比较，在这个过程中，会用到模板编译时的静态标记配合 key 来跳过对比静态节点；

## 创建基础类

封装基本类型判断方法

```javascript
class Util {
  constructor() {}
  // 检测基础类型
  _isPrimitive(value) {
    return (typeof value === 'string' || typeof value === 'number' || typeof value === 'symbol' || typeof value === 'boolean')
  }
  // 判断值不为空
  _isDef(v) {
    return v !== undefined && v !== null
  }
}
// 工具类的使用
const util = new Util()
```

## 创建Vnode

> 本质上是用一个对象去描述一个真实的 DOM 元素；

元素的 tag 标签，元素的属性集合 data,元素的子节点 children ，text 为元素的文本节点；

```javascript
class VNode {
  constructor(tag, data, children) {
    this.tag = tag;
    this.data = data;
    this.children = children;
    this.elm = ''
    // text属性用于标志Vnode节点没有其他子节点，只有纯文本
    this.text = util._isPrimitive(this.children) ? this.children : ''
  }
}
```

## 模拟渲染过程

创建一个类模拟将 render 函数转换为 Vnode，并将 Vnode 渲染为真实 DOM 的过程，定义这个类为 Vn；

Vn 有两个基本方法 createVnode，createElement，分别是创建虚拟 Vnode 和创建真实 Dom 的过程；

### createVnode

> createVnode 模拟 Vue 中 render 函数的实现思路，目的是将数据转换为虚拟的 Vnode；

```javascript
// index.html

<script src="diff.js">
<script>

// 创建Vnode

let createVnode = function() {
  let _c = vn.createVnode;
  return _c('div', { attrs: { id: 'test' } }, arr.map(a => _c(a.tag, {}, a.text)))
}

// 元素内容结构
let arr = 
  [{
    tag: 'i',
    text: 2
  }, {
    tag: 'span',
    text: 3
  }, {
    tag: 'strong',
    text: 4
  }]
</script>

// diff.js
(function(global) {
  class Vn {
    constructor() {}
    // 创建虚拟Vnode
    createVnode(tag, data, children) {
      return new VNode(tag, data, children)
    }
  }
  global.vn = new Vn()
}(this))
```

### createElement

> 渲染真实 DOM 的过程就是遍历 Vnode 对象，递归创建真实节点的过程；

```javascript
class Vn {
  createElement(vnode, options) {
      let el = options.el;
      if(!el || !document.querySelector(el)) return console.error('无法找到根节点')
      let _createElement = vnode => {
        const { tag, data, children } = vnode;
        const ele = document.createElement(tag);
        // 添加属性
        this.setAttr(ele, data);
        // 简单的文本节点，只要创建文本节点即可
        if (util._isPrimitive(children)) {
          const testEle = document.createTextNode(children);
          ele.appendChild(testEle)
        } else {
        // 复杂的子节点需要遍历子节点递归创建节点。
          children.map(c => ele.appendChild(_createElement(c)))
        }
        return ele
      }
      document.querySelector(el).appendChild(_createElement(vnode))
    }
}
```

### setAttr

> setAttr 是为节点设置属性的方法，利用 DOM 原生的 setAttribute 为每个节点设置属性值；

```javascript
class Vn {
  setAttr(el, data) {
    if (!el) return
    const attrs = data.attrs;
    if (!attrs) return;
    Object.keys(attrs).forEach(a => {
      el.setAttribute(a, attrs[a]);
    })
  }
}
```

## diff算法实现

> diff 算法将需要修改的数据进行比较并只渲染必要的 DOM；

- diff 算法的核心在于尽可能小变动的前提下找到需要更新的节点，直接调用原生相关 DOM 方法修改视图；
- 算法比较不同节点时，只会进行同层节点的比较，不会跨层进行比较，大大减少了算法复杂度；

### diffVnode

> diffVnode 的逻辑，会对比新旧节点的不同，并完成视图渲染更新；

```javascript
class Vn {
  ···
  diffVnode(nVnode, oVnode) {
    if (!this._sameVnode(nVnode, oVnode)) {
      // 直接更新根节点及所有子节点
      return ***
    }
    this.generateElm(vonde);
    this.patchVnode(nVnode, oVnode);
  }
}
```

### _sameVnode

> 假定节点相同的判断是 tag 标签是否一致；

```javascript
class Vn {
  _sameVnode(n, o) {
    return n.tag === o.tag;
  }
}
```

### generateElm

> generateElm 的作用是跟踪每个节点实际的真实节点，方便在对比虚拟节点后实时更新真实 DOM 节点；

执行 generateElm  方法后，我们可以在旧节点的 vnode 中跟踪到每个 virture dom 的真实节点信息；

```javascript
class Vn {
  generateElm(vnode) {
    const traverseTree = (v, parentEl) => {
      let children = v.children;
      if(Array.isArray(children)) {
        children.forEach((c, i) => {
          c.elm = parentEl.childNodes[i];
          traverseTree(c, c.elm)
        })
      }
    }
    traverseTree(vnode, this.el);
  }
}
```

### patchVnode

> patchVnode 是新旧 Vnode 对比的核心方法；

1. 节点相同，且节点除了拥有文本节点外没有其它子节点，这种情况下直接替换文本内容；
2. 新节点没有子节点，旧节点有子节点，则删除旧节点所有子节点；
3. 旧节点没有子节点，新节点有子节点，则用新的所有子节点去更新旧节点；
4. 新旧都存在子节点，则对比子节点内容做操作；

```javascript
class Vn {
  patchVnode(nVnode, oVnode) {
    if(nVnode.text && nVnode.text !== oVnode) {
      // 当前真实dom元素
      let ele = oVnode.elm
      // 子节点为文本节点
      ele.textContent = nVnode.text;
    } else {
      const oldCh = oVnode.children;
      const newCh = nVnode.children;
      // 新旧节点都存在。对比子节点
      if (util._isDef(oldCh) && util._isDef(newCh)) {
        this.updateChildren(ele, newCh, oldCh)
      } else if (util._isDef(oldCh)) {
        // 新节点没有子节点
      } else {
        // 老节点没有子节点
      }
    }
  }
}
```

### updateChildren

- 旧节点的起始位置为 oldStartIndex，截止位置为 oldEndIndex，新节点的起始位置为 newldStartIndex，截止位置为 newEndIndex；	
- 新旧 children 的起始位置两两对比，顺序是：
  1. newStartVnode, oldStartVnode；
  2. newEndVnode, oldEndVnode；
  3. newEndVnode, oldStartVnode；
  4. newStartIndex, oldEndIndex；
- newStartVnode, oldStartVnode 节点相同，执行一次 patchVnode 过程，也就是递归对比相应子节点，并替换节点的过程，newStartVnode, oldStartVnode 都向右移动一位；
- newEndVnode, oldEndVnode 节点相同，执行一次 patchVnode 过程，递归对比相应子节点，并替换节点，oldEndIndex， newEndIndex 都像左移动一位；
- newEndVnode, oldStartVnode 节点相同，执行一次 patchVnode 过程，并将旧的 oldStartVnode 移动到尾部,oldStartIndex 右移一位，newEndIndex 左移一位；
- newStartIndex, oldEndIndex 节点相同，执行一次 patchVnode 过程，并将旧的 oldEndVnode 移动到头部,oldEndIndex 左移一位， newStartIndex右移一位；
- 四种组合都不相同，则会搜索旧节点所有子节点，找到这个旧节点和 newStartVnode 执行 patchVnode 过程；
- 不断对比的过程使得 oldStartVnode 不断逼近 oldEndIndex，newStartIndex 不断逼近 newEndIndex；
  1. 当 oldEndIndex <= oldStartIndex ，说明旧节点已经遍历完了，此时只要批量增加新节点即可；
  2. 当 newEndIndex <= newStartIndex，说明旧节点还有剩下，此时只要批量删除旧节点即可；

```javascript
class Vn {
  updateChildren(el, newCh, oldCh) {
    // 新children开始标志
    let newStartIndex = 0;
    // 旧children开始标志
    let oldStartIndex = 0;
    // 新children结束标志
    let newEndIndex = newCh.length - 1;
    // 旧children结束标志
    let oldEndIndex = oldCh.length - 1;
    let oldKeyToId;
    let idxInOld;
    let newStartVnode = newCh[newStartIndex];
    let oldStartVnode = oldCh[oldStartIndex];
    let newEndVnode = newCh[newEndIndex];
    let oldEndVnode = oldCh[oldEndIndex];
    // 遍历结束条件
    while (newStartIndex <= newEndIndex && oldStartIndex <= oldEndIndex) {
      // 新children开始节点和旧开始节点相同
      if (this._sameVnode(newStartVnode, oldStartVnode)) {
        this.patchVnode(newCh[newStartIndex], oldCh[oldStartIndex]);
        newStartVnode = newCh[++newStartIndex];
        oldStartVnode = oldCh[++oldStartIndex]
      } else if (this._sameVnode(newEndVnode, oldEndVnode)) {
      // 新childre结束节点和旧结束节点相同
        this.patchVnode(newCh[newEndIndex], oldCh[oldEndIndex])
        oldEndVnode = oldCh[--oldEndIndex];
        newEndVnode = newCh[--newEndIndex]
      } else if (this._sameVnode(newEndVnode, oldStartVnode)) {
      // 新childre结束节点和旧开始节点相同
        this.patchVnode(newCh[newEndIndex], oldCh[oldStartIndex])
        // 旧的oldStartVnode移动到尾部
        el.insertBefore(oldCh[oldStartIndex].elm, null);
        oldStartVnode = oldCh[++oldStartIndex];
        newEndVnode = newCh[--newEndIndex];
      } else if (this._sameVnode(newStartVnode, oldEndVnode)) {
        // 新children开始节点和旧结束节点相同
        this.patchVnode(newCh[newStartIndex], oldCh[oldEndIndex]);
        el.insertBefore(oldCh[oldEndIndex].elm, oldCh[oldStartIndex].elm);
        oldEndVnode = oldCh[--oldEndIndex];
        newStartVnode = newCh[++newStartIndex];
      } else {
        // 都不符合的处理，查找新节点中与对比旧节点相同的vnode
        this.findIdxInOld(newStartVnode, oldCh, oldStartIdx, oldEndIdx);
      }
    }
    // 新节点比旧节点多，批量增加节点
    if(oldEndIndex <= oldStartIndex) {
      for (let i = newStartIndex; i <= newEndIndex; i++) {
        // 批量增加节点
        this.createElm(oldCh[oldEndIndex].elm, newCh[i])
      }
    }
  }

  createElm(el, vnode) {
    let tag = vnode.tag;
    const ele = document.createElement(tag);
    this._setAttrs(ele, vnode.data);
    const testEle = document.createTextNode(vnode.children);
    ele.appendChild(testEle)
    el.parentNode.insertBefore(ele, el.nextSibling)
  }

  // 查找匹配值
  findIdxInOld(newStartVnode, oldCh, start, end) {
    for (var i = start; i < end; i++) {
      var c = oldCh[i];
      if (util.isDef(c) && this.sameVnode(newStartVnode, c)) { return i }
    }
  }
}
```

## diff 算法优化

当四种比较节点都找不到匹配时，会调用 findIdxOld 找到旧节点中和新的比较节点一致的节点；

唯一属性 key 这个唯一的标志位，可以对旧节点建立简单的字典查询，只要有 key 值便可以方便的搜索到符合要求的旧节点；

```javascript
class Vn {
  updateChildren() {
    ···
    } else {
      // 都不符合的处理，查找新节点中与对比旧节点相同的vnode
      if (!oldKeyToId) oldKeyToId = this.createKeyMap(oldCh, oldStartIndex, oldEndIndex);
      idxInOld = util._isDef(newStartVnode.key) ? oldKeyToId[newStartVnode.key] : this.findIdxInOld(newStartVnode, oldCh, oldStartIndex, oldEndIndex);
      // 后续操作
    }
  }
  // 建立字典
  createKeyMap(oldCh, start, old) {
    const map = {};
    for(let i = start; i < old; i++) {
      if(oldCh.key) map[key] = i;
    }
    return map;
  }
}
```

