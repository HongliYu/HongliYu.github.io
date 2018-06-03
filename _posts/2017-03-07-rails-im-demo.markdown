---
layout: post
title: "使用Rails5构建简单的IM"
date: 2017-03-07 17:49:55 +0200
categories: jekyll update
---

Rials5带来了很多新的功能，能让我们更加容易地书写web app, json API 的接口，但最激动人心的应该是 actioncable，长链接 websocket 变得如此简单，来我们用30分钟来写一个简单的即时通讯的 web app，前端用 JS，服务端用 Ruby。actioncable 的[外链](https://github.com/rails/rails/tree/master/actioncable)

应用的功能是这样的，打开网页，注册，或者已注册用户登录，新建聊天室，或进入已有的聊天室，同一个聊天室容纳多个人，一个聊天室内所有人会收到新的聊天信息的推送。

## 建立应用，User建模，数据迁移

环境：Ruby Version: 2.3.3，Rails Version: 5.0.2

打开ternimal, cd到相关目录下，建立工程

{% highlight shell %}
rails new im_demo -T
{% endhighlight %}

先不写测试了，所以没有加入自带的测试框架

打开Gemfile

{% highlight ruby %}
...
gem 'devise'
gem 'redis', '~> 3.2'
gem 'bootstrap', '4.0.0.alpha6'
...

group :development, :test do
  gem 'sqlite3'
end

...
group :production do
  gem 'pg'
  gem 'rails_12factor'
end
{% endhighlight %}

这个Device是用户认证用的，简化注册登录流程，[外链](https://github.com/plataformatec/devise)

bootstrap 主要是样式，scss 是面向对象的 css，可以嵌套使用，而且默认提供了一部分组件样式。redis 是内存数据库，因为 instance message 需要及时响应，mac 上 brew install redis，完成以后记得bundle install

增加scss样式表，同时删除原来的css

{% highlight shell %}
remove app/assets/stylesheets/application.css
touch app/assets/stylesheets/application.scss
{% endhighlight %}

写入@import "bootstrap";

### User建模

{% highlight shell %}
rails generate devise:install
rails generate devise User
rails generate devise:views
rails db:migrate
{% endhighlight %}

增加约束，进入网站其他页面，必须是认证用户

{% highlight shell %}
subl app/controllers/application_controller.rb
{% endhighlight %}

写入

{% highlight ruby %}
before_action :authenticate_user!
{% endhighlight %}

### 聊天室

{% highlight shell %}
rails g model ChatRoom title:string user:references
rails db:migrate
{% endhighlight %}

检查 chat_room.rb，确认看到 belongs_to :user
检查 users.rb，确认看到 has_many :chat_rooms, dependent: :destroy

聊天室需要一个标题，在用户登录以后，选择聊天室时可见，而且聊天室是由用户创建的，所以和创建他的用户绑定，如果创建他的用户被删除了，那么聊天室的内容也会被删除，类似于SQL中的 DELETE CASCADE 效果，完了记得映射 model 到数据库 migrate 操作

为chatroom创建controller

{% highlight shell %}
touch app/controllers/chat_rooms_controller.rb
{% endhighlight %}

写入

{% highlight ruby %}
class ChatRoomsController < ApplicationController
  def index
    @chat_rooms = ChatRoom.all
  end

  def new
    @chat_room = ChatRoom.new
  end

  def show
    @chat_room = ChatRoom.includes(:messages).find_by(id: params[:id])
    @message = Message.new
  end

  def create
    @chat_room = current_user.chat_rooms.build(chat_room_params)
    if @chat_room.save
      flash[:success] = 'Chat room added!'
      redirect_to chat_rooms_path
    else
      render 'new'
    end
  end

  private

  def chat_room_params
    params.require(:chat_room).permit(:title)
  end
end
{% endhighlight %}

注意 Message 这个类现在还没有创建。index，展现所有 chatrooms，这回事一个 list，在首页，用户登录以后会看到。new 是 chatroom 的新建操作。create 是登录用户可以新建一个聊天室，聊天室创建以后会和当前用户关联。chat_room_params 中的 permit是参数限制，防止 sql 注入攻击

建立简单的 view，显示所有聊天室的 view，views/chat_rooms/index.html.erb

{% highlight ruby %}
<h1>Chat rooms</h1>

<p class="lead"><%= link_to 'New chat room', new_chat_room_path, class: 'btn btn-primary' %></p>

<ul>
  <%= render @chat_rooms %>
</ul>
{% endhighlight %}

views/chat_rooms/_chat_room.html.erb，进入聊天室的，点击按钮，以及事件

{% highlight ruby %}
<li><%= link_to "Enter #{chat_room.title}", chat_room_path(chat_room) %></li>
{% endhighlight %}

views/chat_rooms/new.html.erb，用户新建聊天室，看到那些参数，就是 controller 中会用到的

{% highlight ruby %}
<%= form_for @chat_room do |f| %>
  <div class="form-group">
    <%= f.label :title %>
    <%= f.text_field :title, autofocus: true, class: 'form-control' %>
  </div>

  <%= f.submit "Add!", class: 'btn btn-primary' %>
<% end %>
{% endhighlight %}

接下去建立 message 模块

{% highlight shell %}
rails g model Message body:text user:references chat_room:references
rails db:migrate
{% endhighlight %}

检查 models/chat_room.rb，看到 has_many :messages, dependent: :destroy
检查 models/users.rb，看到 has_many :messages, dependent: :destroy
检查 models/message.rb，看到 belongs_to :user，belongs_to :chat_room

1:n 的关系，在 Rails 中是1的 model 对应 has_many，n的 model 对应 belongs_to，一个聊天室对应多个消息，没毛病。现在 chat_rooms_controller.rb 中的 show 操作，变得有意义了，因为添加了 message 模块。说人话，进入对应的聊天室，展现所有的消息，不管是谁发的，对吧

views/chat_rooms/show.html.erb，聊天室详细页面，展现该聊天室的所有消息

{% highlight ruby %}
<h1><%= @chat_room.title %></h1>

<div id="messages">
  <%= render @chat_room.messages %>
</div>
{% endhighlight %}

views/messages/_message.html.erb，每一条消息的展现，默认组件 card

{% highlight ruby %}
<div class="card">
  <div class="card-block">
    <div class="row">
      <div class="col-md-1">
        <%= gravatar_for message.user %>
      </div>
      <div class="col-md-11">
        <p class="card-text">
          <span class="text-muted"><%= message.user.name %> at <%= message.timestamp %> says</span><br>
          <%= message.body %>
        </p>
      </div>
    </div>
  </div>
</div>
{% endhighlight %}

gravatar_for 头像，message.user.name 用户名字，message.timestamp 消息时间戳，message.body 消息内容

用户注册的邮件不能直接显示在列表中，属于个人隐私，所以我们只截取@之前的那一部分作为用户的名字，models/user.rb

{% highlight ruby %}
def name
  email.split('@')[0]
end
{% endhighlight %}

数据库中的时间，对用户不是很友好，我们改下显示格式，models/message.rb

{% highlight ruby %}
def timestamp
  created_at.strftime('%H:%M:%S %d %B %Y')
end
{% endhighlight %}

记得写在 gravatar_for 方法，在 application_helper.rb 中

{% highlight ruby %}
module ApplicationHelper
  def gravatar_for(user, opts = {})
    opts[:alt] = user.name
    image_tag "https://www.gravatar.com/avatar/#{Digest::MD5.hexdigest(user.email)}?s=#{opts.delete(:size) { 40 }}",
              opts
  end
end
{% endhighlight %}

就是 www.gravatar.com 这个网站的服务，你的邮箱和你的头像绑定的第三方服务

默认的样式不好看，但是在 application.scss 中可以添加样式

最后加上跳转的路由，config/routes.rb

{% highlight ruby %}
resources :chat_rooms, only: [:new, :create, :show, :index]
root 'chat_rooms#index'
{% endhighlight %}

根目录就是聊天室的列表页面

好的，模型，界面基本完成，我们要增加核心功能

###  ActionCable封装好的websocket服务，发消息，全局广播，即刻响应

看到config/cable.yml

{% highlight ruby %}
development:
  adapter: async

test:
  adapter: async

production:
  adapter: redis
  url: <%= ENV["REDISCLOUD_URL"] %>
{% endhighlight %}

REDISCLOUD_URL 是线上环境变量

看到config/routes.rb，增加

{% highlight ruby %}
mount ActionCable.server => '/cable'
{% endhighlight %}

检查javascripts/cable.js，看到

{% highlight ruby %}
//= require action_cable
//= require_self
//= require_tree ./channels

(function() {
 this.App || (this.App = {});

 App.cable = ActionCable.createConsumer();

}).call(this);
{% endhighlight %}

看到javascripts/application.js，增加

{% highlight ruby %}
//= require cable
{% endhighlight %}

让cable.js的代码生效

什么是Consumer，给出[外链](https://github.com/rails/rails/tree/master/actioncable#terminology)
主要是解决，一个用户在多个频道，用户又有多个终端，比如不同的浏览器中同步消息

consumer 可以订阅（subscribe）多个 cable channels, 每个 channel 封装了逻辑单元，consumer 创建以后至少订阅一个 channel。consumer 订阅 channel 以后就成为了订阅者，一个 consumer 可以多次成为同一个 channel 的订阅者，订阅以后消息的发送和接收是双向的。你会看到其他用户发的消息，是 server 传输给你的，你发的消息先传给 server，然后 server广播给其他用户，在同一个 channel上，对吧。对于 websocket 的链接来说，consumer 是 client side, channel 是 server side，channel 类似于 controller，但是他处理的是 streaming 数据流，不是 http 的 request，因为一点连接建立以后，传递的是数据流，而用 http 的长链接做轮训，本质上还是会 close 一个 session 的，效率很低

我们来新建一个channel吧，javascripts/channels/rooms.coffee

{% highlight ruby %}
jQuery(document).on 'turbolinks:load', ->
  messages = $('#messages')
  if $('#messages').length > 0
 
    App.global_chat = App.cable.subscriptions.create {
        channel: "ChatRoomsChannel"
        chat_room_id: messages.data('chat-room-id')
      },
      connected: ->
        # Called when the subscription is ready for use on the server
 
      disconnected: ->
        # Called when the subscription has been terminated by the server
 
      received: (data) ->
        messages.append data['message']
 
      send_message: (message, chat_room_id) ->
        @perform 'send_message', message: message, chat_room_id: chat_room_id
 
    $('#new_message').submit (e) ->
      $this = $(this)
      textarea = $this.find('#message_body')
      if $.trim(textarea.val()).length > 1
        App.global_chat.send_message textarea.val(), messages.data('chat-room-id')
        textarea.val('')
      e.preventDefault()
      return false
{% endhighlight %}

CoffeeScript 是一门编译到 JavaScript 的小巧语言，[外链](http://coffee-script.org)

逻辑上，如果有在页面上有 messages block，那么 client subscribe to the channel，订阅这个频道，订阅的动作依赖于room’s id，submit就是前端提交message，app会在channel内广播消息，同时输入框清空

views/chat_rooms/show.html.erb，输入消息的表单

{% highlight ruby %}
<%= form_for @message, url: '#' do |f| %>
  <div class="form-group">
    <%= f.label :body %>
    <%= f.text_area :body, class: 'form-control' %>
    <small class="text-muted">From 2 to 1000 characters</small>
  </div>

  <%= f.submit "Post", class: 'btn btn-primary btn-lg' %>
<% end %>
{% endhighlight %}

@message 应该在 controller 中初始化，看到 chat_rooms_controller.rb 中 show 方法中是不是创建了一个 @message 对象

增加message的约束，models/message.rb

{% highlight ruby %}
validates :body, presence: true, length: {minimum: 2, maximum: 1000}
{% endhighlight %}

message的body属性，必须存在，2-1000个字符之间

看到 views/chat_rooms/show.html.erb，@chat_room.id 这个参数是 subscriptions 中需要的
好了，下面看下 Server 的代码。channels/chat_rooms_channel.rb

{% highlight ruby %}
class ChatRoomsChannel < ApplicationCable::Channel
  def subscribed
    stream_from "chat_rooms_#{params['chat_room_id']}_channel"
  end

  def unsubscribed
    # Any cleanup needed when channel is unsubscribed
  end

  def send_message(data)
    # process data sent from the page
        current_user.messages.create!(body: data['message'], chat_room_id: data['chat_room_id'])
  end
end
{% endhighlight %}

感受下，是不是对应了js代码，订阅，取消订阅，发送消息，之前 set 的参数，在这里 get。好了么？还没有，Devise 的 current_user，我们还没定义，channels/application_cable/connection.rb

{% highlight ruby %}
module ApplicationCable
  class Connection < ActionCable::Connection::Base
    
        identified_by :current_user

    def connect
      self.current_user = find_verified_user
      logger.add_tags 'ActionCable', current_user.email
    end

    protected

    def find_verified_user # this checks whether a user is authenticated with devise
      if verified_user = env['warden'].user
        verified_user
      else
        reject_unauthorized_connection
      end
    end

  end
end
{% endhighlight %}

完了以后，current_user在channel 开始生效，没有认证的用户无法广播他们的消息。logger.add_tags 是在 console 中查看 debug 信息。Devise 的认证是建立在Warden的基础上的，[外链](https://github.com/hassox/warden)，env['warden'].user 是取得当前的 login 用户

前端数据到server内存，对了还有数据库，我得把消息存起来，models/message.rb，增加

{% highlight ruby %}
after_create_commit { MessageBroadcastJob.perform_later(self) }
{% endhighlight %}

看字面意思，广播消息以后，存到数据库

打开，jobs/message_broadcast_job.rb

{% highlight ruby %}
class MessageBroadcastJob < ApplicationJob
  queue_as :default

  def perform(message)
    ActionCable.server.broadcast "chat_rooms_#{message.chat_room.id}_channel",
                                 message: render_message(message)
  end

  private

  def render_message(message)
    MessagesController.render partial: 'messages/message', locals: {message: message}
  end
end
{% endhighlight %}

perform 方法做了广播的事情，得到的消息需要 controller 来处理，新建 MessagesController，messages_controller.rb

{% highlight ruby %}
class MessagesController < ApplicationController
end
{% endhighlight %}

重新看到客户端 rooms.coffee，貌似已经 ready 了。但是你会发现，新的消息不是在最后，但我要的效果是slack那种，新的消息在底部显示，但是聊天窗口会滚动，新增

{% highlight ruby %}
...
  if $('#messages').length > 0
    messages_to_bottom = -> messages.scrollTop(messages.prop("scrollHeight"))
    messages_to_bottom()
...
  received: (data) ->
      messages.append data['message']
      messages_to_bottom()
...
{% endhighlight %}

最后，稍微美化下导航栏。layouts/application.html.erb

{% highlight ruby %}
<!DOCTYPE html>
<html>
<head>
  <title>IMDemo</title>
  <%= csrf_meta_tags %>

  <%= stylesheet_link_tag    'application', media: 'all', 'data-turbolinks-track': 'reload' %>
  <%= javascript_include_tag 'application', 'data-turbolinks-track': 'reload' %>
</head>

<body>

<nav class="navbar navbar-toggleable-md navbar-light bg-faded">
  <%= link_to 'IMDemo', root_path, class: 'navbar-brand' %>
  <% if user_signed_in? %>
    <ul class="navbar-nav mr-auto">
      <li class="nav-item active">
        <%= link_to 'Log out', destroy_user_session_path, method: :delete, class: 'nav-link' %>
      </li>
    </ul>
    <span class="navbar-text">Logged in as <strong><%= current_user.name %></strong></span>

  <% end %>
</nav>


<div class="container">
  <%= yield %>
</div>

<footer>
  <div class="container">
    <p class="text-muted">Created by <%= link_to 'Hongli', 'http://hlyu.cn' %></p>
  </div>
</footer>
</body>
</html>
{% endhighlight %}

现在完成了，打开 rails s，打开浏览器: (http://localhost:3000)，注册，以后直接就登录了，在不同浏览器，打开同一个聊天室，不同的账号登陆，试一试。
最后推送到heroku线上环境，首先你要redis服务，免费的插件(https://elements.heroku.com/addons), 就选Rediscloud吧，30m免费

看到config/environments/production.rb

{% highlight ruby %}
config.action_cable.allowed_request_origins = ['https://radiant-fjord-14017.herokuapp.com',
'https://radiant-fjord-14017.herokuapp.com']
config.action_cable.url = 'wss://radiant-fjord-14017.herokuapp.com/cable'
{% endhighlight %}

mark (https://radiant-fjord-14017.herokuapp.com) 这是我的线上地址，需要替换成你自己的地址。

这是线上的demo地址: (https://radiant-fjord-14017.herokuapp.com/)

源代码的地址：(https://github.com/HongliYu/im_demo)
