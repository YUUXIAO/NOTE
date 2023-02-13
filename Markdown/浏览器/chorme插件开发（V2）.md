Chrome 插件是一个用 Web 技术开发、用来增强浏览器功能的软件，它其实就是一个由HTML、CSS、JS、图片等资源组成的一个.crx 后缀的压缩包；

## 核心文件

### manifest.json

> manifest.json 文件用来配置所有和插件相关的配置，必须放在根目录下；

manifest_version、name、version 三个配置项是必不可少的，description 和 icons 是推荐的；

```javascript
{
  "manifest_version": 2,  //【固定值】
  "name": "我的扩展程序",  // 插件的名字
  "version": "版本字符串", //  插件的版本
  "description": "", // 插件简介栏
}
```

### popup

> popup 就是点击 browser_action 或者 page_action 图标时打开的一个小窗口网页，焦点离开网页就立即关闭，一般用来做一些临时性的交互；

1. popup 可以包含任意你想要的HTML内容，并且会自适应大小；
2. popup 页面的生命周期一般随弹窗开启而开启，关闭而关闭；
3. 可以通过 default_popup 字段来指定popup页面，也可以调用 setPopup() 方法；
4. popup 中可以直接通过 chrome.extension.getBackgroundPage() 获取background的window对象。

```javascript
{
	"browser_action":{
		"default_icon": "img/icon.png",
		"default_title": "这是一个示例Chrome插件",  
		"default_popup": "popup.html"
	}
}
```

### background

1. background 可以通过 page 指定一个网页，也可以通过 scripts 直接指定一个 JS，Chrome 会自动为这个 JS 生成一个默认的网页；
2. background 的权限非常高，可以调用大部分的 Chrome 扩展API（除了devtools），而且可以无限制跨域。
3. 它的生命周期是插件中所有类型页面中最长的，随着浏览器的打开而打开，关闭而关闭；
4. 通常把需要一直运行的、启动就运行的、全局的代码放在 background 中；

```javascript
{
	"background":{
		"page": "background.html", // 方式一: 直接指定一个网页
		//"scripts": ["js/background.js"]  // 方式二: 指定一个 JS，会自动生成一个背景页面
	},
}
```

### event-pages

1. event-pages 与 background 的区别就是生命周期的长短，它比 background 多了一个 persistent 参数；
2. 它在被需要时加载，在空闲时被关闭，比如第一次安装、插件更新、有content-script向它发送消息等等；

```javascript
{
	"background":{
		"scripts": ["event-page.js"],
		"persistent": false
	},
}
```

### content-scripts

> content-scripts 可以向页面注入 JS/CSS 代码；

1. content-scripts 和原始页面共享 DOM，但不共享 JS；
2. 虽然 content-script 可以向页面中注入 JS 代码，但是 DOM 不能操作它。
3. content_scripts 注入的CSS优先级非常高，几乎仅次于浏览器默认样式，尽量避免写一些影响全局的样式；
4. content-scripts 不能访问绝大部分 Chrome.xxx.api，除了下面4种；
   - chrome.extension(getURL , inIncognitoContext , lastError , onRequest , sendRequest)
   - chrome.i18n
   - chrome.runtime(connect , getManifest , getURL , id , onConnect , onMessage , sendMessage)
   - chrome.storage

```javascript
{
  "content_scripts": 
  [
      {
          //"matches": ["http://*/*", "https://*/*"],
          "matches": ["<all_urls>"], // "<all_urls>" 表示匹配所有地址
          "js": ["js/jquery-1.8.3.js", "js/content-script.js"],  // 多个JS按顺序注入
          "css": ["css/custom.css"],
          // 代码注入的时间，可选值： "document_start", "document_end", or "document_idle"，最后一个表示页面空闲时，默认document_idle
          "run_at": "document_start"
      }
  ],
}
```

### injected-script

> 指的是通过 DOM 操作的方式向页面注入的一种  JS；

