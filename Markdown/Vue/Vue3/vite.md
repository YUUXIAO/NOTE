1. <script setup>
   - 减少组件声明和导出 
   - import 直接导入组件，无需声明
   - defineProps 声明 props
   - defineEmit 声明 emit
   - 获取上下文： useContext()，挂载了 attrs、slots、emit、expose
   - ctx.expose 向外暴露
2. Mock插件： vite-plugin-mock
3. vue-router@4.x和vuex@4.x
4. 样式管理 
5. 引入 element3
   - plugins 插件形式
   - export default function (app)
   - app.use(element3)
6. 基础布局