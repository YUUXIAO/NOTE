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

AST 产生阶段对事件指令 v-on 的处理方式是为 AST 添加 events 属性，普通指令会在 AST 树上添加 directives 属性；

```javascript
// 添加directives属性
function addDirective (el,name,rawName,value,arg,isDynamicArg,modifiers,range) {
  (el.directives || (el.directives = [])).push(rangeSetItem({
    name: name,
    rawName: rawName,
    value: value,
    arg: arg,
    isDynamicArg: isDynamicArg,
    modifiers: modifiers
  }, range));
  el.plain = false;
}
```

最终 AST 树多了到今天为止 属性对象，其中 modifiers 代表模板中添加的修饰符，如：lazy，number，trim 等；

```javascript
// AST
{
  directives: {
    {
      rawName: 'v-model',
      value: 'value',
      name: 'v-model',
      modifiers: undefined
    }
  }
}
```

### render函数生成

- input 标签所有属性，包括指令相关的内容都是以 data 属性的形式作为参数的整体传入 _c 函数；
- input type 的类型，在 data 属性中，以 attrs 键值对存在；
- v-model 会有对应的 directives 属性描述指令的相关信息；
- 从 render 函数的结果看出，它最终以两部分形式存在于 input 标签中，一个是将 value 以 props 的形式存在（domProps）中，另一个是以事件的形式存储 input 事件，并保留在 on 属性中；
- 事件用 $event.target.composing 属性来保证不会在输入法组合文字的过程中更新数据；

render 函数生成阶段，其中 genData 会对模板的诸多属性进行处理，最终返回拼接好的字符串模板，而对指令的处理会进入 genDirectives 流程；

```javascript
function genData(el, state) {
  var data = '{';
  // 指令的处理
  var dirs = genDirectives(el, state);
  // 其他属性，指令的处理
  ··· 
  // 针对组件的v-model处理，放到后面分析
  if (el.model) {
    data += "model:{value:" + (el.model.value) + ",callback:" + (el.model.callback) + ",expression:" + (el.model.expression) + "},";
  }
  return data
}
```

#### genDirectives

genDirectives 会拿到之前 AST 树中保留的 directives 对象，并遍历解析指令对象，最终以 ' directoves:[ ' 包裹的字符串返回；

```javascript
// directives render字符串的生成
function genDirectives (el, state) {
  // 拿到指令对象
  var dirs = el.directives;
  if (!dirs) { return }
  // 字符串拼接
  var res = 'directives:[';
  var hasRuntime = false;
  var i, l, dir, needRuntime;
  for (i = 0, l = dirs.length; i < l; i++) {
    dir = dirs[i];
    needRuntime = true;
    // 对指令ast树的重新处理
    var gen = state.directives[dir.name];
    if (gen) {
      // compile-time directive that manipulates AST.
      // returns true if it also needs a runtime counterpart.
      needRuntime = !!gen(el, dir, state.warn);
    }
    if (needRuntime) {
      hasRuntime = true;
      res += "{name:\"" + (dir.name) + "\",rawName:\"" + (dir.rawName) + "\"" + (dir.value ? (",value:(" + (dir.value) + "),expression:" + (JSON.stringify(dir.value))) : '') + (dir.arg ? (",arg:" + (dir.isDynamicArg ? dir.arg : ("\"" + (dir.arg) + "\""))) : '') + (dir.modifiers ? (",modifiers:" + (JSON.stringify(dir.modifiers))) : '') + "},";
    }
  }
  if (hasRuntime) {
    return res.slice(0, -1) + ']'
  }
}
```

Vue 中针对浏览器端有三个重要的指令选项：

```javascript
var directive$1 = {
  model: model,
  text: text,
  html, html
}
var baseOptions = {
  ···
  // 指令选项
  directives: directives$1,
};
// 编译时传入选项配置
createCompiler(baseOptions)
```

state.directives[‘model’] 就是对应 model 函数；

- model 会对表单控件的 AST 树做进一步处理，表单有不同的类型，每种类型对应的事件处理响应机制也不同；
- 需要针对不同的表单控件生成不同的 render 函数，因此需要产生不同的 AST 属性；

