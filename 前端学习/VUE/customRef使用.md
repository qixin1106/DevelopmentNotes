# ref 和 customRef

在 Vue 3 中，`ref` 和 `customRef` 都是用来创建响应式数据的函数，但它们的使用方式和目的有所不同。

- `ref`: `ref` 用于创建一个包装器对象，可以将基本类型值或对象转换为响应式数据。`ref` 对象仅有一个 `.value` 属性，指向该内部值。如果将对象分配为 `ref` 值，则它将被 `reactive` 函数处理为深层的响应式对象。在 `template` 内使用 `ref` 对象，会自动解包。

- `customRef`: `customRef` 创建一个自定义的 `ref`，并对其依赖项跟踪和更新触发进行显式控制。它需要一个工厂函数，该函数接收 `track` 和 `trigger` 两个函数作为参数，并且应该返回一个带有 `get` 和 `set` 的对象。自定义 `ref` 的 `get` `set`，用于某些数据变更操作的额外功能封装，可以理解为 `computed` 指令类似的功能之类的。

总的来说，`ref` 是一个更通用的工具，适用于大多数情况，而 `customRef` 则提供了更高级的控制，允许你自定义响应式数据的行为。

## ref例子

一个ref的简单用法，当你在输入框输入内容时，可以同时刷新`<h2>`标签内容

```html
<template>
  <div class="app">
    <h2>{{ msg }}</h2>
    <input type="text" v-model="msg">
  </div>
</template>

<script setup lang="ts" name="App">
import { ref } from 'vue'
let msg = ref('你好')
</script>

<style scoped></style>
```

## customRef例子

定义一个最简单的customRef

```ts
// 导入
import { customRef } from 'vue'

// 初始值
let initValue = '你好'
let msg = customRef((track, trigger) => {
  return {
    // 读取时调用
    get() {
      // 追踪数据，一旦变化就去更新。
      track()
      // 返回
      return initValue
    },
    // 修改时调用
    set(value) {
      // console.log('set', value);
      initValue = value
      // 触发刷新，通知vue，数据变化了。
      trigger()
    }
  }
})
```

在 Vue 3 中，`customRef` 函数接收一个工厂函数作为参数，这个工厂函数接受 `track` 和 `trigger` 两个函数作为参数，并返回一个带有 `get` 和 `set` 方法的对象

- `track`: `track` 函数通常在 `get` 方法中调用，用于追踪数据的变化。当你读取数据时，`track` 会被调用，以便 Vue 可以知道哪些副作用（例如计算属性或渲染函数）依赖于该数据。这样，当数据变化时，Vue 就知道需要重新运行这些副作用。

- `trigger`: `trigger` 函数通常在 `set` 方法中调用，用于触发响应。当你修改数据时，`trigger` 会被调用，以通知 Vue 数据已经改变，需要重新运行依赖于该数据的任何副作用。

总的来说，`track` 和 `trigger` 允许你显式地控制 Vue 的依赖跟踪和更新触发机制，这在创建自定义响应式数据时非常有用

> 这里你可以根据自己的特殊需求逻辑，来定义你触发的规则，这就是自定义的好处。

举个例子：如果我们想在数据变更时，延迟500毫秒再触发。

```ts
// 导入
import { customRef } from 'vue'

// 初始值
let initValue = '你好'
// 定义一个timer
let timer: number
// 定义一个customRef
let msg = customRef((track, trigger) => {
  return {
    // 读取时调用
    get() {
      // 追踪数据，一旦变化就去更新。
      track()
      // 返回
      return initValue
    },
    // 修改时调用
    set(value) {
      // 清除之前的timer，类似做防抖的思路。
      clearTimeout(timer)
      // 设置timer，延迟500ms触发。
      timer = setTimeout(() => {
        initValue = value
        // 触发刷新，通知vue，数据变化了。
        trigger()
      }, 500)
    }
  }
})
```



## 结合Hooks使用

通常在项目中我们会结合hooks方式使用，因为每次使用都要写一坨代码，显然不符合开发的基本思路。

#### 创建`useMsgRef.ts`文件

```ts
// 导入
import { customRef } from "vue";

/// HOOKS
/// 带延迟的ref
/// @param { initValue } 初始化的值
/// @param { delay } 延迟毫秒
export default function (initValue: string, delay: number) {
  // 定义一个timer
  let timer: any;
  // 定义一个customRef
  const msg = customRef((track, trigger) => {
    return {
      // 读取时调用
      get() {
        // 追踪数据，一旦变化就去更新。
        track();
        // 返回
        return initValue;
      },
      // 修改时调用
      set(value) {
        // 清除之前的timer，类似做防抖的思路
        clearTimeout(timer);
        // 设置timer
        timer = setTimeout(() => {
          initValue = value;
          // 触发刷新，通知vue，数据变化了。
          trigger();
        }, delay);
      },
    };
  });
  // 返回msg对象
  return { msg };
}
```



#### 使用该hook



```html
<template>
  <div class="app">
    <h2>{{ msg }}</h2>
    <input type="text" v-model="msg">
  </div>
</template>

<script setup lang="ts" name="App">
// 导入
import useMsgRef from '@/hooks/useMsgRef'
// 使用
let { msg } = useMsgRef('Hello', 500)
</script>

<style scoped></style>

```
