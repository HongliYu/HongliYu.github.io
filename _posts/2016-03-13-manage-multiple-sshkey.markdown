---
layout: post
title: "在 Mac 上管理多个 SSH Key"
date: 2016-03-13 17:49:55 +0200
categories: jekyll update
---
SSH Key主要是用来绑定当前使用的设备和Server，防止没有权限的人在没有许可的设备上对代码进行操作。之前讲过怎么添加SSH Key，现在讲多个SSH Key的管理。
先看到用户目录下的隐藏文件：
{% highlight shell %}
cd ~/.ssh
{% endhighlight %}
pub后缀的是公钥，没有后缀的是私钥，公钥要放到Server上，属于非对称加密
看下已经添加的密钥
{% highlight shell %}
ssh-add -l
{% endhighlight %}
让我们从头开始，删除私钥列表中的所有已经添加条目。放心，目录下的文件都还在，只是删除了他们的引用
{% highlight shell %}
ssh-add -D
{% endhighlight %}
这里我们假设id_rsa对应的是github上的私钥，现在要在加另外一个的ssh key
{% highlight shell %}
ssh-keygen -t rsa -C "yuhongli"
{% endhighlight %}
回车，选择文件路径：~/.ssh/id_rsa_baidu，连续回车直到结束
{% highlight shell %}
cat ~/.ssh/id_rsa_baidu.pub
{% endhighlight %}
复制所有内容，也就是SSH key到Server，然后配置config文件
{% highlight shell %}
subl ~/.ssh/config
{% endhighlight %}
添加以下内容
{% highlight shell %}
Host github.com
    HostName github.com
    PreferredAuthentications publickey
    IdentityFile ~/.ssh/id_rsa
Host icode.baidu.com
    HostName icode.baidu.com
    PreferredAuthentications publickey
    IdentityFile ~/.ssh/id_rsa_baidu
{% endhighlight %}
然后保存退出，以这个格式为模版可以添加更多。测试是否配置成功
{% highlight shell %}
ssh -T git@github.com
ssh -T ssh yuhongli@icode.baidu.com -p 端口号
{% endhighlight %}


