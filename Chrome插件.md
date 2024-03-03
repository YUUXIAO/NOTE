相关文件：

https://zhuanlan.zhihu.com/p/634471047

写法相关：

- 【React生命周期】:https://blog.csdn.net/luobo2345/article/details/122818947
- 【props、state、refs】https://blog.csdn.net/qq_43260366/article/details/127969013
  - props 传递方法、组件、数据

路由相关（React-router-dom V6）

- https://blog.csdn.net/weixin_68658847/article/details/130499709
- https://blog.csdn.net/qq_45750263/article/details/128292161
- https://www.cnblogs.com/vevian/p/17561545.html
- https://devpress.csdn.net/beijing/64f7432ee0aa6850f5a20f7f.html?dp_token=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpZCI6MTQ2Nzk1NSwiZXhwIjoxNjk3NTA4NTk1LCJpYXQiOjE2OTY5MDM3OTUsInVzZXJuYW1lIjoibTBfMzgwOTk2MDcifQ.xh4BbmrdKSEpSbf2_vnJ6mXB0rUKcJeDBri8GwPs4Vw
- https://blog.csdn.net/qq_30769437/article/details/128149273
- https://zhuanlan.zhihu.com/p/431389907 


## 初始化项目

https://zhuanlan.zhihu.com/p/649468450

## 安装构建工具

1. 卸载脚手架自带的 react-sctipts（这个也是内部通过webpack构建的）

   ![img](C:\Users\slb0930\AppData\Local\Temp\企业微信截图_16969028332640.png)

   react-scripts 可以参考这片文章： https://blog.csdn.net/zxl1990_ok/article/details/108857437

2. 安装 less

   vite 自带了less，直接在vite.config.js 

   ```
    css: {
       preprocessorOptions: {
         less: {
           additionalData: `@import "${path.resolve(
             __dirname,
             "src/assests/var.less"
           )}";` // 引入全局自定义变量
         }
       }
     },
   ```

   ​

3. 安装vite、

   ```
   npm install --save-dev vite @vitejs/plugin-react
   ```

   1. vite.config.js

      ```javascript
      iimport { defineConfig } from "vite"
      import path from "path"
      import react from "@vitejs/plugin-react"
      import reactRefresh from "@vitejs/plugin-react-refresh"
      import AutoImport from "unplugin-auto-import/vite"

      export default defineConfig({
        plugins: [
          react(),
          reactRefresh(), // 热更新
          AutoImport({
            include: [/\.[tj]sx?$/],
            imports: ["react", "react-router"]
          })
        ],
        css: {
          preprocessorOptions: {
            less: {
              additionalData: `@import "${path.resolve(
                __dirname,
                "src/assests/var.less"
              )}";` // 引入全局自定义变量
            }
          }
        },

        resolve: {
          alias: {
            "@": path.resolve(__dirname, "src")
          }
        }
      })
      ```


      ```

   2. vite 插件引入

      - 样式懒加载 :Antd 的样式使用了 Less 作为开发语言，需要 antd、less，要在 vite.config.js 中指定 css 的预处理器，为了减小 antd 的 css，变全局引入为按需引入，所以需要安装插件 vite-plugin-style-import 

        ```
        npm install vite-plugin-style-import -D
        ```

        ```javascript
        // vite.config.js
        import { defineConfig } from "vite";
        import react from "@vitejs/plugin-react";
        import { createStyleImportPlugin, AntdResolve } from "vite-plugin-style-import";

        // https://vitejs.dev/config/
        export default defineConfig({
          plugins: [react(), createStyleImportPlugin({ resolve: [AntdResolve] })],
        });

        ```

        ​

3. 安装 typescripts


### 配置路由

```
npm install react-dom react-router-dom
```

- 根目录创建router文件夹

- **创建router对象**，配置路由字典，这里路由模式为哈希模式（createHashRouter）

- 注意：这里要用createHashRouter， createBrowserRouter浏览器可以运行但是build之后在插件popup会报错404，感觉可能是默认路由路径匹配的问题

https://www.cnblogs.com/newBugs/p/15965558.html
- ​

  注意这里的路由组件要大写开头

  ```javascript
  import { createHashRouter } from "react-router-dom"
  import PopupHome from "@/popup/pages/Home"
  import Entry from "@/popup/pages/Entry"
  import ContentHome from "@/content/index"

  const routerConfigs = createHashRouter([
    {
      path: '/',
      element: <PopupHome />,
    },
    {
      path: '/later',
      element: <LaterPage />,
    },
  ])

  export default routerConfigs
  ```

- main.js 引入路由，再把路由配置传入就行

  ```javascript
  import {RouterProvider} from 'react-router-dom'
  import routers from '@/router/router.jsx';
  ```

     function Entry(){
         return <RouterProvider router={routers} ></RouterProvider>
     }
    
     ```
    
     ​

1. 安装 redux、react-redux

   ```
   npm install --save redux react-reduxredux
   ```

### redux

1. https://blog.csdn.net/weixin_45605541/article/details/127078701
2. https://www.cnblogs.com/chccee/p/17138406.html
3. https://www.jb51.net/article/266604.htm
4. https://blog.csdn.net/weixin_46029529/article/details/130832689

- redux和react-redux的区别？？

2. 配置eslint、preitter

   ```
   npm install eslint eslint-plugin-react-hooks eslint-config-react-app --D
   ```

- 请求
1. 安装axios，封装
2. background.js 只能使用 fetch 发起请求，所以 background.js 用fetch请求

3. build 项目处理

vite 多页面处理


## content.js

content.js 主要是可以让我们的插件与浏览器页面进行交互（主要是对 DOM 元素进行增删改查等操作），比如修改页面样式，增加按钮、浮窗等控件，方便用户对网页进行操作，

content.js 还可以向当前页面注入 js 脚本，来自动测试网页、爬取网页数据等

### 新增vite.content.config.js

- vite 默认打包入口是根目录的 index.html,

### 