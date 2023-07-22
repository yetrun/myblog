---
title: 'Vue 3 避坑指南（三）：让 watch 只响应内部的变化'
date: 2023-07-22 15:33:53
tags:
- Vue 3
- 避坑指南
---

`watch` 通常只能监听值的改变，也就是我们平常所说的“浅”监听。如果要监听深层次的属性，需要加一个选项 `deep: true`. 但这时，如果监听的变量本身值的改变，和内部属性值的改变，都会触发监听函数。如果我们要区分这两种情况，需要用到一些手段。

<!-- more -->

## 背景：一个绘图的案例

假设我们正在做一个绘图的应用，所谓图形是指圆形、正方形、三角形等。`shapeId` 表示当前选中的图形 ID，`shapeOptions` 表示当前选中的图形的选项。当 `shapeOptions` 改变时，我们应当重新绘制当前选中的图形。下面是示例代码：

```javascript
const shapeId = ref(null)
const shapeOptions = ref(shapeOptions)

// 监听 shapeId，同步更新 shapeOptions
watch(shapeId, () => {
  if (shapeId) {
    shapeOptions.value = fetchShapeOptions(shapeId.value)
  } else {
    shapeOptions.value = null // 没有选中的情况
  }
})
// 监听 shapeOptions，同步更新当前选中的图形
watch(shapeOptions, () => {
  if (shapeOptions.value) {
    updateShape(shapeId.value, shapeOptions.value)
  }
}, { deep: true })
```

乍一看没有问题。但这里有个潜在的问题，当我们仅仅是选中图形而没有更新图形选项时，第二个监听函数也会执行。为了防止潜在的 Bug，更新应当在选项确实被更新时才可执行。

> 选中图形的流程：设置 shapeId.value 为一个新值 -> 触发第一个监听函数 -> 触发第二个监听函数。

## 只监听内部变化

这里我们做一个约束：选中新图形会导致 `shapeOptions` 被设为新值；换言之，`shapeOptions` 被设为新值则一定是选中新图形的情况。这要求我们在应用中，如果要修改图形的选项，一定要直接以修改 `shapeOptions` 内部值的方式，也就是说：

```javascript
shapeOptions.value.foo = 'new value'
```

是对的。而：

```javascript
shapeOptions.value = Object.assign({}, shapeOptions.value, { foo: 'new value '})
```

是错误的。

因此，我们可以在监听 `shapeOptions` 时，通过判断值有没有改变而决定是否更新图形：

```javascript
watch(shapeOptions, (newShapeOptions, oldShapeOptions) => {
  if (newShapeOptions === oldShapeOptions && shapeOptions.value) {
    updateShape(shapeId.value, shapeOptions.value)
  }
}, { deep: true })
```

## 通过 `shapeId` 控制监听函数

如果对以上的编码限制很方案，还可以通过自己控制监听函数的方式来做到，只是代码没有上述方案那么简单了。

```javascript
let unwatchShapeOptions = null
watch(shapeId, () => {
  unwatchShapeOptions && unwatchShapeOptions() // 当 shapeId 改变时，旧的监听函数取消
    
  if (shapeId) {
    shapeShapeOptions.value = fetchShapeOptions(shapeId.value)
    unwatchShapeOptions = watch(shapeOptions, () => { // 此刻才开始监听 shapeOptions
	  if (shapeOptions.value) {
 	    updateShape(shapeId.value, shapeOptions.value)
	  }
  	}, { deep: true })
  } else {
    shapeOptions.value = null // 没有选中的情况
    unwatchShapeOptions = null
  }
})
```

这时候，不论你是通过 `shapeOptions.value.foo = 'new value'` 的方式，还是通过 `shapeOptions.value = newShapeOptions` 的方式，都能触发当前图形的更新。并且，当选中图形改变时不会触发任何图形的更新动作。

> 有个数据流的困惑，以上监听函数里，如果先改变 shapeOptions，然后改变 shapeId，则执行流程是怎样的？Vue 有时候就是这样让人困惑，很难直观地确定数据流，只有一种听天由命的感觉。

