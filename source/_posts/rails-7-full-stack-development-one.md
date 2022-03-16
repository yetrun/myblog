---
title: Rails 7 全栈开发：全页使用 Turbo Frame
date: 2022-03-04 15:44:00
tags:
  - Rails
  - 全栈开发
---

## 需求背景

如果说因为某种原因，需要在整个页面地方使用 Turbo Frame，采用这种结构有哪些需要注意的地方呢？本篇文章就是讨论这种情况的。

<!-- more -->

我给出一个基本布局，这在 Rails 中往往名为 `layout/application.html.erb`：

```erb
<body>
  <div id="container">
    <!-- 这里是页面的可显示内容 -->
    <div id="header">...</div>
    <div id="main">
      <%= yield %>
    </div>
    <div id="footer">...</div>
  </div>

  <div id="modal">
    <!-- 这是一个模态框，用于替代 window.alert -->
  </div>

  <div id="overlay">
    <!-- 一个全局使用的遮罩层，它每隔 15 分钟弹出，需要用户点击确认才消失 -->
  </div>
</body>
```

一个基本的想法是，当在页面内部请求新的页面时，只要刷新 `#container` 部分的内容。这时，Turbo Frame 技术就用得上了。如果不考虑 `#modal` 和 `#overlay` 的刷新，只需要将 `#container` 用 Turbo Frame 包裹起来就可以了（`layout/application.html.erb` 的对应部分修改为）：

```html
<body>
  <turbo-frame id="pageContent">
    <div id="container">
      ...
    </div>
  </turbo-frame>
</body>
```

这样 `<turbo-frame>`中的链接默认只会刷新 `<turbo-frame>` 内部的内容。当然，这要求所有可视页面的模板需要包含 `<turbo-frame id="pageContent">` 部分。这里给个示例（`articles/show.html.erb`）：

```erb
<turbo-frame id="pageContent">
  <%= render @article %>
</turbo-frame>
```

## 遇坑一二三

我先来说第一个注意事项，其实也是一个坑。注意到 `application` 布局的 `<%= yield %>` 部分，说明即使是 `#pageContent` 内部的 Turbo Frame，仍然是有布局的。所以，以上的 `articles/show.html.erb` 实际上应该写成这样：

```erb
<turbo-frame id="pageContent">
  <div id="container">
    <!-- 这里是页面的可显示内容 -->
    <div id="header">...</div>
    <div id="main">
      <%= render @article %>
    </div>
    <div id="footer">...</div>
  </div>
</turbo-frame>
```

如果每个页面都这样写就好冗长哦，DRY 哪里去了？第一想法是利用默认的 `application` 布局，在页面里去掉 `<turbo-frame>`。这样 `articles/show.html.erb` 只用写成这样：

```erb
<%= render @article %>
```

我们希望是能够用上 `application` 布局的，但事与愿违，最后页面仅渲染了 `articles/show.html.erb` 的内容，似乎完全没用上 `application` 布局。

我的初步猜想是布局内的 Turbo Frame 必须在顶层（这是 Rails 的处理机制，我也不知道是为什么），所以为视图 `articles/show.html.erb` 另写了一个布局文件（`layout/page_content.html.erb`）：

```erb
<turbo-frame id="pageContent">
  <div id="container">
    <!-- 这里是页面的可显示内容 -->
    <div id="header">...</div>
    <div id="main">
      <%= yield %>
    </div>
    <div id="footer">...</div>
  </div>
</turbo-frame>
```

然后 `GET /articles/:id` 请求使用布局 `page_content`. 

### 基于是否是 Turbo Frame 请求使用动态布局

但这样引来一个问题，当我在浏览器中直接输入 `/articles/:id` 这样的链接时，它只会加载 `page_content` 布局的内容。而这样的首次加载需要使用 `application` 布局才是。所以，我使用了动态布局加以解决：

```ruby
class ApplicationController
  layout -> { turbo_frame_request? ? 'page_content' : 'application' }
end
```

可以看到，如果是 Turbo Frame 请求，使用 `page_content` 布局，否则使用 `application` 布局。完美解决！

### Turbo Frame 中支持更新地址栏以及历史跳转记录

还有另一个问题，就是 `#pageContent` 内点击链接只是一个表单请求，不再更新地址栏了。我们只是用 Turbo Frame 结构解决我们的一个页面布局问题，而并不是把它当作一个局部区域。（我们能够看到，完整的页面显示都在 `<turbo-frame id="pageContent">` 中）所以，地址栏不更新不符合我们的要求。

解决这个问题只用一招就搞定了，添加一个属性 `turbo-action="advance"` 即可：

```html
<turbo-frame id="pageContent" data-turbo-action="advance">
```

### Turbo Frame 内部不再监听到 `turbo:before-visit` 事件

如果我们想在 Turbo Frame 内部监听点击链接，监听 `turbo:before-visit` 事件办不到了。Turbo Frame 的链接点击和表单提交都放到了 `turbo:before-fetch-request` 事件了。所以，应当监听的事件是 `turbo:before-fetch-request`：

```javascript
addEventListener('turbo:before-fetch-request', listener, false)
```
