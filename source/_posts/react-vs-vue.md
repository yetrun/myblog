---
title: React 与 Vue 3 比较（我为什么选择 React）
date: 2022-10-31 16:15:58
tags:
  - Vue
  - React
---

虽然说 Vue 更新到第三版之后它的表达能力更强了，但相比于 React 还是有几个对我而言不可接受的缺点，现列举一二。

<!-- more -->

## 缺乏对范型组件的支持

React 的组件其实是一个函数，所以它天然地具备范型的能力。而 Vue 3 组件是通过 `defineComponent` 函数生成的，该函数没有对应的范型 API，所以 Vue 3 组件是不具备范型的能力。

参考资料：

- [知乎：Vue 3 怎么写带泛型的组件？](https://www.zhihu.com/question/440637299).

范型组件有什么用呢？假设我要用一个 List 组件，它具备两个属性：

```html
<List :items="[1, 2, 3]" :renderItem="(item: string) => item" />
```

以上调用在 Vue 中是不会报错的，因为 `renderItems` 的类型无法根据 `items` 的类型予以约束。

## 插槽缺乏对类型的支持

Vue 3 中的插槽是这么写（貌似 Vue 2 中也是同样的写法）。

首先，组件（`Comp.vue`）中定义插槽：

```html
<template>
  <slot name="header" text="banana" />
</template>
```

应用组件时使用插槽：

```html
<Comp>
  <template #header v-slot="{ test }">
    {{ test }}
  </template>
</Comp>
```

可以看出，Vue 3 没有预留任何定义类型的地方，以上代码不会报错。
