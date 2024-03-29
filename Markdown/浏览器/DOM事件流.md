## DOM事件级别

DOM有4次版本更新，与 DOM 版本变更，由于DOM 1级中没有事件的相关内容，所以没有DOM 1级事件；

### DOM 0级事件处理

1. on-event 写法绑定事件：

   ```javascript
   // HTML 属性
   <input onclick="alert('xxx')"/>
   
   // 非HTML 属性
   window.onload = function(){
     document.write("Hello world!");
   };
   
   // 实体元素
   <button id="btn">Click</button>
   var btn = document.getElementById('btn');
   btn.onclick = function(){
        alert('xxx');
   }
   
   // 如果要解除事件，重新指定 on-event 为 null
   btn.onclick = null
   ```

2. 同一个元素的同一种事件只能绑定一个函数，否则后面的函数会覆盖之前的函数；
3. 不存在兼容性问题；
4. ie事件模型在这种事件模型中，一次事件有两个过程，1是事件处理阶段、2是事件冒泡阶段

### DOM 2级事件处理

1. Dom 2级事件是通过 addEventListener 绑定的事件；
2. 同一个元素的同种事件可以绑定多个函数，按照绑定顺序执行；
3. 解绑 Dom 2级事件时，使用 removeEventListener；

   - removeEventListener（）不能移除匿名添加的函数；

4. 在这种事件模型中，一次事件有三个过程，1是事件捕获阶段，2是事件处理阶段、3是事件冒泡阶段

### DOM 3级事件处理

DOM 3级事件在DOM2级事件的基础上添加了更多的事件类型，可以理解为**丰富了很多交互式的事件**；

1. **UI事件**，当用户与页面上的元素交互时触发，如：load、scroll；
2. **焦点事件**，当元素获得或失去焦点时触发，如：blur、focus；
3. **鼠标事件**，当用户通过鼠标在页面执行操作时触发如：dblclick、mouseup；
4. **滚轮事件**，当使用鼠标滚轮或类似设备时触发，如：mousewheel；
5. **文本事件**，当在文档中输入文本时触发，如：textInput；
6. **键盘事件**，当用户通过键盘在页面上执行操作时触发，如：keydown、keypress；
7. **合成事件**，当为IME（输入法编辑器）输入字符时触发，如：compositionstart；
8. **变动事件**，当底层DOM结构发生变化时触发，如：DOMsubtreeModified；
9. DOM 3级事件允许使用者自定义一些事件；

## DOM事件流

事件流是指**网页元素接收事件的顺序**；事件流可以分成两种机制：

1. **IE 的事件流是事件冒泡流（dom0级）；**
2. **标准浏览器的事件流是事件捕获流（dom1级）；**

当一个事件发生后，会在子元素和父元素之间传播，分为三个阶段：

- **捕获阶段：**事件从window对象自上而下向目标节点传播的阶段；
- **目标阶段：**真正的目标节点正在处理事件的阶段；
- **冒泡阶段：**事件从目标节点自下而上向 window 对象传播的阶段；

### 事件冒泡

事件冒泡指的是从事件开始的具体元素，一级级向上传播到较不具体的节点；

现代浏览器都支持事件冒泡，IE9、Firefox、Chrome和Safari则将事件一直冒泡到window对象；

### 事件捕获

事件捕获指的是从启动事件的元素节点开始，逐层往下传递，直到最下层节点；

IE9、Firefox、Chrome和Safari目前也支持这种事件流模型，但是有些老版本的浏览器不支持，所以很少人使用事件捕获，而是用事件冒泡的多一点；

### DOM事件流

“DOM2级事件”规定的事件流包括三个阶段：事件捕获阶段、处于目标阶段和事件冒泡阶段；

多数支持 DOM 事件流的浏览器都实现了一种特定的行为；即使“DOM2级事件”规范明确要求捕获阶段不会涉及事件目标，但IE9、Safari、Chrome、Firefox和Opera9.5及更高版本都会在捕获阶段触发事件对象上的事件，结果就是有两个机会在目标对象上操作事件；

