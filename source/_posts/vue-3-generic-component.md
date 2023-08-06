---
title: Vue 3 避坑指南（五）：缺乏范型组件的支持
date: 2023-08-06 16:36:40
tags:
- Vue
- 避坑指南
---

众所周知，相比于 Vue 2，Vue 3 对于 TypeScript 的支持更加完善；但是，依然有一些不够完善的地方。至于有哪些不完善的地方，也不是一下就能说得出来，好像插槽算一个。不管哪一个，对范型组件缺乏支持是它逃不掉的一点。

我们在使用 UI 组件库的时候，像 Select、List 这样的组件，它们天然地不支持范型。这会让你在编写代码时丢失一些必要的类型信息。

<!-- more -->

下面是一个简单的 List 组件，父组件通过一个自定义的函数来渲染 key 和 label：

```html
<!-- ListItems.vue -->
<template>
  <ul>
    <li v-for="item in items" :key="renderKey(item)">
      {{ renderLabel(item) }}
    </li>
  </ul>
</template>

<script setup lang="ts">
defineProps<{
  renderKey: (item: any) => string,
  renderLabel: (item: any) => string
}>()
</script>
```

父组件这样调用：

```vue
<template>
  <ListItems :items="users" 
             :renderKey="renderUserKey" 
             :renderLabel="renderUserLabel" />
</template>

<script lang="ts" setup>
const items = ref<User[]>([])
const renderUserLabel = (user: User) => user.name
const renderUserKey = (user: User) => user.id
</script>
```

有关 `ListItems.vue`，它定义的属性 `renderKey` 和 `renderLabel` 都是函数，其中函数接受的参数类型定义为 `any`. 由于缺乏对于范型的支持，只能将 `item` 定义为 `any`. 但这样会缺乏必要的类型检验。比如，下面的错误调用并不会报错：

```vue
<template>
  <ListItems :items="users" 
             :renderKey="renderUserKey" 
             :renderLabel="renderAddressKey" />
</template>

<script lang="ts" setup>
const items = ref<User[]>([])
const renderUserKey = (user: User) => user.id
const renderAddressLabel = (address: Address) => address.details
</script>
```

甭管我们为什么会将 `renderAddressLabel` 传递给 `renderLabel` 属性，反正这是一个错误。但整段代码是能编译通过的。而如果我们能有范型，将 `ListItems` 用 `User` 类型绑定，那么 `renderLabel` 属性就会提示我们错误。但可惜，并不能使用范型。

*归根结底是因为，Vue 3 的单文件模板不支持范型组件。*
