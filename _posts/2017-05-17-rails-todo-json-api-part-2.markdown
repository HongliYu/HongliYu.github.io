---
layout: post
title: "使用 Rails 构建一个 TO-DO 应用的 JSON API (Part II)"
date: 2017-05-17 17:49:55 +0200
categories: jekyll update
---

在第一部分中，我们创建了基于 JSON API 的 Rails 应用，搭建了 Rspec 测试框架，也介绍了简单的测试驱动的开发。在第二部分中，我们要建立 JWT 认证(JSON Web Token)，包换注册和登录，当然还是使用 TDD。认证原理的[外链](https://www.pluralsight.com/guides/ruby-ruby-on-rails/token-based-authentication-with-ruby-on-rails-5-api)

认证涉及到用户，那先创建用户

{% highlight shell %}
$ rails g model User name:string email:string password_digest:string
# run the migrations
$ rails db:migrate
# make sure the test environment is ready
$ rails db:test:prepare
{% endhighlight %}

建立model的测试，spec/models/user_spec.rb

{% highlight ruby %}
require 'rails_helper'

RSpec.describe User, type: :model do
  it { should have_many(:todos) }
  it { should validate_presence_of(:name) }
  it { should validate_presence_of(:email) }
  it { should validate_presence_of(:password_digest) }
end
{% endhighlight %}

添加固件，spec/factories/users.rb

{% highlight ruby %}
require 'rails_helper'

RSpec.describe User, type: :model do
FactoryGirl.define do
  factory :user do
    name { Faker::Name.name }
    email 'foo@bar.com'
    password 'foobar'
  end
end
{% endhighlight %}

添加model的约束，app/models/user.rb

{% highlight ruby %}
class User < ApplicationRecord
  # encrypt password
  has_secure_password

  # Model associations
  has_many :todos, foreign_key: :created_by
  # Validations
  validates_presence_of :name, :email, :password_digest
end
{% endhighlight %}

has_secure_password，rails5 会自动添加相关属性 password_digest，经过信息摘要处理的 password，总不能在数据库里存储明文密码吧。检查 gemfile 确认存在

{% highlight ruby %}
gem 'bcrypt', '~> 3.1.7'
{% endhighlight %}

接下去要做的几件事情：

1. JsonWebToken: Encode & decode jwt tokens
2. AuthorizeApiRequest: 每一个API请求的认证
3. AuthenticateUser:  认证用户
4. AuthenticationController：认证流程

## JsonWebToken

什么是基于token的认证，[外链](https://stackoverflow.com/questions/1592534/what-is-token-based-authentication)，确认Gemfile里面有
{% highlight ruby %}
gem 'jwt'
{% endhighlight %}

然后我们会把生成 jwt token 的 class 放在 lib 目录下，因为这不是领域特定的。但是 lib 目录下的所有内容是自动加载的。但是 Rails 5 中因为线程问题，取消了自动加载，关于这个问题的讨论，[外链](https://github.com/rails/rails/issues/13142)

{% highlight ruby %}
mkdir app/lib && touch app/lib/json_web_token.rb
{% endhighlight %}

定义这个单例，app/lib/json_web_token.rb

{% highlight ruby %}
class JsonWebToken
  # secret to encode and decode token
  HMAC_SECRET = Rails.application.secrets.secret_key_base

  def self.encode(payload, exp = 24.hours.from_now)
    # set expiry to 24 hours from creation time
    payload[:exp] = exp.to_i
    # sign token with application secret
    JWT.encode(payload, HMAC_SECRET)
  end

  def self.decode(token)
    # get payload; first index in decoded Array
    body = JWT.decode(token, HMAC_SECRET)[0]
    HashWithIndifferentAccess.new body
    # rescue from expiry exception
  rescue JWT::ExpiredSignature, JWT::VerificationError => e
    # raise custom error to be handled by custom handler
    raise ExceptionHandler::ExpiredSignature, e.message
  end
end
{% endhighlight %}

这是标准写法😄，包含 encode 和 decode 两个方法。encode 方法负责生产 token，使用 payload，这里是 user id，还有确定失效时间。因为每个 Rails 应用都有一个唯一的 secret key, 干脆就依据它来产生 token。能看到 decode 是一个相反的过程，使用了相同的 secret key。如果token过期或者无法 decode，那么所有错误我们在 Exception Handler 中统一处理

{% highlight ruby %}
module ExceptionHandler
  # provides the more graceful `included` method
  extend ActiveSupport::Concern

  # Define custom error subclasses - rescue catches `StandardErrors`
  class AuthenticationError < StandardError; end
  class MissingToken < StandardError; end
  class InvalidToken < StandardError; end
  class ExpiredSignature < StandardError; end

  included do
    # Define custom handlers
    rescue_from ActiveRecord::RecordInvalid, with: :four_twenty_two
    rescue_from ExceptionHandler::AuthenticationError, with: :unauthorized_request
    rescue_from ExceptionHandler::MissingToken, with: :four_twenty_two
    rescue_from ExceptionHandler::InvalidToken, with: :four_twenty_two
    rescue_from ExceptionHandler::ExpiredSignature, with: :unauthorized_request

    rescue_from ActiveRecord::RecordNotFound do |e|
      json_response({ message: e.message }, :not_found)
    end
  end

  private
  # JSON response with message; Status code 422 - unprocessable entity
  def four_twenty_two(e)
    json_response({ message: e.message }, :unprocessable_entity)
  end

  # JSON response with message; Status code 401 - Unauthorized
  def unauthorized_request(e)
    json_response({ message: e.message }, :unauthorized)
  end

end
{% endhighlight %}

## 每一个Request的认证

用户认证以后，每次请求当中都会带一个token表示当前用户。现在创建认证服务

{% highlight shell %}
mkdir app/auth && touch app/auth/authorize_api_request.rb
mkdir spec/auth && touch spec/auth/authorize_api_request_spec.rb
{% endhighlight %}

编辑 spec/auth/authorize_api_request_spec.rb

{% highlight shell %}
require 'rails_helper'

RSpec.describe AuthorizeApiRequest do
  # Create test user
  let(:user) { create(:user) }
  # Mock `Authorization` header
  let(:header) { { 'Authorization' => token_generator(user.id) } }
  # Invalid request subject
  subject(:invalid_request_obj) { described_class.new({}) }
  # Valid request subject
  subject(:request_obj) { described_class.new(header) }

  # Test Suite for AuthorizeApiRequest#call
  # This is our entry point into the service class
  describe '#call' do
    # returns user object when request is valid
    context 'when valid request' do
      it 'returns user object' do
        result = request_obj.call
        expect(result[:user]).to eq(user)
      end
    end

    # returns error message when invalid request
    context 'when invalid request' do
      context 'when missing token' do
        it 'raises a MissingToken error' do
          expect { invalid_request_obj.call }
            .to raise_error(ExceptionHandler::MissingToken, 'Missing token')
        end
      end

      context 'when invalid token' do
        subject(:invalid_request_obj) do
          # custom helper method `token_generator`
          described_class.new('Authorization' => token_generator(5))
        end

        it 'raises an InvalidToken error' do
          expect { invalid_request_obj.call }
            .to raise_error(ExceptionHandler::InvalidToken, /Invalid token/)
        end
      end

      context 'when token is expired' do
        let(:header) { { 'Authorization' => expired_token_generator(user.id) } }
        subject(:request_obj) { described_class.new(header) }

        it 'raises ExceptionHandler::ExpiredSignature error' do
          expect { request_obj.call }
            .to raise_error(
              ExceptionHandler::InvalidToken,
              /Signature has expired/
            )
        end
      end
    end
  end
end
{% endhighlight %}

简单解释下，call 方法是入口，先要一个认证的用户，登陆也好，注册也好，结果就是有一个用户被 server 认证了。看到两个方法还没有被定义，token_generator 和 expired_token_generator

{% highlight shell %}
touch spec/support/controller_spec_helper.rb
{% endhighlight %}

创建并编辑它

{% highlight ruby %}
module ControllerSpecHelper
  # generate tokens from user id
  def token_generator(user_id)
    JsonWebToken.encode(user_id: user_id)
  end

  # generate expired tokens from user id
  def expired_token_generator(user_id)
    JsonWebToken.encode({ user_id: user_id }, (Time.now.to_i - 10))
  end

  # return valid headers
  def valid_headers
    {
      "Authorization" => token_generator(user.id),
      "Content-Type" => "application/json"
    }
  end

  # return invalid headers
  def invalid_headers
    {
      "Authorization" => nil,
      "Content-Type" => "application/json"
    }
  end
end
{% endhighlight %}

这太明显了，看字面意思就是了，不多解释。然后要把这个 module 包含在 rails helper 中，我们才能使用。编辑 rails_helper.rb，全局有效，不只是 request

{% highlight ruby %}
  config.include FactoryGirl::Syntax::Methods  
  config.include RequestSpecHelper
  config.include ControllerSpecHelper
{% endhighlight %}

接下去，编辑 app/auth/authorize_api_request.rb，生成对应的路由和方法

{% highlight ruby %}
class AuthorizeApiRequest
  def initialize(headers = {})
    @headers = headers
  end

  # Service entry point - return valid user object
  def call
    {
      user: user
    }
  end

  private

  attr_reader :headers

  def user
    # check if user is in the database
    # memoize user object
    @user ||= User.find(decoded_auth_token[:user_id]) if decoded_auth_token
    # handle user not found
  rescue ActiveRecord::RecordNotFound => e
    # raise custom error
    raise(
      ExceptionHandler::InvalidToken,
      ("#{Message.invalid_token} #{e.message}")
    )
  end

  # decode authentication token
  def decoded_auth_token
    @decoded_auth_token ||= JsonWebToken.decode(http_auth_header)
  end

  # check for token in `Authorization` header
  def http_auth_header
    if headers['Authorization'].present?
      return headers['Authorization'].split(' ').last
    end
      raise(ExceptionHandler::MissingToken, Message.missing_token)
  end
end
{% endhighlight %}

看到 Message.missing_token 对吧，我们对所有的详消息做一个封装，app/lib/message.rb

{% highlight ruby %}
class Message
  def self.not_found(record = 'record')
    "Sorry, #{record} not found."
  end

  def self.invalid_credentials
    'Invalid credentials'
  end

  def self.invalid_token
    'Invalid token'
  end

  def self.missing_token
    'Missing token'
  end

  def self.unauthorized
    'Unauthorized request'
  end

  def self.account_created
    'Account created successfully'
  end

  def self.account_not_created
    'Account could not be created'
  end

  def self.expired_token
    'Sorry, your token has expired. Please login to continue.'
  end
end
{% endhighlight %}

## 认证用户

就是登陆注册的功能

{% highlight shell %}
touch app/auth/authenticate_user.rb
touch spec/auth/authenticate_user_spec.rb
{% endhighlight %}

先写测试，看到 authenticate_user_spec.rb

{% highlight ruby %}
require 'rails_helper'

RSpec.describe AuthenticateUser do
  # create test user
  let(:user) { create(:user) }
  # valid request subject
  subject(:valid_auth_obj) { described_class.new(user.email, user.password) }
  # invalid request subject
  subject(:invalid_auth_obj) { described_class.new('foo', 'bar') }

  # Test suite for AuthenticateUser#call
  describe '#call' do
    # return token when valid request
    context 'when valid credentials' do
      it 'returns an auth token' do
        token = valid_auth_obj.call
        expect(token).not_to be_nil
      end
    end

    # raise Authentication Error when invalid request
    context 'when invalid credentials' do
      it 'raises an authentication error' do
        expect { invalid_auth_obj.call }
          .to raise_error(
            ExceptionHandler::AuthenticationError,
            /Invalid credentials/
          )
      end
    end
  end
end
{% endhighlight %}

AuthenticateUser 入口是 call 方法，注释已经说明一切，编辑 authenticate_user.rb

{% highlight ruby %}
class AuthenticateUser
  def initialize(email, password)
    @email = email
    @password = password
  end

  # Service entry point
  def call
    JsonWebToken.encode(user_id: user.id) if user
  end

  private

  attr_reader :email, :password

  # verify user credentials
  def user
    user = User.find_by(email: email)
    return user if user && user.authenticate(password)
    # raise Authentication error if credentials are invalid
    raise(ExceptionHandler::AuthenticationError, Message.invalid_credentials)
  end
end
{% endhighlight %}

收到 email 和 password，检查如果是有效的就返回一个依据 userid 产生的 token。测试下

{% highlight shell %}
bundle exec rspec spec/auth -fd
{% endhighlight %}

应该都是通过的

## 认证流程

要建立controller

{% highlight shell %}
rails g controller Authentication
{% endhighlight %}

先写测试，看到 spec/requests/authentication_spec.rb

{% highlight ruby %}
require 'rails_helper'

RSpec.describe 'Authentication', type: :request do
  # Authentication test suite
  describe 'POST /auth/login' do
    # create test user
    let!(:user) { create(:user) }
    # set headers for authorization
    let(:headers) { valid_headers.except('Authorization') }
    # set test valid and invalid credentials
    let(:valid_credentials) do
      {
        email: user.email,
        password: user.password
      }.to_json
    end
    let(:invalid_credentials) do
      {
        email: Faker::Internet.email,
        password: Faker::Internet.password
      }.to_json
    end

    # set request.headers to our custon headers
    # before { allow(request).to receive(:headers).and_return(headers) }

    # returns auth token when request is valid
    context 'When request is valid' do
      before { post '/auth/login', params: valid_credentials, headers: headers }

      it 'returns an authentication token' do
        expect(json['auth_token']).not_to be_nil
      end
    end

    # returns failure message when request is invalid
    context 'When request is invalid' do
      before { post '/auth/login', params: invalid_credentials, headers: headers }

      it 'returns a failure message' do
        expect(json['message']).to match(/Invalid credentials/)
      end
    end
  end
end
{% endhighlight %}

编辑对应的控制器，app/controllers/authentication_controller.rb

{% highlight ruby %}
class AuthenticationController < ApplicationController
  # return auth token once user is authenticated
  def authenticate
    auth_token =
      AuthenticateUser.new(auth_params[:email], auth_params[:password]).call
    json_response(auth_token: auth_token)
  end

  private

  def auth_params
    params.permit(:email, :password)
  end
end
{% endhighlight %}

最后增加路径，编辑 config/routes.rb

{% highlight ruby %}
post 'auth/login', to: 'authentication#authenticate'
{% endhighlight %}

在认证用户之前，这个用户总得注册对吧，这属于 User 的行为，放到 User Controller 中解决，建立控制器和对应的 request 测试文件

{% highlight shell %}
rails g controller Users
touch spec/requests/users_spec.rb
{% endhighlight %}

还是先写测试，看到spec/requests/users_spec.rb

{% highlight ruby %}
require 'rails_helper'

RSpec.describe 'Users API', type: :request do
  let(:user) { build(:user) }
  let(:headers) { valid_headers.except('Authorization') }
  let(:valid_attributes) do
    attributes_for(:user, password_confirmation: user.password)
  end

  # User signup test suite
  describe 'POST /signup' do
    context 'when valid request' do
      before { post '/signup', params: valid_attributes.to_json, headers: headers }

      it 'creates a new user' do
        expect(response).to have_http_status(201)
      end

      it 'returns success message' do
        expect(json['message']).to match(/Account created successfully/)
      end

      it 'returns an authentication token' do
        expect(json['auth_token']).not_to be_nil
      end
    end

    context 'when invalid request' do
      before { post '/signup', params: {}, headers: headers }

      it 'does not create a new user' do
        expect(response).to have_http_status(422)
      end

      it 'returns failure message' do
        expect(json['message'])
          .to match(/Validation failed: Password can't be blank, Name can't be blank, Email can't be blank, Password digest can't be blank/)
      end
    end
  end
end
{% endhighlight %}

编辑路由，config/routes.rb，增加

{% highlight ruby %}
  post 'signup', to: 'users#create'
{% endhighlight %}

编辑控制器

{% highlight ruby %}
class UsersController < ApplicationController
  # POST /signup
  # return authenticated token upon signup
  def create
    user = User.create!(user_params)
    auth_token = AuthenticateUser.new(user.email, user.password).call
    response = { message: Message.account_created, auth_token: auth_token }
    json_response(response, :created)
  end

  private

  def user_params
    params.permit(
      :name,
      :email,
      :password,
      :password_confirmation
    )
  end
end
{% endhighlight %}

嗯，还差一件事情，就是直接访问的 API 的时候，必须要有 token，在基类控制器中可以做一个全局的限制。先写测试，spec/controllers/application_controller_spec.rb

{% highlight ruby %}
require "rails_helper"

RSpec.describe ApplicationController, type: :controller do
  # create test user
  let!(:user) { create(:user) }
   # set headers for authorization
  let(:headers) { { 'Authorization' => token_generator(user.id) } }
  let(:invalid_headers) { { 'Authorization' => nil } }

  describe "#authorize_request" do
    context "when auth token is passed" do
      before { allow(request).to receive(:headers).and_return(headers) }

      # private method authorize_request returns current user
      it "sets the current user" do
        expect(subject.instance_eval { authorize_request }).to eq(user)
      end
    end

    context "when auth token is not passed" do
      before do
        allow(request).to receive(:headers).and_return(invalid_headers)
      end

      it "raises MissingToken error" do
        expect { subject.instance_eval { authorize_request } }.
          to raise_error(ExceptionHandler::MissingToken, /Missing token/)
      end
    end
  end
end
{% endhighlight %}

修改控制器基类，app/controllers/application_controller.rb

{% highlight ruby %}
class ApplicationController < ActionController::API
  include Response
  include ExceptionHandler

  # called before every action on controllers
  before_action :authorize_request
  attr_reader :current_user

  private

  # Check for valid request token and return user
  def authorize_request
    @current_user = (AuthorizeApiRequest.new(request.headers).call)[:user]
  end
end
{% endhighlight %}

现在是对所有的 request 做限制，但是我在注册和登录操作中是没有 token 的，所以再加一句话

{% highlight ruby %}
class AuthenticationController < ApplicationController
  skip_before_action :authorize_request, only: :authenticate
  # [...]
end
{% endhighlight %}

还有 user controller，app/controllers/users_controller.rb

{% highlight ruby %}
class UsersController < ApplicationController
  skip_before_action :authorize_request, only: :create
  # [...]
end
{% endhighlight %}

如果现在运行测试，肯定不通过，要更新测试用例，现在加了认证，编辑spec/requests/todos_spec.rb

{% highlight ruby %}
require 'rails_helper'

RSpec.describe 'Todos API', type: :request do
  # add todos owner
  let(:user) { create(:user) }
  let!(:todos) { create_list(:todo, 10, created_by: user.id) }
  let(:todo_id) { todos.first.id }
  # authorize request
  let(:headers) { valid_headers }

  describe 'GET /todos' do
    # update request with headers
    before { get '/todos', params: {}, headers: headers }

    # [...]
  end

  describe 'GET /todos/:id' do
    before { get "/todos/#{todo_id}", params: {}, headers: headers }
    # [...]
    end
    # [...]
  end

  describe 'POST /todos' do
    let(:valid_attributes) do
      # send json payload
      { title: 'Learn Elm', created_by: user.id.to_s }.to_json
    end

    context 'when request is valid' do
      before { post '/todos', params: valid_attributes, headers: headers }
      # [...]
    end

    context 'when request is invalid' do
      let(:valid_attributes) { { title: nil }.to_json }
      before { post '/todos', params: valid_attributes, headers: headers }
      # [...]
    end
  end

  describe 'PUT /todos/:id' do
    let(:valid_attributes) { { title: 'Shopping' }.to_json }

    context 'when the record exists' do
      before { put "/todos/#{todo_id}", params: valid_attributes, headers: headers }
      # [...]
    end
  end

  describe 'DELETE /todos/:id' do
    before { delete "/todos/#{todo_id}", params: {}, headers: headers }
    # [...]
  end
end
{% endhighlight %}

还要更新 todos_controller.rb，增加 user 和 todo 的关联

{% highlight ruby %}
class TodosController < ApplicationController
  # [...]
  # GET /todos
  def index
    # get current user todos
    @todos = current_user.todos
    json_response(@todos)
  end
  # [...]
  # POST /todos
  def create
    # create todos belonging to current user
    @todo = current_user.todos.create!(todo_params)
    json_response(@todo, :created)
  end
  # [...]
  private

  # remove `created_by` from list of permitted parameters
  def todo_params
    params.permit(:title)
  end
  # [...]
end
{% endhighlight %}

同样的，我们更新 spec/requests/items_spec.rb

{% highlight ruby %}
require 'rails_helper'

RSpec.describe 'Items API' do
  let(:user) { create(:user) }
  let!(:todo) { create(:todo, created_by: user.id) }
  let!(:items) { create_list(:item, 20, todo_id: todo.id) }
  let(:todo_id) { todo.id }
  let(:id) { items.first.id }
  let(:headers) { valid_headers }

  describe 'GET /todos/:todo_id/items' do
    before { get "/todos/#{todo_id}/items", params: {}, headers: headers }

    # [...]
  end

  describe 'GET /todos/:todo_id/items/:id' do
    before { get "/todos/#{todo_id}/items/#{id}", params: {}, headers: headers }

    # [...]
  end

  describe 'POST /todos/:todo_id/items' do
    let(:valid_attributes) { { name: 'Visit Narnia', done: false }.to_json }

    context 'when request attributes are valid' do
      before do
        post "/todos/#{todo_id}/items", params: valid_attributes, headers: headers
      end

      # [...]
    end

    context 'when an invalid request' do
      before { post "/todos/#{todo_id}/items", params: {}, headers: headers }

      # [...]
    end
  end

  describe 'PUT /todos/:todo_id/items/:id' do
    let(:valid_attributes) { { name: 'Mozart' }.to_json }

    before do
      put "/todos/#{todo_id}/items/#{id}", params: valid_attributes, headers: headers
    end

    # [...]
    # [...]
  end

  describe 'DELETE /todos/:id' do
    before { delete "/todos/#{todo_id}/items/#{id}", params: {}, headers: headers }

    # [...]
  end
end
{% endhighlight %}

现在运行测试用例，应该是全部通过的，如果有异常，看提示自行处理。没问题的话，自己在本地用 Postman 或者 httpie 手动测试下