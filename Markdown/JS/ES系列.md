## ES11

### matchAll

> matchAll 是基于 String 原型上的一个新方法，允许我们匹配一个字符串和一个正则表达式，返回的是所有匹配结果的迭代器；

- 使用 for...of 遍历或使用操作符 ...  和 Array.from 将其转换成数组；

```javascript
const reg = /[0-3]/g;
const data = '2020'; 
// 返回值是一个迭代器,返回 RegExpStringIterator {}
console.log(data.matchAll(reg)); //
console.log([...data.matchAll(reg)]);

// 返回结果
 * 0: ["2", index: 0, input: "2020", groups: undefined]
 * 1: ["0", index: 1, input: "2020", groups: undefined]
 * 2: ["2", index: 2, input: "2020", groups: undefined]
 * 3: ["0", index: 3, input: "2020", groups: undefined]
```

#### 应用场景

1. 想要匹配出字符串中所有的目标子串；

### Dynamic import

> ES2020 中 import（）可以根据条件导入模块，返回一个 promise 对象；

- 在ES6中 import/export 声明只能出现在顶层作用域，不支持按需加载、懒加载；静态导入有利于静态分析工具和 tree shaking 发挥作用；
- @babel/preset-env 已经包含了 @babel/plugin-syntax-dynamic-import，如果要使用 import( ) 语法，配置 @babel/preset-env 即可；

```javascript
// menu.js
export default {
    menu: 'menu'
}

// index.js
if(true) {
  let menu = import('./menu');
  console.log(menu);	 // Promise {<pending>
  menu.then(data => console.log(data));
  // Module {default: {menu: "menu"}, __esModule: true, Symbol(Symbol.toStringTag): "Module"}
}
```

#### 与import比较

1. 能够在函数、分支等非顶层作用域使用，可以按需加载和懒加载；
2. 模块标识支持变量传入，可以动态计算模块标识；
3. 使用不仅限于 module，普通 script 中也能使用；
4. 减少导入加载时间，很多模块并不需要首屏渲染

#### 应用场景

1. 苛求首屏性能：动态加载模块可以避免在首屏初始化时引用所有的模块，优化首屏性能；
2. 难以提前确认目标模块：比如根据用户语言选项加载不同的语言模块；
3. 在特殊情况下才需要加载某些模块：比如异常情况下加载降级模块；

### import.meta

> import.meta 用来透出模块特定的元信息，是个对象，原型为 null；

1. 模块的 URL 或文件名：例如 Node.js 里的 __ dirname、__ filename；
2. 所处的 script 标签：例如浏览器支持的 document.currentScript；
3. 入口模块：例如 Node.js 里的 process.mainModule；

```javascript
// 模块的 URL（浏览器环境）
import.meta.url
// 当前模块所处的 script 标签
import.meta.scriptElement
```

### export-ns-from

ES11 新增了 export * as XX from 'module' 和  import * as XX from 'module'；

```javascript
//menu.js
export * as ns from './info';

// 可以理解为是将下面两条语句进行合并
import * as ns from './info';
export { ns };
```

### Promise.allSettled

> Promise.allSettled 方法返回一个在所有给定的 promise 都已经 fufilled 或 rejected 后的 promise，并带有一个返回数组，每个对象表示对应的 promise 结果；

返回的每一个对象都有一个 status 属性，值为 fufilled 或  rejected；

- 如果 status 的值是 fufilled，那对象还有一个 value 属性对应 promise 成功的结果；
- 如果 status 的值是 rejected，那对象还有一个 reason属性对应 promise 失败的原因；

```javascript
const promise1 = Promise.resolve(100);
const promise2 = new Promise((resolve, reject) => setTimeout(reject, 100, 'info'));
const promise3 = new Promise((resolve, reject) => setTimeout(resolve, 200, 'name'))

Promise.allSettled([promise1, promise2, promise3]).
    then((results) => console.log(result));

// 返回结果
[
  { status: 'fulfilled', value: 100 },
  { status: 'rejected', reason: 'info' },
  { status: 'fulfilled', value: 'name' }
]
```

#### 应用场景

1. 需要知道所有项的所有结果做一些操作，并不关心其执行结果是否成功；

### BigInt

> BigInt 是一种数字类型的数据，它可以表示任意精度格式的整数；

1. BigInt 只用来表达整数，没有位数的限制，任何位数的整数都可以精确表示；
2. 为了和 Number 类型进行区分，BigInt 类型的数据必须添加后缀 n；