```javascript
function model (el,dir,_warn) {
  warn$1 = _warn;
  // 绑定的值
  var value = dir.value;
  var modifiers = dir.modifiers;
  var tag = el.tag;
  var type = el.attrsMap.type;
  {
    // 这里遇到type是file的html，如果还使用双向绑定会报出警告。
    // 因为File inputs是只读的
    if (tag === 'input' && type === 'file') {
      warn$1(
        "<" + (el.tag) + " v-model=\"" + value + "\" type=\"file\">:\n" +
        "File inputs are read only. Use a v-on:change listener instead.",
        el.rawAttrsMap['v-model']
      );
    }
  }
  //组件上v-model的处理
  if (el.component) {
    genComponentModel(el, value, modifiers);
    // component v-model doesn't need extra runtime
    return false
  } else if (tag === 'select') {
    // select表单
    genSelect(el, value, modifiers);
  } else if (tag === 'input' && type === 'checkbox') {
    // checkbox表单
    genCheckboxModel(el, value, modifiers);
  } else if (tag === 'input' && type === 'radio') {
    // radio表单
    genRadioModel(el, value, modifiers);
  } else if (tag === 'input' || tag === 'textarea') {
    // 普通input，如 text, textarea
    genDefaultModel(el, value, modifiers);
  } else if (!config.isReservedTag(tag)) {
    genComponentModel(el, value, modifiers);
    // component v-model doesn't need extra runtime
    return false
  } else {
    // 如果不是表单使用v-model，同样会报出警告，双向绑定只针对表单控件。
    warn$1(
      "<" + (el.tag) + " v-model=\"" + value + "\">: " +
      "v-model is not supported on this element type. " +
      'If you are working with contenteditable, it\'s recommended to ' +
      'wrap a library dedicated for that purpose inside a custom component.',
      el.rawAttrsMap['v-model']
    );
  }
  // ensure runtime directive metadata
  // 
  return true
}
```

#### genDefaultModel

> 普通 input 标签的处理；

1. 针对修饰符产生不同的事件处理字符串；
2. 为 v-model 产生的 AST 树添加属性和事件相关的属性；

```javascript
function genDefaultModel (el,value,modifiers) {
    var type = el.attrsMap.type;

    // v-model和v-bind值相同值，有冲突会报错
    {
      var value$1 = el.attrsMap['v-bind:value'] || el.attrsMap[':value'];
      var typeBinding = el.attrsMap['v-bind:type'] || el.attrsMap[':type'];
      if (value$1 && !typeBinding) {
        var binding = el.attrsMap['v-bind:value'] ? 'v-bind:value' : ':value';
        warn$1(
          binding + "=\"" + value$1 + "\" conflicts with v-model on the same element " +
          'because the latter already expands to a value binding internally',
          el.rawAttrsMap[binding]
        );
      }
    }
    // modifiers存贮的是v-model的修饰符。
    var ref = modifiers || {};
    // lazy,trim,number是可供v-model使用的修饰符
    var lazy = ref.lazy;
    var number = ref.number;
    var trim = ref.trim;
    var needCompositionGuard = !lazy && type !== 'range';
    // lazy修饰符将触发同步的事件从input改为change
    var event = lazy ? 'change' : type === 'range' ? RANGE_TOKEN : 'input';

    var valueExpression = '$event.target.value';
    // 过滤用户输入的首尾空白符
    if (trim) {
      valueExpression = "$event.target.value.trim()";
    }
    // 将用户输入转为数值类型
    if (number) {
      valueExpression = "_n(" + valueExpression + ")";
    }
    // genAssignmentCode函数是为了处理v-model的格式，允许使用以下的形式： v-model="a.b" v-model="a[b]"
    var code = genAssignmentCode(value, valueExpression);
    if (needCompositionGuard) {
      //  保证了不会在输入法组合文字过程中得到更新
      code = "if($event.target.composing)return;" + code;
    }
    //  添加value属性
    addProp(el, 'value', ("(" + value + ")"));
    // 绑定事件
    addHandler(el, event, code, null, true);
    if (trim || number) {
      addHandler(el, 'blur', '$forceUpdate()');
    }
  }

function genAssignmentCode (value,assignment) {
  // 处理v-model的格式，v-model="a.b" v-model="a[b]"
  var res = parseModel(value);
  if (res.key === null) {
    // 普通情形
    return (value + "=" + assignment)
  } else {
    // 对象形式
    return ("$set(" + (res.exp) + ", " + (res.key) + ", " + assignment + ")")
  }
}

```

addHandler 会为 AST 树添加事件相关的属性；

addProp 会为 AST 树添加 props 属性；

