---
title: Vue 3 避坑指南（一）：避免使用 watch(route)
date: 2023-07-09 10:43:58
tags:
  - Vue
  - 避坑指南
---

## 问题背景

当组件需要响应路由更新的时候，有人提到可以用 `watch(route)` 方法。这个方法可以监听路由的一切变化。比如，对于一个文章组件，我们可以通过监听路由来达到重新获取文章的目的：

```javascript
// 监听路由的变化。当我们的路由从 `/articles/1` 跳转到 `/articles/2` 时，下面的执行逻辑是正确的。
watch(route, async () => {
  const articleId = route.params.id
  article.value = await fetchArticle(articleId)
})
```

但是，请千万别这么用。此时组件不仅监听了当前组件路由的变化，也监听了别的组件的路由变化。当路由跳转到不属于当前组件的路由时，上述 `watch` 响应*可能*会触发，从而导致错误的执行逻辑：

```javascript
watch(route, async () => {
  // 如果跳转到别的路由，比如跳转到 `/users/1`，此时获取到的 articleId 是错误的。
  const articleId = route.params.id
  article.value = await fetchArticle(articleId)
})
```

所以，请**避免使用 `watch(route)`，因为它可能会监听到不属于自身的路由变化**。

<!-- more -->

## 修复办法

针对这种情况，有两种办法可以改善。

**第一种，**在 `watch(route)` 内部添加一个判断条件：

```javascript
watch(route, async () => {
  if(route.name === 'article') {
    const articleId = route.params.id
    article.value = await fetchArticle(articleId)
  }
})
```

**第二种，**使用 vue-router 提供的导航守卫：

```javascript
import {onBeforeRouteUpdate} from 'vue-router'

onBeforeRouteUpdate(async (route) => {
  const articleId = route.params.id
  article.value = await fetchArticle(articleId)
})
```

## 出错情况

事实上，由于 vue-router 内部执行机制的原因，出现这样错误的概率是很低的。也正因为很低，往往会被人们所忽略，而且一旦出现问题往往不知所措。我在这里举出一个能够出现这样错误的例子。

在应用里添加 `RouterView`，遇到路由跳转时一般不会遇到这个问题。因为此时组件已经卸载下来了，不会再触发 `watch` 响应。但我在应用里**添加子路由**后，情况就发生了变化。

我的应用里有一个标签页列表（`tabs`）这样的组件，当切换标签页（`tab`）时切换到不同的视图。一共有两种形式的视图和它们对应的路由：

```javascript
// router.js
const routes = [
  {
    path: '/views',
    name: 'views',
    component: HomeView,
    redirect: { name: 'tableView' },
    children: [
      {
        path: 'tableView', // 路由格式 "/views/tableView"
        name: 'tableView',
        component: TableView
      },
      {
        path: 'boardView', // 路由格式 "/views/boardView?boardId=xxx"
        name: 'boardView',
        component: BoardView
      }
    ]
  }
]

export default createRouter({
  history: createWebHashHistory(),
  routes
}
```

```vue
<!-- HomeView.vue：这个是标签页视图的主页面 -->
<template>
  <div>
    <header>
      <!-- 这里直接使用按钮模拟标签页，点击按钮切换到不同的路由视图 -->
      <button @click="router.replace({name: 'tableView'})">
        表格视图
      </button>
      <button @click="router.replace({name: 'boardView', query: {boardId: 1}})">
        看板一
      </button>
      <button @click="router.replace({name: 'boardView', query: {boardId: 2}})">
        看板二
      </button>
      <button @click="router.replace({name: 'boardView', query: {boardId: 3}})">
        看板三
      </button>
    </header>
    <main>
      <RouterView /> <!-- 子路由视图，在这里加载 TableView 或 BoardView -->
    </main>
  </div>
</template>
```

标签页列表下有一个 `TableView`，同时有多个 `BoardView`. 我们在 `BoardView` 中通过路由参数获取数据，并且响应路由变化：

```javascript
<!-- BoardView.vue -->
const fetchData = async () => {
  const boardId = route.query.boardId
  if(boardId) {
    console.log(`Fetch board data of id ${boardId}`) // 正常加载数据
  } else {
    throw new Error(`Illegal board id: ${boardId}`)  // 异常的 boardId，抛错
  }
}

onMounted(fetchData) // 页面加载时和路由切换时都重新加载数据
watch(route, fetchData)
```

**错误出现在这个时候。当我们在看板视图之间切换时没有问题，但我们从看板视图切换到表格视图时，问题就出现了：`watch(route,  fetchData)` 意外地会执行，从而导致错误 `Illegal board id: undefined` 发生。**

在单层路由下不会出现这个问题，而在子路由下会出现这个问题。为了能够完整地复现这个用例，我补充一下入口函数：

```javascript
// main.js
import { createApp } from 'vue'
import { RouterView } from "vue-router"
import router from './router.js'

const app = createApp(RouterView)
app.use(router)
app.mount('#app')
```

*至于为什么单层路由不会出现问题，而子路由却会，我并不清楚，希望懂行的同学能够给予我解答。*
