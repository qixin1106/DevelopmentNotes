# Vue3编程路由跳转

## 0.安装组件

`npm i vue-router`

## 1.创建路由配置

先创建一个`routers`文件夹, 并在其中创建`index.ts`

![](/Users/qixin/Library/Application%20Support/marktext/images/2024-01-29-14-17-44-image.png)

```ts
// routers/index.ts
import { createRouter, createWebHistory } from 'vue-router'

import HomePage from '@/pages/HomePage.vue'
import AboutPage from '@/pages/AboutPage.vue'
import ProductsPage from '@/pages/ProductsPage.vue'
import ContactPage from '@/pages/ContactPage.vue'

// 配置路由规则
const router = createRouter({
  routes: [
    {
      name: 'homepage',
      path: '/home',
      component: HomePage,
    },
    {
      name: 'aboutpage',
      path: '/about',
      component: AboutPage

    },
    {
      name: 'productspage',
      path: '/products',
      component: ProductsPage
    },
    {
      name: 'contactpage',
      path: '/contact',
      component: ContactPage,
      // 该方式,只能使用params传参方式,并在path中需设置参数名,如'/contact/:name/:phoneNo/:address'
      // props: true,
      // 函数方式,支持query传参方式,并且不需要定义参数,表现为'localhost:5173/contact?name=xxx&phoneNo=xxxx&address=xxxx'
      props(route) {
        return route.query
      }
    },
    {
      path: '/',
      redirect: '/home'
    }
  ],
  history: createWebHistory()
})

// 暴露
export default router
```

## 2.加载路由配置

修改`main.ts`

```ts
import { createApp } from 'vue'
import App from './App.vue'
// 导入路由配置
import router from '@/routers/index'

// 创建app
const app = createApp(App)
// 使用路由器
app.use(router)
// 挂载app
app.mount('#app')
```

### App.vue

简单的布局

```html
<!-- App.vue -->
<template>
  <div>
    <!-- 头区域 -->
    <Header/>
    <!-- 主区域 -->
    <div class="container">
      <!-- 左侧导航区 -->
      <Navigation class="navigation"/>
      <!-- 右侧内容区, 路由跳转切换的部分 -->
      <Content class="content"/>
    </div>
  </div>
</template>

<script setup lang="ts" name="App">
import Header from '@/components/HeaderView.vue'
import Navigation from '@/components/NavigationView.vue'
import Content from '@/components/ContentView.vue'
</script>


<style scoped>
.container {
  display: flex;
}

.navigation {
  flex: 0 0 auto; /* 设置导航区域不可伸缩 */
  width: 200px; /* 设置导航区域的宽度 */
}

.content {
  flex: 1; /* 设置内容区域可伸缩，占据剩余空间 */
}

</style>
```

![](/Users/qixin/Library/Application%20Support/marktext/images/2024-01-29-14-39-38-image.png)

### ContentView.vue

展示内容区域

```html
<!-- ContentView.vue -->
<template>
    <div>
        <!-- 路由跳转的区域,占位 -->
        <RouterView></RouterView>
    </div>
</template>

<script setup lang="ts" name="Content">
import { RouterView } from 'vue-router'
</script>

<style scoped>
</style>
```

### NavigationView.vue

导航页面设置

```html
<!-- NavigationView.vue -->
<template>
    <nav>
        <ul style="display:flex; flex-direction: column;">
            <li><button @click="showHome">首页</button></li>
            <li><button @click="showAbout">关于</button></li>
            <li><button @click="showProducts">产品</button></li>
            <li><button @click="showContact">联系</button></li>
        </ul>
    </nav>
</template>

<script setup lang="ts" name="Navigation">
// 导入库
import { useRouter } from 'vue-router';

// 获得router
const router = useRouter()

// 跳转方法
function showHome() {
    router.push("/home")
}
function showAbout() {
    router.push("/about")
}
function showProducts() {
    router.push("/products")
}
function showContact() {
    // push `to`对象方式,支持传递参数
    router.push({
        // query传参可以使用'path'/'name'都可以
        // params传参只能使用'name'
        path: 'contact',
        query: {
            phoneNo: '17500510512',
            name: '泰森',
            address: '神域阿斯加德'
        }
    })
}
</script>

<style scoped></style>
```
