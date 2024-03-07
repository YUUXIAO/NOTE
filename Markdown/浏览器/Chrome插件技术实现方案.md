## 初始化项目

## 配置插件相关文件

关于谷歌插件的开发基础写法可以先看一下：[chrome 插件开发](chrome%E6%8F%92%E4%BB%B6%E5%BC%80%E5%8F%91.md)

一个完整的谷歌插件必须要有manifest.json（配置文件）、popup（窗口文件）、logo图标；

可选后台脚本(background scripts)、内容脚本(content scripts)

因为项目是基于 React+Vite 处理的，所以最后生成的文件夹就是我们打包的最终文件夹

- 首先我们在根目录下创建public 文件夹（[vite关于public文件夹的说明](https://cn.vitejs.dev/guide/assets#the-public-directory)），里面放manifest.json和logo图片
- 在build的时候将popup.html、background.js、content.js 文件一起打包组成安装包（下面会将打包流程）

### 创建manifest

创建插件最重要的一份文件就是 manifest.json 文件了

- manifest_version：注意是3
- action：配置在浏览器页面网址右边的插件图标、名称和打开页面（html）
- background：后台js文件

```json
{
  "short_name": "标签管理器",
  "name": "TabsManager",
  "version": "1.0.1",
  "manifest_version": 3,
  "permissions": ["storage", "contextMenus", "tabs", "background", "bookmarks"],
  "action": {
    "default_popup": "index.html",
    "default_icon": {
      "16": "./logo.png",
      "32": "./logo.png",
      "48": "./logo.png",
      "128": "./logo.png"
    },
    "default_title": "窗口管理器"
  },
  "background": {
    "service_worker": "background.js",
    "type": "module"
  },
  "icons": {
    "16": "logo.png",
    "32": "logo.png",
    "48": "logo.png",
    "128": "logo.png"
  }
}

```

### 封装插件API

在根目录下创建文件夹extentionUtils，里面放项目里所有调用的插件api（在manifest文件的permission里生命的权限），我项目中用到了tabs、window、storage和bookmarks所以创建了bookmarks.js、storage.js和tabUtils.js

但每次如果修改了代码就要重新打包文件再到chrome:://extentions刷新数据那也太不方便了，为了方便我们在本地浏览器开发插件，所以封装一个判断环境的方法：如果为浏览器环境就返回mockdata，否则就返回正常数据，拿tabUtils.js为例

```javascript
/* eslint-disable no-undef */

import { mockWindowsData, mockTabsData } from '@/api/popup.js'
import { isExtentionEnv } from '@/utils.js'

/**
 * 获取tabs列表
 */
export const getTabLists = (queryInfo = {}) => {
  return new Promise(resolve => {
    if (isExtentionEnv()) {
      chrome.tabs.query(queryInfo, tabs => resolve(tabs))
    } else {
      resolve(mockTabsData)
    }
  })
}

// 创建新tab
export const createNewTab = (queryInfo = {}) => {
  return chrome.tabs.create(queryInfo)
}

// 创建新窗口
export const createNewWindow = (params = {}, callback) => {
  chrome.windows.create(params, window => {
    callback && callback(window)
  })
}

// 切换窗口
export const toggleWindow = windowId => {
  return chrome.windows.update(windowId, { focused: true })
}

/**
 * 获取当前tab
 */
export const getCurrentTab = () => {
  return new Promise(resolve => {
    if (isExtentionEnv()) {
      chrome.tabs.query(
        {
          active: true,
          currentWindow: true,
        },
        tabs => resolve(tabs[0])
      )
    } else {
      resolve(1)
    }
  })
}

/**
 * 切换tab
 * @param {number} tab
 * @param {number} windowId 当前窗口ID
 */
export const toggleTab = (tab, windowId) => {
  toggleWindow(windowId)
  chrome.tabs.highlight({ tabs: tab.index })
}

/**
 * 删除tab
 * @param {string}} type
 */
export const deleteTab = ids => {
  return new Promise(resolve => {
    if (isExtentionEnv()) {
      chrome.tabs.remove(ids, () => {
        resolve(true)
      })
    } else {
      resolve(true)
    }
  })
}

// 移动tab
export const moveTabs = (ids, moveProperties) => {
  if (isExtentionEnv()) {
    return chrome.tabs.move(ids, moveProperties)
  }
}

// window

/**
 * 获取当前窗口ID
 */
export const getCurrentWindowId = () => {
  return new Promise(resolve => {
    if (isExtentionEnv()) {
      return chrome.windows.getCurrent(({ id }) => {
        if (!id) return
        resolve(id)
      })
    } else {
      resolve(973095260)
    }
  })
}

/**
 * 获取所有窗口
 */
export const getAllWindow = () => {
  return new Promise(resolve => {
    if (isExtentionEnv()) {
      chrome.windows.getAll({}, windows => {
        resolve(windows)
      })
    } else {
      resolve(mockWindowsData)
    }
  })
}

// 删除一个窗口
export const deleteWindow = windowId => {
  if (isExtentionEnv()) {
    return chrome.windows.remove(windowId)
  }
}

const TabUtils = {
  getAllWindow,
  getCurrentWindowId,
  deleteTab,
  createNewTab,
  deleteWindow,
  moveTabs,
  toggleTab,
  getCurrentTab,
  toggleWindow,
  getTabLists,
  createNewWindow,
}

export default TabUtils

```

## 安装构建工具

### 卸载react-scripts

卸载脚手架自带的 react-sctipts（这个也是内部通过webpack构建的）

### Vite配置

1. 安装vite相关包：

```javascript
npm install --save-dev vite @vitejs/plugin-react
```

2. 配置 vite.config.js 文件：

```javascript
import { defineConfig } from 'vite'
import path from 'path'
import react from '@vitejs/plugin-react'
import reactRefresh from '@vitejs/plugin-react-refresh'
import AutoImport from 'unplugin-auto-import/vite'
import dotenv from 'dotenv'
dotenv.config()


export default defineConfig({
  plugins: [
    react(),
    reactRefresh(), // 热更新
    AutoImport({
      include: [/\.[tj]sx?$/],
      imports: ['react', 'react-router'],
    }),
  ],
  css: {
    // less配置全局变量和公共样式
    preprocessorOptions: {
      less: {
        additionalData: `@import "${path.resolve(__dirname, 'src/assests/index.less')}";`,
      },
    },
  },

  resolve: {
    alias: {
      '@': path.resolve(__dirname, 'src'),
    },
  },
  // 多页面打包，登录页要单独打包，下面会讲到为何这个样处理
  build: {
    emptyOutDir: true,
    rollupOptions: {
      input: {
        index: path.resolve(__dirname, 'index.html'),
        login: path.resolve(__dirname, '/pages/login.html'),
      },
      output: {
        assetFileNames: 'assets/[name]-[hash].[ext]',
        entryFileNames: 'assets/[name]-[hash].js',
        chunkFileNames: 'assets/[name]-[hash].js',
      },
    },
  },
})

```

### Less和全局css变量

[因为vite 本身提供了对.less文件的内置支持](https://cn.vitejs.dev/guide/features#css-pre-processors)，所以直接安装包之后在vite.config.js 配上就好了：

```javascript
 css: {
    preprocessorOptions: {
      less: {
        additionalData: `@import "${path.resolve(
          __dirname,
          "src/assests/var.less"
        )}";`
      }
    }
  },
