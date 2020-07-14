# 执行上下文和执行栈

### 执行上下文

> 执行上下文就是当前Javascript代码被解析和执行的所在环境的抽象概念，Javascript中运行的任何代码都是在执行上下文中执行

#### 类型

- **全局执行上下文：**这是默认的，最基础的执行上下文；不在任何函数中的代码都位于全局执行上下文中。一个程序中只能有一个全局上下文。

  共有两个过程：

  1. 创建有全局对象，在浏览器中这个全局对象就是window；
  2. 将this指针指向这个全局对象；

- **函数执行上下文：**每次调用函数时，都会为该函数创建一个新的执行上下文，每个函数拥有自己的执行上下文，但是只有在函数被调用时才会被创建。一个程序中可以存在多个函数上下文，这些函数按照特定的顺序执行。

- **Eval函数执行上下文：**运行在eval函数中的代码也获得了自己的执行上下文。

#### 如何被创建的

执行上下文分两个阶段创建：1. 创建阶段； 2. 执行阶段

##### 创建阶段

在任意的Javascript代码被执行前，执行上下文处于创建阶段；在创建阶段总共发生了三件事情：

1. 确定this的值，也被称为This Binding;
2. LexicaEnviroment（词法环境）组件被创建；
3. VariableEnviroment（变量环境）组件被创建；

因此，执行上下文可以在概念上表示如下：

```javascript
ExecutionContext = {  
  ThisBinding = <this value>,  
  LexicalEnvironment = { ... },  
  VariableEnvironment = { ... },  
}
```

<https://www.cnblogs.com/lhh520/p/10175420.html>

<https://juejin.im/post/5a05b4576fb9a04519690d42>

<https://juejin.im/post/5ba32171f265da0ab719a6d7#heading-0>

<https://juejin.im/post/5d387f696fb9a07eeb13ea60#heading-8>

### 执行栈

> 执行栈也叫调用栈，具有FIFO结构，用于存储在代码执行期间创建的所有执行上下文；
>
> 当JavaScript引擎首次读取脚本时，会创建一个全局执行上下文并将其Push到当前执行栈中。每当发生函数调用时，引擎都会为该函数创建一个新的执行上下文并Push到当前执行栈的栈顶。
>
> 引擎会运行执行上下文在执行栈栈顶的函数，根据LIFO规则，当此函数运行完成后，其对应的执行上下文将会从执行栈中Pop出，上下文控制权将转到当前执行栈的下一个执行上下文。

看一段代码

```javascript
let a = 'Hello World!';
function first() {  
  console.log('Inside first function');  
  second();  
  console.log('Again inside first function');  
}
function second() {  
  console.log('Inside second function');  
}
first();  
console.log('Inside Global Execution Context');
```

![img](https://img2018.cnblogs.com/blog/1332080/201812/1332080-20181225155822028-960093150.jpg)