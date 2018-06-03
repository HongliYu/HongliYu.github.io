---
layout: post
title: "github 上的 fork 开发流程"
date: 2015-03-01 17:49:55 +0200
categories: jekyll update
---

场景1：从一个开源的版本库，fork出来，到自己的repo，然后clone到本地，我在develop分支上开发新功能，新建一个分支feature/XXX，其他人没有提交PR，我提交以后，maintainer批准通过，我在终端git pull upstream develop 这样本地就sync到最新的代码，最后git push origin develop 把本地最新的develop分支推到我自己的origin上的develop分支
完成一次迭代，开始下一轮

场景2：前提同上，我提交以后，有同事做了修改，发了PR，并且已经通过，当我要发PR的时候，upstream上有新的commit，那么我要先切换到本地develop分支，git pull develop upstream从源库拉最新的develop代码到本地，然后切换到feature/XXX分支，接着git rebase develop，重定义feature/XXX分支的base为最新的develop分支，在这里可能会发生rebase冲突，vi到冲突的patch文件，解决冲突，然后git add -u更新文件索引，就回到情况1的状态了

小技巧：fork后，如果需要快速获取和切换到他人的pr的话，在.git/config里的upstream添加:
fetch = +refs/pull/*/head:refs/remotes/upstream/pr/*
git fetch upsteam 后可以直接用命令如: git co upstream/pr/871来切换到他人的pr上了。如果co没有反应，配置 co 映射到 checkout
{% highlight shell %}
git config --global alias.co checkout
{% endhighlight %}