1. 需要在配置文件中增加配置项 "web_accessible_resources"，配置页面能够直接访问的插件资源列表，不设置是无法访问；
2. content-script 和页面内的脚本（injected-script 自然也属于页面内的脚本）之间唯一共享的东西就是页面的DOM元素；

```javascript
// 在 content-script 中通过 DOM方 式向页面注入 inject-script 代码示例：
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


// manifest.json
{
	"web_accessible_resources": ["js/inject.js"],
}
```

## 表现形式

### browserAction

通过配置 browser_action 可以在浏览器的右上角增加一个图标，一个 browser_action 可以拥有一个图标，一个tooltip ，一个 badge 和一个 popup；

1. 图标：推荐使用宽高都为 19 像素的图片，更大的图标会被缩小，一般通过配置 manifest 中 default_icon 字段配置，也可以调用 setIcon() 方法；
2. tooltip：一般通过配置 manifest 中 default_title 字段配置，也可以调用 setTitle() 方法；
3. badge：就是在图标上显示一些文本，一般用来更新扩展状态提示信息；
   - 空间有限，只支持 4 个以下的字符（英文4个，中文2个）；
   - 无法通过配置文件指定，必须通过代码实现；
   - 设置 badge 的文字和颜色使用 setBadgeText() 和 setBadgeBackgroundColor() 方法；

```javascript
// manifest.json
"browser_action":{
  "default_icon": "img/icon.png",
  "default_title": "这是一个示例插件,
  "default_popup": "popup.html",
}
  
// 设置 badge 的文字和颜色
chrome.browserAction.setBadgeText({text：'new'});
chrome.browserAction.setBadgeBackgroundColor({color:[255, 0, 0, 255]);
```

### pageAction

> pageAction 指的是只有当某些特定页面打开时才显示的图标；

1. pageAction 和普通的 browserAction 一样也是放在浏览器右上角，只不过没有点亮时是灰色的，点亮了才是彩色的；
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
					new chrome.declarativeContent.PageStateMatcher({pageUrl: {urlContains: 'baidu.com'}})
				],
				actions: [new chrome.declarativeContent.ShowPageAction()]
			}
		]);
	});
});
```

### contexMenus

> chrome.contextMenus API 可以实现自定义浏览器的右键菜单；

1. 右键菜单可以出现在不同的上下文中，比如普通页面、选中的文字、图片、链接等；
2. 如果有同一个插件定义了多个菜单，Chrome 会自动组合放到以插件命名的二级菜单里；

```javascript
chrome.contextMenus.create(createProperties, function callback)

chrome.contextMenus.remove(menuItemId)；  // 删除某一个菜单项
chrome.contextMenus.removeAll();  	// 删除所有自定义右键菜单
chrome.contextMenus.update(menuItemId, updateProperties);	 // 更新某一个菜单项
```

createProperties 参数：

- type：右键菜单项的类型，string ["normal", "checkbox", "radio", "separator"] )；
- title：显示的文字，如果类型为“selection”，您可以在字符串中使用 %s 显示选定的文本；
- contexts：上下文环境，["all", "page", "frame", "selection", "link", "editable", "image", "video", "audio"]；
- onclick: function(){}：单击时触发的方法；
- parentId: 右键菜单项的父菜单项ID,指定父菜单项将会使此菜单项成为父菜单项的子菜单；
- documentUrlPatterns：只在某些页面显示此右键菜单

```javascript
// manifest.json
{
  "permissions": ["contextMenus"， "tabs"]
}

