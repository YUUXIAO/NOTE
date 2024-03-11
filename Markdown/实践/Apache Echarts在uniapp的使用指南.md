# echarts使用指南

### 相关文档地址

官方地址：[https://echarts.apache.org/handbook/zh/get-started/](https://echarts.apache.org/handbook/zh/get-started/)

配置项参考：[https://echarts.apache.org/zh/option.html#title](https://echarts.apache.org/zh/option.html#title)

## 引入方式

### **NPM 安装 ECharts**

```jsx
npm install echarts --save
```

### **从 CDN 获取**

推荐从 [jsDelivr](https://www.jsdelivr.com/package/npm/echarts) 引用

### **从 GitHub 获取**

[apache/echarts](https://github.com/apache/echarts) 项目的 [release](https://github.com/apache/echarts/releases) 页面可以找到各个版本的链接。点击下载页面下方 Assets 中的 Source code，解压后 `dist` 目录下的 `echarts.js` 即为包含完整 ECharts 功能的文件。

### **在线定制（推荐）**

如果只想引入部分模块以减少包体积，可以使用 [ECharts 在线定制](https://echarts.apache.org//builder.html)功能。

## 跨平台方案

- web中使用
- [在百度智能小程序中使用 ECharts](https://echarts.apache.org/handbook/zh/how-to/cross-platform/baidu-app)（与微信小程序使用不太相同，不能自行更换echarts 版本或使用自定义构建）
- 微信小程序中使用

  - 【原生】提供了一个小程序的组件：[https://github.com/ecomfe/echarts-for-weixin，可以直接copy下载使用](https://github.com/ecomfe/echarts-for-weixin%EF%BC%8C%E5%8F%AF%E4%BB%A5%E7%9B%B4%E6%8E%A5copy%E4%B8%8B%E8%BD%BD%E4%BD%BF%E7%94%A8)

    ```jsx
    
    <view class="container">
      <ec-canvas id="mychart-dom-bar" canvas-id="mychart-bar" ec="{{ ec }}"></ec-canvas>
    </view>
    ```


  **如果按步骤在项目中仍没有显示组件，注意有没有设置外层父节点的样式：**

  ![](../images/%E5%AE%9E%E8%B7%B5/echart/Untitled.png)

  - 【uni-App】**在uniApp项目中出现的问题：**
  - 该项目提供的是原生小程序项目文件目录及写法，如果要在uniapp 项目中使用的话，有两种方案：

    - **直接把项目相关文件 Copy 到 wxcomponents 文件夹下，并在 page.json 文件中配置usingComponents 属性完成注册**

      - echarts 文件体积很大，即使采用组件动态下载注册的话，echarts.min.js 文件也要 500kb以上，如果在项目全局注册会导致主包体积飙升（不合算，放弃这个方案）
      - 组件本身支持了同层渲染的问题，解决了在微信小程序中由于 canvas 是原生组件导致的层级最高，并且无法覆盖的问题

    - **自己写一个vue写法的组件，当成单组件引入对应分包**

      - 需要处理同层渲染的问题，可以参考 echarts-for-weixin ，处理相关api和 事件
      - 初始化会有问题，要默认是懒加载的方式处理


  - 由于小程序受包大小限制，如果希望减小包体积大小，可以使用[自定义构建](https://echarts.apache.org//builder.html)生成并替换 `echarts.js`


## 配置项的问题

### 点击/拖拽事件，获取当前图表数据

echarts 提供了 touchStart、touchMove、touchEnd 和 error四个事件，click事件对应的是 touchEnd

但是返回的对象里面只有操作位置等信息，不能直接拿不到坐标轴和折线数据

所以采用了 监听 tooltip（提示框浮层） 的formatter 回调方法来判断当前坐标数据已修改（曲线救国

![](../images/%E5%AE%9E%E8%B7%B5/echart/Untitled%201.png)

![](../images/%E5%AE%9E%E8%B7%B5/echart/Untitled%202.png)

![](../images/%E5%AE%9E%E8%B7%B5/echart/Untitled%203.png)

![](../images/%E5%AE%9E%E8%B7%B5/echart/Untitled%204.png)

### 初始化/更新坐标轴

echarts 图表实例对象，提供了**[dispatchAction](https://echarts.apache.org/zh/api.html#echartsInstance.dispatchAction) 方法来触发触发图表行为，**例如图例开关`egendToggleSelect`,高亮指定的数据图形`highlight`，显示提示框`showTip`等等

![](../images/%E5%AE%9E%E8%B7%B5/echart/Untitled%205.png)

### 数据处理

`ECharts` 中实现异步数据的更新非常简单，在图表初始化后不管任何时候只要通过 `setOption`  
 填入数据和配置项就行

### 坐标移动时获取所有折线的的数据

可以通过tooltip配置的formatter属性获取所以折线数据（包括seriesName、axisValue、value等 ）

![](../images/%E5%AE%9E%E8%B7%B5/echart/Untitled%206.png)

### xAxis （X轴配置项）

如果需要在操作图表时，实时获取其他数据，可以考虑在setOption 时，将相关数据拼接在x轴label上，通过拼接截取的方式，在移动坐标时就可以获取需要的数据

![](../images/%E5%AE%9E%E8%B7%B5/echart/Untitled%207.png)

### [**Tooltip（提示框浮层）**](https://echarts.apache.org/zh/option.html#tooltip)

- 【关于自定义浮层内容】 ：formatter 可以在使用 富文本的方式自定义样式，但是在小程序中行不通！！：

  目前发现可以用 ｛marker ｝获取对应每一项的原点样式

  可以用 ｛\n｝换行

  ![](../images/%E5%AE%9E%E8%B7%B5/echart/Untitled%208.png)


## 踩坑指南

### canvas 层级问题

由于canvas属于小程序原生组件，原生组件拥有最高的层级，页面中的其他组件无论设置 z-index 为多少，都无法盖在原生组件上，所以当有弹窗、侧边菜单出现时，会出现被canvas遮挡的情况，解决方法有2种：

1. **使用微信提供的api将echarts转化为图片显示**，也就是说不在页面中显示canvas而只显示一张图片，这样做的缺点是图表失去了交互能力（我们的需求是需要能够拖拽操作图表，所以这个方法不符合，放弃）
2. **canvas开启**[**同层渲染**](https://developers.weixin.qq.com/miniprogram/dev/component/native-component.html#%E5%8E%9F%E7%94%9F%E7%BB%84%E4%BB%B6%E5%90%8C%E5%B1%82%E6%B8%B2%E6%9F%93)

   这样在视觉上我们的图表是按照正常的层级展示，不会覆盖在所有组件之上，但是如果打开弹窗时，操作弹窗依然会渗透到组件上（解决这个问题的采取的方案是，在打开弹窗时禁用图表功能，关闭弹窗时再解开禁用）

   ![](../images/%E5%AE%9E%E8%B7%B5/echart/Untitled%209.png)


### **tooltip在H5模式下tooltips下不显示的问题（echarts4.x和echarts5.x）**

在 echarts 源码里有这一段代码，

```jsx
if (typeof wx === 'object' && typeof wx.getSystemInfoSync === 'function') {
```

解决方法：

- 在echarts 源码里修改成：

  ```jsx
  false && 'function' == typeof wx.getSystemInfoSync
  ```

- 在 `main.js`添加如下代码：

  ```jsx
  window.wx = {}
  ```
