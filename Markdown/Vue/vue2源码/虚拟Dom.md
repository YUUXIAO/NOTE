> Virtual DOM 其实是一棵以 JS 对象（Vnode 节点）作为基础的树，用对象属性来描述节点，实际上它只是一层对真实 DOM 的抽象；

用 JS 模拟一棵 DOM 树，放在浏览器内存中，当需要变更时，虚拟 DOM 使用 diff 算法进行新旧虚拟 DOM 的比较，将变更放到**变更队列中**，反应到实际的 DOM 树，减少了 DOM 操作；

- 这里不是实时修改虚拟dom然后计算变更的，而是先放入缓冲队列，等到下一事件循环再一起更新（vue内部是通过promise回调、settime这些来实现一个event loop），如果是同一个副作用函数还会进行去重

```javascript
<ul id='list'>
  <li class='item'>Item 1</li>
  <li class='item'>Item 2</li>
  <li class='item'>Item 3</li>
</ul>

var element = {
    tagName: 'ul', // 节点标签名
    props: { // DOM的属性，用一个对象存储键值对
        id: 'list'
    },
    children: [ // 该节点的子节点
      {tagName: 'li', props: {class: 'item'}, children: ["Item 1"]},
      {tagName: 'li', props: {class: 'item'}, children: ["Item 2"]},
      {tagName: 'li', props: {class: 'item'}, children: ["Item 3"]},
    ]
}
```

## 使用虚拟DOM的目的

1. 组件高度抽象化，可以更好的实现 SSR，同构渲染等；
2. 具备跨平台的优势：由于 Virtual DOM 是以 JavaScript 对象为基础而不依赖真实平台环境，所以使它具有了跨平台的能力，比如说浏览器平台、Weex、Node 等；
3. 提升渲染性能：Virtual DOM的优势不在于单次的操作，而是在大量、频繁的数据更新下，能够对视图进行合理、高效的更新；

## VNode

> 虚拟 DOM 就是一个特殊的 js 对象；

```javascript
export class VNode {
  constructor (
    tag?: string,
    data?: VNodeData,
    children?: ?Array<VNode>,
    text?: string,
    elm?: Node,
    context?: Component,
    componentOptions?: VNodeComponentOptions,
    asyncFactory?: Function
  ) {
    this.tag = tag
    this.data = data
    this.children = children
    this.text = text
    this.elm = elm
    this.ns = undefined
    this.context = context
    this.functionalContext = undefined
    this.key = data && data.key
    this.componentOptions = componentOptions
    this.componentInstance = undefined
    this.parent = undefined
    this.raw = false
    this.isStatic = false
    this.isRootInsert = true
    this.isComment = false
    this.isCloned = false
    this.isOnce = false
    this.asyncFactory = asyncFactory
    this.asyncMeta = undefined
    this.isAsyncPlaceholder = false
  }
}
```

## Virtual DOM算法

1. 用 js 对象结构表示 DOM 树的结构，用这个树构建一个真正的 DOM 树，插到文档中；
2. 当状态变化时，生成新的 Virtual DOM ，比对新旧对象记录两者差异；
3. 将差异应用到 DOM 上，并保存新的 Virtual DOM，重复第二步；

```javascript
Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
  const vm: Component = this
  if (vm._isMounted) {
    callHook(vm, 'beforeUpdate')
  }
  const prevEl = vm.$el
  const prevVnode = vm._vnode
  // ...
  if (!prevVnode) {
    // 第一次渲染
    vm.$el = vm.__patch__(
      vm.$el, vnode, hydrating, false /* removeOnly */,
      vm.$options._parentElm,
      vm.$options._refElm
    )
  } else {
    // 更新视图
    vm.$el = vm.__patch__(prevVnode, vnode)
  }
  // ...
}
```

## Diff 算法

> diff 算法的目的就是为了减少 DOM 操作的性能开销，尽可能的复用 DOM 元素，所以需要判断出是否有节点需要移动，应该如何移动以及找出那些需要被添加或删除的节点；