// background.js
chrome.contextMenus.create({
	title: '使用百度搜索：%s', // %s表示选中的文字
	contexts: ['selection'], 
	onclick: function(params){
      chrome.tabs.create({
        url: 'https://www.baidu.com/s?ie=utf-8&wd=' + encodeURI(params.selectionText)
      });
	}
});
```

### override

> 使用 override 页可以将 Chrome  默认的一些特定页面替换掉，改为使用扩展提供的页面；

- 历史记录：从工具菜单上点击历史记录时访问的页面；
- 新标签页：当创建新标签的时候访问的页面；
- 书签：浏览器的书签；

注意：

- 一个扩展只能替代一个页面；
- 不能替代隐身窗口的新标签页面；
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

- 自定义一个和多个和 Elements 、Console 、Sources 等同级别的面板；
- 自定义侧边栏(sidebar)，目前只能自定义 Elements 面板的侧边栏；

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

###  option

> option 页就是插件的设置页面，有两个入口，一个是右键图标的”选项“菜单，一个是在插件管理页面；

```javascript
{
   // Chrome40以前的插件配置页写法
   "options_page": "options.html",
      
  // 新版的optionsV2：
  "options_ui":{
      "page": "options.html",
      "chrome_style": true
  },
}
```

### omnibox

> omnibox 是向用户提供搜索建议的一种方式，通过注册某个关键字以触发插件自己的搜索建议界面；

1.  向地址栏注册一个关键字以提供搜索建议，只能设置一个关键字；

```javascript
// manidest.json
{
	"omnibox": { "keyword" : "go" },
}

// background.js
chrome.omnibox.onInputChanged.addListener((text, suggest) => {
	console.log('inputChanged: ' + text);
	if(!text) return;
	suggest([
        {content: '百度搜索 ' + text, description: '百度搜索 ' + text},
        {content: '谷歌搜索 ' + text, description: '谷歌搜索 ' + text},
    ]);
});

