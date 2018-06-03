---
layout: post
title: "使用 Swift 制作命令行工具"
date: 2017-01-12 17:49:55 +0200
categories: jekyll update
---

打开terminal ，执行 xcrun swift 启动 REPL (Read-Eval-Print Loop)环境，类似于Ruby的irb，这是简单的命令行交互式工具：

![swift-repl]({{ "/assets/swift-repl.png" | absolute_url }})

当然也可以把 .swift 文件作为命令行工具的输入，这样里面的代码也会被自动地编译和执行

1. 在.swift 文件最上面加上命令行工具的路径

2. 将文件权限改为可执行

3. 执行这个 .swift 文件

{% highlight ruby %}
subl demo.swift
{% endhighlight %}

编辑文件

{% highlight shell %}
#!/usr/bin/env swift
print("it works")
{% endhighlight %}

修改权限

{% highlight shell %}
chmod 755 demo.swift
./demo.swift
{% endhighlight %}

输出 it works

当然所谓命令行工具，肯定要生成可执行的二进制文件，脱离Xcode环境，用swiftc进行编译即可：
创建文件 Hello.swift，并编辑

{% highlight shell %}
// Hello.swift
class Hello {
  let name = "Veer"
  func hello() {
    print("Hello \(name)")
  }
}
let object = Hello()
object.hello()
{% endhighlight %}

{% highlight shell %}
swiftc Hello.swift
{% endhighlight %}

生成可执行文件 Hello，并移动到bin目录下

{% highlight shell %}
mv ./Hello /usr/local/bin/
{% endhighlight %}

在任何地方输入命令 Hello，都能看到输出 Hello Veer
