---
layout: post
title: "在 Mac 上为 GitHub 创建 SSH Key"
date: 2015-01-24 17:49:55 +0200
categories: jekyll update
---

接下去的步骤会告诉你如何去创建SSH key并且添加到GitHub的账户当中

第一步：首先，我们需要在本地检查已经存在的SSH keys
{% highlight shell %}
ls -al ~/.ssh
{% endhighlight %}
会列出所有已经拥有的SSH keys，它可能会是这个样子
{% highlight shell %}
drwx------   5 hongliyuele  staff   170  1 14 15:14 .
drwxr-xr-x+ 31 hongliyuele  staff  1054  1 24 13:56 ..
-rw-------   1 hongliyuele  staff  1675  1 14 15:04 id_rsa
-rw-r--r--   1 hongliyuele  staff   413  1 14 15:04 id_rsa.pub
-rw-r--r--   1 hongliyuele  staff  1595  1 14 17:24 known_hosts
{% endhighlight %}
如果你的mac是全新的，应该什么都没有

第二步：去创建一个新的SSH key，复制粘贴下面这段内容，然后执行，确认你的email是GitHub上的注册邮箱
{% highlight shell %}
ssh-keygen -t rsa -C "your_email@example.com"
{% endhighlight %}
然后会看到
{% highlight shell %}
Enter passphrase (empty for no passphrase): [Type a passphrase]
Enter same passphrase again: [Type passphrase again]
{% endhighlight %}
连续按回车执行默认命令，最后得到输出
{% highlight shell %}
Your identification has been saved in /Users/you/.ssh/id_rsa.
# Your public key has been saved in /Users/you/.ssh/id_rsa.pub.
# The key fingerprint is:
# 01:0f:f4:3b:ca:85:d6:17:a1:7d:f0:68:9d:f0:a2:db your_email@example.com
{% endhighlight %}
然后把key添加到ssh-agent，
查看ssh-agent
{% highlight shell %}
eval "$(ssh-agent -s)"
{% endhighlight %}
添加key
{% highlight shell %}
ssh-add ~/.ssh/id_rsa
{% endhighlight %}
然后
{% highlight shell %}
cat ~/.ssh/id_rsa.pub
{% endhighlight %}
复制之后出来的那一大段，登录GitHub, Setting页面的SSH keys，点击Add SSH Key，Title随便自己能识别就可以，把刚才复制的粘贴到Key，最后点击Add key搞定。
原文地址: [generating-ssh-keys][[generating-ssh-keys]]

[generating-ssh-keys]: https://help.github.com/articles/generating-ssh-keys/