![img](https://user-gold-cdn.xitu.io/2019/9/5/16d002b80f291140?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

#### genData$2

通过 genDirectives 处理后，原先的 AST 树新增了两个属性，因此在字符串生成阶段同样需要处理 props 和 events 的分支；

```javascript
function genData$2 (el, state) {
  var data = '{';
  // 已经分析过的genDirectives
  var dirs = genDirectives(el, state);
  // 处理props
  if (el.props) {
    data += "domProps:" + (genProps(el.props)) + ",";
  }
  // 处理事件
  if (el.events) {
    data += (genHandlers(el.events, false)) + ",";
  }
}
```

最终 < input type="text" v-model="value" >  的 render 函数的结果是：

```javascript
"_c('input',{directives:[{name:"model",rawName:"v-model",value:(message),expression:"message"}],attrs:{"type":"text"},domProps:{"value":(message)},on:{"input":function($event){if($event.target.composing)return;message=$event.target.value}}})"
```

### patch真实节点

在生成 vnode 的过程中，所有指令、属性会以 data 属性的形式传递到构造函数 Vnode 中，最终的 Vnode 拥有 directives，domprops，on 属性；

![img](https://user-gold-cdn.xitu.io/2019/9/5/16d002e92c477150?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

有了 Vnode 后会执行 patchVnode，patchVnode 过程是一个真实节点创建的过程，其中关键的是 createElm 方法，对相关属性的处理在 invokeCreateHooks 逻辑；

invokeCreateHooks 会调用定义好的钩子函数，对 vnode 上定义的属性，指令，事件等进行真实 DOM 的处理：

1. updateDOMProps 会利用 vnode data 上的 domProps 更新 input 标签的 value 值；
2. updateAttrs 会利用 vnode data 上的 attrs 属性更新节点的属性值；
3. updateDomListeners 利用 vnode data 上的 on 属性添加事件监听；

```javascript
function createElm() {
  ···
  // 针对指令的处理
   if (isDef(data)) {
      invokeCreateHooks(vnode, insertedVnodeQueue);
    }
}
```

## 组件使用 

> 在组件上使用 v-model 本质上是子父组件通信的语法糖；

### 基本用法

```javascript
 var child = {
    template: '<div><input type="text" :value="value" @input="emitEvent">{{value}}</div>',
    methods: {
      emitEvent(e) {
        this.$emit('input', e.target.value)
      }
    },
    props: ['value']
  }
 
 new Vue({
   data() {
     return {
       message: 'test'
     }
   },
   components: {
     child
   },
   template: '<div id="app"><child v-model="message"></child></div>',
   el: '#app'
 })
```

### AST 树的解析

AST 生成阶段和普通表单控件的区别在于，当遇到  child 时，由于不是普通的 html 标签，会执行 getComponentModel 的过程，而 getComponentModel 的结果是在 AST 树上添加 model 属性；

```javascript
function model() {
  if (!config.isReservedTag(tag)) {
    genComponentModel(el, value, modifiers);
  }
}

function genComponentModel (el,value,modifiers) {
  var ref = modifiers || {};
  var number = ref.number;
  var trim = ref.trim;

  var baseValueExpression = '?v';
  var valueExpression = baseValueExpression;
  if (trim) {
    valueExpression =
      "(typeof " + baseValueExpression + " === 'string'" +
      "? " + baseValueExpression + ".trim()" +
      ": " + baseValueExpression + ")";
  }
  if (number) {
    valueExpression = "_n(" + valueExpression + ")";
  }
  var assignment = genAssignmentCode(value, valueExpression);
  // 在ast树上添加model属性，其中有value，expression，callback属性
  el.model = {
    value: ("(" + value + ")"),
    expression: JSON.stringify(value),
    callback: ("function (" + baseValueExpression + ") {" + assignment + "}")
  };
}
```

最终 AST 树的结果：

```javascript
{
  model: {
    callback: "function ($$v) {message=$$v}"
    expression: ""message""
    value: "(message)"
  }
}
```

### render 函数的生成

经过对 AST 树的处理，回到 genData$2 的流程，由于有了 model 属性，父组件拼接字符串会做进一步处理；

```javascript
function genData$2 (el, state) { 
  var data = '{';
  var dirs = genDirectives(el, state);
  ···
  // v-model组件的render函数处理
  if (el.model) {
    data += "model:{value:" + (el.model.value) + ",callback:" + (el.model.callback) + ",expression:" + (el.model.expression) + "},";
  }
  ···
  return data
}
```

最终父组件 render 函数的结果为：

```javascript
"_c('child',{model:{value:(message),callback:function (?v) {message=?v},expression:"message"}})"
```

子组件的创建阶段会执行 createComponent，对于 model 的逻辑：

```javascript
function createComponent() {
  // transform component v-model data into props & events
  if (isDef(data.model)) {
    // 处理父组件的v-model指令对象
    transformModel(Ctor.options, data);
  }
}
```

#### transformModel

- 子组件 vnode 会为 data.props 添加 data.model.value，并且给 data.on 添加 data.model.callback；
- 父组件 v-model 语法糖的本质上可以理解为 < child :value="message" @input="function(e){message = e}" >< /child  > 

```javascript
function transformModel (options, data) {
  // prop默认取的是value，除非配置上有model的选项
  var prop = (options.model && options.model.prop) || 'value';

  // event默认取的是input，除非配置上有model的选项
  var event = (options.model && options.model.event) || 'input'
  // vnode上新增props的属性，值为value
  ;(data.attrs || (data.attrs = {}))[prop] = data.model.value;

  // vnode上新增on属性，标记事件
  var on = data.on || (data.on = {});
  var existing = on[event];
  var callback = data.model.callback;
  if (isDef(existing)) {
    if (
      Array.isArray(existing)
        ? existing.indexOf(callback) === -1
        : existing !== callback
    ) {
      on[event] = [callback].concat(existing);
    }
  } else {
    on[event] = callback;
  }
}
```

