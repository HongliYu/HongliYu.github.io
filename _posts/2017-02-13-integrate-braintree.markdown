---
layout: post
title: "iOS 集成 Braintree, 支持信用卡, PayPal 支付(sandbox)"
date: 2017-02-13 17:49:55 +0200
categories: jekyll update
---

如果需求是客户端只支持PayPal，那么需要import PayPal Mobile SDKs，但在v.zero以后[link](https://developer.paypal.com/docs/bt-vzero-overview/),可以集成多种支付，不需要分别import各自的SDK，就像是统一了行业标准，对开发者来说是件好事情，而且如果使用drop-in SDK自带的UI，开发起来会很省心，不需要自己写界面，一个底部弹出框包含所有支付内容。
看到[官方文档](https://developers.braintreepayments.com/start/overview)，首先先讲两个重要的点：1. Client token，2. Payment method nonce

## Client token
是服务端通过 Braintree 的 SDK 生成的，比如我要用 Rails5 来开发服务端，那么我需要的 SDK 是(gem “braintree”, “~> 2.72.0”)，用它来生成token
这三个参数是生成token的依据，当你注册一个新的账户（https://www.braintreepayments.com/en-ca/sandbox）
给下截图，Account->My User->View Authorizations->API Keys->view，注意那三个参数，mark 下 server 会用到

![braintree-gateway-0]({{ "/assets/braintree-gateway-0.png" | absolute_url }})

![braintree-gateway-1]({{ "/assets/braintree-gateway-1.png" | absolute_url }})

![braintree-gateway-2]({{ "/assets/braintree-gateway-2.png" | absolute_url }})

Rails配置代码：

{% highlight ruby %}
Braintree::Configuration.environment = :sandbox
Braintree::Configuration.merchant_id = "use_your_merchant_id"
Braintree::Configuration.public_key = "use_your_public_key"
Braintree::Configuration.private_key = "use_your_private_key"
{% endhighlight %}

记下以后继续，然后客户端会依据这个 client token 直接和 Braintree 的服务器通信，所以这是一个认证令牌（These should not be reused; a new client token should be generated for each request that’s sent to Braintree.）而且为了安全，每次客户端和 Braintree 服务器的直接通信都需要新的 token，对应于每一个支付 request。就像 APNs 当中，注册 token，然后拿着 token 直接和 apple 服务器通信，一个道理，只不过一个是OS级的，一个是第三方服务，OS级的无非是依赖于Apple ID这个统一的账户系统，本质是一样的

## Payment method nonce
直接翻译叫做：支付方法随机数。其实就是一个字符串，代表一次支付方法的调用，然后服务端，获取这个字符串，通过 Server 的 SDK，对 Braintree服务器发起新的事务请求
这个东西其实在客户端的 callback 方法中有，是 Braintree 服务器直接下发的，我们的服务器不管这个类型。附上解释原理的那张经典图片：

![braintree-principle]({{ "/assets/braintree-principle.png" | absolute_url }})

看下overview的文档，图片下的5个steps，讲得很清楚
以上原理，现在到实践

## 信用卡支付
我们现在简单写个服务器，用 Rails5, 这个是崭新的 Rails 版本，之前这个架构只在 Web App 的开发中有优势，现在默认继承了 gem: rails-api 写 JSON 接口

环境：Ruby-2.3.3, Rails-5.0.1

cd 到工程目录，rails new braintree_api –api -T, 说明：–api 告诉 Rails 我只要 API 应用没有界面元素，-T 就是不要使用 Minitest 默认的测试工具，正常情况下会使用 RSpec 来替代测试，但我这里连 model 都没有，更没有涉及到数据库，直接在内存中执行，测试就不写了

更新gem file:
 
{% highlight ruby %}
source 'https://rubygems.org'

ruby '2.3.3'

git_source(:github) do |repo_name|
  repo_name = "#{repo_name}/#{repo_name}" unless repo_name.include?("/")
  "https://github.com/#{repo_name}.git"
end

gem 'rails', '~> 5.0.1'
gem 'puma', '~> 3.0'
gem "braintree", '~> 2.72.0'

group :development, :test do
  gem 'sqlite3', '1.3.12'
  gem 'byebug', platform: :mri
end

group :development do
  gem 'listen', '~> 3.0.5'
  gem 'spring'
  gem 'spring-watcher-listen', '~> 2.0.0'
end

group :production do
  gem 'pg', '0.18.4'
end

gem 'tzinfo-data', platforms: [:mingw, :mswin, :x64_mingw, :jruby]
{% endhighlight %}
完了 bundle install

touch /app/controllers/concerns/response.rb：

{% highlight ruby %}
module Response

  def json_response(object, status = :ok)
    render json: object, status: status
  end
  
end
{% endhighlight %}

application_controller.rb：
{% highlight ruby %}
class ApplicationController < ActionController::API
  include Response
end
{% endhighlight %}

rails g controller BraintreeAction

touch config/initializers/braintree.rb, 配置文件，写入刚才的三个参数

config/routes.rb

{% highlight ruby %}
Rails.application.routes.draw do
  # For details on the DSL available within this file, see http://guides.rubyonrails.org/routing.html
    root 'braintree_action#home'
    get '/client_token', to: 'braintree_action#client_token'
    post '/checkout', to: 'braintree_action#checkout'
    post '/paypal_checkout', to: 'braintree_action#paypal_checkout'
end
{% endhighlight %}

验证是否有效，我这里用httpie，你用curl也行，brew install httpie, rails s 开启服务，另一个终端界面http get :3000/client_token，看图片这是对的情况：

![braintree-terminal-token]({{ "/assets/braintree-terminal-token.png" | absolute_url }})

后面的checkout验证需要客户端的配合

我们开始写客户端。使用cocoapods，这是Podfile

{% highlight ruby %}
def main_pods
  pod 'BraintreeDropIn'
  pod 'Braintree/PayPal'
  pod 'Braintree/Venmo'
  pod 'Braintree/Apple-Pay'
  pod 'Braintree/3D-Secure'
end

target 'PaypalDemo' do
  main_pods
end
{% endhighlight %}

我这里偷懒了，一次都加完，照理说是用什么加什么。完了 pod install，继续，打开新的 xcworkspace，开始配置，都写在VC里面了，[gist链接](https://gist.github.com/anonymous/e0825fd399a57bbc518f5eb9d7c949f1)

流程是这样的，先拿client token->show dropin UI->post nonce to server, （还有另外一种是走 TokenizationKey，等下讲）注意一点，测试环境的测试账户是固定的，不要自己造，无效！[文档](https://developers.braintreepayments.com/reference/general/testing/ruby)，我们就用4111111111111111这张visa卡片吧，nonce 是"fake-valid-nonce"作为常量写在客户端，正常情况下应该是 result.paymentMethod 中的返回结果

打开浏览器，登陆到沙河环境下的 Braintree 控制台，Documents->Merchant Statement，等下我们支付完成以后刷新这个页面，看看是否有变化，我这里是服务端给出常量交易额度为10美刀，成功以后会看到这样的变化，这是我们想要的结果，现在还没配置完成，先预览下：

![braintree-settlement-outcome-0]({{ "/assets/braintree-settlement-outcome-0.png" | absolute_url }})

![braintree-settlement-outcome-1]({{ "/assets/braintree-settlement-outcome-1.png" | absolute_url }})

再回到服务端，要开启SSL，也就是https，所以我们找到 config/environments/production.rb, 开启 config.force_ssl = true，
然后记得推到 heroku 的线上环境才是有效的，正常情况下我们的本地仓库对应两个远程仓库，一个 origin 我放在 bitbucket 上免费私有的git服务，一个直接推送到 heroku，我这里假设 heroku 的本地环境都已经配置好了

cd到工程目录，heroku create，git add -A，git commit -m “init repo”，git push origin master, git push heroku master，部署到线上环境：

![braintree-heroku-0]({{ "/assets/braintree-heroku-0.png" | absolute_url }})

![braintree-heroku-1]({{ "/assets/braintree-heroku-1.png" | absolute_url }})

看到最后https就是地址，填充到iOS客户端，替代那个URL(https://cryptic-citadel-99786.herokuapp.com，这是我的地址)我们开始测试吧

等下，貌似还忘记了很重要的一件事情，连接 PayPal 测试账户和 braintree 沙河环境，看到[文档](https://developer.paypal.com/docs/accept-payments/express-checkout/ec-vzero/get-started/) Credentials这一栏。这个是 PayPal 的开发者控制台，首先创建测试用的买家，卖家账户，看图左边Accounts目录下，然后再到 My Apps & Credentials 目录下，Express Checkout via Braintree SDK – Test Accounts 连接 Braintree 和 paypal，回到 Accounts 目录，打开卖家 Profile，看到 API Credentials，看到 app name 里面有 braintree 的测试账户，说明连接成功:

![link-braintree&paypal-testaccount-0]({{ "/assets/link-braintree&paypal-testaccount-0.png" | absolute_url }})

![link-braintree&paypal-testaccount-1]({{ "/assets/link-braintree&paypal-testaccount-1.png" | absolute_url }})

好了，终于可以开始测试了，有点麻烦对吧，我也是这么认为的，希望以后能再简单点，现在的沙河环境，限制也是非常多的

输入4111111111111111，过期时间随便选，这里就测试结算-settlement，刷新交易页面，看下43变成53，加了10美刀，成功了，这就是信用卡支付，其实还有其他状态比如settlement_comfirmed，记得测试用例不能随便写，必须是test页面里面的账户。

![braintree-creditcardpay-0]({{ "/assets/braintree-creditcardpay-0.png" | absolute_url }})

![braintree-creditcardpay-1]({{ "/assets/braintree-creditcardpay-1.png" | absolute_url }})

![braintree-creditcardpay-2]({{ "/assets/braintree-creditcardpay-2.png" | absolute_url }})

控制台看到返回的交易事务的ID，当然这些参数可以自定义

## PayPal支付

首先分清楚两种支付模式 Vault 和 Checkout[文档](https://developers.braintreepayments.com/guides/paypal/overview/ios/v4#vault-vs.-checkout)前者适用于多次支付信息复用，后者一次性支付

这次我们用 TokenizationKey，与 Client Token 不同，这是一个静态的 key，不和自己的 Server 做交互，直接与 Braintree 的服务器通信，不支持较早版本的SDK是一个问题(If you’re using our iOS SDK v4 or Android SDK v2, you may also use a tokenization key instead.)，但是对于客户端来说更加简单了，想一下其实本质和导入 fabric 框架是一样的对吧。退一步说，如果还是要走自己的服务器，那么还是 client token 的流程，服务器的代码看到 paypal_checkout 这个方法，统一作为 Braintree 的 Transaction 处理，正如之前讲的我们的服务端不关心 payment_method_nonce 的具体类型。可以自己测试下，结果也是可行的。

好了，现在走 TokenizationKey 的流程，首先我们要一个 TokenizationKey，在这里打开 braintreegateway->Account-My User->View Authorizations->Tokenization Keys这一栏，没有的创建一个。客户端这边，Podfile 增加

{% highlight ruby %}
pod 'Braintree/PayPal'
pod 'Braintree/Venmo'
pod 'Braintree/Apple-Pay'
pod 'Braintree/3D-Secure'
{% endhighlight %}

如果之前和我一样偷懒全部导入了，请忽略，完了pod install，
再看(https://developers.braintreepayments.com/guides/paypal/client-side/ios/v4)，schema 跳转的配置，自定义按钮，打开 setupPayPalUI 的注释，同时让 fetchClientToken 失效，我这里就是个左上角绿色的按钮，点击以后验证这个账户，看图：

![braintree-paypal-pay]({{ "/assets/braintree-paypal-pay.png" | absolute_url }})

好吧，这个页面是个大坑，写着按我能跳，其实是不能跳转的，不支持端对端测试，更加细节的调试，必须要有生产环境，
看[文档](http://stackoverflow.com/questions/41261073/how-to-test-paypal-payment-using-braintree-sandbox
https://developers.braintreepayments.com/guides/paypal/testing-go-live/ruby#end-to-end-testing)

所以在sandbox环境下，我们最远就是走到这里，沙河环境限制很多，希望以后能更加方便吧

支付流程看上去复杂，本质上是一系列的安全措施，证明我的服务器是我的服务器，和 Braintree 的服务器通信，如果是直接通信那么用 TokenizationKey 更加方便，如果需要走自己的服务，那么 client token 是有必要的，尽量每次 session 都更新，但起码每次应用重启的时候更新一次，对吧

