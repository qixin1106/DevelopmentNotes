# Pinia🍍使用

[Pinia](https://pinia.vuejs.org) 是一个用于 Vue.js 的状态管理库。它是 Vuex 的一个轻量级替代品，专为 Vue 3 设计，但也可以在 Vue 2 中使用。Pinia 提供了一种简单且灵活的方式来管理和组织你的应用状态。

## 在 Vue3 项目中使用 Pinia 作为状态管理工具有以下一些优点和缺点：

**优点**：

1. **简单易用**：Pinia 的 API 设计得非常简单，易于理解和使用。它提供了一种更直观的方式来定义和管理状态，使得代码更易于阅读和维护。

2. **灵活性**：Pinia 允许你根据需要创建任意数量的 store，这使得状态管理更加灵活。

3. **更好的 TypeScript 支持**：Pinia 提供了出色的 TypeScript 支持，这对于 TypeScript 项目来说是一个巨大的优势。

4. **自动化的代码拆分**：Pinia 支持自动化的代码拆分，这可以帮助提高应用的性能。

**缺点**：

1. **新的学习曲线**：尽管 Pinia 的 API 设计得相当简单，但对于习惯了 Vuex 的开发者来说，可能需要一些时间来适应。

2. **社区支持**：虽然 Pinia 是由 Vue.js 的核心团队成员开发的，但它相对于 Vuex 来说还是较新的库，因此可能在社区支持和可用资源方面稍逊一筹。

3. **可能的不稳定性**：由于 Pinia 是一个相对较新的库，可能会有一些尚未发现的问题或者不稳定的地方。

## 以下是如何在 Vue 项目中使用 Pinia 的基本步骤：

1. **安装 Pinia**：你可以使用 `npm` 或 `yarn` 来安装 Pinia。在你的项目目录中运行以下命令：
   
   ```bash
   npm install pinia
   # 或者
   yarn add pinia
   ```

2. **创建并使用 Pinia store**：在你的 `main.js` 或 `main.ts` 文件中，需要你创建的 Pinia store。例如：
   
   ```ts
   import { createApp } from 'vue'
   import App from './App.vue'
   // 导入pinia
   import { createPinia } from 'pinia'
   // 创建app
   const app = createApp(App)
   // 创建pinia
   const pinia = createPinia()
   // 使用pinia
   app.use(pinia)
   // 挂载app
   app.mount('#app')
   ```

3. **创建store文件**：在你的 `src` 目录下创建 `store` 目录，这是约定俗成的命名。假设我们创建一个 `countStore.ts` 文件
   
   ```ts
   import { defineStore } from 'pinia'
   /*
   defineStore 是 Pinia 库中的一个函数，用于定义一个状态存储（store）。这个函数接收两个参数：
   [name]：一个字符串，必须是唯一的，用作该 store 的唯一 id。
   [options]：一个对象，包含了 store 的配置项，比如 state（状态），getters（获取器），actions（动作）等。
   */
   export const useCountStore = defineStore('count', {
     // 动作方法
     // 这种方式的优势,是可统一增加执行逻辑,方便复用
     actions: {
       // 增加sum的值，当sum>=10时，停止。
       increment(value: number) {
         if (this.sum < 10) {
           this.sum += value
         }
       }
     },
     // 真正存储数据的地方
     state() {
       return {
         sum: 0
       }
     },
     // 获取器
     getters: {
       bigSum(): number {
         return this.sum * 10
       }
     }
   })
   ```

4. **使用Pinia状态**：在你的vue文件中 `<script></script>` 尝试如下：
   
   ```ts
   // 导入countStroe.ts
   import { useCountStore } from '@/store/countStore'
   // 导入storeToRefs，用来获取store中的响应式变量
   import { storeToRefs } from 'pinia'
   // 获取countStore对象
   const countStore = useCountStore()
   // 获取其中的变量，该变量是响应式的，可以直接使用
   const { sum, bigSum } = storeToRefs(countStore)
   // 定义一个add方法，展示修改变量的3种方式
   function add() {
       // 第一种方式, 简单粗暴方式
       // countStore.sum += 1
   
       // 第二种方式, 好处是可以一次性批量修改
       // countStore.$patch({
       //     sum: countStore.sum += 1
       // })
   
       // 第三种方式actions, 封装可复用逻辑方式
       countStore.increment(1)
   }
   ```
