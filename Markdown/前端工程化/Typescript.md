参考： [https://blog.csdn.net/qq_43869822/article/details/121664818](https://blog.csdn.net/qq_43869822/article/details/121664818)

## 内置工具类型

### Record

> Record 接收两个泛型参数，表示对象键和值的类型，主要是用来定义一个对象的 key 和 value 的类型

先看下 Record 的源码

```javascript
/**
 * Construct a type with a set of properties K of type T
 */
type Record<K extends keyof any, T> = {
    [P in K]: T;
};

// 使用
type recordType = Record<K,T>
```

对象的键的类型都为 K，值的类型为 T，并返回一个新的类型 recordType

## Partial

> Partial 生成一个新类型，该类型拥有与 T 相同的属性，但是所有属性都是可选的

看下源码：

```javascript
type Partial<T> = {
    [P in keyof T]?: Partial<T[P]>
}
```

实际使用有点类似于从对象 A 复制属性，并且生成一个新的类型

```javascript
interface Person {
  name: string
  age: string
  job: string
}

type Child = Partial<Person>
// 等同于
type Child = {
  name?: string
  age?: string
  job?: string
}
```

### Required

> Required 生成一个新类型，该类型与 T 拥有相同的属性，但是所有属性都是必选项

看下源码：

```javascript
type Require<T> = {
    [P in keyof <T>]-?: T[P]
}
```

使用起来是和  Partial 一样的，只是生成的类型属性是必选的



## TS类型

### 基础类型

- 常用：boolean、number、string、array、enum、any、void
- 不常用：tuple、null、undefine、never

```javascript
const count: number = 1;
```

### 对象类型

- interface主要声明的是函数和对象，并且总是需要引入特定的类型
- type 声明可以是任何的数据类型（基本类型别名，联合类型，元组等类型）
- 一般声明都用interface，需要用到其它变量类型使用 type；

```javascript
// 定义一个对象
interface Point {
  x: number;
  y: number;
}

// 定义一个函数
interface SetPoint {
  // key 部分是函数的形参数 value 部分是函数的返回值
  (x: number, y: number): void;  
}

// 继承接口
interface C extends Point {
  // 自己拥有的属性 action
  z: string;
}

```

```javascript
// 定义原生类型
type Name = string;

// 对象
type PartialPointX = { x: number; };
type PartialPointY = { y: number; };

// 联合类型
type PartialPoint = PartialPointX | PartialPointY;

// 元组
type Data = [number, string];
...
```



### 定义数据结构

开发前先定义好所需的数据结构，特别是表单数据、枚举值等；

1. 一个模块或功能定义份文件，命名一般与模块或功能命名，存放在src/types/productLibary文件夹下。
2. src/types/store 文件夹用来存放 Vuex 相关结构。

   - modules 文件夹下对应 @Store/modules 下的模块；
   - 最基础要定义 State 中数据结构

3. src/types/common.d.ts 定义了基础常用的 interface。
4. ​

### 函数的类型

1. 函数参数注意区分，必选/可选参数，可选参数必须在必选参数后面；
2. 需要设置所有的类型，注意区分必选参数和可选参数；

### data类型定义

> data中的对象类型利用类型断言确定数据类型。

```javascript
// 数据类型定义，interface/xxx.ts
export interface Todo {
  id: number;
  name: string;
  completed: boolean;
}

// 使用，views/xxx.vue
<script lang="ts">
import type { Todo } from "@/interface/xx";

export default defineComponent({
  data() {
    return {
      items: [] as Todo[] // 利用类型断言
    }
  },
});
</script>

```

### Props类型定义

> Props 中的对象类型利用类型断言和 PropType< T >

props 属性校验失败会在控制台 warning 报警告。

```javascript
<script lang="ts">
import { PropType } from "vue"; // 属性类型需要PropType支持
import type { TitleInfo } from "@/interface"

export default defineComponent({
  props: {
    // 利用泛型类型约束对象类型
    titleInfo: Object as PropType<TitleInfo>,
  },
})
</script>
```

###computed类型定义

computed要着重标识函数返回类型。

```javascript
computed: {
  doubleCounter(): number {
    return this.counter * 2
  }
}
```

### methods类型定义

标识函数形参和返回类型即可。

普通函数，定义形参类型和返回值类型。

```javascript
export interface Todo{
  id: Number,
  name: String,
  completed: Boolean
}

const todoName = ref("");

function newTodo(todoName: string): Todo {
  return {
    id: items.value.length + 1,
    name: todoName,
    completed: false,
  };
}

function addTodo(todo: Todo) {
  items.value.push(todo);
  todoName.value = "";
}
```



### 声明文件

当使用第三方库时，需要引用它的声明文件，才能获得对应的代码补全、接口提示等功能。

### 第三方库没有提供声明d.ts文件

如果第三方库没有提供声明文件，可以手动添加声明文件

声明文件一般都是放置在src/types文件夹下

```javascript
declare module 'axios'; // 这里的axios声明为any类型
```

### interface

1. 表单或对象数据先定义好数据结构；
2. 尽量避免使用

### 注意

1. 尽量不要把解构和声明写在一起，可读性极差

   ```javascript
   class Node {
     onNodeCheck(checkedKeys: any, { // 解构
         checked, checkedNodes, node, event,
       } : { // 声明
         node: any;
         [key: string]: any;
       }
     ) { 
     }
   }
   ```


## Vuex中使用Ts

### 创建Store

```javascript
// 1. 创建 store 实例, store/index.ts
import { createStore, Store, useStore as baseUseStore } from 'vuex'
import { InjectionKey } from 'vue'
import RootStoreState from '@/types/store/index'
import { getStoreModules } from '@/plugins/index' // 动态引用store模块下文件

export const key: InjectionKey<Store<RootStoreState>> = Symbol()

// 封装useStore，避免每次导入key
export function useStore() {
  return baseUseStore(key)
}

export default createStore<RootStoreState>({
  modules: {
    ...getStoreModules(),
  },
})

// 2. 引入vue, main.ts
import store, { key } from './store'
createApp(App).use(store).mount("#app");

// 3. 使用，views/comp.vue
import { mapState } from "vuex";
export default defineComponent({
  computed: {
    // 映射state counter
    ...mapState(['counter']),
    doubleCounter(): number {
      return this.$store.state.counter * 2;
    },
  },
}
```

### $store类型化

我们需要 `this.$store` 是有明确的类型，就要为组件选项添加一个明确类型的 $store 属性。

可以为 ComponentCustomProperties 扩展 $store 属性。

```javascript
// interface/modules.d.ts
import { Router } from 'vue-router'
import { Store } from 'vuex'
import RootState from './store/index'


declare module '@vue/runtime-core' {
  // provide typings for `this.$store、this.$router`
  interface ComponentCustomProperties {
    $store: Store<RootState>,
    $router: Router
  }
}
```

### useStore()类型化

在 setup 中使用 useStore() 时要类型化，共需要三步：

1. 定义 InjectionKey；
2. app 安装时提供 InjectionKey；
3. 传递 InjectionKey 给 useStore；

```javascript
// 定义一个InjectionKey，约束Store中State类型，store/index.ts
import { InjectionKey } from "vue";
import { State } from "./vuex";

export const key: InjectionKey<Store<State>> = Symbol();

// main.ts 中作为参数2传入vuex插件
import { key } from "./store";
createApp(App).use(store, key).mount("#app");

// 使用时，store就可以有明确类型了，CompSetup.vue
import { useStore } from 'vuex'
import { key } from '../store'

const store = useStore()
const counter = computed(() => store.state.counter);

```

### 简化使用

封装 useStore，避免每次导入 key；

```javascript
import { useStore as baseUseStore } from "vuex";
export function useStore() {
  return baseUseStore(key);
}

// 使用，views/comp.vue
import { useStore } from '../store'
const store = useStore()
```

### 模块化

```javascript
// todo 模块 ，store/modules/todo.ts
import { Module } from "vuex";
import { State } from "../vuex";
import type { Todo } from "../../types";

const initialState = {
  items: [] as Todo[],
};

export type TodoState = typeof initialState;

export default {
  namespaced: true,
  state: initialState,
  mutations: {
    initTodo(state, payload: Todo[]) {
      state.items = payload;
    },
    addTodo(state, payload: Todo) {
      state.items.push(payload)
    }
  },
  actions: {
    initTodo({ commit }) {
      commit("initTodo", [
        {
          id: 1,
          name: "vue3",
          completed: false,
        },
      ]);
    }
  },
} as Module<TodoState, State>;


// 引入子模块，store/index.ts
import todo from "./modules/todo";
const store = createStore({
  modules: {
    todo,
  },
});

// 状态中添加模块信息，vuex.d.ts
import type { TodoState } from "./modules/todo";

export interface State {
  todo?: TodoState;
}
 
  
// 组件中使用，views/comp.vue
export default {
  data() {
    return {
      // items: [] as Todo[],
    };
  },
  computed: {
    items(): Todo[] {
      return this.$store.state.todo!.items
    }
  },
  methods: {
    addTodo(todo: Todo) {
      // this.items.push(todo);
      this.$store.commit("todo/addTodo", todo);
      this.todoName = "";
    },
  },
}

// setup中使用，CompSetup.vue
const items = computed(() => store.state.todo!.items)
store.dispatch('todo/initTodo')

function addTodo(todo: Todo) {
  store.commit('todo/addTodo', todo)
  todoName.value = "";
}
```