![img](https://pic1.zhimg.com/80/v2-4de189d2a42b1e8c74b379e067b67578_720w.jpg)

### 事件处理程序

> 事件处理程序（事件监听器）是指响应某个事件的函数，事件处理程序的名字以 on 开头；

DOM2 提供了两个方法让我们处理和删除事件处理程序；

1. addEventListener（）；
2. removeEventListernr（）；

```javascript
btn.addEventListener(eventType, function () {
}, false);
 
// 该方法应用至dom节点
// 第一个参数为事件名
// 第二个为事件处理程序
// 第三个为布尔值，true为事件捕获阶段调用事件处理程序，false为事件冒泡阶段调用事件处理程序
```

1. 当容器元素及嵌套元素，既在捕获阶段又在冒泡阶段调用事件处理程序时：事件按 DOM 事件流的顺序执行事件处理程序；
2. 当事件处理目标阶段时，事件调用顺序决定于绑定事件的第三个参数；
3. 为了最大限度的兼容，一般都是将事件处理程序添加到事件冒泡阶段；

### 兼容IE浏览器

IE 下添加和删除事件处理程序的写法有点小区别；

```javascript
var EventUtil = {
    addHandler: function (el, type, handler) {
        if (el.addEventListener) {
            el.addEventListener(type, handler, false);
        } else {
            el.attachEvent('on' + type, handler);
        }
    },
    removeHandler: function (el, type, handler) {
        if (el.removeEventListener) {
            el.removeEventListerner(type, handler, false);
        } else {
            el.detachEvent('on' + type, handler);
        }
    }
};
```

### 事件对象

> 触发dom上的某个事件时，会产生一个事件对象，里面包含着所有和事件有关的信息；

```javascript
currentTarget  // 事件处理程序当前正在处理事件的那个元素
preventDefault // 取消事件默认行为,比如链接的跳转
stopPropagation // 取消事件冒泡
target  // 事件的目标
```

### 事件委托&事件代理

因为每个函数都是对象，都会占用内存，内存中的对象越多性能越差，使用事件委托或者事件代理可以让我们大量减少浏览器对元素的监听；

- **事件委托就是利用事件冒泡，将事件委托给父节点：**简单俩说就是在父元素上注册一个事件，在这个事件里处理多个子元素的相同事件

  ```javascript
  const container = document.getElementById('myContainer');
  container.addEventListener('click', function(event) {
    if (event.target.classList.contains('button')) {
      console.log('按钮被点击');  // 处理按钮的点击事件
    } else if (event.target.classList.contains('link')) {
      console.log('链接被点击'); // 处理连接的点击事件
    }
  });
  ```

- **事件代理也是利用事件冒泡，父节点处理多个子元素的相同事件**：（其实我感觉这个和事件委托在使用层面上是差不多意思，都是为了减少对事件处理程序过多的问题）

  ```javascript
  const list = document.getElementById('myList');
    list.addEventListener('click', function(event) {
      if (event.target.tagName === 'LI') {
        console.log('项目被点击');
      }
    });
  ```


要避免直接对 document.body 进行事件委托，可能会带来额外的问题：

- **增加合成器线程于主线程的通信开销：**浏览器在进行页面渲染的时候会有合成的步骤，合成的过程会先将页面分成不同的合成层，而用户与浏览器进行交互的时候需要接收事件。此时浏览器会将页面上具有事件处理程序的区域进行标记，被标记的区域会与主线程进行通信。
- 如果我们  document.body 上被绑定了事件，这时候整个页面都会被标记。即使我们的页面不关心某些部分的用户交互，合成器线程也必须与主线程进行通信，并在每次事件发生时进行等待。这种情况也可以使用 passive: true 选项来解决，但其实这种方案也是得不偿失的：

  - 产生等待是因为合成器线程于主线程进行通信。passive 设置为 true 时，表示 listener 永远不会调用 preventDefault。但是passive 默认为false，这引入了处理某些触摸事件（以及其他）的事件监听器在尝试处理滚动时阻止浏览器的主线程的可能性，从而导致滚动处理期间性能可能大大降低。
