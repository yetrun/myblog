---
title: Vue 3 避坑指南（二）：watchEffect 用法建议
date: 2023-07-15 11:18:49
tags:
- Vue3
- 避坑指南
---

本文通过一个路由和变量绑定的案例介绍 `watchEffect` 的使用技巧、限制和注意事项。我们将知道应当在数据就绪后再参与监听，`watch` 比 `watchEffect` 高级应当始终使用 `watch` 即可。

<!-- more -->

## 案例：将路由绑定到响应式变量

现在说一个简化的需求，假设我们有三个变量：`author`、`committer`、`merger`，它们的 id 作为 URL 的 query 参数实时更新。那么，我们首先在代码中定义这三个响应式样变量：

```javascript
const author = ref(null)
const committer = ref(null)
const merger = ref(null)
```

然后，组件初始挂载时通过路由参数获取到变量的值：

```javascript
onMounted(async () => {
  author.value = await fetchUser(route.query.author_id)
  committer.value = await fetchUser(route.query.committer_id)
  merger.value = await fetchUser(route.query.merger_id)
})
```

现在要求 `author`、`committer`、`merger` 三个变量改变时，对应的 URL 也能变更。这时可以用到 `watchEffect`：

```javascript
watchEffect(() => {
  router.replace({
    query: {
      author_id: author.value.id,
      committer_id: committer.value.id,
      merger_id: merger.value.id
    }
  })
})
```

## 问题一：应在数据就绪后再监听

如果 `watchEffect` 写到 `onMounted` 外面，则初始化时 `author.value`、`committer.value`、`merger.value` 都是 `null`，则上述函数执行时会出错。（即使不出错，一开始时地址栏就会根据三个 `null` 值被替换，这也是不对的）。因此，`watchEffect` 应放在 `onMounted` 内部，即在数据初始化后再执行。

```javascript
onMounted(async () => {
  author.value = await fetchUser(route.query.author_id)
  committer.value = await fetchUser(route.query.committer_id)
  merger.value = await fetchUser(route.query.merger_id)

  watchEffect(() => {
    router.replace({
      query: {
        author_id: author.value.id,
        committer_id: committer.value.id,
        merger_id: merger.value.id
      }
    })
  })
})
```

## 问题二：`watchEffect` 会立即执行

`watchEffect` 因为它的机制问题，需要立即执行一遍以得到响应式依赖。但上述监听立即执行会得到一个问题，就是 `router.replace` 会重复加载已经存在的 URL. 虽然不会出现什么大问题，但控制台会给出一个警告，说明此时 `watchEffect` 立即执行还是有问题的，应该等待 `author`、`committer`、`merger` 其中一个变量有更新时才应当执行。

`watch` 默认是延迟执行的，换成 `watch` 就能解决：

```javascript
watch([author, committer, merger], () => {
  router.replace({
    query: {
      author_id: author.value.id,
      committer_id: committer.value.id,
      merger_id: merger.value.id
    }
  })
})
```

## 总结

我们使用 `watchEffect` 是为了让系统自动帮我们计算依赖，但它有个立即执行的限制，在使用时应当注意。如果使用场景中要求我们延迟执行，此时就要换成 `watch`，并把依赖包装成数组作为监听变量。

不论是使用 `watch` 还是 `watchEffect`，都要记住一点，**应当在数据就绪后才开始监听**。我们要知道监听的依赖以及它们当前所处的状态，是加载好的还是一个未知的状态。因为这个原因，`watchEffect` 自动计算依赖的特性似乎就缺少了吸引力，我们要时刻关注到监听的依赖以及它们的当前状态。因此，现实场景中只使用 `watch` 就好，它还能决定是立即执行还是延迟执行。
