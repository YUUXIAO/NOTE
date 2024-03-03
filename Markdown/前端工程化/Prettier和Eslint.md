## Prettier

Preitter 是一个代码格式化的工具（只起格式化作用，不对代码做质量检查）

但是Prettier 则天然支持对大多数项目文件的格式化，包括 JSX、Vue、TypeScript、CSS、HTML、JSON、Markdown、YAML 等，所以项目中一般会将 prettier 和 eslint 搭配着使用

````
npm i prettier -D
````

### 配置

一共有三种方式支持对 Preitter 进行配置：

1. 根目录下创建 .prettierrc 文件，能够写入 YML、JSON 的配置格式，并且支持 .yml/.yaml/.json/.js 后缀；
2. 根目录下创建 .prettier.config.js 文件，并且对外 export 一个对象；
3. 在 package.json 中新建 preitter 属性；

### 忽略代码

1. preitter 支持对某些文件的格式化忽略，使用 .gitignore 就可以了；
2. 在不需要格式化的代码片段上添加一行 prettier-ignore 注释

````
// prettier-ignore
function log() {
  console.log('log');console.log('format')
}


// 并不会被格式化成
function log() {
    console.log("log");
    console.log("format");
}
````

## ESlint

ESlint 是一个作**代码质量检测、编码风格约束**的工具，它也可以用来格式化你的代码，但是更主要的功能是辅助检查你的 javaScript 代码，

比如:

- **统一团队代码风格问题**：将运算符两边的空、语句末尾的分号...
- **检查语法错误，避免一些低级的代码错误**： 使用了未定义的变量、修改 const 定义的变量...
- **确保代码遵循最佳实践**：可以借助 eslint-config-standard 配置包扩展社区中流行的最佳实践的风格指南

### 配置文件

ESlint 支持配置几种格式的配置文件：

1. 根目录下创建 .eslintrc 文件，能够写入 YML、JSON 的配置格式，并且支持 .yml/.yaml/.json/.js 后缀；
2. package.json：在 package.json 里创建一个 eslintConfig 属性定义你的配置；

如果在项目内有多个层叠配置，ESlint 只会使用一个，优先级如下：

.eslintrc**.js** - eslintrc**.yaml** - eslintrc**.yaml** -.eslintrc**.json** -.eslintrc -package.json

### 配置项解析

**ESlint 是一个高度依赖配置化的工具**，尤其是要注意 extends 和 rules 字段，它们定义了在项目中采用哪种风格（一段代码有没有问题，取决于项目中应用了哪些规则）

#### parser(解析器)

ESlint 默认使用 Espree 作其解析器，但该解析器仅支持最新的 ES5 标准，对于实验性的语法和非标准（例如 Flow 或 TS 类型）语法是不支持的，所以开源社区提供了两种解析器来丰富 ESLint 的功能：

1. **bable-eslint：**Babel  主要用于将  ES6+ 版本的代码转换为向后兼容的 js 语法，如果在项目中使用 es6，就需要将解析器改成 babel-eslint；
2. **@typescript-eslint/parser：**该解析器将 TS 转换成 estree 兼容的形式，允许 ESLint 验证了 ts 源代码；

#### parserOptions(解析器选项)

ESLint 允许你指定你想要的支持的 js 语言选项，默认情况，ESLint 支持 ECMAScript 5 语法，可以覆盖该设置，启用对 ECMAScript 其它版本和 JSX 的支持；

可用的选项有：

1. ecmaVersion：可以使用 6、7、8、9、10 指定要使用的 ECMAScript 版本，也可以使用年份命名的版本号；
2. sourceType：设置为 script（默认）或 module （代码是 ECMAScript 模块）；
3. ecmaFeatures：这是个对象，表示想使用的额外的语言特性：

   - globalReturn：允许在全局作用域下使用 return 语句；
   - impliedStrict：启用全局 strict mode；
   - jsx：使用 JSX；


#### env & globals(环境变量和全局变量)

