## Prettier

> Prettier 是一个专注于代码格式化的工具，美化代码；

Preitter 通过解析代码并匹配自己的一套规则，来强制执行一致的代码展示格式；

### 配置

一共有三种方式支持对 Preitter 进行配置：

1. 根目录下创建 .preitter 文件，能够写入 YML、JSON 的配置格式，并且支持 .yml/.yaml/.json/.js 后缀；
2. 根目录下创建 .prettier.config.js 文件，并且对外 export 一个对象；
3. 在 package.json 中新建 preitter 属性；

### 配合ESLint

在 ESLint 与 Preitter 合作时，对于它们交集的部分规则，会出现冲突；

可以在 ESLint 的配置文件上进行修改，安装特定的 plugin，把其配置在规则的尾部，实现 Preitter 规则对 ESLint 规则的覆盖；

```javascript
// 安装eslint-config-prettier
$ npm install --save-dev eslint-config-prettier

// 在 .eslintrc.* 文件里面的 extends 字段添加：
{
  "extends": [
    ...,
    "已经配置的规则",
+   "prettier",
+   "prettier/@typescript-eslint"
  ]
}

```

完成上面可以实现的是运行 ESLint 命令会按照 Preitter 的规则做相关校验，但是还是要运行 Preitter 进行格式化，可以在使用 eslint --fix 时候，实际使用 Prettier 来替代 ESLint 的格式化功能；

```javascript
// 安装eslint-plugin-prettier
$ npm install --save-dev eslint-plugin-prettier

// 在 .eslintrc.* 文件里面的 extends 字段添加：
{
  "extends": [
    ...,
    "已经配置的规则",
+   "plugin:prettier/recommended"
  ],
  "rules": {
    "prettier/prettier": "error",
  }
}

```

此时运行 eslint --fix 实际使用的是 Prettier 去格式化文件；

在 rules 中添加 "prettier/prettier": "error"，当代码出现 Prettier 校验出的格式化问题，ESLint会报错；

## ESlint

> ESlint 是一个作代码质量检测、编码风格约束等；

### 作用&优势

1. 检查语法错误，避免低级 bug；
   - 比如 api 语法错误、使用了未定义的变量、修改 const 变量；
2. 统一团队代码风格；
   - 比如使用 tab 还是空格，使用单引号还是双引号；
3. 确保代码遵循最佳实践；
   - 比如可以借助 eslint-config-standard 配置包扩展社区中流行的最佳实践的风格指南；

### 配置

ESlint 支持配置几种格式的配置文件：

1. Javascript：使用 .eslintrc.js 然后输出一个配置对象；
2. YAML：使用 .eslintrc.yaml 或 .eslintrc.yml 去定义配置的结构；
3. JSON：使用 .eslintrc.json 去定义配置的结构，ESLint 的 JSON 文件允许 JavaScript 风格的注释；
4. package.json：在 package.json 里创建一个 eslintConfig 属性定义你的配置；

如果在项目内有多个层叠配置，ESlint 只会使用一个，优先级如下：

1. .eslintrc.js
2. .eslintrc.yaml
3. .eslintrc.yml
4. .eslintrc.json
5. .eslintrc
6. package.json

### 配置项解析

#### parser-解析器

ESlint 默认使用 Espree 作其解析器，但该解析器仅支持最新的 ES5 标准，对于实验性的语法和非标准（例如 Flow 或 TS 类型）语法是不支持的，所以开源社区提供了两种解析器来丰富 TSLint 的功能：

1. bable-eslint：Babel  主要用于将  ES6+ 版本的代码转换为向后兼容的 js 语法，如果在项目中使用 es6，就需要将解析器改成 babel-eslint；
2. @typescript-eslint/parser：该解析器将 TS 转换成 estree 兼容的形式，允许 ESLint 验证了 ts 源代码；

#### parserOptions-解析器选项

ESLint 允许你指定你想要的支持的 js 语言选项，默认情况，ESLint 支持 ECMAScript 5 语法，可以覆盖该设置，启用对 ECMAScript 其它版本和 JSX 的支持；

可用的选项有：

1. ecmaVersion：可以使用 6、7、8、9、10 指定要使用的 ECMAScript 版本，也可以使用年份命名的版本号；
2. sourceType：设置为 script（默认）或 module （代码是 ECMAScript 模块）；
3. ecmaFeatures：这是个对象，表示想使用的额外的语言特性：
   - globalReturn：允许在全局作用域下使用 return 语句；
   - impliedStrict：启用全局 strict mode；
   - jsx：使用 JSX；

#### env & globals-环境变量和全局变量

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

可以在 golbals 中使用字符串 off  禁用全局变量来覆盖 env 中的声明；

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

#### rules-规则

可以在配置文件的 rules 属性中配置想要的规则，参考 [规则](https://cn.eslint.org/docs/rules/)；

在 rules 中配置的规则，它会覆盖在拓展或插件中引入的规则；

#### plugins-插件

虽然官方提供了规则可供选择，但是只能检查标准的 JS 语法，如果写的是 JSX 或 TS ，ESLint 的规则不起作用；

 此时需要安装 ESLint 插件来定制一些特定的规则进行检查；

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

#### extends-扩展

> extends 可以理解为是一份配置好的 plugin 和 rules；

extends 属性值可以是：

1. 指定配置的字符串：比如官方提供的两个拓展eslint:recommended 或 eslint:all，可以启用当前安装的 ESLint 中所有的核心规则，省得在 rules 中一一配置；
2. 字符串数组：每个配置继承它前面的配置，ESLint 会递归的扩展配置，然后使用 rules 属性来拓展或覆盖 extends 配置规则；

```javascript
{
    "extends": [
        "eslint:recommended", // 官方拓展
        "plugin:@typescript-eslint/recommended", // 插件拓展
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

   ```javascript
   /* eslint-disable */
   alert('该注释放在文件顶部，整个文件都不会出现 lint 警告');

   /* eslint-disable */
   alert('块注释 - 禁用所有规则');
   /* eslint-enable */

   /* eslint-disable no-console, no-alert */
   alert('块注释 - 禁用 no-console, no-alert 特定规则');
   /* eslint-enable no-console, no-alert */

   ```

2. 行注释：可以在某一特定的行上禁用所有规则可禁用特定规则；

   ```javascript
   /* eslint-disable */
   alert('该注释放在文件顶部，整个文件都不会出现 lint 警告');

   /* eslint-disable */
   alert('块注释 - 禁用所有规则');
   /* eslint-enable */

   /* eslint-disable no-console, no-alert */
   alert('块注释 - 禁用 no-console, no-alert 特定规则');
   /* eslint-enable no-console, no-alert */

   ```

### 比较 Prettier

1. ESLint 主要是检查代码质量并给出提示，所能提供的格式化功能较少；
2. Preitter 在格式化代码方面具有更大的优势；

## VSCode集成

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