```

### 配置路由

```javascript
npm install react-dom react-router-dom
```

1. 根目录创建router文件夹

- **创建router对象**，配置路由字典，这里路由模式为哈希模式（createHashRouter）
- **注意：**这里要用createHashRouter， createBrowserRouter浏览器可以运行但是build之后在插件popup会报错404，感觉可能是默认路由路径匹配的问题​

注意在React的路由组件要大写开头：

```javascript
// router.js
import { createHashRouter } from 'react-router-dom'
import PopupHome from '@/popup/pages/Home'
import Entry from '@/popup/pages/Entry'
import LaterPage from '@/popup/pages/LaterPage'
import TodoKeysPage from '@/popup/pages/TodoKeysPage'
import UrlsGroupPage from '@/popup/pages/UrlsGroupPage'

const routerConfigs = createHashRouter([
  {
    path: '/',
    element: <PopupHome />,
  },
  {
    path: '/popup',
    element: <Entry />,
    children: [
      {
        path: 'later',
        name: '稍后再看',
        element: <LaterPage />,
      },
      {
        path: 'urlGroup',
        name: '网页组',
        element: <UrlsGroupPage />,
      },
      {
        path: 'todoKeys',
        name: '关键词记事本',
        element: <TodoKeysPage />,
      },
    ],
  },
])

