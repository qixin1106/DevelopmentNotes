# Vue组件传参

## 1.Props简单传参

**父组件 ->子组件**：直接在组件后拼参数，非常简单

```html
<template>
    <div class="super">
        <h3>父组件</h3>
        <ChildView :car="car"/>
    </div>
</template>

<script setup lang="ts" name="SuperView">
import ChildView from '@/views/ChildView.vue'
import { ref } from 'vue';
let car = ref('奔驰')
</script>

<style scoped></style>
```

#### 使用方法

1. 父组件需要先创建一个回调方法
   
   ```ts
   import { ref } from 'vue';
   // 定义一个变量保存子组件给父组件的回传的数据
   let toy = ref('')
   // 定义一个回调方法
   function getToy(value: string) {
          // 收到回传的值，赋值给toy变量
       toy.value = value
   }
   ```

2. 将回调方法传递给子组件
   
   ```html
   <!-- sendToy是给子组件定义的方法名 -->
   <ChildView :car="car" :sendToy="getToy"/>
   ```

3. 子组件调用回调方法，将值传递给父组件
   
   ```html
   <template>
       <div class="child">
           <h3>子组件</h3>
           <button @click="sendToy(toy)">给父玩具</button>
       </div>
   </template>
   <script setup lang="ts" name="ChildView">
   import { ref } from 'vue';
   let toy = ref('奥特曼')
   // 获取接受的参数
   defineProps(['car', 'sendToy'])
   </script>
   ```

## 2.自定义event

1. 父组件定义一个回调方法

```html
<template>
   <div class="super">
       <h3>父组件</h3>
       <ChildView @send-toy="saveToy"/>
   </div>
</template>

<script setup lang="ts" name="SuperView">
import ChildView from '@/views/ChildView.vue'
function saveToy(value: string) {
    console.log('收到：', value);
}
</script>
```

2. 子组件触发

```html
<template>
   <div class="child">
       <h3>子组件</h3>
       <!-- 通过emit来触发父组件定义的方法@send-toy -->
       <!-- <ChildView @send-toy="saveToy"/> -->
       <!-- 当 -->
       <button @click="emit('send-toy','平衡车')">测试</button>
   </div>
</template>

<script setup lang="ts" name="ChildView">
// 获取emit
const emit = defineEmits(['send-toy'])
</script>
```

## 3.mitt

### 安装

```bash
npm install mitt
```

### 使用

创建一个叫`emitter.ts`的文件

```ts
// 导入库
import mitt from 'mitt'
// 创建emitter对象，可以绑定事件
const emitter = mitt()


// 绑定事件-----------------------------------
const key1 = 'myEventKey1'
const key2 = 'myEventKey2'
emitter.on(key1, () => {
    console.log(key1, '调用了');
})
emitter.on(key2, () => {
    console.log(key2, '调用了');
})

// 触发事件，每一秒触发一次
setInterval(() => {
    emitter.emit(key1)
    emitter.emit(key2)
}, 1000)

// 3秒后，解除key1事件绑定
setTimeout(() => {
    // 解除单个绑定事件
    emitter.off(key1)
}, 3000);
// -----------------------------------------


// 暴露对象
export default emitter
```

#### 绑定事件

```ts
// 定义一个事件名
const key1 = 'myEventKey1'
// 参数1：事件名 
// 参数2：回调函数
emitter.on(key1, () => {
    console.log(key1, '调用了');
})
```

#### 调用事件

```ts
emitter.emit(key1)
```

#### 解除单个事件绑定

```ts
emitter.off(key1)
```

#### 解除所有事件绑定

```ts
emitter.all.clear()
```

> 请在vue组件解绑时`onUnmounted`，解除事件绑定，否则会产生内存泄漏
> 
> 即绑定和解绑，成对出现。

## 4.v-model方式

`v-model` 的作用就是用来监听我们的视图层数据和逻辑层数据是否发生了改变，如果视图层数据改变了，相应的逻辑层数据也会随之改变，如果逻辑层改变了，相应的视图层也会跟着改变。

#### 使用

在你的父组件中

