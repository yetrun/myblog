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

## 基本介绍

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

## 补充介绍

有关 Vue 3 支持范型组件的方案，其实有支持的办法。方法是通过原生的 `defineComponent` 方法再结合 TSX. 以下是我找到的相关资料：

> [Vue 3 怎么写带泛型的组件？ - 崮生的回答 - 知乎](https://www.zhihu.com/question/440637299/answer/1697377342)

按照这个方案呢，我没有验证。但是有一个明显的问题，每次用到一个类型时，就会创建一个新的组件。也就是说 `List<number>` 和 `List<string>` 是两个组件。所以，我有所怀疑这是不是最好的方案，或者说不存在一个真正的基于原生 `defineComponent` 函数的范型方案。

另外，和我论坛里讨论的热心开发者 **[agileago](https://www.v2ex.com/member/agileago)** 的工作成果，它的框架是可以使用范型组件和类型插槽的。但用起来还有些不明白的地方，需要声明一些额外的东西。有兴趣的读者详见它的 GitHub 成果：

> [agileago/vue3-oop](https://github.com/agileago/vue3-oop)

此外，据小道消息所说 Vue 3 官方打算为单文件组件引入范型，其语法大致是：

```vue
<script setup generic="T">
// 其中的 T 就是范型，例如
defineProps<{
  items: T[]
}
</script>
```

有关范型方面的内容我就只做这些研究了，其中最让人期待的是 Vue 3 单文件组件引入的范型声明语法。毕竟，如果因为范型而放弃单文件组件转而使用 TSX，实在是两难的取舍。
