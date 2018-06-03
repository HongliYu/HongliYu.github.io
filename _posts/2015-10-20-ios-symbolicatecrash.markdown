---
layout: post
title: "iOS 命令行符号化 crash 文件"
date: 2015-10-20 17:49:55 +0200
categories: jekyll update
---

打开命令行输入：
{% highlight shell %}
find /Applications/Xcode.app -name symbolicatecrash -type f
{% endhighlight %}
找到symbolicatecrash工具的路径，可能会花去一些时间，OSX 10.10.5 安装xcode7下的结果是：
{% highlight shell %}
/Applications/Xcode.app/Contents/SharedFrameworks/DTDeviceKitBase.framework/Versions/A/Resources/symbolicatecrash
{% endhighlight %}
遇到Error: “DEVELOPER_DIR” is not defined，处理：
{% highlight shell %}
export DEVELOPER_DIR="/Applications/XCode.app/Contents/Developer"
{% endhighlight %}
把.ipa改名.zip解压以后得到.app文件，.app.dSYM文件，对应的crash文件方法到一个目录下，运行：
{% highlight shell %}
/Applications/Xcode.app/Contents/SharedFrameworks/DTDeviceKitBase.framework/Versions/A/Resources/symbolicatecrash BDPhoneBrowser.crash BDPhoneBrowser.app.dSYM > crashlog.log
{% endhighlight %}
符号化到当前目录的crashlog.log文件中
配置符号化工具到命令行：
{% highlight shell %}
sudo ln -s /Applications/Xcode.app/Contents/SharedFrameworks/DTDeviceKitBase.framework/Versions/A/Resources/symbolicatecrash /usr/bin/symbolicatecrash
{% endhighlight %}
以后这么用：
{% highlight shell %}
symbolicatecrash BDPhoneBrowser.crash BDPhoneBrowser.app.dSYM > crashlog.log
{% endhighlight %}
最后方便大家使用，我用gem写了命令行工具，[crash_analysis](https://rubygems.org/gems/crash_analysis)，打开终端运行
{% highlight shell %}
gem install crash_analysis
{% endhighlight %}