BigInt 解决了下面两个问题：

1. 超过 JS 中安全的最大数字（Number.MAX_SAFE_INTEGER）无法精确显示；
2. 大于或等于2的1024次方的数值，JS 无法表示，会返回 Infinity；

```javascript
//Number类型在超过9009199254740991后，计算结果即出现问题
const num1 = 90091992547409910;
console.log(num1 + 1); //90091992547409900

//BigInt 计算结果正确
const num2 = 90091992547409910n;
console.log(num2 + 1n); //90091992547409911n

//Number 类型不能表示大于 2 的 1024 次方的数值
let num3 = 9999;
for(let i = 0; i < 10; i++) {
    num3 = num3 * num3;
}
console.log(num3); //Infinity

//BigInt 类型可以表示任意位数的整数
let num4 = 9999n;
for(let i = 0n; i < 10n; i++) {
    num4 = num4 * num4;
}
console.log(num4); // ... 正确返回

const bigint = 999999999999999999n
const bigintByMethod = BigInt('999999999999999999')
console.log(bigint) //999999999999999999n
console.log(bigintByMethod) //999999999999999999n
console.log(bigint === bigintByMethod) //true

//布尔值
console.log(BigInt(true)) //1n
console.log(BigInt(false)) //0n
```

BigInt 和 Number 是两种数据类型，不能直接进行四则运算可以进行比较操作；

```javascript
console.log(typeof bigint) // "bigint" 

console.log(99n == 99); //true
console.log(99n === 99); //false 
console.log(99n + 1);
//TypeError: Cannot mix BigInt and other types, use explicit conversionss
```

BigInt之间，除了一元加号（+）运算符外，其他均可用于BigInt；

```javascript
console.log(1n + 2n) //3n
console.log(1n - 2n) //-1n
console.log(+ 1n) 
//Uncaught TypeError: Cannot convert a BigInt value to a number
console.log(- 1n) //-1n
console.log(10n * 20n) //200n
console.log(23n%10n) //3n
console.log(20n/10n) //2n

// 除法运算符会自动向下舍入到最接近的整数
console.log(25n / 10n) // 2n
console.log(29n / 10n) // 2n
```

Boolean类型和BigInt类型的转换时，只要不是0n，BigInt就被视为true；

```javascript
if (5n) {
  // 这里代码块将被执行
}

if (0n) {
  // 这里代码块不会执行
}
```

### globalThis

> globalThis  作为顶层对象引入，在任何环境下都可以通过 globalThis 拿到顶层对象；

在 globalThis 之前，从不同的 Javascript 环境中获取全局对象需要不同的语句：

1. 在 web 中，可以使用 window、self 取到全局对象；
2. 在 web workers 中，使用  self 可以获取全局对象；
3. 在 Node.js 中，使用 global 获取全局对象；

在 globalThis 之前，需要这样去获取全局对象；

```javascript
var getGlobal = function () {
    if (typeof self !== 'undefined') { return self; }
    if (typeof window !== 'undefined') { return window; }
    if (typeof global !== 'undefined') { return global; }
    throw new Error('unable to locate global object');
};
```

### Nullish coalescing Operator

> ES2020 新增了运算符 ??，当左侧操作数为 null 或 undefined 时，返回其右侧操作数，否则返回左侧操作数；

```javascript
const defaultValue = 100;
const someValue = 0;
// 当使用 || 操作符时，当左侧操作数是 0、null、undefined、NaN、false、‘ ’ 时，都会返回右侧操作数；
let value1 = someValue || defaultValue;  // 100
let value2 = someValue ?? defaultValue;   // 0
```

- 不可与其它运算符组合使用，可以使用括号包裹组合使用；

```javascript
'value' || undefined ?? "aaa" 
//Uncaught SyntaxError: Unexpected token '??'
'value' && undefined ?? "aaa" 
//Uncaught SyntaxError: Unexpected token '??'

('value' || undefined) ?? "aaa" // "value"
('value' && null) ?? "aaa"  // "aaa"
```

### Optional Chaining

> 可选链操作符 ?. 能够在属性访问、方法调用前检查其是否存在，不存在则返回 undefined；

```javascript
const user = {
  age: 18
}
const u1 = user.children.name 
// 不然会报错：TypeError: Cannot read property 'name' of undefined 或 TypeError: Cannot read property 'name' of null

// 使用 ?. 操作符可以将上面代码精简
const u2 = user.children?.name  // undefined
```

