---
layout: post
title: "使用 Rails 构建一个 TO-DO 应用的 JSON API (Part I)"
date: 2017-05-17 17:49:55 +0200
categories: jekyll update
---

第一部分包括，介绍，配置并建立应用，基本业务逻辑与测试

介绍：众所周知 Rails 最初是为了迅速构建 Web 应用而建立的框架，但是技术总是会随着发展而变化，移动端的应用不需要很多富文本信息，作为服务端只要提供纯粹的JSON API就够了。所以去年Rails5发行以来，rails-api(https://github.com/rails-api/rails-api) 这个gem被整合到 Rails 内核中。对比于同样使用 Rails 建立的普通的 Web App，他们的区别是：

1. 应用的中间件（middleware）减少了

2. ApplicationController 继承自 ActionController::API，而不是 ActionController::Base

3. 不需要建立的对应的视图文件，不需要HTML，CSS，嵌入式Ruby什么的那一堆东西

接下去，我们来建立这个应用，检查下环境：

![rails-todo-json-api-environment]({{ "/assets/rails-todo-json-api-environment.png" | absolute_url }})

整理下，我们需要的RESTful API，给自己一个小目标😊

{% highlight shell %}
Endpoint            Functionality
POST /signup            Signup
POST /auth/login        Login
GET /auth/logout        Logout
GET /todos              List all todos
POST /todos             Create a new todo
GET /todos/:id          Get a todo
PUT /todos/:id          Update a todo
DELETE /todos/:id   Delete a todo and its items
GET /todos/:id/items    Get a todo item
PUT /todos/:id/items    Update a todo item
DELETE /todos/:id/items Delete a todo item
{% endhighlight %}

好的，开始建立应用

{% highlight shell %}
rails new todos_api --api -T
{% endhighlight %}

–api 是说明应用只是支持JSON API，那些 Web 应用的中间件我们不要。-T 是说，我们不要 Rails 自带的测试框架 Minitest，因为我们要测试 Request，还有他的返回值，RSpec 是首选。看下我们需要的 gem，并给出 list：

[rspec-rails](https://github.com/rspec/rspec-rails) – 测试框架

[factory_girl_rails](https://github.com/thoughtbot/factory_girl_rails) – 替代fixtures，语法更加直接

[shoulda_matchers](https://github.com/thoughtbot/shoulda-matchers) - 提供 RSpec 额外的 matchers

[database_cleaner](https://github.com/DatabaseCleaner/database_cleaner) – 你说呢

[faker](https://github.com/stympy/faker) – 假数据，测试用的

这是最后的Gemfile:

{% highlight ruby %}
source 'https://rubygems.org'

ruby '2.3.3'

gem 'rails', '~> 5.0.2'

gem 'bcrypt', '~> 3.1.7'
gem 'jwt'
gem 'active_model_serializers', '~> 0.10.0'
gem 'will_paginate', '~> 3.1.0'
gem 'faker'

group :development, :test do
  gem 'byebug', platform: :mri
  gem 'rspec-rails', '~> 3.5'
  gem 'sqlite3'
  gem 'factory_girl_rails', '~> 4.0'
  gem 'shoulda-matchers', '~> 3.1'
  gem 'database_cleaner'
  gem 'listen', '~> 3.0.5'
  gem 'spring'
  gem 'spring-watcher-listen', '~> 2.0.0'
end

group :production do
  gem 'pg'
end
{% endhighlight %}

完了 bundle install，我们在本地使用 sqlite3 数据库，但是 heroku 不支持，换成了 Postgres。接下来我们配置下 Rspec

{% highlight shell %}
rails generate rspec:install
{% endhighlight %}

你会发现增加了以下三个文件：.rspec, spec/spec_helper.rb, spec/rails_helper.rb

再增加一个factories目录：

{% highlight shell %}
mkdir spec/factories
{% endhighlight %}

然后我们配置这些添加的Gem, 打开spec/rails_helper.rb

{% highlight ruby %}
# require database cleaner at the top level
require 'database_cleaner'

# [...]
# 配置 shoulda matchers 使用 rspec 作为测试框架，对应的 libraries 是 rails
Shoulda::Matchers.configure do |config|
  config.integrate do |with|
    with.test_framework :rspec
    with.library :rails
  end
end

# [...]
RSpec.configuration do |config|
  # [...]
  # 增加 `FactoryGirl` 的所有methods
  config.include FactoryGirl::Syntax::Methods

  # start by truncating all the tables but then use the faster transaction strategy the rest of the time.
  config.before(:suite) do
    DatabaseCleaner.clean_with(:truncation)
    DatabaseCleaner.strategy = :transaction
  end

  # start the transaction strategy as examples are run
  config.around(:each) do |example|
    DatabaseCleaner.cleaning do
      example.run
    end
  end
  # [...]
end
{% endhighlight %}

现在创建Models，开始书写基本的业务内容

{% highlight shell %}
rails g model Todo title:string created_by:string
{% endhighlight %}

Model 的名字叫 Todo, 有两个属性 title, created_by, 顺便检查下生成的迁移文件，db/migrate/[timestamp]_create_todos.rb

{% highlight ruby %}
class CreateTodos < ActiveRecord::Migration[5.0]
  def change
    create_table :todos do |t|
      t.string :title
      t.string :created_by

      t.timestamps
    end
  end
end
{% endhighlight %}

同样的，我们增加另外一个 Model 叫做 Item

{% highlight shell %}
rails g model Item name:string done:boolean todo:references
{% endhighlight %}

reference 是添加外键，一个 item 点开来以后是多个 todo 待办事项对吧，所以他们的关系也是1:n。打开迁移文件，确认外键关系
{% highlight ruby %}
class CreateItems < ActiveRecord::Migration[5.0]
  def change
    create_table :items do |t|
      t.string :name
      t.boolean :done
      t.references :todo, foreign_key: true

      t.timestamps
    end
  end
end
{% endhighlight %}

最后记得，把model映射到数据库，Rails 有专业术语叫 migrate

{% highlight shell %}
rails db:migrate
{% endhighlight %}

好吧，测试驱动，我们要开始写测试了，看到 spec/models/todo_spec.rb

{% highlight ruby %}
require 'rails_helper'

# Test suite for the Todo model
RSpec.describe Todo, type: :model do
  # Association test
  # 一对多的关系，级联删除
  it { should have_many(:items).dependent(:destroy) }
  # 相关字段的存在性验证
  it { should validate_presence_of(:title) }
  it { should validate_presence_of(:created_by) }
end
{% endhighlight %}

其实我不加注释，也应该能看懂，因为太显而易见了😄。不得不说，Rspec 是非常具有表现力的 DSL(Domain Specific Language)。继续看到 spec/models/item_spec.rb

{% highlight ruby %}
require 'rails_helper'

# Test suite for the Item model
RSpec.describe Item, type: :model do
  it { should belong_to(:todo) }
  it { should validate_presence_of(:name) }
end
{% endhighlight %}

看到 shoulda matchers 在 Rspec中的作用了对吧，字面意思就是 item 应该属于 todo，验证 name 字段的存在性，就是普通的英语短句

如果现在运行测试用例，当然不可能通过，不信自己试下:

{% highlight shell %}
rspec
{% endhighlight %}

都是红的，然后我们一步步让他变绿，看到 app/models/todo.rb

{% highlight ruby %}
class Todo < ApplicationRecord
  has_many :items, dependent: :destroy
  validates_presence_of :title, :created_by
end
{% endhighlight %}

app/models/item.rb

{% highlight ruby %}
class Item < ApplicationRecord
  belongs_to :todo
  validates_presence_of :name
end
{% endhighlight %}

再次运行spec，都变绿了对吧，必须的嘛。现在来写 Controller

{% highlight shell %}
rails g controller Todos
rails g controller Items
{% endhighlight %}

我们不会写 controller 的测试，而是要写 request 的测试，因为覆盖得更加多，包括路径，请求完成以后的回调。创建文件夹和相关的文件
{% highlight shell %}
mkdir spec/requests && touch spec/requests/{todos_request_spec.rb,items_request_spec.rb}
{% endhighlight %}

记得不能写成 todos_spec.rb，会引起重复引用的错误：in `method_missing’: Factory already registered: item (FactoryGirl::DuplicateDefinitionError)

增加固件测试，就是跑测试用例时候的样本文件：

{% highlight shell %}
touch spec/factories/{todos.rb,items.rb}
{% endhighlight %}

编辑文件spec/factories/todos.rb，加上两个假数据

{% highlight ruby %}
FactoryGirl.define do
  factory :todo do
    title { Faker::Lorem.word }
    created_by { Faker::Number.number(10) }
  end
end
{% endhighlight %}

再看 spec/factories/items.rb，类似的，先不管外键

{% highlight ruby %}
FactoryGirl.define do
  factory :item do
    name { Faker::StarWars.character }
    done false
    todo_id nil
  end
end
{% endhighlight %}

最后写关键的 Request 测试部分，看到 spec/requests/todos_spec.rb

{% highlight ruby %}
require 'rails_helper'

RSpec.describe 'Todos API', type: :request do
  # initialize test data 
  let!(:todos) { create_list(:todo, 10) }
  let(:todo_id) { todos.first.id }

  # Test suite for GET /todos
  describe 'GET /todos' do
    # make HTTP get request before each example
    before { get '/todos' }

    it 'returns todos' do
      # Note `json` is a custom helper to parse JSON responses
      expect(json).not_to be_empty
      expect(json.size).to eq(10)
    end

    it 'returns status code 200' do
      expect(response).to have_http_status(200)
    end
  end

  # Test suite for GET /todos/:id
  describe 'GET /todos/:id' do
    before { get "/todos/#{todo_id}" }

    context 'when the record exists' do
      it 'returns the todo' do
        expect(json).not_to be_empty
        expect(json['id']).to eq(todo_id)
      end

      it 'returns status code 200' do
        expect(response).to have_http_status(200)
      end
    end

    context 'when the record does not exist' do
      let(:todo_id) { 100 }

      it 'returns status code 404' do
        expect(response).to have_http_status(404)
      end

      it 'returns a not found message' do
        expect(response.body).to match(/Couldn't find Todo/)
      end
    end
  end

  # Test suite for POST /todos
  describe 'POST /todos' do
    # valid payload
    let(:valid_attributes) { { title: 'Learn Elm', created_by: '1' } }

    context 'when the request is valid' do
      before { post '/todos', params: valid_attributes }

      it 'creates a todo' do
        expect(json['title']).to eq('Learn Elm')
      end

      it 'returns status code 201' do
        expect(response).to have_http_status(201)
      end
    end

    context 'when the request is invalid' do
      before { post '/todos', params: { title: 'Foobar' } }

      it 'returns status code 422' do
        expect(response).to have_http_status(422)
      end

      it 'returns a validation failure message' do
        expect(response.body)
          .to match(/Validation failed: Created by can't be blank/)
      end
    end
  end

  # Test suite for PUT /todos/:id
  describe 'PUT /todos/:id' do
    let(:valid_attributes) { { title: 'Shopping' } }

    context 'when the record exists' do
      before { put "/todos/#{todo_id}", params: valid_attributes }

      it 'updates the record' do
        expect(response.body).to be_empty
      end

      it 'returns status code 204' do
        expect(response).to have_http_status(204)
      end
    end
  end

  # Test suite for DELETE /todos/:id
  describe 'DELETE /todos/:id' do
    before { delete "/todos/#{todo_id}" }

    it 'returns status code 204' do
      expect(response).to have_http_status(204)
    end
  end
end
{% endhighlight %}

一个 describe 对应一个 Request 测试，各种断言，一次返回10条 todos，json 数据不能为空，而且状态码是200，太明显了对吧，Rspec确实语法简练。然后我们要定义 json 方法，解析 JSON 数据到 Ruby hash，创建

{% highlight shell %}
mkdir spec/support && touch spec/support/request_spec_helper.rb
{% endhighlight %}

写入spec/support/request_spec_helper

{% highlight ruby %}
module RequestSpecHelper
  # Parse JSON response to ruby hash
  def json
    JSON.parse(response.body)
  end
end
{% endhighlight %}

support 目录不是自动添加到工程的，打开这个文件 spec/rails_helper.rb 写入

{% highlight ruby %}
Dir[Rails.root.join('spec/support/**/*.rb')].each { |f| require f }
RSpec.configuration do |config|
  # [...]
  config.include RequestSpecHelper, type: :request
  # [...]
end
{% endhighlight %}

然后我们配置跳转的路由，打开config/routes.rb

{% highlight ruby %}
Rails.application.routes.draw do
  resources :todos do
    resources :items
  end
end
{% endhighlight %}

对，就是这么神奇，一个循环😄

{% highlight shell %}
rails routes
{% endhighlight %}

看下路由对不对，然后配置 controller 中的方法，app/controllers/todos_controller.rb

{% highlight ruby %}
class TodosController < ApplicationController
  before_action :set_todo, only: [:show, :update, :destroy]

  # GET /todos
  def index
    @todos = Todo.all
    json_response(@todos)
  end

  # POST /todos
  def create
    @todo = Todo.create!(todo_params)
    json_response(@todo, :created)
  end

  # GET /todos/:id
  def show
    json_response(@todo)
  end

  # PUT /todos/:id
  def update
    @todo.update(todo_params)
    head :no_content
  end

  # DELETE /todos/:id
  def destroy
    @todo.destroy
    head :no_content
  end

  private

  def todo_params
    # whitelist params
    params.permit(:title, :created_by)
  end

  def set_todo
    @todo = Todo.find(params[:id])
  end
end
{% endhighlight %}

还记得那个返回状态码的 json_response 方法，配置下，app/controllers/concerns/response.rb
{% highlight ruby %}
module Response
  def json_response(object, status = :ok)
    render json: object, status: status
  end
end
{% endhighlight %}

还有统一的异常处理，app/controllers/concerns/exception_handler.rb

{% highlight ruby %}
module ExceptionHandler
  # provides the more graceful `included` method
  extend ActiveSupport::Concern

  included do
    rescue_from ActiveRecord::RecordNotFound do |e|
      json_response({ message: e.message }, :not_found)
    end

    rescue_from ActiveRecord::RecordInvalid do |e|
      json_response({ message: e.message }, :unprocessable_entity)
    end
  end
end
{% endhighlight %}

在 TodosController 中我们使用 create!，而不是 create 方法，这样就直接抛出异常 ActiveRecord::RecordInvalid，在 ExceptionHandler 中实现异常的复用。然后在 app/controllers/application_controller.rb 加入帮助文件，让他们生效

{% highlight ruby %}
class ApplicationController < ActionController::API
  include Response
  include ExceptionHandler
end
{% endhighlight %}

最后测试下，正常情况下都是绿的
接下去类似的写下item的测试，app/requests/items_spec.rb

{% highlight ruby %}
require 'rails_helper'

RSpec.describe 'Items API' do
  # Initialize the test data
  let!(:todo) { create(:todo) }
  let!(:items) { create_list(:item, 20, todo_id: todo.id) }
  let(:todo_id) { todo.id }
  let(:id) { items.first.id }

  # Test suite for GET /todos/:todo_id/items
  describe 'GET /todos/:todo_id/items' do
    before { get "/todos/#{todo_id}/items" }

    context 'when todo exists' do
      it 'returns status code 200' do
        expect(response).to have_http_status(200)
      end

      it 'returns all todo items' do
        expect(json.size).to eq(20)
      end
    end

    context 'when todo does not exist' do
      let(:todo_id) { 0 }

      it 'returns status code 404' do
        expect(response).to have_http_status(404)
      end

      it 'returns a not found message' do
        expect(response.body).to match(/Couldn't find Todo/)
      end
    end
  end

  # Test suite for GET /todos/:todo_id/items/:id
  describe 'GET /todos/:todo_id/items/:id' do
    before { get "/todos/#{todo_id}/items/#{id}" }

    context 'when todo item exists' do
      it 'returns status code 200' do
        expect(response).to have_http_status(200)
      end

      it 'returns the item' do
        expect(json['id']).to eq(id)
      end
    end

    context 'when todo item does not exist' do
      let(:id) { 0 }

      it 'returns status code 404' do
        expect(response).to have_http_status(404)
      end

      it 'returns a not found message' do
        expect(response.body).to match(/Couldn't find Item/)
      end
    end
  end

  # Test suite for PUT /todos/:todo_id/items
  describe 'POST /todos/:todo_id/items' do
    let(:valid_attributes) { { name: 'Visit Narnia', done: false } }

    context 'when request attributes are valid' do
      before { post "/todos/#{todo_id}/items", params: valid_attributes }

      it 'returns status code 201' do
        expect(response).to have_http_status(201)
      end
    end

    context 'when an invalid request' do
      before { post "/todos/#{todo_id}/items", params: {} }

      it 'returns status code 422' do
        expect(response).to have_http_status(422)
      end

      it 'returns a failure message' do
        expect(response.body).to match(/Validation failed: Name can't be blank/)
      end
    end
  end

  # Test suite for PUT /todos/:todo_id/items/:id
  describe 'PUT /todos/:todo_id/items/:id' do
    let(:valid_attributes) { { name: 'Mozart' } }

    before { put "/todos/#{todo_id}/items/#{id}", params: valid_attributes }

    context 'when item exists' do
      it 'returns status code 204' do
        expect(response).to have_http_status(204)
      end

      it 'updates the item' do
        updated_item = Item.find(id)
        expect(updated_item.name).to match(/Mozart/)
      end
    end

    context 'when the item does not exist' do
      let(:id) { 0 }

      it 'returns status code 404' do
        expect(response).to have_http_status(404)
      end

      it 'returns a not found message' do
        expect(response.body).to match(/Couldn't find Item/)
      end
    end
  end

  # Test suite for DELETE /todos/:id
  describe 'DELETE /todos/:id' do
    before { delete "/todos/#{todo_id}/items/#{id}" }

    it 'returns status code 204' do
      expect(response).to have_http_status(204)
    end
  end
end
{% endhighlight %}

编辑 app/controllers/items_controller.rb

{% highlight ruby %}
class ItemsController < ApplicationController
  before_action :set_todo
  before_action :set_todo_item, only: [:show, :update, :destroy]

  # GET /todos/:todo_id/items
  def index
    json_response(@todo.items)
  end

  # GET /todos/:todo_id/items/:id
  def show
    json_response(@item)
  end

  # POST /todos/:todo_id/items
  def create
    @todo.items.create!(item_params)
    json_response(@todo, :created)
  end

  # PUT /todos/:todo_id/items/:id
  def update
    @item.update(item_params)
    head :no_content
  end

  # DELETE /todos/:todo_id/items/:id
  def destroy
    @item.destroy
    head :no_content
  end

  private

  def item_params
    params.permit(:name, :done)
  end

  def set_todo
    @todo = Todo.find(params[:todo_id])
  end

  def set_todo_item
    @item = @todo.items.find_by!(id: params[:id]) if @todo
  end
end
{% endhighlight %}

所有测试都变绿通过