vm._ patch _ 函数就是 Virtual DOM Diff 的核心，是它最后把真实 DOM 插入页面中的；

1. 判断老节点是否存在；
2. 如果不存在老节点则为首次渲染，直接创建元素；
3. 如果存在老节点则先判断新旧节点根节点是否相同 ；
4. 如果根节点相同则使用 patchVnode 给老节点打补丁；
5. 如果根节点不相同则使用新节点直接替换老节点；

```javascript
function patch (oldVnode, vnode, hydrating, removeOnly, parentElm, refElm) {
  if (isUndef(vnode)) {
    if (isDef(oldVnode)) invokeDestroyHook(oldVnode)
    return
  }

  let isInitialPatch = false
  const insertedVnodeQueue = []

  if (isUndef(oldVnode)) {
    // 老节点不存在，直接创建元素
    isInitialPatch = true
    createElm(vnode, insertedVnodeQueue, parentElm, refElm)
  } else {
    const isRealElement = isDef(oldVnode.nodeType)
    if (!isRealElement && sameVnode(oldVnode, vnode)) {
      // 新节点和老节点根节点相同，则给老节点打补丁
      patchVnode(oldVnode, vnode, insertedVnodeQueue, removeOnly)
    } else {
      // ... 省略ssr代码
      // 新节点和老节点根节点不相同，直接替换老节点
      const oldElm = oldVnode.elm
      const parentElm = nodeOps.parentNode(oldElm)
      createElm(
        vnode,
        insertedVnodeQueue,
        oldElm._leaveCb ? null : parentElm,
        nodeOps.nextSibling(oldElm)
      )
    }
  }
  // .....
  return vnode.elm
}
```

### sameVnode（判断节点是否相同）

1. **节点类型的比较：**先比较两个节点的类型是否一致，比如元素节点、文本节点、组件节点，如果节点类型不同那肯定不相同
2. **节点标签比较：**如果两个节点都是元素节点，那么就比较标签名是否一致
3. **节点的key属性比较：**如果两个节点都有key属性，那就比较key值，key相同就相同，不相同就不相同
4. **其他属性比较：**如果节点类型和标签相同，并且没有key属性，那么就继续比较其他的属性：样式、class、事件等，只有其他属性都相同才是相同的
5. 如果**两个元素有子节点，也是会递归比较他们的子节点**，只有子节点完全相同才认为两个节点完全相同
6. **组件节点比较：**对于组件节点，除了比较构造函数或者组件名称和实例引用之外，vue还会比较组件的props属性是否相同，如果props相同就表示相同

```javascript
function sameVnode (a, b) {
  return (
    a.key === b.key && (
      (
        a.tag === b.tag &&
        a.isComment === b.isComment &&
        isDef(a.data) === isDef(b.data) &&
        sameInputType(a, b)
      ) || (
        isTrue(a.isAsyncPlaceholder) &&
        a.asyncFactory === b.asyncFactory &&
        isUndef(b.asyncFactory.error)
      )
    )
  )
}
```

### patchVnode

vue分别对新旧节点的子节点做判断，如果两者都没有或者一者有一者没有，直接删除或者新增即可，如果两者都有子节点，就进行对比和复用操作

patchVnode 方法就是比较节点的子节点，这也是diff算法真正大显威力的时候：