```html
<template>
    <div class="super">
        <h3>父组件</h3>
        <!-- v-model用在组件标签上 -->
        <!-- <QixinInput :modelValue="username" @update:modelValue="username = $event" /> -->
        <QixinInput v-model="username"/>
    </div>
</template>

<script setup lang="ts" name="SuperView">
import QixinInput from '@/views/QixinInput.vue'
import { ref } from 'vue'
let username = ref('qixin')
</script>
```

在子组件中

```html
<template>
    <input type="text" :value="modelValue" @input="emit('update:modelValue', (<HTMLInputElement>$event.target).value)">
</template>


<script setup lang="ts" name="QixinInput">
defineProps(['modelValue'])
const emit = defineEmits(['update:modelValue'])
</script>
```

> `:value="modelValue"` 接收数据
> 
> `@input="emit('update:modelValue', ($event.target).value)"` 回传数据
> 
> 双向绑定完成

#### 改名

```html
<QixinInput v-model="username"/>
```

改成

```html
<QixinInput v-model:userName="username"/>
```

> 给v-model传入的值进行命名，可以绑定多个v-model，用名字来区分

```html
<QixinInput v-model:userName="username" v-model:passWord="password"/>
```

那么<QixinInput />中需要做出对应调整

```html
<template>
    <input type="text" :value="userName" @input="emit('update:userName', (<HTMLInputElement>$event.target).value)">
    <input type='password' :value="passWord" @input="emit('update:passWord', (<HTMLInputElement>$event.target).value)">
</template>


<script setup lang="ts" name="QixinInput">
defineProps(['userName', 'passWord'])
const emit = defineEmits(['update:userName', 'update:passWord'])
</script>
```

| 原                                                         | 改为                                                      |
| --------------------------------------------------------- | ------------------------------------------------------- |
| :value="modelValue"                                       | :value="userName"                                       |
| @input="emit('update:modelValue', ($event.target).value)" | @input="emit('update:userName', ($event.target).value)" |
| defineProps(['modelValue'])                               | defineProps(['userName'])                               |
| const emit = defineEmits(['update:modelValue'])           | const emit = defineEmits(['update:userName'])           |

## 5.$attrs方式

#### 父->子->孙，传递参数

这是一种透传的方式，如果子没有使用传递的参数，这些没用过的参数会存在`attrs`中，那么孙组件可以使用子组件中未使用的参数。

在父组件中给`Son`组件传递一堆参数

```html
<template>
    <div class="father">
        <h3>父组件</h3>
        <h4>a: {{ a }}</h4>
        <h4>b: {{ b }}</h4>
        <h4>c: {{ c }}</h4>
        <h4>d: {{ d }}</h4>
        <!-- 给Son组件传值 -->
        <Son :a="a" :b="b" :c="c" :d="d" v-bind="{ x: 100, y: 200 }"/>
    </div>
</template>

<script setup lang="ts" name="Father">
import Son from '@/views/attr-type/SonView.vue'
import { ref } from 'vue'

let a = ref(1)
let b = ref(2)
let c = ref(3)
let d = ref(4)

</script>
```

在`Son`组件中，我们只获取一个`a`，其他的都不动，让其存在`attrs`中，并将`attrs`绑定给`GrandSon`组件

```html
<template>
    <div class="son">
        <h3>子组件</h3>
        <!-- 展示a的值 -->
        <h4>a: {{ a }}</h4>
        <!-- 当其他参数传递过来但是并未获取的情况，会存在attrs中 -->
        <h4>其他attrs {{ $attrs }}</h4>
        <!-- 绑定$attrs -->
        <GrandSon v-bind="$attrs"/>
    </div>
</template>

<script setup lang="ts" name="Son">
import GrandSon from '@/views/attr-type/GrandSonView.vue'
// 仅获取a的值，其他值不需要
defineProps(['a'])
</script>
```

`GrandSon`组件此时可以获取透传的参数

```html
<template>
    <div class="grandson">
        <h3>孙组件</h3>
        <h4>a: {{ a }}</h4>
        <h4>b: {{ b }}</h4>
        <h4>c: {{ c }}</h4>
        <h4>d: {{ d }}</h4>
        <h4>x: {{ x }}</h4>
        <h4>y: {{ y }}</h4>
    </div>
</template>

<script setup lang="ts" name="Grandson">
// 在孙组件中获取
defineProps(['b', 'c', 'd', 'x', 'y'])
</script>
```

#### 孙->父，回传方式