export default routerConfigs
```

2. main.js 注册路由，项目的默认入口页面是popup页面

```javascript
import React from "react"
import ReactDOM from "react-dom/client"
import Popup from "@/popup/index"

const root = ReactDOM.createRoot(document.getElementById("root"))
root.render(<Popup />)

```

```javascript
// popup/index.js
​import { RouterProvider } from 'react-router-dom'
import routers from '@/router/router.jsx'

function Entry() {
  return <RouterProvider router={routers}></RouterProvider>
}

export default Entry
```

### 安装 redux、react-redux

```javascript
npm install --save redux react-reduxredux
```



### 配置eslint、preitter规范

```nginx
npm install eslint eslint-plugin-react-hooks eslint-config-react-app --D
```

安装eslit和preitter就是项目基本操作了，具体可以看另一篇[Prettier和Eslint.md](../%E5%89%8D%E7%AB%AF%E5%B7%A5%E7%A8%8B%E5%8C%96/Prettier%E5%92%8CEslint.md)

### axios封装和fetch文件

为什么有了axios文件还需要用fetch发请求呢？因为按照规范在background.js只能使用fetch发送请求，所以在项目里除了background.js里其他地方都使用axios处理请求

axios配置文件在/api/http.js 

```javascript
import axios from 'axios'
import storageUtils from '@/extentionUtils/storage'
import Store from '@/store/index'

axios.defaults.baseURL = import.meta.env.BASE_URL
axios.defaults.timeout = 10000

// 请求拦截器
axios.interceptors.request.use(
  async config => {
    return storageUtils.getStorageItem('token').then(val => {
      val && (config.headers.Authorization = `Bearer ${val}`)
      return config
    })
  },
  error => {
    return Promise.error(error)
  }
)

// 响应拦截器
axios.interceptors.response.use(
  response => {
    if (response.status === 200 && response.data.error === 0) {
      return Promise.resolve(response)
    } else {
      return Promise.reject(response)
    }
  },
  // 服务器状态码不是2开头的的情况
  error => {
    if (error?.response?.status) {
      switch (error.response.status) {
        case 401:
        case 403:
          Store.dispatch({
            type: 'get_user',
            payload: {},
          })
          break

        default:
          console.log(JSON.stringify(error.response.data.msg))
      }
      return Promise.reject(error.response)
    }
    return Promise.reject(error)
  }
)

/**
 * get方法，对应get请求
 * @param {String} url [请求的url地址]
 * @param {Object} params [请求时携带的参数]
 */
export function get(url, params) {
  return new Promise((resolve, reject) => {
    axios
      .get(url, {
        params: params,
      })
      .then(res => {
        resolve(res.data)
      })
      .catch(err => {
        reject(err)
      })
  })
}

/**
 * post方法，对应post请求
 * @param {String} url [请求的url地址]
 * @param {Object} params [请求时携带的参数]
 */
export function post(url, params) {
  return new Promise((resolve, reject) => {
    axios
      .post(url, params)
      .then(res => {
        resolve(res.data)
      })
      .catch(err => {
        reject(err)
      })
  })
}

