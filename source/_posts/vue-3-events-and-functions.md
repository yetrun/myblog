---
title: Vue 3 避坑指南（六）：使用事件还是函数
date: 2023-08-20 09:47:32
tags:
- 避坑指南
- Vue 3
---

Vue 3 使用 `v-on` 指令表示事件。我们知道 React 中没有事件的，它只有属性的概念。事件是通过函数属性表示的，也就是“回调函数”。既然这样，同样的一个效果，在 Vue 3 中既可以使用事件实现，也可以使用函数属性实现，在实际开发中我们应该怎么抉择呢？

<!-- more -->

## 示例一：`FetchLoading` 组件

`FetchLoading` 是一个组件，它内部可以执行一个异步任务，在任务执行期间一直显示 *loading* 效果。该异步任务可通过事件或函数属性传递，我们分别来实现。

### 使用函数属性

```vue
<!-- FetchLoading.vue -->
<script setup lang="ts">
const props = defineProps<{
  fetch: () => Promise<void>
}>()

const loaded = ref(false)
onMounted(async () => {
  await props.fetch()
  loaded.value = true
})
</script>
```

则使用者可以通过传递一个异步函数即可实现 loading 效果（假设 `fetchData` 就是一个异步函数）：

```vue
<FetchLoading :fetch="fetchData" />
```

### 使用事件

```vue
<!-- FetchLoading.vue -->
<script setup lang="ts">
const emit = defineEmits<{
  (e: 'loading', loaded: () => void)
}>()

const loaded = ref(false)
onMounted(async () => {
  emit('loading', () => loaded.value = true)
})
</script>
```

则使用者在响应事件时，需要在异步任务执行后通知 `FetchLoading`：

```vue
<FetchLoading @loading="async (loaded) => {
  await fetchData()
  loaded()
}" />
```

> 通过对比发现，响应事件需要传递额外的参数以实现后续的控制，而函数属性没有这样的要求。一般来说，这种**持续性的状态响应不适合用事件来处理**，事件在点控的时机最为合适，比如 `on-click` 等。放在这里应该是 `before-loading`、`after-loading`. 另一方面，作为执行逻辑的一环，使用函数式属性比较合适。这可以看作是设计模式中的模板模式。

## 示例二：`TodoList` 组件的新增动作

**场景：**点击“新增”按钮，出现一个输入框，输入内容后按下 Enter 键，执行新增动作并让输入框消失。新增的动作可能出错，出错信息需要显示在输入框的下方，此时输入框不消失。也就是说，只有当新增动作成功后，输入框才可以消失，否则一直显示错误信息。

```vue
<!-- TodoList.vue -->
<template>
  <button @click="showInput = true">新增</button>
  <div v-show="showInput" >
    <input v-model="newItem" @keyup:enter="addItem">
    <span>{{ errorMessge }}</span>
  </div>
</template>
<script setup lang="ts">
const showInput = ref(false) // 是否显示新增输入框
const errorMessage = ref('') // 显示错误信息
const newItem = ref('')      // 绑定输入框的内容
</script>
```

以上的关键在于是通过事件接受新增动作还是函数属性，以及 `addItem` 如何实现。

### 使用函数

```typescript
const props = defineProps<{
  addItem: () => Promise<void>
}>()

const addItem = async () => {
  try {
    await props.addItem(newItem.value)
    showInput.value = false // 隐藏输入框
    newItem.value = ''      // 清空输入框的内容
  } catch (e) {
    if (e instanceof AddItemError) { // 仅接受 AddItemError
      errorMessage.value = e.message
    } else {
      throw e
    }
  }
}
```

使用者：

```vue
<template>
  <TodoList :add-item="addItem" />
</template>

<script>
const addItem = async (value: string) => {
  try {
    await remoteSaveItem(value)
  } catch (e) {
    throw new AddItemError(e.message) // 转换成 TodoList 要求的异常格式
  }
}
</script>
```

### 使用事件

```typescript
const emit = defineEmits<{
  (e: 'add-item', value: string): void
}>()

// finish 函数，其接受一个错误消息参数。如果错误消息为空白，则让输入框消失。
const finish = (errorMessage: string) => {
  if (errorMessage) {
    errorMessage.value = errorMessage
  } else {
    showInput.value = false // 隐藏输入框
    newItem.value = ''      // 清空输入框的内容
  }
}
const addItem = () => {
  emit('add-item', newItem.value, finish)
}
```

使用者：

```vue
<template>
  <TodoList @add-item="addItem" />
</template>

<script setup>
const addItem = async (value: string，finish: (errorMessage: string) => void) => {
  try {
    await remoteSaveItem(value)
    finish(null)
  } catch (e) {
    finish(e.message)
  }
}
</script>
```

> 这是一个使用事件较为合适的例子。可以发现，在这个例子里，无论是使用函数属性还是事件，其代码量不相上下。这种作为点控的操作模式适合事件来处理，我个人倾向于使用事件，它比需要转化为一个特定异常的函数式属性更为合适。当然，也就只有函数属性能实现这样的效果，因为事件无法获取回调函数执行后的结果。
