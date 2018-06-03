---
layout: post
title: "MVVM 与迸发架构中的 Realm"
date: 2017-01-12 17:49:55 +0200
categories: jekyll update
---

Realm是为移动端打造的数据库，面向对象的数据库，重写了存储引擎，告别ROM，纯粹原生的对象存储。[介绍](https://realm.io/cn/news/realm-object-centric-present-day-database-mobile-applications/) 划下重点：1. 数据层库和 ORMs 2. Realm 不是重复现在的技术

Realm 移动数据库的一个标志性特性：你可以在主线程里面读写！你不用担心这会阻塞你的 UI。但我原来的结构是并发的，数据的增删改查都是异步的，怎么接入是个问题。__underscores__（以这并不是一个最佳实践，只是临时的处理方式）

首先介绍下当前的MVVM是这样的结构：

## Model
先是 RawModel, 这是 Server 传输给我的 JSON 数据的映射产生的 Model，最纯粹，最原始的数据，没有任何加工，但我不会在数据库中去存储这个Model

## ViewModel
它是和View的显示是直接相关联的，view是容器，我们初始化容器以后，容器怎么呈现给用户，完全依赖于注入的ViewModel。典型的比如UITableView的cell，只通过一个方法bindData方法绑定ViewModel完成渲染。ViewModel中包含一个RawModel的property，所以他们之间的关系是组合关系，当然也可以是继承，方便省去访问RawModel的属性的路由代码，比如viewmodel.model.property需要简写成viewmodel.property，那么需要ViewModel内部写一个属性映射，类似于Rails中的路由操作，组合还是继承看具体模块情况而定，没有定式。ViewModel还包含，用户前端操作需要持久化的一些属性，比如cell中的button按下以后，状态改变，在重启app以后改变还是有效的，那么这个改变需要持久化，所以我选择在db中存储的是ViewModel，
包含Server的数据和Client用户的定制数据，在Sqlite3中以ViewModel为基础做table，或者在OO的数据库中，以ViewModel为基础建立持久化对象。然后VC之间切换，修改ViewModel中的属性，传统方法是KVO，view响应式更新，Swift没有现成的KVO，需要Object继承NSObject，所以建议走系统API，获取那个cell，并重新绑定数据

## Manager
MainManager 是个 Singleton，其中包含类似于 UITableView 中的数据源，比如一个 Array。MainManager 会完成数据和 view 绑定之前最后内存中的操作，比如filter，按照条件过滤数据，还有各种 action 以后会回掉给其他 VC 的 block，影响下一次数据请求的临时变量，以后 Model 要是有改动也必须是调用 Manager 中的方法，记住是数据展现前的最后内存中的操作。另外，进入或者退出某个模块的数据初始化操作。MainManager有一个 property 是 DataManager，他负责所有的数据存取，以及通过不同渠道拿到数据以后的 merge，会有这些 properties：DBRoot，NetworkService，FileManager，DispatchQueue，包含的意义是数据的来源，disk 上的db或者 file system，还是 network，有自己的 GCD 管理的并发队列，只要在 IDE 的断点看到这个队列，那么就是在这个 DataManager 的操作。其实对于上层 MainManager 来说，他只是请求数据，并不关心来源，需要制定来源可以在 methd 中增加参数，比如第一次进入VC的时候数据请求的顺序是db，select -> present data -> network request -> present data -> db insert，但是如果是当前VC的 refresh 操作就不需要请求数据库对吧，这是不一样的

## ViewController
每次进入VC会做这么几件事情，basicUI，requestBasicData，bindActions，registNotifications，遇到业务复杂的核心VC，actions会特别多，建议分离到business业务层, 所以又多一个操作叫 bindBusiness，记得给一个 weak reference 解决 retain cycle的问题

![integrate-realm-to-mvvm-swift-0]({{ "/assets/integrate-realm-to-mvvm-swift-0.png" | absolute_url }})

![integrate-realm-to-mvvm-swift-1]({{ "/assets/integrate-realm-to-mvvm-swift-1.png" | absolute_url }})

## DBRoot
Realm 对象只能通过从最初创建的他的那个线程访问，增删改查的所有操作都必须在这个线程上，因为本来设计是在主线程上，而我的框架是并发的😓。那么，该怎么办？涉及到一个小知识，就是runloop，子线程存活，一个 while true 的循环，等待响应，虽然在 DataManager 的并发的队列上，但是 realm 的线程只有一个，直到退出模块，销毁所有内容。然后那个 NSCondition 对象会保证，添加到 task pool 中的 block 顺序进行不会有脏数据产生。还有要注意的一点是，realm 对象得到的 collection 是引用是 lazy init，所以对 collection 的直接操作必须还是在 realm 线程上的，尽管可能已经抛出 result 到 DataManager 这一层，注意当前代码所在的队列和线程，好在如果操作不在 realm 线程上，assert一个断言立刻crash，及时修复错误。
道理都懂，看DBRoot.swift的代码: [DBRoot](https://gist.github.com/anonymous/0e5e60bffc677044ab84a1994be0e90d)
