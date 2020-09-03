### 变量提升与函数提升

1. 在js中js引擎会优先解析var变量和function定义，在预解析完成后从上到下逐步进行；
2. 解析var变量时，会把值存储在“执行环境”中，而不会去赋值，值是存储作用；
3. 在解析function时会把函数整体定义，在内部机制中第一步实行的是把以function方式定义的函数先声明了（预处理）；

### 执行上下文

> 执行上下文就是当前Javascript代码被解析和执行的所在环境的抽象概念，Javascript中运行的任何代码都是在执行上下文中执行；

#### 类型

1. 全局执行上下文：这是默认的，最基础的执行上下文；不在任何函数中的代码都位于全局执行上下文中，一个程序中只能有一个全局上下文；

   共有两个过程：

   1. 创建有全局对象，在浏览器中这个全局对象就是window；
   2. 将this指针指向这个全局对象；

2. 函数执行上下文：每次调用函数时，都会为该函数创建一个新的执行上下文，每个函数拥有自己的执行上下文，一个程序中可以存在多个函数上下文，这些函数按照特定的顺序执行；
3. Eval函数执行上下文：运行在eval函数中的代码也获得了自己的执行上下文；

#### 如何被创建的

执行上下文分两个阶段创建：1. 创建阶段； 2. 执行阶段

##### 创建阶段

在任意的Javascript代码被执行前，执行上下文处于创建阶段；在创建阶段总共发生了三件事情：

1. 确定this的值，也被称为This Binding;
2. LexicaEnviroment（词法环境）组件被创建；
3. VariableEnviroment（变量环境）组件被创建；

执行上下文可以在概念上表示如下：

```javascript
ExecutionContext = {  
  ThisBinding = <this value>,  
  LexicalEnvironment = { ... },  
  VariableEnvironment = { ... },  
}
```

##### This Biling:

1. 在全局执行上下文中，this 的值指向全局对象，在浏览器中，this 的值指向 window 对象。
2. 在函数执行上下文中，this 的值取决于函数的调用方式。如果它被一个对象引用调用，那么 this 的值被设置为该对象，否则 this 的值被设置为全局对象或 undefined（严格模式下）。

##### Lexical Environment:

词法环境是一个包含标识符变量映射的结构。（这里的标识符表示变量/函数的名称，变量是对实际对象【包括函数类型对象】或原始值的引用）

1. 两个组成：
   1. 环境记录器：是存储变量和函数声明的实际位置；
   2. 对外部环境的引用：意味着它可以访问其外部词法环境（父级词法环境（作用域））；

2. 两种类型：

   1. **全局环境**（在全局执行上下文中）是没有外部环境引用的词法环境。全局环境的外部环境引用为null。它拥有内建的 Object/Array/等、在环境记录器内的原型函数（关联全局对象，比如 window 对象）还有任何用户定义的全局变量，并且this的值指向这个全局对象。

   2. **函数环境**，用户在函数中定义的变量被存储在环境记录中，包含了arguments对象。对外部环境的引用可以是全局环境或者任何包含此内部函数的外部函数。

      对于函数环境而言，环境记录还包含了一个arguments对象，该对象包含了索引和传递给函数的参数之间的映射以及传递给函数的参数的长度（数量）。

   ```javascript
   function foo(a, b) {  
     var c = a + b;  
   }  
   foo(2, 3);

   // arguments 对象  
   Arguments: {0: 2, 1: 3, length: 2},
   ```

环境记录同样也有两种类型：

1. 声明性环境记录：存储变量，函数和参数。一个函数环境包含声明性环境记录。
2. 对象环境记录：用于定义在全局执行上下文中出现的变量和函数的关联。全局环境包含对象环境记录。

抽象地说，词法环境的伪代码中看起来像这样：

```javascript
GlobalExectionContext = {  // 全局执行上下文
  LexicalEnvironment: {  // 词法环境
    EnvironmentRecord: {  // 环境记录
      Type: "Object",    //全局环境
      // 标识符绑定在这里 
    outer: <null>   // 对外部环境的引用
  }  
}

FunctionExectionContext = {   //函数执行上下文
  LexicalEnvironment: {  //词法环境
    EnvironmentRecord: {  //环境记录
      Type: "Declarative",  //函数环境
      // 标识符绑定在这里 
    outer: <Global or outer function environment reference>  //对外部环境的引用
  }  
}
```

##### Variable Environment ：

变量环境是一个词法环境，其环境记录器持有变量声明语句在执行上下文中创建的绑定关系，它具有上面定义的词法环境的所有属性。

在 ES6 中，词法环境和 变量环境的区别在于前者用于存储函数声明和变量（ let  和 const ）绑定，而后者仅用于存储变量（ var ）绑定。

使用例子进行介绍：

```javascript
let a = 20;  
const b = 30;  
var c;

function multiply(e, f) {  
 var g = 20;  
 return e * f * g;  
}

c = multiply(20, 30);
```

执行上下文如下：

```javascript
GlobalExectionContext = {

  ThisBinding: <Global Object>,

  LexicalEnvironment: {  
    EnvironmentRecord: {  
      Type: "Object",  
      // 标识符绑定在这里  
      a: < uninitialized >,  
      b: < uninitialized >,  
      multiply: < func >  
    }  
    outer: <null>  
  },

  VariableEnvironment: {  
    EnvironmentRecord: {  
      Type: "Object",  
      // 标识符绑定在这里  
      c: undefined,  
    }  
    outer: <null>  
  }  
}

FunctionExectionContext = {  
   
  ThisBinding: <Global Object>,

  LexicalEnvironment: {  
    EnvironmentRecord: {  
      Type: "Declarative",  
      // 标识符绑定在这里  
      Arguments: {0: 20, 1: 30, length: 2},  
    },  
    outer: <GlobalLexicalEnvironment>  
  },

  VariableEnvironment: {  
    EnvironmentRecord: {  
      Type: "Declarative",  
      // 标识符绑定在这里  
      g: undefined  
    },  
    outer: <GlobalLexicalEnvironment>  
  }  
}
```

### 执行栈

1. 执行栈也叫调用栈，具有FIFO结构，用于存储在代码执行期间创建的所有执行上下文；
2. 在全局代码执行前, JS引擎就会创建一个栈来存储管理所有的执行上下文对象；
3. 在全局执行上下文(window)确定后, 将其添加到栈中(压栈)；
4. 在函数执行上下文创建后, 将其添加到栈中(压栈)；
5. 在当前函数执行完后,将栈顶的对象移除(出栈)；
6. 当所有的代码执行完后, 栈中只剩下window；

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

##### 堆栈溢出

当达到调用栈最大的大小的时候就会发生这种情况，特别是在你写递归的时候却没有全方位的测试它。

```javascript
    function foo() {
      foo();
    }
    foo();

```

当我们的引擎开始执行这段代码的时候，它从 foo 函数开始。然后这是个递归的函数，并且在没有任何的终止条件的情况下开始调用自己。因此，每执行一步，就会把这个相同的函数一次又一次地添加到调用堆栈中。然后它看起来就像是这样的：

![img](https://user-gold-cdn.xitu.io/2017/11/11/3925f8363d7a763e6474709ccddf7d96?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)