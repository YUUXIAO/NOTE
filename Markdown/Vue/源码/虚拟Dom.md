> 用 JS 模拟一棵 DOM 树，放在浏览器内存中，当需要变更时，虚拟 DOM 使用 diff 算法进行新旧虚拟 DOM 的比较，将变更放到变更队列中，反应到实际的 DOM 树，减少了 DOM 操作；

- 把 template 模板描述成 VNode，然后一系列操作之后通过 VNode 形成真实 DOM 进行挂载；
- 更新的时候对比新旧 VNode ，只更新有变化的一部分，提高视图更新速度；

## 使用虚拟DOM的目的

1. 组件高度抽象化；
2. 可以更好的实现 SSR，同构渲染等；
3. 框架跨平台；


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

### sameVnode

> sameVnode 方法用来判断两个节点是否相同；

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

> patchVnode 方法就是比较节点的子节点；

- 分别对新旧节点的子节点做判断，如果两者都没有或者一者有一者没有，直接删除或者新增即可；
- 如果两者都有子节点，就进行对比和复用操作，实现的方法是 updateChildren；

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
    if (isUndef(oldStartVnode)) {
      // 旧节点的第一个子节点不存在，旧节点头指针就往下一个移动
      oldStartVnode = oldCh[++oldStartIdx] 
    } else if (isUndef(oldEndVnode)) {
      // 旧节点的最后一个子节点不存在，旧节点尾指针就往上一个移动
      oldEndVnode = oldCh[--oldEndIdx]
    } else if (sameVnode(oldStartVnode, newStartVnode)) {
      // 新节点的第一个和旧节点的第一个相同，patch该节点并且新旧节点头指针分别往下一个移动
      patchVnode(oldStartVnode, newStartVnode, insertedVnodeQueue)
      oldStartVnode = oldCh[++oldStartIdx]
      newStartVnode = newCh[++newStartIdx]
    } else if (sameVnode(oldEndVnode, newEndVnode)) {
      // 新节点的最后一个和旧节点的最后一个相同，patch该节点并且新旧节点尾指针分别往上一个移动
      patchVnode(oldEndVnode, newEndVnode, insertedVnodeQueue)
      oldEndVnode = oldCh[--oldEndIdx]
      newEndVnode = newCh[--newEndIdx]
    } else if (sameVnode(oldStartVnode, newEndVnode)) { 
      // 新节点的最后一个和旧节点的第一个相同，patch该节点并且新节点尾指针往上一个移动，旧节点头指针往下一个移动
      patchVnode(oldStartVnode, newEndVnode, insertedVnodeQueue)
      canMove && nodeOps.insertBefore(parentElm, oldStartVnode.elm, nodeOps.nextSibling(oldEndVnode.elm))
      oldStartVnode = oldCh[++oldStartIdx]
      newEndVnode = newCh[--newEndIdx]
    } else if (sameVnode(oldEndVnode, newStartVnode)) {
      // 新节点的第一个和旧节点的最后一个相同, patch该节点并且老节点尾指针往上一个移动，新节点头指针往下一个移动
      patchVnode(oldEndVnode, newStartVnode, insertedVnodeQueue)
      canMove && nodeOps.insertBefore(parentElm, oldEndVnode.elm, oldStartVnode.elm)
      oldEndVnode = oldCh[--oldEndIdx]
      newStartVnode = newCh[++newStartIdx]
    } else {
      // 创建老节点key to index的映射
      if (isUndef(oldKeyToIdx)) oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx)
      idxInOld = isDef(newStartVnode.key)
        ? oldKeyToIdx[newStartVnode.key] // 假如新节点第一个有key,找该key下老节点的index
        : findIdxInOld(newStartVnode, oldCh, oldStartIdx, oldEndIdx) // 假如新节点没有key,直接遍历找相同的index
      if (isUndef(idxInOld)) { // New element
        // 假如没有找到index,则创建节点
        createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm)
      } else {
        // 假如有index,则找出这个需要move的老节点
        vnodeToMove = oldCh[idxInOld]
        /* istanbul ignore if */
        if (process.env.NODE_ENV !== 'production' && !vnodeToMove) {
          warn(
            'It seems there are duplicate keys that is causing an update error. ' +
            'Make sure each v-for item has a unique key.'
          )
        }
        if (sameVnode(vnodeToMove, newStartVnode)) {
          // move老节点和新节点的第一个基本相同则开始patch
          patchVnode(vnodeToMove, newStartVnode, insertedVnodeQueue)
          // 设置老节点空
          oldCh[idxInOld] = undefined
          canMove && nodeOps.insertBefore(parentElm, vnodeToMove.elm, oldStartVnode.elm)
        } else {
          // 不同则还是创建新节点
          // same key but different element. treat as new element
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

