---
title: 前端框架和技术实践汇总
date: 2021-12-13 10:05:21
tags:
- 前端
- Vue
---

这段时间在拿一个小项目练手，力图寻找出一个适合自己的前端框架合集。我们知道 Vue 框架为我们实现了数据绑定和单页应用，让我们能够基于数据状态而不是 DOM 状态开发前端项目。但我们仍然知道，Vue 只是实现了视图层的逻辑而已，围绕 Vue 框架我们依然需要为前端项目的其他层次选择合适的技术方案。

就我个人观念而言，前端项目的以下技术方案至关重要：

- 视图层的数据绑定，这个可以由 Vue 框架帮我们做到
- 表单验证
- 与后端 API 的交互

这篇文章将介绍后两者的方案。

<!-- more -->

## 与后端 API 的交互

在与后端 API 的交互过程中，我选择了 [js-data](https://www.js-data.io/) 作为我的实现载体。`js-data` 是一个 JavaScript 框架，起初为 Angular 框架开发，现在能够适配任意框架。当然，在应用于 Vue 的时候还有小小的不适应性，待会我会提到。

除了可以调用后端提供的 API 之外，`js-data` 提供了模型层几乎需要的东西，包括：

- 模式定义
- 关系映射
- 数据缓存
- 多数据源的支持

### 模式定义

模式定义为模型定义模式，这样约束了模型的字段，同时还可以为其做数据验证。例如我们定义了以下的模式：

```javascript
const userSchema = new Schema({
  type: 'object',
  properties: {
    id: { type: 'number' },
    name: { type: 'string' },
    age: { type: 'number' }
  }
})
```

我们只能为 `user` 对象提供 `id`、`name`、`age` 参数，并且如果它们的类型不一致也会报错：

```javascript
new User({ 
  name: 'James Dean', 
  age: 18, 
  firstName: 'James', lastName: 'Dean' // firstName 和 LastName 将忽略
})

new User({
  name: 'James Dean',
  age: '18' // 报错，字符串类型无法应用到数字类型
})
```

### 关系映射

关系映射是四个东西里我最不在意的，因为它带来的便利仅仅是编码更简洁了（也许是我没有理解透彻，望看官指正）。首先我们可以如下定义两个模型之间的关系：

```javascript
store.defineMapper('user', {
  relations: {
    hasMany: {
      post: {
        foreignKey: 'userId',
        localField: 'movies'
      }
    }
  }
})

store.defineMapper('movie', {
  relations: {
    belongsTo: {
      user: {
        foreignKey: 'userId',
        localField: 'user'
      }
    }
  }
})
```

便利性的体现有很多，这里我仅举两个较容易理解的场景：

**场景一：同时创建**

```javascript
await store.create('user', {
  name: 'James Dean',
  age: 24,
  movies: [
    {
      title: 'East of Eden',
      year: '1955'
    },
    {
      title: 'Rebel Without a Cause',
      year: '1955'
    },
    {
      title: 'Giant',
      year: '1956'
    }
  ]
})
```

**场景二：同时返回**

```javascript
await store.find('user', 1, { with: 'movies' })
```

将返回：

```javascript
{
  id: 1,
  name: 'James Dean',
  age: 24,
  movies: [
    {
      id: 11,
      userId: 1,
      title: 'East of Eden',
      year: '1955'
    },
    {
      id: 12,
      userId: 1,
      title: 'Rebel Without a Cause',
      year: '1955'
    },
    {
      id: 13,
      userId: 1,
      title: 'Giant',
      year: '1956'
    }
  ]
}
```

我说过四个东西里最不在意的就是“关系映射”，但这里却用了最大的篇幅介绍这货，这说明关系映射确实是这里面最复杂的（而不是我对它有什么偏赖）。不在意是因为即使没有定义关系，我们仍然可以用别的方式实现（显而易见），只不过是样板代码稍微多了点。

```javascript
// 针对场景一
const user = await store.create('user', { ... })
for (const post of posts) {
  post.userId = user.id
}
await store.createMany('post', posts)

// 针对场景二
const user = await store.find('user', 1)
const posts = await store.findAll('post', { userId: 1 })
```

说实话，在开发前端项目时，以上写法也许更常见。

### 数据缓存

当我们通过调用后端 API（更一般地，从数据源）取得数据时，再度请求同样的数据会发生什么？例如在列表页里我们已经取得 `posts` 列表数据，如今点击其中一项进入详情页，此时详情所需的 `post` 数据已经在 `posts` 列表内了，我们有必要再请求一遍吗？当然不必要了，可现实是我们往往又重新请求了一遍，完全没有用到缓存机制。

`vuex` 尝试解决这个问题。但说实话，这种全局状态绑定的模式我是真心喜欢不起来。

`js-data` 通过缓存机制无感地为我们解决了这个问题。首先，我们在列表页已经请求过一遍数据：

```javascript
await findAll('post')
```

然后点击其中一项进入详情页，请求 `post` 详情数据依然用同样的调用方式：

```javascript
await find('post', 1)
```

代码没有任何变化，内在逻辑的改变 `js-data` 悄悄地为我们做了。如果我们是从列表页进入详情页的，会从缓存中拿到数据而不会再向后端发送请求；如果我们是直接进入详情页的，会向后端发送请求。

### 多数据源的支持

数据存储在哪里？数据从哪里获取？对于前端项目来说，大多数不是一个问题：数据当然是通过调用后端 API 获取的。`js-data` 同时提供多数据源的支持，利用适配器模式，几乎不用修改任何代码就可以将数据源从后端服务器切换为 IndexedDB. 你可以不用，但有时候也许用得上。

## 表单验证

虽然 `js-data` 为我们提供了数据验证的功能，但数据验证不同于表单验证。表单验证面向的是终端用户，为用户提供即时的反馈。

基于 Vue 的控件多数提供了表单验证的功能，例如 `Element UI`、`iView` 等，如果你用的是这类 UI 组件，直接使用它们的表单验证机制即可。

有时候表单控件没有提供验证的能力，这时我们可以使用 [async-validator](https://github.com/yiminghe/async-validator) 包配置我们的验证逻辑。

## `js-data` 的槽点和坑

我们有了 `js-data`，表示我们有了模型层的封装。我们有了表单验证，为我们解决了一个开发上的大难题。剩下的就交给 Vue 框架和 Vue 生态（例如 `vue-router`）。有了这三大利器，我想不到还缺少什么（我指的是几乎所有项目都需要的那种东西）。

`js-data` 占了很大的一个比重，但它不是完美的。就我的实践发现，它主要有两个坑：

### 面向的不是普通的 JavaScript 对象

`store.findAll` 返回不是对象数组，而是 `Record` 实例数组。

```javascript
const users = await store.findAll('user')
users[0] instanceof Record // true
```

这有什么关系呢？用在 Vue 框架就大大的有关系了，因为 Vue 框架有个明确的建议：只能用于普通的 JavaScript 对象。如果绑定的不是一个纯粹的 JavaScript 对象，而是一个类的实例，有可能遇到潜在的问题。不幸的是，`js-data` 正好中枪了。

解决的办法是：将 Record 转化为普通的对象。Record 提供了 `toJSON` 方法，它将会转化为普通的对象：

```javascript
let users = await store.findAll('user')
users = users.map(user => user.toJSON())

let user = await store.find('user', 1)
user = user.toJSON()
```

Record 会提供额外的便利的方法，有时候我们会用到它。况且每次请求时调用 `toJSON` 方法转一道不觉得繁琐吗？所以我在实践中会采用折衷的办法，有时候使用 Record，有时候使用普通的对象。

那么何时使用 Record，何时又要转化为普通的对象呢？其窍门在于，我们何时用何种方式触发 Vue 的响应式。因此，我得出的结论是：在仅展示的页面使用 Record，在数据会被修改的地方使用普通的 JavaScript 对象。

列表页是一个仅展示的页面，它在初始化时触发响应式：

```javascript
async mounted () {
  await fetchAll()
},
methods: {
  async fetchAll () {
    const users = await store.findAll('user')
    
    // ...也许你想要为 users 做些别的事
    
    // 直接赋值给 `userRecords` 属性。由于 `userRecords` 属性被赋予了另一个值，所以总能触发响应式。
    // 带上 `Records` 后缀是我的一个命名习惯，提醒我这里是一个 `js-data` 的 Record 对象。
    this.userRecords = users 
  }
}
```

而在编辑页模型的内部状态会被修改，不适用于 Record：

```vue
<template>
  <form>
    <!-- 这里会触发 user 内部状态改变，如果 Record 实例，则无法触发响应式 -->
    <input v-model="user.name" />
    <input v-model="user.age" />
  </form>
</template>

<script>
export default {
  data () {
    return {
      user: {}
    }
  },
  async mounted () {
    await fetchUser()
  },
  methods: {
    async fetchUser () {
      const userRecord = await store.find('user', 1)
      this.user = userRecord.toJSON()
    }
  }
}
</script>
```

### 它最佳的适用模式是数据源拥有一套标准的 CURD 方法

针对数据源的操作无非是 Create、Update、Retrieve、Delete 四种模式。无论是关系型数据库、文档型数据库、Redis、ElasticSearch、localStorage、WebSQL、IndexedDB 等，它们都有一套标准的 CURD 方法，为它们编写适配器是一种统一的工作。这也是 `js-data` 也能应用于后端的原因。

而前端最常面对的是后端提供的 API，它没有统一的 CURD 实现标准。试想一下，后端 API 的风格千奇百怪，尤其是中国这个特殊的大环境下。所以，不同的项目有不同的实现标准，甚至同一个项目的不同模块都有不同的实现标准，这是我们在应用 `js-data` 最大的难题。

另一个方面，由于后端提供的 API 是面向业务逻辑的，它提供的接口未闭会完全参考 CURD 的模式。例如，有一个审核的接口，它可能定义为 `/posts/{id}/verify`，可以传递一个 `value` 参数为 `true` 或 `false`，以使得这篇博文审核被通过或被拒绝：

```bash
curl -XPOST /posts/1/verify -d '{ value: true }' # 审核通过
curl -XPOST /posts/1/verify -d '{ value: false }' # 审核不通过
```

它的主要作用是修改 `post` 资源的 `verified` 字段，也许会有其他附加的动作如添加审核人和审核时间等。无论如何，它已经被设计为 Restful 资源下的一个动作了，关于这一点，`js-data` 的扩展能力及其之低。

我在前面说过，我做了一个小项目实验 `js-data` 的特性，由于都是标准的 CURD 方法，并且数据源是本地的 IndexedDB，所以也就没有遇到这个潜在的问题了。但我隐隐觉得这可能是应用 `js-data` 最力不从心的地方，后续实践进一步验证。

## 总结

总结一下前端项目如何实践：

- 适用 Vue 作为视图层数据状态的绑定和单页逻辑。
- 添加表单验证，它们或是 Vue 组件自带的功能，或用 `async-validator` 库自行实现一套。
- 适用 `js-data` 作为与数据源的交互，以及定制模型层。

我们还提到 `js-data` 并不是完美的，但是它提供的几个特性如模式定义、数据缓存是我们在开发中必不可少的。并没有任何东西是完美的，Vue 也有很多不完美的地方，但开发的任务却是不容等待。在我们的认知局限下选择最适合我们的方式，才是当务之机。
