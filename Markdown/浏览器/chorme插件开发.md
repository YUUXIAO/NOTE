> Chrome插件是一个用Web技术开发、用来增强浏览器功能的软件，它其实就是一个由HTML、CSS、JS、图片等资源组成的一个.crx 后缀的压缩包

## 开发与调试

## 项目结构

插件的核心是 manifest.json 清单文件，提供有关扩展程序的各种信息；

```javascript
{
  "manifest_version": 2,  // 固定值
  "name": "我的扩展程序",
  "version": "版本字符串",
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

安装插件后每打开一个网页可以将 content 脚本注入到时页面中，内容脚本可以读取浏览器访问的网页的细节，并且可以修改页面；

可以将内容脚本看作是已加载页面的一部分，在哪些页面中注入脚本和注入什么脚本在 manifest.json 文件中配置；

```javascript
{
  "content_scripts": [
    {
      "matches": ["*://*/*", "file://*"], // 可以设置一个匹配表达式来过滤需要注入脚本的网站
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

## 参考文档 

https://crxdoc-zh.appspot.com/extensions/api_other（中文文档）