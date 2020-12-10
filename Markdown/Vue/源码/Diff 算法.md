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