ESLint 会检测未声明的变量，并发出警告，但有些变量是引入的库声明的，这里就需要提前在配置中声明，每个变量有三个选项， writable、readonly 和 off，表示 可重写、不可重写和禁用；

```json
{
  "globals":{
    "$": false，// true 表示该变量设置为 writeable，false 表示 readonly
    "jQuery"：false
  }
}
```

在 globals 中一个个进行声明有些繁琐，这时需要用到 env，这是对一个环境定义的一组全局变量的预设；

```json
{
  "env":{
    "browser": true,
    "es2021": true,
    "jquery": true, // 环境中开启jquery，表示声明了jquery相关的全局变量，无需在globals二次声明
  }
}
```

可以在 globals 中使用字符串 off  禁用全局变量来覆盖 env 中的声明；

```json
// 在大多数 ES2015 全局变量可用但 Promise 不可用的环境中
{
    "env": {
        "es6": true
    },
    "globals": {
        "Promise": "off"
    }
}
```

如果是微信小程序开发，env 并没有定义微信小程序变量，需要在 globals 中手动声明全局变量，否则在文件中引入变量，会提示报错；

```javascript
{
  globals: {
    wx: true,
    App: true,
    Page: true,
    Component: true,
    getApp: true,
    getCurrentPages: true,
    Behavior: true,
    global: true,
    __wxConfig: true,
  },
}

```

#### rules(规则)

