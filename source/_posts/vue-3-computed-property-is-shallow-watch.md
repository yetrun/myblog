---
title: Vue 3 避坑指南（四）：计算属性是浅监听
date: 2023-07-30 15:24:16
tags:
- Vue 3
- 避坑指南
---

这几篇写的都是有关于 watch 监听的，殊不知计算属性也有监听的味道在里面。类似于 watchEffect，计算属性会自动计算依赖，并依据依赖变化自动更新变量值。

```javascript
const firstName = ref('')
const lastName = ref('')
const fullName = computed(() => firstName.value + ' '  + lastName.value)
```

接下来通过一个菜单组件的例子，介绍计算属性的浅监听可能带来的问题，以及如何克服这样的问题。但文章最后还是提出一个困扰，关于 Vue 提供的深度监听的特性和函数式思维的抉择是我们开发上的一个难点。

<!-- more -->

## 一个菜单组件的例子

计算属性也只是浅监听，这点可能和 `watchEffect` 一样，如果没注意到这一点，可能导致问题的产生。

假设我们编写一个组件，其中定义一个属性 `users` 用以传递用户数据。然后我们希望以菜单的形式展示用户数据，这时就需要将属性转化为菜单要求的格式。

```vue
<!-- UsersMenu.vue -->
<template>
  <n-menu options="menuOptions" />
</template>

<script setup>
const props = defineProps({
  users: Array
})
const menuOptions = computed(() => {
  return props.data.map(user => ({
    key: user.id,
    label: user.firstName + ' ' + user.lastName
  }))
})
</script>
```

以上定义的 `UsersMenu` 组件，父级组件可以这样使用它：

```vue
<template>
  <UsersMenu users="users" />
</template>

<script setup>
const users = ref([])
const addUser = () => {
  users.value.push({ id: 1, firstName: 'James', lastName: 'Dean' })
}
</script>
```

## 计算属性无法监听到内部变化

如果我们尝试修改 `users` 内部的数据，像上面的 `addUser` 函数做的那样，之后会发现 `UsersMenu` 组件没有任何变化。原因是计算属性只跟踪变量引用的变化，而不会跟踪引用的值内部的变化。所以，当遇到计算属性时，我们要将跟踪的变量赋新值，`addUser` 改成下面这样就有效果了：

```javascript
const addUser = () => {
  users.value = [...users.value, { id: 1, firstName: 'James', lastName: 'Dean' }]
}
```

## 或者使用深度监听

现实中我们很难猜到 `UsersMenu` 组件不监听内部元素的变化，某种感觉上不像 Vue 的思维。我们要想让 `UsersMenu` 组件能够监听内部元素的变化，针对 `users`  属性我们就不用计算函数了，改用深度监听：

```javascript
<!-- UsersMenu.vue -->
<template>
  <n-menu options="menuOptions" />
</template>

<script setup>
const props = defineProps({
  users: Array
})
const menuOptions = []
watch(() => props.users, () => {
  menuOptions = props.data.map(user => ({
    key: user.id,
    label: user.firstName + ' ' + user.lastName
  }))
}, { immediate: true, deep: true })
</script>
```

这样在使用 `UsersMenu` 上就没有什么限制了，父组件可以恢复成最开始的写法。

> 但是，这样就没有问题了吗？频繁地使用深度监听会降低应用的性能，因此完全杜绝深度监听而使用函数式思维，可能是打开 Vue 使用的另一扇大门。但这样的话，我为什么不用 React 呢？
