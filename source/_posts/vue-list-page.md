---
title: Vue 数据列表页的写法参考
date: 2021-11-02 11:52:54
tags:
- vue
---

常规的管理系统开发（以及部分用户端原型的开发），涉及到列表页、详情页、编辑页、新建页。据我多年的开发经验获悉，每一种类型的页面，开发者都应有一套固定的写法。这一篇讲列表页的固定写法。

列表页，如果考虑功能齐全，应包括：

- 展示一个数据列表
- 支持若干筛选项目
- 支持分页浏览
- 支持全文搜索
- 支持排序

本文分两个阶段介绍，第二阶段相比第一阶段增加了 vue-router 的考虑。

<!-- more -->

## 第一阶段

### 设计数据格式

用 Vue 开发组件，应首先着眼于数据。对于满足以上功能的列表页来说，可设计如下的数据结构：

```javascript
data () {
  return {
    // 展示给列表的数据
    users: [],
    // 分页相关
    pagination: {
      current: 1,
      pageSize: 10,
      total: 0
    },
    // 筛选相关
    filters: {
      // 根据名字筛选
      name: '',
      // 根据手机号筛选
      mobile: '',
      // 其他的不再一一展示
    }
  }
}
```

由于我不喜欢为自己的数据取一些通用的名字，诸如 `data`、`records` 之类，所以这里我列出了一个业务相关的数据样例。这是一个展示用户列表的数据结构，也是我最喜欢用的一个示例。

### 设计 UI 布局

接下来应设计 UI 布局。用户列表类的数据比较适合用表格展示，这里我使用 Element UI 的 Table 组件展示用户列表数据，其他人可根据实际需要自行选择其他组件。

```vue
<template>
  <div>
    <!-- 筛选项目表单 -->
    <el-form :inline="true">
      <el-form-item label="姓名">
        <el-input v-model="filters.name" placeholder="输入姓名筛选"></el-input>
      </el-form-item>
      <el-form-item label="手机号">
        <el-input v-model="filters.mobile" placeholder="输入手机号筛选"></el-input>
      </el-form-item>
    </el-form>

    <!-- 展示用户数据列表 -->
    <el-table :data="users" style="width: 100%">
      <el-table-column
        prop="name"
        label="姓名"
        width="180">
      </el-table-column>
      <el-table-column
        prop="mobile"
        label="手机号"
        width="180">
      </el-table-column>
      <el-table-column
        prop="address"
        label="地址">
      </el-table-column>
      <el-table-column
        prop="date"
        label="登记日期"
        width="180">
      </el-table-column>
    </el-table>

    <!-- 控制分页的组件 -->
    <el-pagination
      layout="prev, pager, next"
      :currentPage="pagination.current"
      :pageSize="pagination.pageSize"
      :total="pagination.total"
      @current-change="pagination.current = $event"
    ></el-pagination>
  </div>
</template>
```

这里有如下几个注意要点：

- 表单的筛选项是直接和 filters 内的属性双向绑定的。注意这里并没有一个查询按钮，假设需求是边输入边执行筛选的实时效果。
- 数据列表直接使用 Element UI 的 Table 组件，注意其中绑定的数据是 `users`.
- 注意分页组件的数据绑定。

### 组件加载时初始化数据

数据一开始是个空数组，这样什么也不会展示。需要在组件初始化时加载数据，一般是从远端 API 请求数据。本文使用模拟的 `fetchUsers` 函数表示从远端 API 获取数据的方法，它接收一个可选的查询参数：

```javascript
function fetchUsers (params = {})
```

注意，组件初始化要用到 `mounted` 钩子：

```javascript
mounted () {
  const response = await fetchUsers()

  this.users = response.data.users
  this.pagination.total = response.data.total
}
```

### 响应筛选和分页变化

响应筛选和分页的逻辑，我们还是要从数据的视角出发。这里的做法是，监听筛选和分页数据的变化，重新获取数据：

```javascript
watch: {
  filters: {
    handler () {
      this.fetchUsers()
    },
    deep: true
  },
  'pagination.current' () {
    this.fetchUsers()
  }
}
```

注意这里的 `deep: true`，一旦 `filters.name`、`filters.mobile` 和 `pagination.current` 的值发生变化，都会重新执行一遍 `fetchUsers` 动作。需要补充 `fetchUsers` 方法：

