## 原理

> Tree-shaking 的作用是把项目中没必要的模块全部抖掉，用于在不同的模块之间消除无用的代码；

Tree-shaking 的消除原理是依赖 ES6 的模块特性；

- 只能作为模块顶层语句出现；
- import 的模块名只能是字符串常量；
- import binding 是 immutable的；

ES6 模块依赖关系是确定的，和运行时的状态无关，可以进行可靠的静态分析，这个就是 Tree-shaking 的基础；

