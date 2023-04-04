---
title: 为 Rails 应用添加参数验证、返回值渲染和生成 Swagger 文档的能力
date: 2023-04-04 13:22:18
tags:
---

现实中为 Rails 添加参数验证的 Gems 很多，添加 Swagger 文档生成的也不少，但能够同时支持参数验证、返回值渲染和 Swagger 文档生成的少之又少，而且将这些能力紧密结合在一起的就几乎没有了。

大家伙在团队开发中，是否会有过这些困惑：

- 缺少一个比较详细的 API 文档，有些甚至没有文档。
- 文档和实现往往不一致，明明文档中说有这个字段，结果在实际调用时发现没有这个字段，或者反之。
- 文档很难实现规约，比如参数集的字段与返回值的字段需要区分开来，以及不同场景下返回值的字段也应有所区分。所以，现实中往往只能抛出一个大而全的字段表放在 API 文档中。

<!-- more -->

如果有，你就真应该看看我的这个 gem：[meta-api](https://github.com/yetrun/web-frame)，它天生就是为了解决这些问题的。我们把 API 文档看成是前后端开发者都需要遵守的契约，不仅是前端调用错误需要报错，后端使用错误也需要报错。

## 初窥一览

*温馨提示：*访问仓库 [web-frame-rails-example](https://github.com/yetrun/web-frame-rails-example) 可查看示例项目。你可以现在就去体验，或者任何时候回过头来体验均可。

### 安装

要在 Rails 中使用 `meta-api`，请遵循以下步骤。

**第一步，**在 Gemfile 中添加：

```ruby
gem 'meta-api'
```

然后执行 `bundle install`.

**第二步，**在 `config/initializers` 目录下创建一个文件，例如 `meta_rails_plugin.rb`，并写入：

```ruby
require 'meta/rails'

Meta::Rails.setup
```

**第三步，**在 `application_controller.rb` 下添加：

```ruby
class ApplicationController < ActionController::API
  # 引入宏命令
  include Meta::Rails::Plugin

  # 处理参数验证错误，以下仅是个示例，请根据实际需要编写
  rescue_from Meta::Errors::ParameterInvalid do |e|
    render json: e.errors, status: :bad_request
  end
end
```

以上步骤完成后，就可以在 Rails 中编写宏命令定义参数和返回值了。

### 编写示例

下面编写一个 `UsersController`，作为使用宏命令的示例：

```ruby
class UsersController < ApplicationController
  route '/users', :post do
    title '创建用户'
    params do
      param :user, required: true do
        param :name, type: 'string', description: '姓名'
        param :age, type: 'integer', description: '年龄'
      end
    end
    status 201 do
      expose :user do
        expose :id, type: 'integer', description: 'ID'
        expose :name, type: 'string', description: '姓名'
        expose :age, type: 'integer', description: '年龄'
      end
    end
  end
  def create
    user = User.create(params_on_schema[:user])
    render json_on_schema: { 'user' => user }, status: 201
  end
end
```

以上首先使用了 `route` 命令定义一个 `POST /users` 的路由，并提供了一个代码块，所有的参数定义和返回值定义都在这个代码块内。当这一切定义就绪之后，实际执行时会带来两个效果：

1. `params_on_schema`：它返回根据定义解析后的参数，比如你如果传递了这么一个参数：

    ```json
    {
      "user": {
        "name": "Jim",
        "age": "18",
        "foo": "foo"
      }
    }
    ```

    它会帮你过滤掉不必要的字段，并且做适度地类型转换，从而让应用内真正获取的参数变成（通过 `params_on_schema`）：

    ```json
    {
      "user": {
        "name": "Jim",
        "age": 18
      }
    }
    ```

    这样你就可以放心无虞地将其传递给 `User.create!` 方法，不用担心任何错误。

2. `json_on_schema`：通过它传递的对象会经由返回值定义解析，因为有字段的过滤、类型的转换和数据的验证，后端可以控制哪些字段返回给前端、如何返回以及在字段出错时得到提醒等。比如你的 users 表包括以下字段：

    ```
    id, name, age, password, created, updated
    ```

    根据定义，它只会返回如下字段：

    ```
    id, name, age
    ```

### 生成 Swagger 文档

*一切皆是以生成 Swagger 文档为核心。*

如果你想要生成文档，请按照我的步骤执行。

第一步，想好你的 Swagger 文档放哪？比如我新建一个接口用于返回 Swagger 文档：

```ruby
class SwaggerController < ApplicationController
  def index
    Rails.application.eager_load! # 需要提前加载所有控制器常量
    doc = Meta::Rails::Plugin.generate_swagger_doc(
      ApplicationController,
      info: {
        title: 'Web API 示例项目',
        version: 'current'
      },
      servers: [
        { url: 'http://localhost:9292', description: 'Web API 示例项目' }
      ]
    )
    render json: doc
  end
end
```

第二步，为它配置一个路由：

```ruby
# config/routes.rb

get '/swagger_doc' => 'swagger#index'
```

第三步，启动应用即可查看到 Swagger 文档的效果，在浏览器内输入：

```
http://localhost:3000/swagger_doc
```

JSON 格式的 Swagger 文档映入眼帘。

> 如果你是使用 Swagger 文档的老人了，应该知道接下来怎么做。如果你不知道，请按照如下步骤渲染 Swagger 文档：
>
> 1. 为应用添加跨域支持：
>
>     ```ruby
>     # Gemfile
>     gem 'rack-cors'
>
>     # config/initializers/cors.rb
>     Rails.application.config.middleware.insert_before 0, Rack::Cors do
>       allow do
>         origins 'openapi.yet.run'
>         resource "*", headers: :any, methods: :all
>       end
>     end
>     ```
>
> 2. 如果你用的是 Google Chrome 浏览器，需要做一些额外的设置才能支持 localhost 域名的跨域。或者你需要将其挂在一个正常的域名下。
>
> 3. 打开 http://openapi.yet.run/playground，在输入框内输入 `http://localhost:3000/swagger_doc`.

至此，一切初试体验完毕，前端可以获得一份定义明确的文档，后端也是严格按照这个文档执行的。后端开发者不用做额外的工作，它只是定义参数、定义返回值，然后，一切就完备了。

## 用不同的场合响应不同的字段

这一章节提供了一个比较充分的主题，`meta-api` 如何处理不同场合下的字段区分。不同的场合，比如：

1. 某些字段只用作参数，而某些字段只作为返回值。
2. 用作返回值时，某些接口返回字段 `a, b, c`，而某些接口返回字段 `a, b, c, d`.
3. 用作参数时同 2，某些接口用到字段 `a, b, c`，而某些接口用到字段 `a, b, c, d`.
4. 如何处理 HTTP 中对 `PUT` 和 `PATCH` 请求的语义区分。

这里 `meta-api` 插件提供了实体的概念，我们可以将字段放到实体里，然后在接口中引用它。实体只用定义一遍，不需要任何的重复定义，并且最重要的是，文档和你的定义能够始终保持一致。请接下去看！

### 使用实体简化参数和返回值的定义

我们将上例中的参数和返回值归纳为为一个实体：

```ruby
class UserEntity < Meta::Entity
  property :id, type: 'integer', description: 'ID', param: false
  property :name, type: 'string', description: '姓名'
  property :age, type: 'integer', description: '年龄'
end
```

然后参数和返回值的定义都可以引用这个实体：

```ruby
route '/users', :post do
  params do
    param :user, required: true, ref: UserEntity
  end
  status 201 do
    expose :user, ref: UserEntity
  end
end
```

注意到实体定义中，`id` 属性有一个 `param: false` 选项，它表示这个字段只能被用作返回值而不能用作参数。这与之前的定义是一致的。

除了限定字段只能用作返回值外，也可以限定它只能用作参数。在 `UserEntity` 内添加一个 `password` 字段作为参数，可以如下定义：

```ruby
class UserEntity < Meta::Entity
  # 省略其他字段
  property :password, type: 'string', description: '密码', render: false
end
```

**总之，只用写一遍实体，同时用作参数和返回值。**

### 如何根据不同场景返回不同的字段

实际接口运用中，不同的场景返回的字段范畴往往是不同的，那些不分场合一股脑返回同样字段的接口实现是错误的。这里提到的不同的出场景，比方说：

1. 列表页下返回基本字段，详情页返回更多字段。
2. 普通用户查看时只能看到一部分字段，管理员查看时可查看全部字段。

这些都可以用 `meta-api` 提供的 scope 机制加以区分。我仅以列表页接口举例。

假设 `/articles` 接口返回文章列表，每一篇文章只用返回：`title` 字段；`/articles/:id`  接口需要返回文章详情，每一篇文章需要返回 `title`、`content` 字段。定义两个实体会显得繁琐加混乱，只用定义一个实体并用 `scope:` 选项加以区分：

```ruby
class ArticleEntity < Meta::Entity
  property :title
  property :content, scope: 'full'
end
```

这时，列表页接口和详情页接口用如下的方式实现：

```ruby
class ArticlesController < ApplicationController
  route '/articles', :get do
    status 200 do
      expose :articles, type: 'array', ref: ArticleEntity
    end
  end
  def index
    articles = Article.all
    render json_on_schema: { 'articles' => articles }
  end
    
  route '/articles/:id', :get do
    status 200 do
      expose :article, type: 'object', ref: ArticleEntity
    end
  end
  def show
    article = Article.find(params[:id])
    render json_on_schema: { 'article' => article }, scope: 'full'
  end
end
```

与常规实现唯一的变化，就是 `show` 方法内渲染数据时用到的这一行代码：

```ruby
render json_on_schema: { 'article' => article }, scope: 'full'
```

它传递了一个选项 `scope: 'full'`，告诉实体要同时渲染出 `scope` 标注为 `full` 的字段（也就是 `content` 字段）。

除了在调用时传递特定的选项外，这里还有另外一个技巧，使用 `meta-api` 提供的**锁定**技术将引用的实体锁定在特定的选项上。

我们修改一下上例 `show` 方法的定义和实现，如下：

```ruby
class ArticlesController < ApplicationController
  route '/articles/:id', :get do
    status 200 do
      expose :article, type: 'object', ref: ArticleEntity.locked(scope: 'full')
    end
  end
  def show
    article = Article.find(params[:id])
    render json_on_schema: { 'article' => article }
  end
end
```

这里有两点变化：

1. 在引用实体时，我们对实体作了 `ArticleEntity.locked(scope: 'full')`，它返回新的被锁定的实体。
2. 在渲染数据时，选项 `scope: 'full'` 不用再传递了。

不要小看这点小小的变化，我们在定义时锁定实体，实际上是在接口设计阶段就将实体的作用范畴定义好了。我的观点是，设计优先于实现，因此我更推荐第二种方案。此外，锁定还会影响到文档的生成，文档中的实体只会渲染出锁定 `scope` 的字段，不会生成不会返回的其他的字段。这对前端查看文档时更友好。

我们在 `ArticleEntity` 内添加一个新的用不上的字段：

```ruby
class ArticleEntity < Meta::Entity
  property :title
  property :content, scope: 'full'
  property :foo, scope: 'foo'
end
```

则 `ArticleEntity.locked(scope: 'full')` 不仅实现时只会返回 `title`、`content` 字段，文档渲染时也只会出现 `title`、`content` 这两个字段。

**这一节的思想是：只用写一遍实体，在不同的场合下使用。**这条思想同时也是下一节的思想。

### 参数上的锁定：在参数上区分不同的场合

与返回值一样，参数也需要根据不同的场合修改不同的字段。我们定义一个参数实体：

```ruby
class ExampleEntity < Meta::Entity
  property :name
  property :age
  property :password, scope: 'master'
  property :created_at, scope: 'admin'
  property :updated_at, scope: 'admin'
end
```

它设定有两个 `scope`，在两种场合下能够修改的字段不同。我们在实现时，可以在参数定义时使用锁定技术：

```ruby
class Admin::UsersController < ApplicationController
  route '/admin/users', :put do
    params do
      param :user, ref: ExampleEntity.locked(scope: 'admin')
    end
  end
  def update
  end
end

class UsersController < ApplicationController
  route '/user', :put do
    params do
      param :user, ref: ExampleEntity.locked(scope: 'master')
    end
  end
  def example
  end
end
```

这在实现上和文档上能同时响应。

### 参数上的锁定：处理缺失参数

先声明一个 HTTP 相关的背景。对于缺失的参数，HTTP 提供了两种语义的方法：

1. `PUT` 请求需要提供完整的实体，包括所有字段。如果某个字段缺失，将按照该字段传递了一个 `nil` 值处理。
2. `PATCH` 请求只用提供需要更新的字段。如果某个字段缺失，将表示这个字段不需要被更新。

这里面涉及到的是对于参数中缺失值如何处理。换言之，对于实体：

```ruby
class UserEntity < Meta::Entity
  property :name
  property :age
end
```

如果用户请求只传递了 `{ "name": "Jim" }`，调用 `params_on_schema` 会返回什么呢？是

```json
{ "name": "Jim", "age": null }
```

还是

```json
{ "name": "Jim" }
```

呢？

两者的区分在于前者将缺失值设为 `nil`，后者丢弃了缺失值。注意这两种数据作为参数传递到 `user.update!` 方法中，其效果是不同的。

控制这种效果的方式也是通过锁定设置 `discard_missing:` 选项。

```ruby
class UsersController < ApplicationController
  # PUT 效果，默认情况，参数始终返回完整实体
  route '/users/:id', :put do
    params do
      param :user, ref: UserEntity # 或 UserEntity.locked(discard_missing: false)，等效
    end
  end
  def replace
  end
    
  # PATCH 效果，参数丢弃未传递的字段
  route '/users/:id', :patch do
    params do
      param :ref: UserEntity.locked(discard_missing: true)
    end
  end
  def update
  end
end
```

**总结：只用写一遍实体，在不同的场合下使用。**

## 深入细节

最后，我们讲讲 `meta-api` 有关细节的一些东西吧。它具备基本的参数验证、默认值和转化等逻辑，详细的内容可参考 [教程](https://github.com/yetrun/web-frame/blob/master/docs/%E6%95%99%E7%A8%8B.md) 相关部分。如果文档中出现不友好的部分，欢迎提 ISSUE 改进。

### 默认值

你可以为参数（或实体内的字段）提供默认值，如下：

```ruby
class UserEntity < Meta::Entity
  property :age, type: 'integer', default: 18
end
```

### 数据验证

有一些内建的数据验证器，比如：

```ruby
class UserEntity < Meta::Entity
  property :birth_date, type: 'string', format: /\d\d\d\d-\d\d-\d\d/
end
```

或自定义：

```ruby
class UserEntity < Meta::Entity
  property :birthday, 
    type: 'string', 
    validate: ->(value) { raise Meta::JsonSchema::ValidationError('日期格式不对') unless value =~ /\d\d\d\d-\d\d-\d\d/}
end
```

### 枚举

```ruby
class UserEntity < Meta::Entity
  property :age, type: 'integer', allowable: [1,2,3,4,5,6] # 只允许 6 岁以下儿童
end
```

### 多态

真的，它有处理多态的能力，就像 Swagger 文档中提供的那样。我不想在这里讲解了，但请相信，它真的有处理多态的能力。

## 最后的说辞

啥也不说了，我的项目地址目前是在：

> http://github.com/yetrun/web-frame

所有的意见和建议对我都是有用的。

如果你遇到 Bug，或者想要新的特性，请提 [ISSUE](https://github.com/yetrun/web-frame/issues).

如果你遇到任何使用上的问题，希望快速得到解决，请加群：489579810.

如果这个项目对你有用，请为它添加一个 star，不胜感激。

如果你有兴趣为这个项目贡献，欢迎提供 PR.

我希望为 Rails 提供插件的方式没有对你现在的项目带来任何干扰，并为生成文档方面提供助力。