```

## popup页面

popup页面主要是点击浏览器网页右边那个小图标展示的html，这里具体看你的业务需求

### 登录处理（多页面打包）

目前登录是采用的邮箱📮+验证码的方式，但是由于是插件的原因，在用户获取验证码之后打开新页面查看邮件的时候，浏览器是默认把插件的popup关闭，这样会导致拿到code后需要再次打开popup进入登录页面输入邮箱，所以考虑了两种方式：

1. 因为验证码的有效期的30分钟，所以在每次用户打开popup页面时，判断用户为未登录状态&&已经点击过获取验证码(有效期内)，自动跳转登录页面并回填邮箱
2. **【目前实现】**把登录页面抽离插件成为一个单独的网页窗口，这样就可以避开切换选项卡popup页面关闭的问题了

如果把登录页面单独打包成一个页面，那就涉及了vite多页面打包的配置：

```javascript
build: {
    emptyOutDir: true,
    rollupOptions: {
      input: {
        index: path.resolve(__dirname, 'index.html'),
        // 把除了popup之外需要单独打包的页面都统一管理在pages文件夹下
        login: path.resolve(__dirname, '/pages/login.html'), 
      },
      output: {
        assetFileNames: 'assets/[name]-[hash].[ext]',
        entryFileNames: 'assets/[name]-[hash].js',
        chunkFileNames: 'assets/[name]-[hash].js',
      },
    },
  },

```

## background.js

background.js 就像是poup页面的js文件，但是不同的是：

- popup页面如果弹窗关闭就销毁了
- background 只要浏览器打开就是一直存在后台运行，可以在这里监听生命周期处理数据

这个项目在background.js 文件只要是在浏览器打开时，插件注册成功的时候初始化了右键菜单相关功能：

```javascript
chrome.runtime.onInstalled.addListener(function () {
  chrome.contextMenus.create({
    id: 'tabs_extention_keys',
    title: '添加到记事本',
    type: 'normal',
    contexts: ['selection'],
  })

  chrome.contextMenus.create({
    id: 'tabs_extention_later',
    title: '添加到稍后再看',
    type: 'normal',
    contexts: ['all'],
  })
  chrome.contextMenus.onClicked.addListener(async (menuInfo, tabInfo) => {
    const token = await storageUtils.getStorageItem('token')
    const { menuItemId } = menuInfo
    switch (menuItemId) {
      case 'tabs_extention_keys': // 关键词
      case 'tabs_extention_later': // 稍后再看
        postMenusMaps(menuItemId, menuInfo, tabInfo, token)
        break
      default:
        break
    }
  })
})
```

## content.js

content.js 主要是可以让我们的插件与浏览器页面进行交互（主要是对 DOM 元素进行增删改查等操作），比如修改页面样式，增加按钮、浮窗等控件，方便用户对网页进行操作，

content.js 还可以向当前页面注入 js 脚本，来自动测试网页、爬取网页数据等

后续悬浮窗等功能上线了，再来更新这里

## 打包

我们需要将popup、background.js和content.js一起打包到dist文件，所以配置了三份vite.config.js文件，通过build.js 先将background和content打包到临时目录，再将文件copy到目标目录dist文件中

### build.js

```javascript
import path from 'path'
import fs from 'fs'

// 复制临时目录的文件到目标文件夹
const copyTemp2Build = (tempDir, tarDir) => {
  if (!fs.existsSync(tarDir)) {
    fs.mkdir(tarDir) // 如果不存在目标目录就创建目录
  }

  fs.readdirSync(tempDir).forEach(file => {
    const tempPath = path.join(tempDir, file)
    const tarPath = path.join(tarDir, file)

    if (fs.lstatSync(tempPath).isDirectory()) {
      copyTemp2Build(tempPath, tarPath)
    } else {
      fs.copyFileSync(tempPath, tarPath)
    }
  })
}

// 删除临时目录
const deleteTempDir = tempDir => {
  if (fs.existsSync(tempDir)) {
    fs.readdirSync(tempDir).forEach(file => {
      const tempPath = path.join(tempDir, file)
      if (fs.lstatSync(tempPath).isDirectory()) {
        deleteTempDir(tempPath)
      } else {
        fs.unlinkSync(tempPath)
      }
    })
    fs.rmdirSync(tempDir)
  }
}

// content-script 临时打包目录
const tempContentDir = path.resolve(process.cwd(), process.env.TEMP_CONTENT_DIR)
const tempBackgroundDir = path.resolve(process.cwd(), process.env.TEMP_BACKGROUND_DIR)
const targetBuildDir = path.resolve(process.cwd(), process.env.TARGET_BUILD_DIR)

