> Teleport 组件主要是用来将模板内的 DOM 元素移动到其它的位置；

其实就是多了一种情况的判断，然后用vue2的mount()方法挂载到指定dom元素上；

```php+HTML
<template>
  <Teleport to="body">
      <div> teleport to body </div>  
   </Teleport>
</template>
```

## 源码解析

Teleport 组件经过模板编译后生成的代码：

```javascript
function render(_ctx, _cache) {
  with (_ctx) {
    const { createVNode, openBlock, createBlock, Teleport } = Vue
    return (openBlock(), createBlock(Teleport, { to: "body" }, [
      createVNode("div", null, " teleport to body ", -1 /* HOISTED */)
    ]))
  }
}
```

### createBlock 

> Teleport 组件通过 createBlock 进行创建；

1. 传入 createBlock 的第一个参数为 Teleport，最后得到的 vnode 会有一个 shapeFlag 属性，该属性用来表示 vnode 的类型；
2. isTeleport(type) 得到的结果为 true，所以 shapeFlag 属性的值为 ShapeFlags.TELEPORT（1 << 6）;

```javascript
// packages/runtime-core/src/renderer.ts
export function createBlock(
	type, props, children, patchFlag
) {
  const vnode = createVNode(
    type,
    props,
    children,
    patchFlag
  )
  // ......
  return vnode
}

export function createVNode(
  type, props, children, patchFlag
) {
  // class & style normalization.
  if (props) {
    // ...
  }

  // encode the vnode type information into a bitmap
  const shapeFlag = isString(type)
    ? ShapeFlags.ELEMENT
    : __FEATURE_SUSPENSE__ && isSuspense(type)
      ? ShapeFlags.SUSPENSE
      : isTeleport(type)
        ? ShapeFlags.TELEPORT
        : isObject(type)
          ? ShapeFlags.STATEFUL_COMPONENT
          : isFunction(type)
            ? ShapeFlags.FUNCTIONAL_COMPONENT
            : 0

  const vnode: VNode = {
    type,
    props,
    shapeFlag,
    patchFlag,
    key: props && normalizeKey(props),
    ref: props && normalizeRef(props),
  }

  return vnode
}

// packages/runtime-core/src/components/Teleport.ts
export const isTeleport = type => type.__isTeleport
export const Teleport = {
  __isTeleport: true,
  process() {}
}
```

### ShapeFlags

```javascript
// packages/shared/src/shapeFlags.ts
export const enum ShapeFlags {
  ELEMENT = 1,
  FUNCTIONAL_COMPONENT = 1 << 1,
  STATEFUL_COMPONENT = 1 << 2,
  TEXT_CHILDREN = 1 << 3,
  ARRAY_CHILDREN = 1 << 4,
  SLOTS_CHILDREN = 1 << 5,
  TELEPORT = 1 << 6,
  SUSPENSE = 1 << 7,
  COMPONENT_SHOULD_KEEP_ALIVE = 1 << 8,
  COMPONENT_KEPT_ALIVE = 1 << 9
}
```

### render

> 在组件的 render 节点，会根据 type 和 shapeFlag 走不同的逻辑；

```javascript
// packages/runtime-core/src/renderer.ts
const render = (vnode, container) => {
  if (vnode == null) {
    // 当前组件为空，则将组件销毁
    if (container._vnode) {
      unmount(container._vnode, null, null, true)
    }
  } else {
    // 新建或者更新组件
    // container._vnode 是之前已创建组件的缓存
    patch(container._vnode || null, vnode, container)
  }
  container._vnode = vnode
}

// patch 是表示补丁，用于 vnode 的创建、更新、销毁
const patch = (n1, n2, container) => {
  // 如果新旧节点的类型不一致，则将旧节点销毁
  if (n1 && !isSameVNodeType(n1, n2)) {
    unmount(n1)
  }
  const { type, ref, shapeFlag } = n2
  switch (type) {
    case Text:
      // 处理文本
      break
    case Comment:
      // 处理注释
      break
    // ...
    default:
      if (shapeFlag & ShapeFlags.ELEMENT) {
        // 处理 DOM 元素
      } else if (shapeFlag & ShapeFlags.COMPONENT) {
        // 处理自定义组件
      } else if (shapeFlag & ShapeFlags.TELEPORT) {
        // 处理 Teleport 组件,调用 Teleport.process 方法
        type.process(n1, n2, container...);
      } // else if ...
  }
}
```

### Teleport.process

> 在处理 Teleport 时，最后会调用 Teleport.process 方法；

```javascript
// packages/runtime-core/src/components/Teleport.ts
const isTeleportDisabled = props => props.disabled
export const Teleport = {
  __isTeleport: true,
  process(n1, n2, container) {
    const disabled = isTeleportDisabled(n2.props)
    const { shapeFlag, children } = n2
    if (n1 == null) {
      const target = (n2.target = querySelector(n2.prop.to))      
      const mount = (container) => {
        // compiler and vnode children normalization.
        if (shapeFlag & ShapeFlags.ARRAY_CHILDREN) {
          mountChildren(children, container)
        }
      }
      if (disabled) {
        // 开关关闭，挂载到原来的位置
        mount(container)
      } else if (target) {
        // 将子节点，挂载到属性 `to` 对应的节点上
        mount(target)
      }
    }
    else {
      // n1不存在，更新节点即可
    }
  }
}
```

