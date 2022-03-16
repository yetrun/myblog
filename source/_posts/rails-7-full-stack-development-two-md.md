---
title: Rails 7 全栈开发：基于模型验证的的方式
date: 2022-03-16 13:57:51
tags:
  - Rails
  - 全栈开发
---

主流的开放方式分为两种：以前端为主的前后端分离模式和以后端为主的全栈开发模式。Rails（全称 Ruby on Rails）是一个全栈开发框架，自然要用 Ruby 语言做更多的事。我在应用 Rails 实践的过程中，发现它的开发模式并不是有多种，只有一种。这一种，就是“基于模型验证的方式”。

<!-- more -->

## Rails 的表单提交案例

*以一个基本案例入手，解释何谓“基于模型验证的方式”。*

以创建和更新用户表单为例：

```erb
<%= form_with @user do |form| %>
  <div>
    <%= form.label :name %>
    <%= form.text_field :name %>
    <% unless @user.errors[:name].empty? %>
      <p><%= @user.errors[:name].join(', ')</p>
    <% end %>
  </div>

  <div>
    <%= form.label :age %>
    <%= form.text_field :age %>
    <% unless @user.errors[:age].empty? %>
      <p><%= @user.errors[:age].join(', ')</p>
    <% end %>
  </div>

  <%= form.submit %>
<% end %>
```

这是一个集创建、更新、表单错误显示等多重要素于一体的代码。对于 Rails 开发者来说，这种模式是自然的、基本的。

用这种方式写代码有多种好处：

1. 不用写任何 JavaScript 代码；
2. 考虑到后端验证是必不可少的，所以即使保留前端验证也不可忽略后端验证。

以上两种从必要性和充分性两点考虑了这种写法的必要性和唯一性。这对 Rails 开发来说是自然的。

考虑到表单验证使用的频繁性，添加一个帮助方法，它可以将表单代码简化为如下形式：

```ruby
<%= form_with @user do |form| %>
  <div>
    <%= form.label :name %>
    <%= form.text_field :name %>
    <%= report_field_error(@user, :name) %>
  </div>

  <div>
    <%= form.label :age %>
    <%= form.text_field :age %>
    <%= report_field_error(@user, :age) %>
  </div>

  <%= form.submit %>
<% end %>
```

## 基于模型验证的方式

这里说出了一个新的概念，将一切接口应用为模型验证的方式。为什么要这么做呢？说出两个优点：

1. 既然基于模型验证的方式是优越的，那么一切接口借用这种方式实现，就可获得表单验证带来的便利；
2. 我们可以将应用逻辑放在模型中，这样接口内的逻辑比较单一，便于我们的测试分离。

OK. 既然如此，那么我必须要解决应用这种方式的难点，毕竟很多同学可能有点疑惑。这里必须要强调实现和理念上的两点：

### Restful 框架

我们知道 Rails 是一个 Restful 框架。Restful 简单说下：将一切对象看成资源，针对资源的操作仅包括创建、删除、更新、查阅，分别对应 GET、POST、PUT、PATCH、DELETE 方法（汉字和英文顺序不是一一对应的）。所以，我们要做的第一条，是将一切对象看成资源。

###  将对象映射为模型

Rails 可以天然地将对象映射为模型，并对模型应用验证方法。详见 ActiveModel.

## 案例一：重置密码

如今要做一个重置密码的功能，它的表单形式大致是（页面原型靠自己想象）：

```erb
<%= form_with do |form| %>
  <div>
    <%= form.label :email %>
    <%= form.text_field :email %>
  </div>
  <div>
    <%= form.label :new_password %>
    <%= form.text_field :new_password %>
  </div>
  <div>
    <%= form.label :verification_code %>
    <%= form.text_field :verification_code %>
  </div>

  <%= form.submit %>
<% end %>
```

这个时候，我们要将“重置密码”这个动作视为一个资源，将表单资源映射为 `GET /reset_passwords/new`，将提交动作映射为 `POST /reset_passwords`：

```ruby
# config/routes.rb
resources :rest_passwords
```

如此，上面的表单我们可以改写成：

```erb
<%= form_with @reset_password do |form| %>
  <div>
    <%= form.label :email %>
    <%= form.text_field :email %>
    <%= report_field_error(@reset_password, :email) %>
  </div>
  <div>
    <%= form.label :new_password %>
    <%= form.text_field :new_password %>
    <%= report_field_error(@reset_password, :new_password) %>
  </div>
  <div>
    <%= form.label :verification_code %>
    <%= form.text_field :verification_code %>
    <%= report_field_error(@reset_password, :verification_code) %>
  </div>

  <%= form.submit %>
<% end %>
```

注意是添加了数据验证和 `@reset_password` 变量。

这时候，对应的 Controller 只要写成：

```ruby
class ResetPasswordsController < ApplicationController
  def new
  end
    
  def create
    @reset_password = ResetPassword.new(reset_password_params)
    if @reset_password.reset_password
      redirect_to logins_path
    else
      render :new, status: :unprocessable_entity
    end
  end
end
```

而最最核心的 ResetPassword 类，大致如下：

```ruby
class ResetPassword
  include ActiveModel::API
  
  attr_accessor :email, :new_password, :verification_code
    
  validates_presence_of :email, :new_password, :verification_code
  
  def reset_password
    return false unless valid?
    
    user = User.find_by(email: email)
    unless user
      errors.add :email, :not_exist, message: '邮箱不存在'
      return false
    end
      
    unless user.verification_code == verification_code
      errors.add :verification_code, :not_matched, message: '验证码不匹配'
      return false
    end
      
    user.update(verification_code: true)
    return true
  end
end
```

## 案例二：发送验证码

现在基于以上案例添加一个发送验证码的功能。按照 Rails 只有一种实现方式的原则，我们首先为发送验证码添加一个表单：

```erb
<%= form_with @reset_password do |form| %>
  <!-- email 和 new_password 的表单控件略 -->
  <div>
    <%= form.label :verification_code %>
    <div>
      <%= form.text_field :verification_code %>
      <%= form_with @send_verification_code do |form| %>
        <%= form.text_field :email, hidden: true %>
        <%= form.submit %>
      <%= end %>
    </div>
    <%= report_field_error(@reset_password, :verification_code) %>
  </div>

  <%= form.submit %>
<% end %>
```

说明：我在表单内嵌套一个表单。由于放松验证码时要传递 `email` 参数，所以要添加一个隐藏的 `email` 控件。可通过 Stimulus 让其与外层的 `email` 控件保持一致。（这段代码省略）

但实践中这种方法是**行不通**的，因为根据 HTML5 的标准，表单内不准内嵌表单。（这操蛋的标准）

思来想去还是决定通过 JavaScript 向后端发送请求，请求返回一个 Turbo Stream 模板，这样可以复用利用 Turbo Stream 更新视图的逻辑（不用自己在写代码了）。

怎么做到呢？收到响应后，关键在于调用 `Turbo.renderStreamMessage(html)`.
