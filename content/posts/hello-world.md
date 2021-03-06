---
title: "你好世界！"
subtitle: "Hello, World!"
date: 2020-07-10T15:12:25+08:00
draft: false
author: ""
authorLink: ""
description: "你好世界！可喜可贺的第一篇 Hugo 博文，讲述了这个博客重生的故事。"

tags:
  - 你好世界
  - Kubernetes
categories:
  - 茶馆

hiddenFromHomePage: false
hiddenFromSearch: false

featuredImage: "https://assets.anislet.dev/images/blog/posts/hello-world/cover.jpg"
featuredImagePreview: "https://assets.anislet.dev/images/blog/posts/hello-world/cover.jpg"

toc:
  enable: true
math:
  enable: false
lightgallery: true
license: ""
---

<!--more-->

## 起

> 以前完全搞不懂写日记或者Blog有啥子意义。就好比你说我一个学新闻的学生，怎么就写起代码开发起游戏来了呢？莫名其妙。
>
> 但开一个博客的想法一直都在我脑子里盘旋，特别是经过这一段时间新闻的狂轰乱炸之后，我愈发体会到舆情生命周期之短暂和自身记忆能力之低下。把一些关注的事件、想说的话写下来，似乎变成了一个顺理成章的选择。
>
>{{< style "text-align:right" >}}—— 我，2020年4月7日{{< /style >}}

## 承

于是在两年悠久而漫长的岁月之后，2020年4月份，我再次拥有了自己的博客。

那个时候我刚刚新购买了几台服务器，与以前就有的合共五台一起将他们以功能进行划分并最终组成了一个两节点的 Kubernetes 集群。博客的 git 储存和自动部署都仰仗于集群内的软件。折腾完事儿之后还挺痛快的，甚至放出了以下豪言壮语：

>仔细想想作为一个知识储备飘忽于文理之间的人，能写的东西似乎还蛮多的。文可说历史评时事，理可写代码焊电路，再辅以少量读书心得和游戏设计理念，岂不美哉？也许在我和文豪之间唯一的阻碍，就是懒了。
>
>不过作为一个学新闻的学生，也许播客才是更加符合我自然态的表达形式，虽然牵扯到技术问题时怎样用播客来口述代码仍旧是一个令我感到头疼的问题，但未来真的会搞几期播客玩儿也不一定。
>
>{{< style "text-align:right" >}}—— 我，2020年4月7日{{< /style >}}

后来大概五月份的时候，我查收好久没看的收件箱时发现 Master Node 所在的服务器商 Contabo 于4月6日启用了位于圣路易斯的新数据中心。那个时候集群中除了 Master Node 之外的所有服务器都在北美境内，只有它一个孤家寡人在德国，而正好我也开始对过远距离通讯带来的高延迟感到厌烦，就决定对整体的集群架构进行大改。

为了让集群尽可能[高效]^(省钱)，我剔除了单独作为 Ceph 和 ETCD 使用的两台服务器（每个月省下了将近十美元）。集群的 PVC 我打算直接用 Longhorn 来代替（后来发现 Rook 才是真正优秀的方案）基本没有问题。同时 ETCD 服务器虽然取消了，但我仍旧希望集群能在未来继续加入新的 Master Node，又不想在任何一个 Node 上单独运行 ETCD 集群。于是我决定斗胆使用 K3S 的实验性功能：用嵌入式的 Dqlite 数据库做 HA 实现。实验很成功，集群运转的很顺利。

## 转

民国时期非知名不专业程序员汤师爷曾经曰过：

>实验性功能，任何时候都不要用！一用就死，你们想想，你刚折腾完上线，回了家，带着老婆出了城，吃着火锅还唱着歌，突然这功能就被取消了……所以，没有实验性功能的程序才是好程序！
>
>{{< style "text-align:right" >}}—— 汤师爷，1920年于鹅城{{< /style >}}

2020年6月2日，K3S `v1.18.3+k3s.1` 版本发布，查看更新日志的我突然就发现了一个已经关闭的 Pull request 叫 [WIP: Delete dqlite](https://github.com/rancher/k3s/pull/1760)。本着这玩意儿已经 Close 了肯定是没通过的原则我点开了它看看内容。

然后发现其实是换了个 [Pull request](https://github.com/rancher/k3s/pull/1770)。

这来的一下那我的心可是哇凉哇凉的，激动的心、颤抖的手，就差背过气儿去了。按照相关的 RoadMap 嵌入式 ETCD 的支持将随 K8S `v1.19.x` 版本一起推出（大概是八、九月份），原有的嵌入式 Dqlite 支持将被完全抛弃，且没有迁移数据的可能性。我看着刚刚跑起来没多久的这个 Dqlite 集群，就像看着一坨:shit:，浑身难受。

现在就改吧？改成啥？外置数据库的服务器已经被我退掉了再买也不是个事儿，用内置的 Sqlite 方案集群就只能有一个 Master Node 而不能有更多，在 Node 上单独运行 ETCD 更是完全违背了对集群架构更改的本质。

更要命的是，我心心念念的 cgroups v2 支持也将会随着 K8S 的 `v1.19.x` 版本一起推出。那就只能等了？等三个月，这个一次性的集群在这里空转着，图个啥？

图个寂寞？

## 合

本着**色即是空、空即是色**的原则，我忍了。但同时本着[易迁移]^(懒成猪)的原则我也没有继续往集群上部署新的应用。一个偌大的集群，只跑着一个 Treafik、一个 Elasticsearch、一个 Kibana、和一架云梯。

那老子的博客往哪里放好？

曾经的我本着对“自己的数据掌握在自己手中”的执着和对大部分 [SaaSS](https://www.gnu.org/philosophy/who-does-that-server-really-serve.html) 的不信任，誓要将任何的程序和数据都部署在完全受我控制的环境中。所有的服务器我都会使用 LUKS2 进行全盘加密，搞得我每次重启服务器都得打开 VNC 输一遍密码才能开机（偏偏我还用的是 Fedora Server）；所有的程序我都尽可能去寻找开源的替代进行部署和使用，搞得我花在筛选质量参差不齐的开源软件上的时间比部署还多（虽然选出来了之后还是很爽的）。

但现在，我顿悟了：

> 如果一个 Selfhost 的环境并不稳定，那还 host 你:horse:呢。
>
>{{< style "text-align:right" >}}—— 超我，2020年7月10日{{< /style >}}

特别是类似静态博客这种本身所有的资源就可以被外界直接访问的东西，真的是没有什么 Selfhost 的必要。

于是，这个博客就又重生了。

阿门。

至于这博客究竟是怎么重生的，我们按下不表，日后再说。