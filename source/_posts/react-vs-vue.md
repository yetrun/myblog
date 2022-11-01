---
title: React 与 Vue 3 比较（我为什么选择 React）
date: 2022-10-31 16:15:58
tags:
  - Vue
  - React
---

## Vue 的致命缺点

虽然说 Vue 更新到第三版之后它的表达能力更强了，但相比于 React 还是有几个对我而言不可接受的缺点，现列举一二。

<!-- more -->

### 缺乏对范型组件的支持

React 的组件其实是一个函数，所以它天然地具备范型的能力。而 Vue 3 组件是通过 `defineComponent` 函数生成的，该函数没有对应的范型 API，所以 Vue 3 组件是不具备范型的能力。

参考资料：

- [知乎：Vue 3 怎么写带泛型的组件？](https://www.zhihu.com/question/440637299).

范型组件有什么用呢？假设我要用一个 List 组件，它具备两个属性：

```html
<List :items="[1, 2, 3]" :renderItem="(item: string) => item" />
```

以上调用在 Vue 中是不会报错的，因为 `renderItems` 的类型无法根据 `items` 的类型予以约束。

### 插槽缺乏对类型的支持

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

## React 的致命缺点

这么说 React 就没有显著的缺点了吗？不是这样的。React 有一些奇怪的缺陷我也要列举一二。

### useEffect 不支持 async 函数

答案是 `useEffect` 要返回一个销毁函数，所以它不支持 async 函数（此时返回的是一个 Promise 而不是一个函数）。具体情况可参见：

> [React useEffect 不支持 async function 你知道吗？](https://zhuanlan.zhihu.com/p/425129987)

讲得很好，也讲到如何解决。只不过我觉得无论是内部创建一个异步函数还是采用立即执行函数的写法都很傻逼。致命缺点。

好在文章中也讲到，可以通过 *自定义 Hook* 来解决。也算是解决了我的一个痛点吧。

### React 的 Hook 必须写在顶层

因为 React 是依赖特定的顺序来识别同一个 Hook 的，因此使用 Hook 必须写在组件函数的顶层。尤其是不能在 `if`、`else` 等嵌套块中使用 Hook. 这极大地提高了编写工作的心智负担。

好在 React 提供了 `eslint-plugin-react-hooks` 这个包用于检查，也避免了我们不正确地调用 Hook 函数。

## Vue 的可喜优点

Vue 就不是完全无优点的吗？也不是。只不过 Vue 致命缺点对我来说更加不可接受而已。我想 Vue 对比于 React 的显著优点是：

### 单文件结构

单文件结构显著地减少了目录里文件的数量，干干板板地很清爽。

而且 Vue 还提供了只作用于单个组件的 CSS 集成方案，即 `<style scoped>`. React 没有提供这种方案，需要自己手动地构造。

有时候，去看 Vue 单文件的时候，我发现将脚本和模板分开的写法也是一种很清爽的方案。只不过 React 将模板看成 JS 的思路能创造出更大的可能性。

### 我很熟悉

从 Vue 2 开始，直到 Vue 3，我用起 Vue 也多多少少有五六年了。而 React 尚且接受不到一个月。