```html
<template>
    <div class="father">
        <h3>父组件</h3>
        <h4>a: {{ a }}</h4>
        <h4>b: {{ b }}</h4>
        <h4>c: {{ c }}</h4>
        <h4>d: {{ d }}</h4>
        <!-- 给Son组件传值，增加回调方法传参-->
        <Son :a="a" :b="b" :c="c" :d="d" v-bind="{ x: 100, y: 200 }" :updateA="updateA"/>
    </div>
</template>

<script setup lang="ts" name="Father">
import Son from '@/views/attr-type/SonView.vue'
import { ref } from 'vue'

let a = ref(1)
let b = ref(2)
let c = ref(3)
let d = ref(4)

// 增加回传方法
function updateA(value: number) {
    a.value = value
}

</script>
```

```html
<template>
    <div class="grandson">
        <h3>孙组件</h3>
        <h4>a: {{ a }}</h4>
        <h4>b: {{ b }}</h4>
        <h4>c: {{ c }}</h4>
        <h4>d: {{ d }}</h4>
        <h4>x: {{ x }}</h4>
        <h4>y: {{ y }}</h4>
        <!-- 通过点击回传数据 -->
        <button @click="updateA(6)">更新A</button>
    </div>
</template>

<script setup lang="ts" name="Grandson">
// 增加获取updateA方法，配置在按钮中
defineProps(['a', 'b', 'c', 'd', 'x', 'y', 'updateA'])
</script>
```

## 6.$refs方式

#### 子暴露属性给父操作

父组件传递给子组件们

先在父组件中创建两个子组件

```html
<template>
    <Child1 />
    <Child2 />
</template>
```

创建2个ref标签

```ts
import { ref } from 'vue'
let c1 = ref()
let c2 = ref()
```

给子组件添加标签

```html
<template>
    <Child1 ref="c1" />
    <Child2 ref="c2" />
</template>
```

在子组件中，创建2个变量，并暴露

```ts
import { ref } from 'vue'
let toy = ref('小黄鸭')
let bookCount = ref(100)
// 暴露给父组件可以操作
defineExpose({ toy, bookCount })
```

在父组件中，可对子组件暴露的属性进行修改

```ts
// 修改子组件暴露的属性
c1.value.toy = '小猪佩奇'
c1.value.bookCount = 50
```

关于`$refs`的使用，相当于获取父组件声明的所有ref

```html
<button @click="getAllChild($refs)">给所有孩子买3本书</button>
```

这里我们得到的`key`分别是`"c1"`+`"c2"`

```ts
function getAllChild(refs: { [key: string]: any }) {
    for (const key in refs) {
        refs[key].book += 3
    }
}
```

> 主要是要利用`defineExpose({ toy, bookCount })`来给父组件暴露属性

#### 父暴露属性给子控制

那么父组件其实也可以暴露给子组件属性

```ts
// 父组件中创建一个属性
let money = ref(100)
```

```ts
// 暴露money属性
defineExpose({ money })
```

在子组件中，我们可以获取父的属性通过`$parent`

```html
<button @click="minusMoney($parent)">花父亲钱</button>
```

```ts
function minusMoney(parent: any) {
    // 每次花掉1元钱，此时此刻，可以直接改变父暴露的属性
    parent.money -= 1
}
```

## 7.provide+inject方式

之前使用了`$attrs`方式透传参数，但是这样会打扰到子组件，使用`provide + inject`方式可以避免打扰，直接传值

先在父中创建一个属性

```ts
let money = ref(100)
```

导入provide，然后定义两个属性。通过`provide`暴露。

```ts
import { ref, reactive, provide } from 'vue'
let money = ref(100)
let computer = reactive({
    brand: 'Apple',
    price: '10000',
})
provide('💰', money)
provide('💻', computer)
```

然后你可以在父组件下面的所有子组件，孙组件等，都可以通过`inject`方式获取参数

```ts
import { inject } from 'vue'
// 第一个参数是key，第二个参数是默认值，你可以通过填写默认值的，方式来帮助ts做类型推断。
let money = inject('💰', '默认')
let computer = inject('💻', {brand: "", price: ""})
```

> 想要回传数据，依然是通过传入回调方法的方式进行

## 8.pinia

略

## 9.slot插槽

略