```javascript
methods: {
  async fetchUsers () {
    // 构造筛选和分页参数
    const filterParams = Object.entries(this.filters)
      .reduce((finalParams, [key, value]) => {
        if (value) {
          finalParams[key] = value
        }
        return finalParams
      }, {})
    const paginationParams = {
      page: this.pagination.current,
      perPage: this.pagination.pageSize
    }

    // 执行远端请求
    const data = await fetchUsers({
      ...filterParams,
      ...paginationParams
    })

    // 修改当前数据
    this.users = data.users
    this.pagination.total = data.total
  }
}
```

当然，还需提到一点，以上 `mounted` 的实现方式欠妥，应该修正为：

```javascript
mounted () {
  this.fetchUsers()
}
```

### 第一阶段总结

至此，一个拥有基本流程的列表页已经全部完成了，它拥有：

- 展示一个数据列表
- 支持筛选
- 支持分页浏

有关搜索和排序的实现，它们的实现逻辑与筛选、分页一致，也是从数据出发，再监听数据的变化。所以这里不再重复实现了，读者可以自行完成。

总结一下，实现这样的一个页面，它的要点是：

1. 首先定义数据结构，我们在这里定义了三个数据：`filters`、`pagination`、`users`.
2. 定义数据之间的关系，这里的关系是，数据 `users` 受到 `filters` 和 `pagination` 的支配。

关于以上第 2 点，可以进一步说明。因为数据 `users` 受到 `filters` 和 `pagination` 的支配，用户一旦想要重新渲染 `users` 数据，只需要调整 `filters` 和 `pagination` 数据即可。这里采用 watch 机制实现，事实上也可以采用一个 UI 交互，例如点击一个按钮，加以实现。

## 第二阶段

第二阶段增加了 vue-router 的考虑。在真正实现之前，先考虑这样一个问题：列表页与路由有什么关系呢？实现列表页时是可以不考虑路由的，但加上路由会带来两个好处：

1. 当需要分享特定列表给其他人时，路由中带上筛选、分页等参数可以让对方直接进入目标页；
2. 当执行特定的筛选、分页等操作后，及时调整路由参数可以从详情页返回时回到当前列表页的状态。

我们先前定义了 `filters` 和 `pagination` 数据的默认值，它们是这样的：

```javascript
pagination: {
  current: 1,
  pageSize: 10,
  total: 0
},
filters: {
  name: '',
  mobile: ''
}
```

带上路由考虑时，重点是维护 `filters`、 `pagination` 和路由参数的一致性。这要考虑两个方面：

1. 在页面初始化时从路由参数中加载 `filters`、`pagination` 的默认值；
2. 在 `filters`、`pagination` 改变时修改地址栏中显示的路由参数。

### 维护路由参数和组件内数据的状体一致性

好了，既然目标清洗了，着手去做就是了。首先要调整 `mounted` 的代码，它要做两件事：

```javascript
mounted () {
  this.loadRoute()
  this.fetchUsers()
}
```

然后看如何加载路由参数：

```javascript
methods: {
  loadRoute () {
    // 因为我待会要修改路由参数对象，所以先拷贝一个副本
    const routeParams = Object.assign({}, this.$route.query)

    // 路由中名为 `page` 的参数代表当前页
    this.pagination.current = routeParams.page

    // 剩下的参数全部给 filters
    delete routeParams.page
    this.filters = routeParams
  }
}
```

这样第一步便完成了，第二步即要修改 `filters`、`pagination` 改变后的代码：

```javascript
watch: {
  filters: {
    async handler () {
      await this.fetchUsers()
      this.refreshRoute()
    },
    deep: true
  },
  async 'pagination.current' () {
    await this.fetchUsers()
    this.refreshRoute()
  }
}
```

`refreshRoute` 即把 `loadRoute` 的逻辑反过来：

```javascript
methods: {
  refreshRoute () {
    const filters = Object.entries(this.filters)
      .reduce((finalFilters, [key, value]) => {
        if (value) {
          finalFilters[key] = value
        }
        return finalFilters
      }, {})

    this.$router.replace({
      query: {
        page: this.pagination.current,
        ...filters
      }
    })
  }
}
```

等等，这样就结束了吗？

### 调整监听的时机

如果这样实现之后，打开控制台，你会发现一个控制台会抛出一个 `NavigationDuplicated` 错误。仔细观察，你会发现，在 `mounted` 逻辑内会设置 `filters` 和 `pagination.current` 的值，从而触发监听逻辑，最终导致 `refreshRoute` 的逻辑执行。而此时执行 `refreshRoute` 就会加载一份同地址栏一模一样的路由，从而引起 vue-router 的警告。