可以在配置文件的 rules 属性中配置想要的规则，参考 [规则](https://cn.eslint.org/docs/rules/)；

在 rules 中配置的规则，它会覆盖在拓展或插件中引入的规则；

#### plugins(插件)

虽然官方提供了规则可供选择，但是只能检查标准的 JS 语法，如果写的是 JSX 或 TS ，ESLint 的规则不起作用,此时需要安装 ESLint 插件来定制一些特定的规则进行检查；

ESLint 的插件与有固定的命名格式，以 eslint-plugin- 开头，使用时可以省略这个头；

如果要在项目中使用 TypeScript，需要将解析器改为 @typescript-eslint/parser，同时需要安装 @typescript-eslint/eslint-plugin 插件来拓展规则，添加的 plugins 中的规则默认是不开启的，需要在 rules 中选择要使用的规则，也就是说 plugins 是要和 rules 结合使用的；

```json
// npm i --save-dev @typescript-eslint/eslint-plugin 


{
  "parser": "@typescript-eslint/parser",
  "plugins": ["@typescript-eslint"],   // 引入插件
  "rules": {
    "@typescript-eslint/rule-name": "error"    // 使用插件规则
    '@typescript-eslint/adjacent-overload-signatures': 'error',
    '@typescript-eslint/ban-ts-comment': 'error',
    '@typescript-eslint/ban-types': 'error',
    '@typescript-eslint/explicit-module-boundary-types': 'warn',
    '@typescript-eslint/no-array-constructor': 'error',
    'no-empty-function': 'off',
    '@typescript-eslint/no-empty-function': 'error',
    '@typescript-eslint/no-empty-interface': 'error',
    '@typescript-eslint/no-explicit-any': 'warn',
    '@typescript-eslint/no-extra-non-null-assertion': 'error',
    ...
  }
}
```

#### extends（扩展）

> extends 可以理解为是一份配置好的 plugin 和 rules

要注意 ESlint 的解析顺序是按照 **从下往上** 的顺序来加载的

extends 属性值可以是：

1. 指定配置的字符串：比如官方提供的两个拓展eslint:recommended 或 eslint:all，可以启用当前安装的 ESLint 中所有的核心规则，省得在 rules 中一一配置；
2. 字符串数组：每个配置继承它前面的配置，ESLint 会递归的扩展配置，然后使用 rules 属性来拓展或覆盖 extends 配置规则；

```javascript
{
    "extends": [
        "eslint:recommended", // 官方拓展
        "plugin:@typescript-eslint/recommended", // ts插件拓展
        "standard", // npm包，开源社区流行的配置方案，比如：eslint-config-airbnb、eslint-config-standard
    ],
    "rules": {
    	"indent": ["error", 4], // 拓展或覆盖extends配置的规则
        "no-console": "off",
    }
}
```



### 在注释中使用 ESLint

在代码中，也可以使用注释把 lint 规则嵌入到源码中；

1. 块注释：可以在整个文件或代码块中禁用所有规则或禁用特定规则；
2. 行注释：可以在某一特定的行上禁用所有规则可禁用特定

   ```javascript
   /* eslint-disable */
   alert('该注释放在文件顶部，整个文件都不会出现 lint 警告');
   
   /* eslint-disable */
   alert('块注释 - 禁用部分代码的lint');
   /* eslint-enable */
   
   /* eslint-disable no-console, no-alert */
   alert('块注释 - 禁用 no-console, no-alert 特定规则');
   /* eslint-enable no-console, no-alert */
   
   ```


### vue组件选项自动排序

前提在我们项目的 eslint 配置文件中的 extends属性中加入： `"plugin:vue/recommended"`

#### vue/order-in-components（选项式排序）

这个主要是用于处理script**选项式排序**比如data，watch，methods等，但是不处理vue3 setup里面的排序

这个规则在 `"plugin:vue/vue3-recommended"` and `"plugin:vue/recommended"` 是自带的，需要在 eslint 的配置文件里的 rules 属性加上下面的配置，我们也可以按照自己的项目风格手动调整顺序

```javascript
rules: {
  // ...
  {
    "vue/order-in-components": ["error", {
      "order": [
        "el",
        "name",
        "key",
        "parent",
        "functional",
        ["delimiters", "comments"],
        ["components", "directives", "filters"],
        "extends",
        "mixins",
        ["provide", "inject"],
        "ROUTER_GUARDS",
        "layout",
        "middleware",
        "validate",
        "scrollToTop",
        "transition",
        "loading",
        "inheritAttrs",
        "model",
        ["props", "propsData"],
        "emits",
        "setup",
        "asyncData",
        "data",
        "fetch",
        "head",
        "computed",
        "watch",
        "watchQuery",
        "LIFECYCLE_HOOKS",
        "methods",
        ["template", "render"],
        "renderError"
      ]
    }]
  }
}

```

#### [vue/attributes-order （](https://eslint.vuejs.org/rules/attributes-order.html)attributes排序[）](https://eslint.vuejs.org/rules/attributes-order.html)

这个主要是用于处理template 里面attribute的排序，比如class、:xxx、v-if、filterable这种

这个规则在 `"plugin:vue/vue3-recommended"` and `"plugin:vue/recommended"` 是自带的，需要在 eslint 的配置文件里的 rules 属性加上下面的配置，我们也可以按照自己的项目风格手动调整顺序

```json
rules: {
  // ...
  {
    "vue/attributes-order": ["error", {
      "order": [
        "DEFINITION", // 比如is、
        "LIST_RENDERING", // 比如 v-for
        "CONDITIONALS", // 比如 v-if、v-show
        "RENDER_MODIFIERS", // 比如 v-once、v-pre
        "GLOBAL", // 比如 id
        ["UNIQUE", "SLOT"], // 比如 ref、key
        "TWO_WAY_BINDING", // 比如 v-model
        "OTHER_DIRECTIVES", // 比如 v-custom-directive
        "OTHER_ATTR", // 比如 :customProp='xxx'
        "EVENTS", // 比如 @click
        "CONTENT"// 比如 v-html
      ],
      "alphabetical": false // 是否按照字母顺序排列
    }]
  }
}

```



## Prettier vs ESLint

ESLint 和 Prettier 都会对 AST（语法树）进行检查，

- 但 Prettiter 只会进行语法分析（只能检查并归正代码的**格式问题**）
- 而 ESLint 还会进一步对代码进行语义分析，能发现格式问题和代码模式问题（比如用 let 定义了一个变量，后面这个变量没有重新赋值，Preitter 会通过，ESLint 会检查出这里用 const 声明更好）

**上面说到 ESLint 也能做格式化工具，那为什么还需要 Prettier?**

因为 ESLint 只能检查 JavaScript 代码以及 TypeScript、JSX 等衍生代码（需要配置解析器），无法检查项目中的 CSS、HTML 等代码

Prettier 则天然支持对大多数项目文件的格式化，包括 JSX、Vue、TypeScript、CSS、HTML、JSON、Markdown、YAML 等

## 同时使用 Prettier 和 ESLint

仔细了解就会发现，这两种工具在某些文件上的格式化功能是会有所重合的，那如果在项目中同时启用了这两种工具，我们需要保证这些文件只采用其中一种进行格式化，避免这两个工具的风格配置相互冲突，导致一个校验通过一个校验不通过

如果需要同时使用二者，就需要关闭 ESLint 中可能和 Prettier 冲突的规则，这时 **eslint-config-prettier** 就派上用场了

```javascript
npm i eslint-config-prettier --save-dev
```

然后在 .eslintrc.js 配置 extends 属性：

```javascript
extends: [
    // 覆盖eslint格式配置, 需要写在extends最后面
    'prettier',
],

```

完成上面可以实现的是运行 ESLint 命令会按照 Preitter 的规则做相关校验，但是还是要运行 Preitter 进行格式化，可以在使用 eslint --fix 时候，实际使用 Prettier 来替代 ESLint 的格式化功能；

```javascript
// 安装eslint-plugin-prettier
$ npm install --save-dev eslint-plugin-prettier

// 在 .eslintrc.* 文件里面的 extends 字段添加：
{
  "extends": [
    ...,
+   "plugin:prettier/recommended"
  ],
  "rules": {
    "prettier/prettier": "error", // 当代码出现 Prettier 校验出的格式化问题，ESLint会报错；
  }
}

```

此时运行 eslint --fix 实际使用的是 Prettier 去格式化文件；

## VSCode集成

ESLint 原本是一个命令行工具，如果在 VSCode 中使用需要安装 ESLint 插件。

**插件实现的原理也很简单**：

- 在我们写代码的时候，插件在后台自动执行 eslint 命令分析代码，并根据结果实时回显到编辑器中
- 使用插件需要我们当前项目内安装了 ESLint，如果目录中没有安装，插件会尝试使用全局安装的（插件本身并不包含 ESLint 核心库，而是读取本地或全局安装的 ESLint，并使用查找读取项目内的 eslintrc.* 配置文件
- 所以当使用本地安装运行ESlint 程序时，每个项目都是独立的，不会冲突

### 保存自动格式化

1. 安装 Preitter - Code formatter 插件；
2. 打开 setting.json 文件；
3. 添加代码将 Preitter 设置为默认格式化程序；

```javascript
{
  // 设置全部语言在保存时自动格式化
  "editor.formatOnSave": ture,
  // 设置全部语言的默认格式化程序为prettier
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  // 设置特定语言的默认格式化程序为prettier
  "[javascript]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode",
    "editor.formatOnSave": true
  },
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": true, // 为ESLint启用“保存时自动修复”
  }
}


// 支持的语言有如下
javascript;
javascriptreact;
typescript;
typescriptreact;
json;
graphql;

```

## Editor Config

Editor Config 主要是用来解决项目成员之间使用了不同的 IDE 编辑器导致的代码风格不统一的问题

在项目根目录中创建 `editorconfig` 文件，一般这份文件主要是处理字符集、缩进、换行等基础风格

```javascript
# 表示是最顶层的 EditorConfig 配置文件
root = true


# 表示所有文件适用
[*]
charset = utf-8 # 设置文件字符集为 utf-8
indent_style = space # 缩进风格（tab | space）
indent_size = 2 # 缩进大小
end_of_line = lf # 控制换行类型(lf | cr | crlf)
trim_trailing_whitespace = true # 去除行首的任意空白字符
insert_final_newline = true # 始终在文件末尾插入一个新行


[*.{yml,yaml,json}]
indent_style = space
indent_size = 2


[*.md]
max_line_length = off
trim_trailing_whitespace = false


[Makefile]
indent_style = tab

```