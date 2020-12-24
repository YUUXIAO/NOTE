## 表单绑定

### 基础使用

```javascript
// 普通输入框
<input type="text" v-model="value1">

// 多行文本框
<textarea v-model="value2" cols="30" rows="10"></textarea>

// 单选框
<div class="group">
  <input type="radio" value="one" v-model="value3"> one
  <input type="radio" value="two" v-model="value3"> two
</div> 

// 多选框  (原始值： value4: [])
<div class="group">
  <input type="checkbox" value="jack" v-model="value4">jack
  <input type="checkbox" value="lili" v-model="value4">lili
</div>

// 下拉选项
<select name="" id="" v-model="value5">
  <option value="apple">apple</option>
  <option value="banana">banana</option>
  <option value="bear">bear</option>
</select>
```

### AST 树的解析

Vue 模板属性由两部分组成，一部分是指令，另一部分是普通 html 标签属性；

在指令的细分领域，又将 v-on，v-bind 做特殊处理，其它的普通分支会执行 addDirective 过程；

```javascript
// 处理模板属性
function processAttrs(el) {
  var list = el.attrsList;
  var i, l, name, rawName, value, modifiers, syncGen, isDynamic;
  for (i = 0, l = list.length; i < l; i++) {
    name = rawName = list[i].name; // v-on:click
    value = list[i].value; // doThis
    if (dirRE.test(name)) { // 1.针对指令的属性处理
      ···
      if (bindRE.test(name)) { // v-bind分支
        ···
      } else if(onRE.test(name)) { // v-on分支
        ···
      } else { // 除了v-bind，v-on之外的普通指令
        ···
        // 普通指令会在AST树上添加directives属性
        addDirective(el, name, rawName, value, arg, isDynamic, modifiers, list[i]);
        if (name === 'model') {
          checkForAliasModel(el, value);
        }
      }
    } else {
      // 2. 普通html标签属性
    }

  }
}
```



## 组件使用