除此之外，此时响应监听让我心生一种不安的感觉，除了 `refreshRoute`，它极有可能再一次执行 `fetchUsers`，从而导致重复加载。我没有实际去验证是否发生了这种情况，但既然不安，就是隐患。

问题的症结在于监听的时机不对，页面应该在加载完初始数据之后，待它的状态稳定下来才开始监听。所以，这里我摒弃了 `watch` 选项，改用 `$watch` API：

```javascript
async mounted () {
  this.loadRoute()
  await this.fetchUsers()

  this.$watch('filters', {
    handler: this.respondFetchParamsChange,
    deep: true
  })
  this.$watch('pagination.current', this.respondFetchParamsChange)
}
```

而 `respondFetchParamsChange` 即是原来监听下的处理函数：

```javascript
methods: {
  async respondFetchParamsChange () {
    await this.fetchUsers()
    this.refreshRoute()
  }
}
```

至此，才算是大功告成。

## 完整代码

```vue
<template>
  <div>
    <!-- 筛选项目表单 -->
    <el-form :inline="true">
      <el-form-item label="姓名">
        <el-input v-model="filters.name" placeholder="输入姓名筛选"></el-input>
      </el-form-item>
      <el-form-item label="手机号">
        <el-input v-model="filters.mobile" placeholder="输入手机号筛选"></el-input>
      </el-form-item>
    </el-form>

    <!-- 展示用户数据列表 -->
    <el-table :data="users" style="width: 100%">
      <el-table-column
        prop="name"
        label="姓名"
        width="180">
      </el-table-column>
      <el-table-column
        prop="mobile"
        label="手机号"
        width="180">
      </el-table-column>
      <el-table-column
        prop="address"
        label="地址">
      </el-table-column>
      <el-table-column
        prop="date"
        label="登记日期"
        width="180">
      </el-table-column>
    </el-table>

    <!-- 控制分页的组件 -->
    <el-pagination
      layout="prev, pager, next"
      :current-page="pagination.current"
      :page-size="pagination.pageSize"
      :total="pagination.total"
      @current-change="pagination.current = $event"
    ></el-pagination>
  </div>
</template>

<script>
import fetchUsers from './fetchUsers'

export default {
  data () {
    return {
      // 展示给列表的数据
      users: [],
      // 分页相关
      pagination: {
        current: 1,
        pageSize: 10,
        total: 0
      },
      // 筛选相关
      filters: {
        name: '',
        mobile: ''
      }
    }
  },
  async mounted () {
    this.loadRoute()
    await this.fetchUsers()

    this.$watch('filters', {
      handler: this.respondFetchParamsChange,
      deep: true
    })
    this.$watch('pagination.current', this.respondFetchParamsChange)
  },
  methods: {
    // 根据组件当前的筛选、分页数据更新用户数据
    async fetchUsers () {
      // 构造筛选和分页参数
      const filterParams = Object.entries(this.filters)
        .reduce((finalParams, [key, value]) => {
          if (value) {
            finalParams[key] = value
          }
          return finalParams
        }, {})
      const paginationParams = {
        page: this.pagination.current,
        perPage: this.pagination.pageSize
      }
  
      // 执行远端请求
      const data = await fetchUsers({
        ...filterParams,
        ...paginationParams
      })
  
      // 修改当前数据
      this.users = data.users
      this.pagination.total = data.total
    },
    // 页面首次加载时调用，从路由参数初始化组件的筛选、分页数据
    loadRoute () {
      // 因为我待会要修改路由参数对象，所以先拷贝一个副本
      const routeParams = Object.assign({}, this.$route.query)

      // 路由中名为 `page` 的参数代表当前页
      this.pagination.current = routeParams.page ? parseInt(routeParams.page) : 1

      // 剩下的参数全部给 filters
      delete routeParams.page
      this.filters = routeParams
    },
    // 由组件当前的筛选、分页数据重置路由（即地址栏的显示）
    refreshRoute () {
      const filters = Object.entries(this.filters)
        .reduce((finalFilters, [key, value]) => {
          if (value) {
            finalFilters[key] = value
          }
          return finalFilters
        }, {})

      this.$router.replace({
        query: {
          page: this.pagination.current,
          ...filters
        }
      })
    },
    // 监听函数，响应筛选、分页数据的变化
    async respondFetchParamsChange () {
      await this.fetchUsers()
      this.refreshRoute()
    }
  }
}
</script>
```
