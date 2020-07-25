---
title: "让 Timeshift 的 btrfs 模式在 Fedora 32 上运行"
subtitle: "Make Timeshift's btrfs mode work with Fedora 32"
date: 2020-07-24T22:20:58+08:00
draft: false
author: ""
authorLink: ""
description: "Fedora 默认的 btrfs subvolume 命名策略并不被 timeshift 所接受，想要让它正常运行就必须好好的折腾一番。"

tags:
  - Fedora
  - Timeshift
categories:
  - 脑机

hiddenFromHomePage: false
hiddenFromSearch: false

#featuredImage: "https://assets.anislet.dev/images/blog/posts/make-timeshift-btrfs-work-with-fedora/cover.png"
#featuredImagePreview: "https://assets.anislet.dev/images/blog/posts/make-timeshift-btrfs-work-with-fedora/cover.png"

toc:
  enable: true
math:
  enable: false
lightgallery: true
license: ""
---

<!--more-->

## 缘起

因为疫情被困在家里的期间，手中只有一台性能羸弱的 HP ENVY 13 笔记本[^1]。本来跑的是 [Manjaro](https://manjaro.org)，但上网课所需软件大多只支持 Windows，虚拟机也跑不起来，无奈之下只好换回 Windows 暂且将就。

暑假开始之后自然就没有上网课的需求，也不必受 Windows 的折磨了。兴奋的我决定把这台笔记本系统换回 Linux。因为我有在 Kubernetes 1.19 版本发布后将集群内所有的机器都迁移到 [Feodra Coreos](https://getfedora.org/coreos/) 上的打算，所以我就装了个 [Fedora](https://getfedora.org)。又听闻 FESCo 最近刚批准了在 Fedora 33 将 btrfs 取代 xfs 作为 Workstation Edition 的默认文件系统的议程，过了 10 年时间终于实现也是满难得的，再加上用 Manjaro 的时候就用的 btrfs，和 [Timeshift](https://github.com/teejee2008/timeshift) 相互配合，快照备份何其快乐，就决定干脆 Fedora 也做成 btrfs 好了。

Fedora 的安装界面真的是业界良心，好用至极，安装过程丝般顺滑。装完了之后下载 Timeshift，配置 btrfs 快照类型，然后就失败了。

## 浅析

因为是事后再写的文章，那个时候报错的具体文字描述已然是记不得了，唯一可以肯定的是截止到今天，基本所有使用 btrfs 的 Fedora 都会有此问题。

为什么我敢这么肯定呢？从 Timeshift 的自述文件中就可以一探究竟。

> **Supported System Configurations**
>
> - ...
>
> - **BTRFS** - OS installed on BTRFS volumes (with or without LUKS)
>   - Only Ubuntu-type layouts with **@** and **@home** subvolumes are supported
>   - **@** and **@home** subvolumes may be on same or different BTRFS volumes
>   - **@** may be on BTRFS volume and **/home** may be mounted on non-BTRFS partition
>   - Other layouts are not supported

源自 Ubuntu 的 btrfs subvolume 命名策略影响十分深远，且受到了很多发行版与程序的认可，Timeshift 也不例外。但 Fedora 并没有采纳这种命名策略，也就进一步导致 Timeshift 无法正常运行。

```
[orwill@lotus ~]$ sudo btrfs subvolume list /
ID 256 gen 7587 top level 5 path root
ID 257 gen 7587 top level 5 path home
```

这个问题的解决办法也很简单，只要在 Fedora 安装的时候将 btrfs volume 的命名修改为正确的就 OK 了，但现实却远没有那么简单。

## 深入

如果你试图在系统安装过程中将 btrfs subvolume 的命名从默认的 `root` 修改为 `@` 的话，你就会惊奇的发现，它自己又变回去了。

这究竟是为什么呢？

原来，这是因为 Fedora 的系统安装程序 [Anaconda](https://fedoraproject.org/wiki/Anaconda) 所使用的磁盘分区模块 [Blivet](https://github.com/storaged-project/blivet) 本身就将 `@` 作为非法字符的一种进行处理。虽说如此但 Anaconda 又不会弹出非法字符提示，于是就自己变回去了。

而如果你在 BlivetGUI 的高级模式下进行同样操作的话，所输入的 `@` 会被直接忽略掉，生成 `btrfs.xxxx` 格式的随机命名。

但无论是怎样的操作，它就是不给你弹非法字符提示。

经过简单的搜索，我在 Ask Fedora 上找到了相关的讨论[^2]并找到了解决办法，只要将 Fedora 安装程序 ISO 中的 `/usr/lib/python3.8/site-packages/blivet/blivet.py` 文件进行适当的修改，就可以让 `@` 作为合法字符被正常处理。

```python
- tmp = re.sub("[^0-9a-zA-Z._-]", "", tmp)
+ tmp = re.sub("[^0-9a-zA-Z._@-]", "", tmp)
```

这问题似乎是解决了，但最麻烦的部分还在最后。

## 解决

现代的 Linux Install ISO 都会通过各种写保护来确保 ISO 内部文件的安全，针对 LiveCD 进行的 Hack 行为变得十分复杂且难以迅速完成。

如果从头构建新的 ISO 文件，且不论国内拉跨的网络环境从仓库拉取文件究竟要拉取多长时间，就这台性能羸弱的小本子要构建多长时间都不一定。

就为了添加一个 `@`，难道真的要弄这么麻烦？

其实最后的解决方法十分简单，但思索的过程却异常艰难。在问题解决之后回首思索问题时的我有感，请允许我吟一首诗来表达我当下的心情。

{{< style "text-align:center" >}}
>夏日题悟空上人院
>
>杜荀鹤（唐）
>
>三伏闭门披一衲，兼无松竹荫房廊。
>
>安禅不必须山水，灭得心中火自凉。
{{< /style >}}

顺便，下阕也是日本战国时期非著名和尚快川绍喜的辞世诗。

启动 Fedora 的安装程序后，他会利用多个不同的 tty 来实现不同的功能。其中 tty2 是终端，tty6 是 Anaconda 的图形界面。

那问题就简单了，首先用 `ctrl + alt + f2` 切换到 tty2，然后对上文中提到的文件进行修改，让 `@` 成为合法字符。

然后使用 `systemctl restart anaconda.service` 来重启整个 Anaconda 程序。重启的 Anaconda 会加载修改后的 Blivet 模块。这样我们就可以对 btrfs subvolume 进行正确的命名了。

之后一切的安装过程都和正常安装没有区别。启动之后便可以验证相关的设置是否正确。

```
[orwill@lotus ~]$ sudo btrfs subvolume list /
ID 256 gen 7587 top level 5 path @
ID 257 gen 7587 top level 5 path @home
```

安装 Timeshift 也能正常启用 btrfs 模式。

{{< image src="https://assets.anislet.dev/images/blog/posts/make-timeshift-btrfs-work-with-fedora/timeshift.png" alt="Timeshift" caption="**Timeshift** 运行正常" >}}

[^1]: 因为是高考完之后买的，所以是 i7-7500U，两核心四线程，买了没多久 8 代 U 就出了。还额外带着张 MX150 导致我没办法搞黑苹果，从各种意义上来说都是一笔相当亏的买卖。

[^2]: https://ask.fedoraproject.org/t/timeshift-and-btrfs/5325