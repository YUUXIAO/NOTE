Chrome 插件是一个用 Web 技术开发、用来增强浏览器功能的软件，它其实就是一个由HTML、CSS、JS、图片等资源组成的一个.crx 后缀的压缩包；

## 可以做些什么

1. 书签控制；
2. 下载控制；
3. 窗口控制；
4. 标签控制；
5. 网络请求控制，各类事件监听；
6. 自定义原生菜单；
7. 完善的通信机制；
8. 等等；

## 开发环境与调试

Chorme 插件没有严格的项目结构要求，只要保证目录下有一个 manifest.json 文件，不需要专门的 IDE ，普通的  web 开发工具即可；

从右上角菜单->更多工具->扩展程序可以进入 插件管理页面，也可以直接在地址栏输入 [chrome://extensions]() 访问；

勾选开发者模式即可以文件夹的形式直接加载插件，否则只能安装.crx格式的文件；

### 调试



## 项目结构

### manifest.json

> 这个是 Chrome 插件最重要的文件，用来配置所有和插件相关的配置，必须放在根目录下；

就像 node.js 的package.json一样，每一个插件必须有一个manifest.json，作为最初配置文件；

manifest_version、name、version 三个配置项是必不可少的，description 和 icons 是推荐的；

```javascript
{
  "manifest_version": 2,  //【固定值】
  "name": "我的扩展程序",  // 插件的名字
  "version": "版本字符串", //  插件的版本
  "description": "", // 插件简介栏
}
```

### content-scripts

> 这是 Chrome 插件中向页面注入脚本的一种形式；

虽然名为 script ，但还包括 css；

借助 content-scripts 可以实现通过配置的方式向指定的页面注入 JS 和 CSS，比如广告屏蔽、页面CSS 定制等；

1. 通过 content_scripts 注入的CSS优先级非常高，几乎仅次于浏览器默认样式，所以尽量避免写一些影响全局的样式；

```javascript
{
  // 需要直接注入页面的JS
  "content_scripts": 
  [
      {
          //"matches": ["http://*/*", "https://*/*"],
          // "<all_urls>" 表示匹配所有地址
          "matches": ["<all_urls>"],
          // 多个JS按顺序注入
          "js": ["js/jquery-1.8.3.js", "js/content-script.js"],
          // JS的注入可以随便一点，但是CSS的注意就要千万小心了，因为一不小心就可能影响全局样式
          "css": ["css/custom.css"],
          // 代码注入的时间，可选值： "document_start", "document_end", or "document_idle"，最后一个表示页面空闲时，默认document_idle
          "run_at": "document_start"
      }
  ],
}
```

content-scripts 和原始页面共享 DOM，但是不共享 JS，如果要访问页面 JS，比如某个 JS 变量，只能通过 injected js 来实现；

content-scripts 不能访问绝大部分 Chrome.xxx.api，除了下面4种；

- chrome.extension(getURL , inIncognitoContext , lastError , onRequest , sendRequest)
- chrome.i18n
- chrome.runtime(connect , getManifest , getURL , id , onConnect , onMessage , sendMessage)
- chrome.storage

如果非要调用其它的 API，可以通过通信来实现让 background 来帮助调用； 

###  background

> 这是一个常驻页面，它的生命周期是插件中所有类型页面中最长的，随着浏览器的打开而打开，关闭而关闭，通常把需要一直运行的、启动就运行的、全局的代码放在 background 中；

background 的权限非常高，几乎可以调用所有的 Chrome 扩展API（除了devtools），而且它可以无限制跨域，也就是可以跨域访问任何网站而无需要求对方设置 CORS；

配置中，background 可以通过 page 指定一个网页，也可以通过 scripts 直接指定一个 JS，Chrome 会自动为这个 JS 生成一个默认的网页；

```javascript
{
	"background":{
		"page": "background.html", // 方式一: 直接指定一个网页
		//"scripts": ["js/background.js"]  // 方式二: 指定一个 JS，会自动生成一个背景页面
	},
}
```

### event-pages

鉴于background生命周期太长，长时间挂载后台可能会影响性能，所以Google又弄一个 event-pages ，在配置文件上，它与 background 的唯一区别就是多了一个 persistent 参数：

```javascript
{
	"background":{
		"scripts": ["event-page.js"],
		"persistent": false
	},
}
```

它的生命周期是：在被需要时加载，在空闲时被关闭，什么叫被需要时呢？比如第一次安装、插件更新、有content-script向它发送消息，等等。

除了配置文件的变化，代码上也有一些细微变化，一般情况下background也不会很消耗性能的。

### popup

> popup 就是点击 browser_action 或者 page_action 图标时打开的一个小窗口网页，焦点离开网页就立即关闭，一般用来做一些临时性的交互；

popup 可以包含任意你想要的HTML内容，并且会自适应大小。可以通过 default_popup 字段来指定popup页面，也可以调用 setPopup() 方法；

```javascript
{
	"browser_action":{
		"default_icon": "img/icon.png",
		"default_title": "这是一个示例Chrome插件",  // 图标悬停时的标题，可选
		"default_popup": "popup.html"
	}
}
```

由于单击图标打开popup，焦点离开又立即关闭，所以popup页面的生命周期一般很短，需要长时间运行的代码千万不要写在popup里面。

在权限上，它和background非常类似，它们之间最大的不同是生命周期的不同，popup中可以直接通过 chrome.extension.getBackgroundPage() 获取background的window对象。

### injected-script

> 指的是通过 DOM 操作的方式向页面注入的一种  JS；

虽然有 Content-script 可以向页面中注入 JS 代码，但是 DOM 不能操作它，也就是无法在 DOM 中通过绑定事件的方式调用 content-script 中的代码，包括直接写 onclick 和 addEventListener 的方式；

在 content-script 中通过DOM方式向页面注入 inject-script 代码示例：

```javascript
// 向页面注入JS
function injectCustomJs(jsPath){
	jsPath = jsPath || 'js/inject.js';
	var temp = document.createElement('script');
	temp.setAttribute('type', 'text/javascript');
	// 获得的地址类似：chrome-extension://ihcokhadfjfchaeagdoclpnjdiokfakg/js/inject.js
	temp.src = chrome.extension.getURL(jsPath);
	temp.onload = function(){
		this.parentNode.removeChild(this);
	};
	document.head.appendChild(temp);
}

// 需要在配置文件中增加配置项
{
	// 普通页面能够直接访问的插件资源列表，如果不设置是无法直接访问的
	"web_accessible_resources": ["js/inject.js"],
}
```

### homepage_url

## 8种表现形式

### browserAction(浏览器右上角)

通过配置 browser_action 可以在浏览器的右上角增加一个图标，一个 browser_action 可以拥有一个图标，一个tooltip ，一个 badge 和一个 popup；

1. 图标：推荐使用宽高都为 19 像素的图片，更大的图标会被缩小，一般通过配置 manifest 中 default_icon 字段配置，也可以调用 setIcon() 方法；
2. tooltip：般通过配置 manifest 中 default_title 字段配置，也可以调用 setTitle() 方法；
3. badge：就是在图标上显示一些文本，一般用来更新扩展状态提示信息；
   - 空间有限，只支持 4 个以下的字符（英文4个，中文2个）；
   - 无法通过配置文件指定，必须通过代码实现；
   - 设置 badge 的文字和颜色使用 setBadgeText() 和 setBadgeBackgroundColor() 方法；

```javascript
"browser_action":{
  "default_icon": "img/icon.png",
  "default_title": "这是一个示例插件,
  "default_popup": "popup.html",
}
  
// 设置 badge 的文字和颜色
chrome.browserAction.setBadgeText({text：'new'});
chrome.browserAction.setBadgeBackgroundColor({color:[255, 0, 0, 255]);
```

### pageAction(地址栏右侧)

> pageAction 指的是只有当某些特定页面打开时才显示的图标；

它和 browserAction 最大的区别是一个始终都显示，一个只有在特定的情况才显示；

1. pageAction和普通的browserAction一样也是放在浏览器右上角，只不过没有点亮时是灰色的，点亮了才是彩色的；
2. 调整之后的 pageAction 我们可以简单地把它看成是可以置灰的 browserAction；

```javascript
chrome.pageAction.show(tabId) // 显示图标
chrome.pageAction.hide(tabId) // 隐藏图标
```

示例(只有打开百度才显示图标)：

```javascript
// manifest.json
{
	"page_action":
	{
		"default_icon": "img/icon.png",
		"default_title": "我是pageAction",
		"default_popup": "popup.html"
	},
	"permissions": ["declarativeContent"]
}

// background.js
chrome.runtime.onInstalled.addListener(function(){
	chrome.declarativeContent.onPageChanged.removeRules(undefined, function(){
		chrome.declarativeContent.onPageChanged.addRules([
			{
				conditions: [
					// 只有打开百度才显示pageAction
					new chrome.declarativeContent.PageStateMatcher({pageUrl: {urlContains: 'baidu.com'}})
				],
				actions: [new chrome.declarativeContent.ShowPageAction()]
			}
		]);
	});
});
```

### 右键菜单

> 通过 chrome.contextMenus API 可以实现自定义浏览器的右键菜单；

右键菜单可以出现在不同的上下文中，比如普通页面、选中的文字、图片、链接等；

如果有同一个插件定义了多个菜单，Chrome 会自动组合放到以插件命名的二级菜单里；

```javascript
// 添加右键百度搜索

// manifest.json
{"permissions": ["contextMenus"， "tabs"]}

// background.js
chrome.contextMenus.create({
	title: '使用度娘搜索：%s', // %s表示选中的文字
	contexts: ['selection'], // 只有当选中文字时才会出现此右键菜单
	onclick: function(params){
      // 注意不能使用location.href，因为location是属于background的window对象
      chrome.tabs.create({url: 'https://www.baidu.com/s?ie=utf-8&wd=' + encodeURI(params.selectionText)});
	}
});
```

#### 常见语法

```javascript
chrome.contextMenus.create({
	type: 'normal'， // 类型，可选：["normal", "checkbox", "radio", "separator"]，默认 normal
	title: '菜单的名字', // 显示的文字，除非为“separator”类型否则此参数必需，如果类型为“selection”，可以使用%s显示选定的文本
	contexts: ['page'], // 上下文环境，可选：["all", "page", "frame", "selection", "link", "editable", "image", "video", "audio"]，默认page
	onclick: function(){}, // 单击时触发的方法
	parentId: 1, // 右键菜单项的父菜单项ID。指定父菜单项将会使此菜单项成为父菜单项的子菜单
	documentUrlPatterns: 'https://*.baidu.com/*' // 只在某些页面显示此右键菜单
});
// 删除某一个菜单项
chrome.contextMenus.remove(menuItemId)；
// 删除所有自定义右键菜单
chrome.contextMenus.removeAll();
// 更新某一个菜单项
chrome.contextMenus.update(menuItemId, updateProperties);
```

### override(覆盖特定页面)

> 使用 override 页可以将Chrome 默认的一些特定页面替换掉，改为使用扩展提供的页面；

扩展可以替代以下页面：

1. 历史记录：从工具菜单上点击历史记录时访问的页面，或者从地址栏直接输入 [chrome://history](chrome://history/)
   - 一个扩展只能替代一个页面；
2. 新标签页：当创建新标签的时候访问的页面，或者从地址栏直接输入 [chrome://newtab](chrome://newtab/)
   - 不能替代隐身窗口的新标签页面；
3. 书签：浏览器的书签，或者直接输入 [chrome://bookmarks](chrome://bookmarks/)
   - 网页必须设置 title，否则用户可能会看不到网页的 URL；

```javascript
"chrome_url_overrides":{
	"newtab": "newtab.html",
	"history": "history.html",
	"bookmarks": "bookmarks.html"
}
```

### devtools

Chrome允许插件在开发者工具(devtools)上动手脚，主要表现在：

1. 自定义一个和多个和 Elements 、Console 、Sources 等同级别的面板；
2. 自定义侧边栏(sidebar)，目前只能自定义 Elements 面板的侧边栏；

#### 介绍

1. 每打开一个开发者工具窗口，都会创建一个 devtools 页面的实例；
2. F12 窗口关闭，页面也随着关闭，所以 devtools 页面的生命周期和 devtools 窗口是一致的；
3. devtools 页面可以访问一新组特有的 DevTools API 以及有限的扩展 API，这组特定的 DevTools API 只有 devtools 页面才可以访问，background 是无法访问的；

```javascript
chorme.devtools.panels  	// 面板相关
chrome.devtools.inspectedWindwo 	// 获取被审查窗口的有关信息
chrome.devtools.network	 	// 获取有关网络请求的信息
```

大部分扩展API都无法直接被 DevTools 页面调用，但它可以像 content-script 一样直接调用 chrome.extension 和 chrome.runtime API，同时它也可以像 content-script 一样使用 Message 交互的方式与 background 页面进行通信。

```javascript
// manifest.json
{
  "devtools_page": "devtools.html"  // 只能指向一个HTML文件，不能是JS文件
}

// devtools.html
<!DOCTYPE html>
<html>
<head></head>
<body>
	<script type="text/javascript" src="js/devtools.js"></script>
</body>
</html>

// devtools.js
// 创建自定义面板，同一个插件可以创建多个自定义面板
// 几个参数依次为：panel标题、图标（其实设置了也没地方显示）、要加载的页面、加载成功后的回调
chrome.devtools.panels.create('MyPanel', 'img/icon.png', 'mypanel.html', function(panel){
	console.log('自定义面板创建成功！'); // 注意这个log一般看不到
});

// 创建自定义侧边栏
chrome.devtools.panels.elements.createSidebarPane("Images", function(sidebar){
	// sidebar.setPage('../sidebar.html'); // 指定加载某个页面
	sidebar.setExpression('document.querySelectorAll("img")', 'All Images'); // 通过表达式来指定
	//sidebar.setObject({aaa: 111, bbb: 'Hello World!'}); // 直接设置显示某个对象
});
```

```javascript
// 检测jQuery
document.getElementById('check_jquery').addEventListener('click', function(){
	// 访问被检查的页面DOM需要使用inspectedWindow
	// 简单例子：检测被检查页面是否使用了jQuery
	chrome.devtools.inspectedWindow.eval("jQuery.fn.jquery", function(result, isException){
		var html = '';
		if (isException) html = '当前页面没有使用jQuery。';
		else html = '当前页面使用了jQuery，版本为：'+result;
		alert(html);
	});
});

// 打开某个资源
document.getElementById('open_resource').addEventListener('click', function(){
	chrome.devtools.inspectedWindow.eval("window.location.href", function(result, isException){
		chrome.devtools.panels.openResource(result, 20, function(){
			console.log('资源打开成功！');
		});
	});
});

// 审查元素
document.getElementById('test_inspect').addEventListener('click', function(){
	chrome.devtools.inspectedWindow.eval("inspect(document.images[0])", function(result, isException){});
});

// 获取所有资源
document.getElementById('get_all_resources').addEventListener('click', function(){
	chrome.devtools.inspectedWindow.getResources(function(resources){
		alert(JSON.stringify(resources));
	});
});
```

#### 调试技巧

修改了devtools页面的代码时，需要先在 chrome://extensions 页面按下`Ctrl+R`重新加载插件，然后关闭再打开开发者工具即可，无需刷新页面（而且只刷新页面不刷新开发者工具的话是不会生效的）。

###  option(选项页)

> option 页就是插件的设置页面，有两个入口，一个是右键图标的”选项“菜单，一个是在插件管理页面；

```javascript
{
   // Chrome40以前的插件配置页写法
   "options_page": "options.html",
      
  // 新版的optionsV2：
  "options_ui":{
      "page": "options.html",
      // 添加一些默认的样式，推荐使用
      "chrome_style": true
  },
}
```

### omnibox

> omnibox 是向用户提供搜索建议的一种方式，通过注册某个关键字以触发插件自己的搜索建议界面；

```javascript
// manidest.json
{
	// 向地址栏注册一个关键字以提供搜索建议，只能设置一个关键字
	"omnibox": { "keyword" : "go" },
}

// background.js
chrome.omnibox.onInputChanged.addListener((text, suggest) => {
	console.log('inputChanged: ' + text);
	if(!text) return;
	if(text == '美女') {
		suggest([
			{content: '中国' + text, description: '你要找“中国美女”吗？'},
			{content: '日本' + text, description: '你要找“日本美女”吗？'},
			{content: '泰国' + text, description: '你要找“泰国美女或人妖”吗？'},
			{content: '韩国' + text, description: '你要找“韩国美女”吗？'}
		]);
	}
	else if(text == '微博') {
		suggest([
			{content: '新浪' + text, description: '新浪' + text},
			{content: '腾讯' + text, description: '腾讯' + text},
			{content: '搜狐' + text, description: '搜索' + text},
		]);
	}
	else {
		suggest([
			{content: '百度搜索 ' + text, description: '百度搜索 ' + text},
			{content: '谷歌搜索 ' + text, description: '谷歌搜索 ' + text},
		]);
	}
});

// 当用户接收关键字建议时触发
chrome.omnibox.onInputEntered.addListener((text) => {
	console.log('inputEntered: ' + text);
	if(!text) return;
	var href = '';
	if(text.endsWith('美女')) href = 'http://image.baidu.com/search/index?tn=baiduimage&ie=utf-8&word=' + text;
	else if(text.startsWith('百度搜索')) href = 'https://www.baidu.com/s?ie=UTF-8&wd=' + text.replace('百度搜索 ', '');
	else if(text.startsWith('谷歌搜索')) href = 'https://www.google.com.tw/search?q=' + text.replace('谷歌搜索 ', '');
	else href = 'https://www.baidu.com/s?ie=UTF-8&wd=' + text;
	openUrlCurrentTab(href);
});
// 获取当前选项卡ID
function getCurrentTabId(callback){
	chrome.tabs.query({active: true, currentWindow: true}, function(tabs)
	{
		if(callback) callback(tabs.length ? tabs[0].id: null);
	});
}

// 当前标签打开某个链接
function openUrlCurrentTab(url){
	getCurrentTabId(tabId => {
		chrome.tabs.update(tabId, {url: url});
	})
}
```

### 桌面通知

Chrome提供了一个 chrome.notifications API以便插件推送桌面通知，暂未找到 chrome.notifications 和 HTML5自带的 Notification 的显著区别及优势。

在后台JS中，无论是使用 chrome.notifications 还是 Notification 都不需要申请权限；

```javascript
chrome.notifications.create(null, {
	type: 'basic',
	iconUrl: 'img/icon.png',
	title: '这是标题',
	message: '您刚才点击了自定义右键菜单！'
});

```

## 消息通信

Chrome插件中存在的5种 JS：injected-script、content-script、popup-js、background-js、devtools-js；

### popup和background

> popup可以直接调用 background 中的 JS 方法，也可以直接访问 background 的 DOM；

```javascript
// background.js
function test(){
	alert('我是background！');
}

// popup.js
var bg = chrome.extension.getBackgroundPage();
bg.test(); // 访问bg的函数
alert(bg.document.body.innerHTML); // 访问bg的DOM


// background 访问 popup
var views = chrome.extension.getViews({type:'popup'});
if(views.length) {
	console.log(views[0].location.href);
}
```

### popup/bg和content

```javascript
// background.js 或 popup.js
function sendMessageToContentScript(message, callback){
	chrome.tabs.query({active: true, currentWindow: true}, function(tabs){
		chrome.tabs.sendMessage(tabs[0].id, message, function(response){
			if(callback) callback(response);
		});
	});
}
sendMessageToContentScript({cmd:'test', value:'你好，我是popup！'}, function(response){
	console.log('来自content的回复：'+response);
});


// content-script.js接收
chrome.runtime.onMessage.addListener(function(request, sender, sendResponse){
	if(request.cmd == 'test') alert(request.value);
	sendResponse('我收到了你的消息！');
});
```

### content 和 background/popup

1. content-scripts 向 popup 发送消息的前提是 popup 必须打开，否则需要利用 background 作中转；
2. 如果 background 和 popup 同时监听，它们可以同时收到消息，但只有一个可以 sendResponse ，一个先发送了，另一个再发送会无效； 

```javascript
// content-script
chrome.runtime.sendMessage({greeting: '你好，我是content-script呀，我主动发消息给后台！'}, function(response) {
	console.log('收到来自后台的回复：' + response);
});

// background.js 或 popup.js：
chrome.runtime.onMessage.addListener(function(request, sender, sendResponse){
	console.log('收到来自content-script的消息：');
	console.log(request, sender, sendResponse);
	sendResponse('我是后台，我已收到你的消息：' + JSON.stringify(request));
});
```

### injected-script和content-script

content-script 和页面内的脚本（injected-script 自然也属于页面内的脚本）之间唯一共享的东西就是页面的DOM元素；

1. 可以通过 window.postMessage 和 window.addEventListener 来实现二者消息通讯；

   ```javascript
   // injected-script
   window.postMessage({"test": '你好！'}, '*');

   // content-script
   window.addEventListener("message", function(e){
   	console.log(e.data);
   }, false);
   ```

2. 通过自定义 DOM 事件来实现；

   ```javascript
   // injected-script
   var customEvent = document.createEvent('Event');
   customEvent.initEvent('myCustomEvent', true, true);
   function fireCustomEvent(data) {
   	hiddenDiv = document.getElementById('myCustomEventDiv');
   	hiddenDiv.innerText = data
   	hiddenDiv.dispatchEvent(customEvent);
   }
   fireCustomEvent('你好，我是普通JS！');

   // content-script.js
   var hiddenDiv = document.getElementById('myCustomEventDiv');
   if(!hiddenDiv) {
   	hiddenDiv = document.createElement('div');
   	hiddenDiv.style.display = 'none';
   	document.body.appendChild(hiddenDiv);
   }
   hiddenDiv.addEventListener('myCustomEvent', function() {
   	var eventData = document.getElementById('myCustomEventDiv').innerText;
   	console.log('收到自定义事件消息：' + eventData);
   });
   ```

### 长连接和短连接

1. 短连接

   - 指的是 chrome.tabs.sendMessage 和 chrome.runtime.sendMessage；
   - 短连接就是我发送一下，你收到了再回复一下，如果对方不回复，只能重新发；

2. 长连接

   - chrome.tabs.connect 和 chrome.runtime.connect；
   - 长连接类似于 webSocket 会一直建立连接，双方可以随时互发消息；

   ```javascript
   // popup-js
   getCurrentTabId((tabId) => {
   	var port = chrome.tabs.connect(tabId, {name: 'test-connect'});
   	port.postMessage({question: '你是谁啊？'});
   	port.onMessage.addListener(function(msg) {
   		alert('收到消息：'+msg.answer);
   		if(msg.answer && msg.answer.startsWith('我是')){
   			port.postMessage({question: '哦，原来是你啊！'});
   		}
   	});
   });

   // content-script.js
   chrome.runtime.onConnect.addListener(function(port) {
   	console.log(port);
   	if(port.name == 'test-connect') {
   		port.onMessage.addListener(function(msg) {
   			console.log('收到长连接消息：', msg);
   			if(msg.question == '你是谁啊？'){
                 port.postMessage({answer: '我是你爸！'});
   			}
   		});
   	}
   });
   ```

## 其它

### 动态注入代码

虽然在 background 和 popup 中无法直接访问页面DOM，但可以通过 chrome.tabs.executeScriptchrome.tabs.executeScript 来执行脚本，从而实现访问web页面的DOM，这种方式也不能直接访问页面 JS；

```javascript
// manifest.json
{
	"name": "动态CSS注入演示",
	...
	"permissions": [
      	// 可以通过executeScript或者insertCSS访问的网站
		"tabs", "http://*/*", "https://*/*" 
	],
	...
}

// 动态执行JS代码
chrome.tabs.executeScript(tabId, {code: 'document.body.style.backgroundColor="red"'});
// 动态执行JS文件
chrome.tabs.executeScript(tabId, {file: 'some-script.js'});
// 动态执行CSS文件
chrome.tabs.insertCSS(tabId, {file: 'some-style.css'});
```

### 本地存储

1. chrome.storage 是针对插件全局的，即使在 background 中保存的数据，在content-script 也能获取到；
2. chrome.storage.sync 可以跟随当前登录用户自动同步，这台电脑修改的设置会自动同步到其它电脑，如果没有登录或者未联网则先保存到本地，等登录了再同步至网络；
3. 需要在 manifest.json 中声明 storage 权限；

```javascript
// 读取数据，第一个参数是指定要读取的key以及设置默认值
chrome.storage.sync.get({color: 'red', age: 18}, function(items) {
	console.log(items.color, items.age);
});
// 保存数据
chrome.storage.sync.set({color: 'blue'}, function() {
	console.log('保存成功！');
});
```

### webRequest

通过 webRequest 系列 API 可以对 HTTP 请求进行任性地修改、定制；

```javascript
//manifest.json
{
	"permissions":[
		"webRequest", // web请求
      	"storage",
		"webRequestBlocking", // 阻塞式web请求
	],
}


// background.js

var showImage; // 是否显示图片
chrome.storage.sync.get({showImage: true}, function(items) {
	showImage = items.showImage;
});
// web请求监听，最后一个参数表示阻塞式，需单独声明权限：webRequestBlocking
chrome.webRequest.onBeforeRequest.addListener(details => {
	// cancel 表示取消本次请求
	if(!showImage && details.type == 'image') return {cancel: true};
	// 简单的音视频检测
	if(details.type == 'media') {
		chrome.notifications.create(null, {
			type: 'basic',
			iconUrl: 'img/icon.png',
			title: '检测到音视频',
			message: '音视频地址：' + details.url,
		});
	}
}, {urls: ["<all_urls>"]}, ["blocking"]);
```

## 项目开发

### 通信方式

#### 事件

在脚本中监听事件：

```javascript
chrome.extension.onMessage.addListener((request, sender, sendResponse) => {
    console.log('收到消息')
    sendResponse('发送返回值');
});

```

发送消息：

```javascript
chrome.extension.sendMessage({msg: 'send a message'},(response) => { 
    console.log(response); 
});
```

#### chrome.extension

popup 和 background 能通过 chrome.extension API来访问脚本；

```javascript
// getBackgroundPage（）返回当前扩展在后台运行脚本的 windows 对象；
chrome.extension.getBackgroundPage()

// getViews（）返回一个数组，含有每一个在当前扩展程序中运行的页面的JavaScript window 对象；
chrome.extension.getViews()

```



### 本地储存

Chrome 插件可以使用 html5 的 localStorage API 来实现数据存储，需要在 manifest.json 文件中的 permissions 配置储存权限；

1. chrome.cookies.* API 来储存 cookies；
2. chrome.storage.* API 来储存数据；

storage 储存分为同步模式和本地模式：

1. chrome.storage.sync.*：同步模式储存的数据如果在另一台设备的浏览器登录同一个账号数据会被同步；
2. chrome.storage.local.*：本地模式；
3. 只读存储可以使用 chrome.storage.managed 来存储； 

```javascript
// 储存数据
chrome.storage.sync.set({key: value}, function() {
  console.log('Value is set to ' + value);
});

// 读取数据
chrome.storage.sync.get(['key'], function(result) {
  console.log('Value currently is ' + result.key);
});
```

## 打包与发布

打包的话直接在插件管理页有一个打包按钮，点击会生成一个 .crx 文件；

如果要发布到 Google 应用商店的话需要先登录你的Google账号，然后花5个$注册为开发者；



## 查看已安装的插件路径



## 调试

1. 打开 Chrome 扩展页面打开开发者模式；
2. 直接将 crx 或者 zip 文件拖入浏览器安装；
3. 也可以通过导航栏的加载已解压扩展程序来安装；

### 调试popup

在工具栏的扩展小图标上右击选择审查弹出内容，会弹出 Chrome 的 DevTools ，调用方式和普通页面一样；

### 调试background

在扩展管理页面，在安装的扩展上有背景页按钮，点击会弹出 background 页面的 DevTools；

### 调试 content

content 脚本是直接注入到页面中，所以直接在页面中打开 DevTools 就能调试； 

## 参考文档 

https://crxdoc-zh.appspot.com/extensions/api_other