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

Chrome允许插件在开发者工具(devtools)上动手脚；

1. 自定义一个和多个和 Elements 、Console 、Sources 等同级别的面板；
2. 自定义侧边栏(sidebar)，目前只能自定义 Elements 面板的侧边栏；

### 介绍

## 项目开发

Chrome 插件可以分为三部分，分别运行在不同的环境；

### 后台页面/事件页面（background）

1. 后台页面是运行在浏览器后台的，随着浏览器的启动开始运行，浏览器关闭就对事运行；
2. 事件页面是需要调用时加载，脚本空闲时被卸载；
3. 两者都是运行在后台；

在 manifest.json 文件中配置需要运行的脚本；

```javascript
{
  "background": {
    "scripts": ["eventPage.js"],
    "persistent": false
  }
}

```

### 用户界面网页（popup）

用户界面指的是点击插件显示的小弹窗，每次点击开始运行，弹窗关闭后结束，可以与 background 脚本交互；

#### 工具栏界面

点击工具栏/地址栏插件图标出来的弹窗其实就是一个html页面，弹窗要显示的文件，和工具栏小图标在 manifest.json 文件中配置；

```javascript
{
  "browser_action": {
    "default_title": "clearRead",
    "default_icon": "icon.png",
    "default_popup": "popup.html"
  },
}
```

### 内容脚本（content）

> 安装插件后每打开一个网页可以将 content 脚本注入到时页面中，内容脚本可以读取浏览器访问的网页的细节，并且可以修改页面；

1. 内容脚本在打开网页时会被注入到网页中，它们在浏览器中已加载页面的上下文中执行；
2. 可以将内容脚本视为已加载页面的一部分；

在哪些页面中注入脚本和注入什么脚本在 manifest.json 文件中配置；

```javascript
{
  "content_scripts": [
    {
      // 可以设置一个匹配表达式来过滤需要注入脚本的网站
      "matches": ["*://*/*", "file://*"], 
      "css": ["style/content.css"],
      "js": ["js/contentScript.js"]
    }
  ]
}
```

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

## 上传与发布

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