// 复制 content-script 和 background-script 的build 文件到最终build 目录中
copyTemp2Build(tempContentDir, targetBuildDir)
copyTemp2Build(tempBackgroundDir, targetBuildDir)

// 删除临时目录
deleteTempDir(tempContentDir)
deleteTempDir(tempBackgroundDir)

```

### vite.config.js

vite 默认打包入口是根目录的 index.html，这个用做popup页面，所以默认的vite.config.js主要是用来打包popup页面的

```javascript
import { defineConfig } from 'vite'
import path from 'path'
import react from '@vitejs/plugin-react'
import reactRefresh from '@vitejs/plugin-react-refresh'
import AutoImport from 'unplugin-auto-import/vite'

export default defineConfig({
  plugins: [
    react(),
    reactRefresh(), // 热更新
    AutoImport({
      include: [/\.[tj]sx?$/],
      imports: ['react', 'react-router'],
    }),
  
  ],
  css: {
    // CSS 预处理器的配置选项
    preprocessorOptions: {
      less: {
        additionalData: `@import "${path.resolve(__dirname, 'src/assests/index.less')}";`, // 全局变量
      },
    },
  },

  resolve: {
    alias: {
      '@': path.resolve(__dirname, 'src'),
    },
  },
  build: {
    emptyOutDir: true,
    rollupOptions: {
      input: {
        // 配置所有页面路径，使得所有页面都会被打包
        index: path.resolve(__dirname, 'index.html'),
        login: path.resolve(__dirname, '/pages/login.html'),
      },
      output: {
        assetFileNames: 'assets/[name]-[hash].[ext]',
        entryFileNames: 'assets/[name]-[hash].js',
        chunkFileNames: 'assets/[name]-[hash].js',
      },
    },
  },
})

```

### vite.background.config.js

```javascript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import path from 'path'

// https://vitejs.dev/config/
export default defineConfig({
  build: {
    outDir: process.env.TEMP_BACKGROUND_DIR,
    lib: {
      entry: [path.resolve(__dirname, 'src/background/index.jsx')],
      formats: ['cjs'],
      // 设置生成文件的文件名
      fileName: () => {
        // 将文件后缀名强制定为js，否则会生成cjs的后缀名
        return 'background.js'
      },
    },
  },
  resolve: {
    alias: {
      '@': path.resolve(__dirname, 'src'),
    },
  },
  plugins: [react()],
})

```

### vite.content.config.js

```javascript
import { defineConfig } from 'vite'
import path from 'path'
import react from '@vitejs/plugin-react'
// import { CRX_CONTENT_OUTDIR } from "./globalConfig"
import dotenv from 'dotenv'
dotenv.config()

export default defineConfig({
  build: {
    outDir: process.env.TEMP_CONTENT_DIR,
    // https://cn.vitejs.dev/config/build-options.html#build-lib
    lib: {
      name: 'contentLib',
      entry: path.resolve(__dirname, 'src/content/index.jsx'),
      // content script不支持ES6，因此不用使用es模式，需要改为cjs模式
      // formats: ["es", "cjs"],
      fileName: () => {
        return 'content.js'
      },
    },
    rollupOptions: {
      input: 'src/content/index.jsx',
      output: {
        assetFileNames: assetInfo => {
          // 附属文件命名，content script会生成配套的css
          return 'content.css'
        },
      },
    },
  },
  resolve: {
    alias: {
      '@': path.resolve(__dirname, 'src'),
    },
  },
  define: {
    'process.env.NODE_ENV': null,
  },
  plugins: [react()],
})

```

### npm执行命令

可以根据自己的需求选择打包单个文件或者全部文件

```javascript
 "scripts": {
    "start": "vite",
    "build": "vite build -c vite.config.js && vite build -c vite.content.config.js&& vite build -c vite.background.config.js && node build.js",
    "build-content": "vite build -c vite.content.config.js && node build.js",
    "build-popup": "vite build -c vite.config.js",
    "build-background": "vite build -c vite.background.config.js",
  },
```



