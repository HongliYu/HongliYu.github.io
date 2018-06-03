---
layout: post
title: "使用 Rails 构建一个 TO-DO 应用的 JSON API (Part III)"
date: 2017-05-17 17:49:55 +0200
categories: jekyll update
---

第三部分，我们要做三件事情

1. 版本管理
2. 序列化
3. 分页

在开始之前，我们先设置seed，假的数据，方便后面的测试

检查Gemfile

{% highlight ruby %}
gem 'faker'
{% endhighlight %}

编辑 db/seeds.rb

{% highlight ruby %}
user = User.create(name: "hongli", email: "yhlssdone@gmail.com", password: "password")

50.times do
  todo = Todo.create(title: Faker::Lorem.word, created_by: User.first.id)
  todo.items.create(name: Faker::Lorem.word, done: false)
end
{% endhighlight %}

然后，本地开发环境重置数据库

{% highlight shell %}
rails db:migrate:reset
rails db:seed
{% endhighlight %}

如果代码已经提交到 heroku，那么，heroku 重置数据库

{% highlight shell %}
heroku pg:reset DATABASE
heroku run rake db:migrate
heroku run rake db:seed
heroku restart
{% endhighlight %}

好的，准备工作完成，开始进入正题

## 版本管理

最常见的场景，客户端1.0用到了一个 API，但是下个版本1.1中，需要服务端改变同个 API 中的返回字段，但是用户群在客户端发布1.1以后不选择升级，那么服务端就需要适配两个版本的接口，需要引入版本管理。在 Rails 中，如果你需要做这件事情，你只需要做两件事：1. 添加路由约束，这会依据每个 request 的 header 选择不同的接口版本。2. 给控制器命名空间，controller 中不同的 namespace 处理不同的版本

编辑 app/lib/api_version.rb

{% highlight ruby %}
class ApiVersion
  attr_reader :version, :default

  def initialize(version, default = false)
    @version = version
    @default = default
  end

  # check whether version is specified or is default
  def matches?(request)
    check_headers(request.headers) || default
  end

  private

  def check_headers(headers)
    # check version from Accept headers; expect custom media type `todos`
    accept = headers[:accept]
    accept && accept.include?("application/vnd.todos.#{version}+json")
  end
end
{% endhighlight %}

