---
layout: post
title: "Mock JSON 数据"
date: 2016-02-29 17:49:55 +0200
categories: jekyll update
---

场景1：服务端和客户端已经制定了JSON数据的格式，但是数据组还没能给出真实数据，客户端的开发走服务端前面

场景2：客户端工程师已经完成story，而服务端数据一直不正常，影响测试，要在后期接口才能ready

这个时候，我们需要mock服务端的JSON数据，方法有很多，比如：

1. 自己在本地架起服务器，手动写入一批JSON数据，在客户端工程中把请求路径改为本地

2. 使用第三方的服务，比如[Mocky](https://www.mocky.io/)，但是如果不能科学上网的话，数据返回很慢

既然mocky是开源的，我们可以在墙内的服务器上搭建

环境是centOS6.5，Linode的主机，ip在新加波，没有用国内的云服务的原因很简单，有些请求拿不到数据，总之就是不能科学上网

最新的代码支持jdk1.8。所以，决定卸载jdk1.7更新到jdk1.8

查看已经安装的jdk：
{% highlight shell %}
yum list installed | grep java
{% endhighlight %}

卸载环境：
{% highlight shell %}
yum -y remove java-1.7.0-openjdk*
{% endhighlight %}

查看yum的java源： 
{% highlight shell %}
yum -y list java*
{% endhighlight %}

安装1.8的jdk：
{% highlight shell %}
yum -y install java-1.8.0-openjdk*
{% endhighlight %}

然后下载PlayFramework2.4.4的包，应为是https，就用curl命令吧

{% highlight shell %}
curl -O https://downloads.typesafe.com/typesafe-activator/1.3.7/typesafe-activator-1.3.7-minimal.zip
{% endhighlight %}

并解压缩

{% highlight shell %}
unzip typesafe-activator-1.3.7-minimal.zip
{% endhighlight %}

然后检查下本机有没有安装git，没有的话，yum install git
接着clone mocky的最新版本到主机上

{% highlight shell %}
git clone https://github.com/studiodev/Mocky.git
{% endhighlight %}

完了以后

{% highlight shell %}
vi Mocky/conf/application.conf
{% endhighlight %}

这里是选择持久化类型，有本地文件系统的，直接存储到云端gist，或者本地mongodb数据库，都行。看说明选一个，我选了用mongdb就配置下

{% highlight shell %}
version=v2
mongodb.uri="mongodb://localhost:27017/mocky1"
{% endhighlight %}

接着，service mongod start 把数据库服务开启
输入mongo进入数据库，use mocky1，exit退出。cd到Mocky目录下

{% highlight shell %}
sudo ../typesafe-activator-1.3.7-minimal/activator
{% endhighlight %}

如果你想把命令配置到usr/bin目录下也行，我不常用就没配置。然后慢慢等，过很长一段时间进入activator的console

{% highlight shell %}
run 端口号
{% endhighlight %}

开启服务，这是开发模式，用浏览器看下是否正常，如果正常，crl+D退出，部署服务

{% highlight shell %}
start 端口号
{% endhighlight %}

这时log会继续记录，可以退出，服务会保持运行的状态

最后记得用charles之类的软件，shift+cmd+m 打开URLmapping功能，把要请求的URL mapping到自己刚刚生成JSON数据的URL，客户端代码不要改就能测试，只要server给的JSON数据是按照约定的，前端工程师的story就可以提测了
