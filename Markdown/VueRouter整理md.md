1. hash模式和histotry模式

   通过location.hash拿取

   |# 后就是hash的内容

   对通过onhashchange监听hash的改变

   history就是正常的路径

   localtion.pathname

   onpopstate()

   ​

2. Vue.use（）方法

   ​

   ​

3. Vue-router的工作流程

   url的改变 =》触发监听事件 =》改变vue-router的current变量 =》监视current变量的监听者 =》获取新的组件 =》render新组件

4. vue插件

   实际就是对一个install方法执行一遍

   可接收方法，单纯执行

   如果有install,执行install方法或者属性

5. init()方法：监听路由改变事件

   根据mode

   设置默认‘/’

   history有个current属性

   load也要设置current

6. vue.utils.defineReative()

   将一个外部变量关联到vue

   # VueRouter源码解析

   ​

   ### 前端路由原理

   ```
   前端路由：输入URL => js解析地址 => 找到对应地址的页面 => 执行页面生成的JS  => 看到页面

   后端路由：输入URL => 发送请求到服务器  => 服务器解析请求的路径  => 拿取对应的页面  => 返回出去
   ```

   前端路由的本质就是监听URL的变化，然后匹配路由规则，显示出对应的页面，无须刷新。

   路由模式

   - Hash模式
   - History模式

