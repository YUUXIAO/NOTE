> Chrome插件是一个用Web技术开发、用来增强浏览器功能的软件，它其实就是一个由HTML、CSS、JS、图片等资源组成的一个.crx 后缀的压缩包

## 开发环境

## 项目结构

就像 node.js 的package.json一样，每一个插件必须有一个manifest.json，作为最初配置文件；

```javascript
{
  "manifest_version": 2,  //【固定值】
  "name": "我的扩展程序",  // 插件的名字
  "version": "版本字符串", //  插件的版本
  "description": "", // 插件简介栏
}
```

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