依据标准写法，检查这个外链，很明显 header 里面有对应版本号的字段。从 request 对象，进入 Accept 的 headers，然后检查版本信息，找到对应的接口，这个过程叫做 content negotiation，[外链](http://guides.rubyonrails.org/routing.html#advanced-constraints)，简单讲就是同样的 URI 对应不同的版本，服务端驱动的版本管理。依据 [Media Type Specification](https://tools.ietf.org/html/rfc6838#section-3.2)，你可以通过 [vendor tree](https://en.wikipedia.org/wiki/Media_type#Vendor_tree) 自己定义 media types，比如 application/vnd.example.resource+json

好的，现在我们已经有了约束的类，接下去需要改变路由。由于我们不会改变 URI 来做版本管理(顺便说一句这是错误的做法，是反设计的，但很多公司仍然在使用😄)，所以我们会使用 module scope 和 namespace。

编辑 config/routes

{% highlight ruby %}
Rails.application.routes.draw do
  # namespace the controllers without affecting the URI
  scope module: :v1, constraints: ApiVersion.new('v1', true) do
    resources :todos do
      resources :items
    end
  end

  post 'auth/login', to: 'authentication#authenticate'
  post 'signup', to: 'users#create'
end
{% endhighlight %}

version constraint 是全局有效的，v1 是默认版本，也就是说没指定版本就是 v1。如果我们指定新的版本，Rails 会从高到低去匹配版本号，使用那个 matches 方法。然后我们看到 controller，建立文件夹

{% highlight shell %}
mkdir app/controllers/v1
{% endhighlight %}

把相关文件移动到这个文件夹下面

{% highlight shell %}
mv app/controllers/{todos_controller.rb,items_controller.rb} app/controllers/v1
{% endhighlight %}

还没完，记得加 module 的版本号，看到 app/controllers/v1/todos_controller.rb

{% highlight shell %}
module V1
  class TodosController < ApplicationController
  # [...]
  end
end
{% endhighlight %}

同样的 app/controllers/v1/items_controller.rb

{% highlight ruby %}
module V1
  class ItemsController < ApplicationController
  # [...]
  end
end
{% endhighlight %}

接下去是v2的版本

{% highlight shell %}
rails g controller v2/todos
{% endhighlight %}

编辑config/routes.rb

{% highlight ruby %}
Rails.application.routes.draw do

  # module the controllers without affecting the URI
  scope module: :v2, constraints: ApiVersion.new('v2') do
    resources :todos, only: :index
  end

  scope module: :v1, constraints: ApiVersion.new('v1', true) do
    # [...]
  end
  # [...]
end
{% endhighlight %}

注意顺序，不是默认的版本要在默认的版本上方

编辑 app/controllers/v2/todos_controller.rb

{% highlight shell %}
class V2::TodosController < ApplicationController
  def index
    json_response({ message: 'Hello There'})
  end
end
{% endhighlight %}

因为只是测试用，所以就给条消息吧

![rails-todo-api-login]({{ "/assets/rails-todo-api-login.png" | absolute_url }})

用 v1 登陆，拿到 token 用给 v2 的 todos 发 get 请求，是有效的也是正确的

## 序列化

这是个很重要的特性，经常有这么个段子，客户端问服务端，我要新加个字段，你要多久。服务端说，是已经有的字段吧，几分钟的事情。嗯？这么快，为什么？那是因为序列化😄

这里举个例子，我需要一个 todo 的内容，还有他对应的所有 items 数组，总不至于请求两次 API 吧。使用 [Active model serializers](https://github.com/rails-api/active_model_serializers) ，检查 Gemfile

{% highlight ruby %}
gem 'active_model_serializers', '~> 0.10.0'
{% endhighlight %}

bundle install 以后

{% highlight shell %}
rails g serializer todo
{% endhighlight %}

会有一个新的目录 app/serializers，以及一个新的文件 todo_serializer.rb，编辑

{% highlight ruby %}
class TodoSerializer < ActiveModel::Serializer
  # attributes to be serialized  
  attributes :id, :title, :created_by, :created_at, :updated_at
  # model association
  has_many :items
end
{% endhighlight %}

item 的哪些属性需要序列化，以及 item 和 todo 的关系。用 httpie 再测试下吧

![rails-todo-api-httpie-login]({{ "/assets/rails-todo-api-httpie-login.png" | absolute_url }})

![rails-todo-api-httpie-add-item]({{ "/assets/rails-todo-api-httpie-add-item.png" | absolute_url }})

![rails-todo-httpie-todo-serializer]({{ "/assets/rails-todo-httpie-todo-serializer.png" | absolute_url }})

看到我请求了 todo，返回了 todo 的属性，还有这个 todo 下的所有 items

## 分页

已经接近完成了，由于在真实环境中，数据可能非常多，我们不可能一次返回所有数据，所以我们需要一个功能叫做分页，让数据批量次序返回，对应于前端的上拉加载更多功能

检查Gemfile：

{% highlight ruby %}
  gem 'will_paginate', '~> 3.1.0'
{% endhighlight %}

记得 bundle install

编辑 app/controllers/v1/todos_controller.rb

{% highlight ruby %}
module V1
  class TodosController < ApplicationController
  # [...]
  # GET /todos
  def index
    # get paginated current user todos
    @todos = current_user.todos.paginate(page: params[:page], per_page: 20)
    json_response(@todos)
  end
  # [...]
end
{% endhighlight %}

可见，每次返回20条数据，用 httpie 试一下是不是20条数据，这里就不贴图了

{% highlight shell %}
http :3000/todos page==1 Accept:'application/vnd.todos.v1+json' Authorization:'eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjoxLCJleHAiOjE0OTUxMjc5MTF9.eab5fH5bfz2lZsJw5jTFeRLlwyx3n-ggpsHPaYF7F64'
{% endhighlight %}

注意如果page==0，那么返回所有数据

使用 Rails 5写 RESTful API，真的很高效，赶快实践下吧，小伙伴们😄

这是我部署在heroku上的应用(https://desolate-sea-54055.herokuapp.com/)

完整的git项目地址(https://github.com/HongliYu/todos_api)
