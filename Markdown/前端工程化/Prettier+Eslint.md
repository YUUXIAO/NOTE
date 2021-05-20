## Prettier

> Prettier 是一个专注于代码格式化的工具，美化代码；

Preitter 通过解析代码并匹配自己的一套规则，来强制执行一致的代码展示格式；

### 配置

一共有三种方式支持对 Preitter 进行配置：

1. 根目录下创建 .preitter 文件，能够写入 YML、JSON 的配置格式，并且支持 .yml/.yaml/.json/.js 后缀；
2. 根目录下创建 .prettier.config.js 文件，并且对外 export 一个对象；
3. 在 package.json 中新建 preitter 属性；

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

安装插件：

```javascript
npm i -D eslint-plugin-prettier
```

eslint-plugin-prettier 插件会调用 prettier 对代码风格进行检查，其原理是先使用 preitter 对代码进行格式化，然后与格式化之前的代码进行对比，如果出现不一致，这个地方会被 preitter 进行标记；

```javascript
// .eslintre.json
{
  "plugins": ["prettier"],
  "rules": {
    "prettier/prettier": "error"
  }
}
```

当 eslint 与 preitter 的规则冲突时，最简单的方式就是使用 eslint-config-preitter 