// 当用户接收关键字建议时触发
chrome.omnibox.onInputEntered.addListener((text) => {
	console.log('inputEntered: ' + text);
	if(!text) return;
	var href = 'https://www.baidu.com/s?ie=UTF-8&wd=' + text;
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

### notifications

Chrome提供了一个 chrome.notifications API以便插件推送桌面通知；

```javascript
chrome.notifications.create(null, {
	type: 'basic',
	iconUrl: 'img/icon.png',
	title: '这是标题',
	message: '这是桌面通知！'
});

```

## 消息通信

Chrome插件中存在的5种 JS：injected-script、content-script、popup-js、background-js、devtools-js；

### popup和background

> popup可以直接调用 background 中的 JS 方法，也可以直接访问 background 的 DOM；

```javascript
// popup.js 获取background页面
var bg = chrome.extension.getBackgroundPage();
bg.test();
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

### content 和 popup/bg

1. content-scripts 向 popup 发送消息的前提是 popup 必须打开，否则需要利用 background 作中转；
2. 如果 background 和 popup 同时监听，它们可以同时收到消息，但只有一个可以 sendResponse ，一个先发送了，另一个再发送会无效； 

```javascript
// content-script
chrome.runtime.sendMessage({greeting: '我是content，我发消息给后台！'}, function(response) {
	console.log('收到来自后台的回复：' + response);
});

// background.js 或 popup.js：
chrome.runtime.onMessage.addListener(function(request, sender, sendResponse){
	console.log('收到来自content-script的消息：');
	sendResponse('我是后台，我已收到你的消息：' + JSON.stringify(request));
});
```

### injected-script和content

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
   // content-script.js
   var port = chrome.extension.connect({ name: 'knockknock' });
   port.postMessage({ answer: 'Knock knock' });

   port.onMessage.addListener(function ({ question }) {
     setTimeout(function () {
       if (question == "Who's there?") port.postMessage({ answer: 'Madame' });
       else if (question == 'Madame who?')
         port.postMessage({ answer: 'Madame... Bovary' });
     }, 2000);
   });

   // popup-js
   chrome.extension.onConnect.addListener(function (port) {
     const connectList = document.getElementById('connect_list');
     port.onMessage.addListener(function ({ answer }) {
         if (answer == 'Knock knock')
           port.postMessage({ question: "Who's there?" });
         else if (answer == 'Madame')
           port.postMessage({ question: 'Madame who?' });
         else if (answer == 'Madame... Bovary')
           port.postMessage({ question: "I don't get it." });
   });
   ```

## 其它

### 动态注入代码

可以通过 chrome.tabs.executeScriptchrome.tabs.executeScript 来执行脚本，从而实现访问web页面的DOM，这种方式也不能直接访问页面 JS；

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

1. 需要在 manifest.json 中声明 storage 权限；
2. chrome.storage 是针对插件全局的，即使在 background 中保存的数据，在 content-script 也能获取到；
3. chrome.storage.sync 可以跟随当前登录用户自动同步，这台电脑修改的设置会自动同步到其它电脑，如果没有登录或者未联网则先保存到本地，等登录了再同步至网络；
4. ​
5. 只读存储可以使用 chrome.storage.managed 来存储

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

### webRequest

通过 webRequest 系列 API 可以对 HTTP 请求进行修改、定制；

![img](http://www.yumefx.com/wp-content/uploads/2020/10/chromeplug_22.png)



```javascript
//manifest.json
{
	"permissions":[
      	"storage",
		"webRequest", // web请求
		"webRequestBlocking", // 阻塞式web请求
	],
}


// 参数blocking表示阻塞式，需单独声明权限：webRequestBlocking
// 每次请求前触发，可以拿到 requestBody 数据，同时可以对本次请求作出干预修改
chrome.webRequest.onBeforeRequest.addListener(details => {
    console.log('onBeforeRequest', details);
}, {urls: ['<all_urls>']}, ['blocking', 'extraHeaders', 'requestBody']);

// 发送header之前触发，可以拿到请求headers，也可以添加、修改、删除headers,有一定限制，一些特殊头部可能拿不到或者存在特殊情况
chrome.webRequest.onBeforeSendHeaders.addListener(details => {
    console.log('onBeforeSendHeaders', details);
}, {urls: ['<all_urls>']}, ['blocking', 'extraHeaders', 'requestHeaders']);

// 开始响应触发，可以拿到服务端返回的headers
chrome.webRequest.onResponseStarted.addListener(details => {
    console.log('onResponseStarted', details);
}, {urls: ['<all_urls>']}, ['extraHeaders', 'responseHeaders']);

// 请求完成触发，能拿到的数据和onResponseStarted一样，无法拿到responseBody
chrome.webRequest.onCompleted.addListener(details => {
    console.log('onCompleted', details);
}, {urls: ['<all_urls>']}, ['extraHeaders', 'responseHeaders']);
```

使用webRequestAPI是无法拿到responseBody的，想要拿到可以采用：

1. 重写XmlHttpRequest和fetch，增加自定义拦截事件，缺点是实现方式可能有点脏，重写不好的话可能影响正常页面；
2. devtools的chrome.devtools.network.onRequestFinishedAPI可以拿到返回的body，缺点是必须打开开发者面板；
3. 使用chrome.debugger.sendCommand发送Network.getResponseBody命令来获取body内容，缺点也很明显，会有一个提示；

### 国际化

1. 插件根目录新建一个名为_locales的文件夹；
2. 在 _locales 下面新建一些语言的文件夹，如en、zh_CN、zh_TW；
3. 在每个语言文件夹放入一个 messages.json；
4. 同时必须在清单文件中设置default_locale；
5. 在manifest.json和CSS文件中通过 __ MSG_messagename __ 引入

```javascript
// _locales\en\messages.json
{
    "pluginDesc": {"message": "A simple chrome extension demo"},
    "helloWorld": {"message": "Hello World!"}
}

// _locales\zh_CN\messages.json
{
    "pluginDesc": {"message": "一个简单的Chrome插件demo"},
    "helloWorld": {"message": "hello word！"}
}

// manifest.json
{
    "description": "__MSG_pluginDesc__",
    "default_locale": "zh_CN",  // 默认语言
}
```



## 打包与发布

1. 在插件管理页面点击打包扩展程序，生成.crx文件。
2. Google账号登录Chrome应用商店（首次成为开发者需要支付5美元）。
3. 上传文件后开始填写插件的图标、说明及截图，截图将会以轮播图的形式展现。