```javascript
function patchVnode (oldVnode, vnode, insertedVnodeQueue, removeOnly) {
  // 新老节点相同
  if (oldVnode === vnode) {
    return
  }
  // ......
  // 假如新节点没有text
  if (isUndef(vnode.text)) {
    // 新旧点都有子节点
    if (isDef(oldCh) && isDef(ch)) {
      // 子节点不相等则更新子节点
      if (oldCh !== ch) updateChildren(elm, oldCh, ch, insertedVnodeQueue, removeOnly)
    } else if (isDef(ch)) {
      // 新节点有子节点，旧节点没有,老节点加上
      if (isDef(oldVnode.text)) nodeOps.setTextContent(elm, '')
      addVnodes(elm, null, ch, 0, ch.length - 1, insertedVnodeQueue)
    } else if (isDef(oldCh)) {
      // 旧节点有子节点，新节点没有，移除旧节点 
      removeVnodes(elm, oldCh, 0, oldCh.length - 1)
    } else if (isDef(oldVnode.text)) {
      // 旧节点有文本，新节点没有文本
      nodeOps.setTextContent(elm, '')
    }
  } else if (oldVnode.text !== vnode.text) {
    // 新节点和旧节点text不相等
    nodeOps.setTextContent(elm, vnode.text)
  }
  if (isDef(data)) {
    if (isDef(i = data.hook) && isDef(i = i.postpatch)) i(oldVnode, vnode)
  }
}
```

### updateChildren

- 有 key 的话，分别比较首尾节点，如果没有匹配到则使用第一个节点 key 去找相同的 key 进行 patch 比较；
- 没有 key 的话，直接遍历找相似的节点，有则 patch 移动，没有则创建新的节点；

```javascript
function updateChildren (parentElm, oldCh, newCh, insertedVnodeQueue, removeOnly) {
  let oldStartIdx = 0
  let newStartIdx = 0
  let oldEndIdx = oldCh.length - 1
  let oldStartVnode = oldCh[0]
  let oldEndVnode = oldCh[oldEndIdx]
  let newEndIdx = newCh.length - 1
  let newStartVnode = newCh[0]
  let newEndVnode = newCh[newEndIdx]
  let oldKeyToIdx, idxInOld, vnodeToMove, refElm

  const canMove = !removeOnly

  while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
    // 这里排除节点占坑为undefind的情况，因为中间入果匹配上了旧节点会把真实dom挪走
    // 并且把旧数组里的数据设置为 undefind
    if (isUndef(oldStartVnode) ) {
      oldStartVnode = oldCh[++oldStartIdx] 
    } else if (isUndef(oldEndVnode)) {
      oldEndVnode = oldCh[--oldEndIdx]
    } 
    // 新节点的第一个和旧节点的第一个相同，patch该节点并且新旧节点头指针分别往下一个移动
    else if (sameVnode(oldStartVnode, newStartVnode)) {
      patchVnode(oldStartVnode, newStartVnode, insertedVnodeQueue)
      oldStartVnode = oldCh[++oldStartIdx]
      newStartVnode = newCh[++newStartIdx]
    } 
    // 新节点的最后一个和旧节点的最后一个相同，patch该节点并且新旧节点尾指针分别往上一个移动
    else if (sameVnode(oldEndVnode, newEndVnode)) {
      patchVnode(oldEndVnode, newEndVnode, insertedVnodeQueue)
      oldEndVnode = oldCh[--oldEndIdx]
      newEndVnode = newCh[--newEndIdx]
    }
    // 新节点的最后一个和旧节点的第一个相同，patch该节点并且新节点尾指针往上一个移动，旧节点头指针往下一个移动 
    else if (sameVnode(oldStartVnode, newEndVnode)) { 
      patchVnode(oldStartVnode, newEndVnode, insertedVnodeQueue)
      canMove && nodeOps.insertBefore(parentElm, oldStartVnode.elm, nodeOps.nextSibling(oldEndVnode.elm))
      oldStartVnode = oldCh[++oldStartIdx]
      newEndVnode = newCh[--newEndIdx]
    }
    // 新节点的第一个和旧节点的最后一个相同, patch该节点并且老节点尾指针往上一个移动，新节点头指针往下一个移动 
    else if (sameVnode(oldEndVnode, newStartVnode)) {
      patchVnode(oldEndVnode, newStartVnode, insertedVnodeQueue)
      canMove && nodeOps.insertBefore(parentElm, oldEndVnode.elm, oldStartVnode.elm)
      oldEndVnode = oldCh[--oldEndIdx]
      newStartVnode = newCh[++newStartIdx]
    } 
    // 如果头尾循环比还没有结束循环，那就开始从新数组里去节点去旧数组里找
    else {
      if (isUndef(oldKeyToIdx)) oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx)
      // 假如新节点第一个有key,找该key下老节点的index，没有key,直接遍历找相同的index
      idxInOld = isDef(newStartVnode.key)
        ? oldKeyToIdx[newStartVnode.key] 
        : findIdxInOld(newStartVnode, oldCh, oldStartIdx, oldEndIdx)
        // 假如没有找到index,则创建节点，startid++
      if (isUndef(idxInOld)) { // New element
        createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm)
      } else {
        // 假如有index,则找出这个需要move的老节点
        vnodeToMove = oldCh[idxInOld]
         // move老节点和新节点的第一个基本相同则开始patch
        if (sameVnode(vnodeToMove, newStartVnode)) {
          patchVnode(vnodeToMove, newStartVnode, insertedVnodeQueue)
          oldCh[idxInOld] = undefined   // 设置老节点空
          canMove && nodeOps.insertBefore(parentElm, vnodeToMove.elm, oldStartVnode.elm)
        } else {
          // 不同则还是创建新节点
          createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm)
        }
      }
      newStartVnode = newCh[++newStartIdx]
    }
  }
  if (oldStartIdx > oldEndIdx) {
    // 假如老节点的头指针超过了尾部的指针
    // 说明缺少了节点
    refElm = isUndef(newCh[newEndIdx + 1]) ? null : newCh[newEndIdx + 1].elm
    addVnodes(parentElm, refElm, newCh, newStartIdx, newEndIdx, insertedVnodeQueue)
  } else if (newStartIdx > newEndIdx) {
    // 假如新节点的头指针超过了尾部的指针
    // 说明多了节点
    removeVnodes(parentElm, oldCh, oldStartIdx, oldEndIdx)
  }

}
```

### 两组子节点元素比对

![](/Users/yabby/code%20Projects/NOTE/Markdown/.images/18e51984638.png)

vue会先对比这四种情况，这四种其实算是比较理想的情况，有点类似for循环里的子组件：

1. 老数组的头与新数组的头：oldStartIdx，oldStartIdx
2. 老数组的尾和新数组的尾：oldEndIdx，newEndIdx
3. 老数组的头和新数组的尾：oldStartVnode, newEndVnode
4. 老数组的尾和新数组的头：oldEndVnode, newStartVnode

- 会先进行上面4中情况的比对，只要有匹配上相同元素的，就会将对应的startidx++或者endidx--，然后再继续循环递归比对这4种情况，如果匹配上就挪对应新旧数组的idx（**当oldStartIdx > oldEndIdx或者oldStartIdx > newEndIdx时结束循环**），如果全都匹配不上就走下一段逻辑；

  - 如果匹配上了是会直接挪真实dom的节点的位置的
  - 如果在这个环节新旧数组哪一组的idx相交了，说明匹配结束：

    - 此时旧节点元素都匹配完了，新节点还有剩那就直接创建新增dom
    - 如果新节点匹配完了，旧节点还有剩那就直接删除对应的dom


- 上面情况如果没有匹配结束，这时候就会判断新节点有没key值，直接去老节点找，如果找到了就直接将真实dom挪位置，并将老节点的位置设置为undefind，**newStartidx++**
- 如果没key值，那么就会从新数组的第一个节点去老数组中查找，找到了就递归更新，没找到就直接创建节点了，**newStartidx++**

  ![](/Users/yabby/code%20Projects/NOTE/Markdown/.images/18e51d26abc.png)

  只要当新数组｜｜老数组中有一组指针相交了说明对比结束，如果此时旧节点元素都匹配完了，新节点还有剩那就直接创建新增dom，如果新节点匹配完了，旧节点还有剩那就直接删除对应的dom








