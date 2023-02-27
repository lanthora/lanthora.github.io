---
title: "主机入侵检测与防御系统"
date: 2022-09-20T22:03:16+08:00
categories: ["操作系统"]
draft: false
---

## 引言

这篇文章介绍一款开源的的主机入侵检测与防御系统.

开源社区有类似的产品,实现方案也大体一致,他们共通的特点是过于庞大,对于我这样一个对 k8s,服务部署都不太了解的人来说,使用起来过于困难.

如果你想尝试类似的产品,同时又喜欢简单轻量的部署方式,又或者你的生产环境资源有限,导致不适合部署复杂应用,这款产品可能可以满足你的需求.

我们提供了一个演示环境,这套环境部署在树莓派 4B 的 Manjaro 上.如果演示环境挂了,请通过博客上提供的联系方式告诉我:

- 服务器地址: https://uranus.armix.cc
- 用户名密码: test/123456

## 初衷

我的初衷是学习内核相关的技术,操作系统的安全产品是一个很好的切入点,它的下限很低同时上限很高.做出一款这样的产品可以检验我的学习成果,同时还能做出对社区有些用的东西.

## 名称的由来

在最初的提交中就确定项目名为 hackernel, 这是 hack 和 kernel 的组合拼写.在[维基百科](https://en.wikipedia.org/wiki/Kludge#Computer_science)中对 hack 这个行为的解释有这样的描述,"问题的解决方案是不效率甚至难以理解的,但是以某种方式起作用".

这个项目实现过程中采用[一些非常规的手段对内核数据进行修改](/posts/disable-kernel-memory-write-protection/).因此 hackernel 是个契合项目的特点的名称.

## 核心功能实现原理

这个项目由内核模块提供核心功能,上层应用可以组合核心功能以达到不同的效果.原理上:

1. 通过修改系统调用表,将系统调用修改为自定义的 hook 函数,根据入参决定是否调用真实的系统调用; 
2. 使用内核提供的 hook 函数, 如网络相关的 netfilter hook 点.

hook 函数有能力决定是否向下层调用.允许向下调用并记录行为为检测,禁止向下调用为防御.我们通过策略来定义某个内核事件发生时的处理行为.

如果你对这部分感兴趣,可以看[内核模块实现](https://github.com/lanthora/hackernel/tree/master/kernel-space)

## 多层封装降低使用难度

内核模块只能一条条的执行策略,而不能给自己下发策略.下发策略是上层的行为,为了降低使用难度,将上层分成了多个层次,越往上越接近最终用户,使用难度也就越低.

### netlink 接口封装成函数

在这一层,我们把内核模块提供的功能封装成了 C/C++ 函数,这套封装与内核的距离最近,灵活性最强.

如果你对这部分感兴趣,可以看[进程操作的封装](https://github.com/lanthora/hackernel/tree/master/user-space/process),相同目录结构下还有文件和网络.

### 函数封装成 unix domain socket 服务

上一层封装与内核直接通信,在内部实现功能存在潜在风险,用 C/C++ 开发难度相对较大.因此在函数的基础上封装出服务,仅在进程内保留最核心的功能并提供 API 调用接口,其他功能在外部.这样做即隔离了进程,又允许用其他语言开发功能.

我们开发了一套事件处理机制,利用这套机制实现了纯异步的请求处理和事件通知.到此为止,所有由 C/C++ 开发的部分结束了.

如果你对这部分感兴趣,可以看[封装 unix domain socket](https://github.com/lanthora/hackernel/tree/master/user-space/ipc)

### unix domain socket 接口封装 Go 应用

经过反复考量,我们决定用 Go 实现用户直接使用的大多数需求.它有很多成熟的库,可以显著提高开发效率.我们实现了三个应用作为给最终用户使用的产品.

## 三个应用

### 测试程序

展示库的使用方法, 验证底层模块是否正常工作.

如果你对这部分感兴趣,可以看 [uranus-sample](https://github.com/lanthora/uranus/tree/master/cmd/sample)

### TG 机器人

通过 Telegram Bot 上报正在发生的进程创建事件.监控入侵.这是相对成熟的应用,目前部署在我的生产环境中,最近一次最长持续运行时间为 2 个月,由于升级操作系统重启了服务.

如果你对这部分感兴趣,可以看 [uranus-telegram](https://github.com/lanthora/uranus/tree/master/cmd/telegram)

### 网页

引言中给出的演示环境就是网页版.前端页面还不成熟,未来可能会出现大幅变动.

如果你对这部分感兴趣,可以看 [uranus-web](https://github.com/lanthora/uranus/tree/master/cmd/web)  
如果你对前端页面感兴趣,可以看 [venus](https://github.com/lanthora/venus)

## 安装

这个项目多个部分解耦独立开发,没有找到一个简单的办法从源码开始构建整套系统.因此单独开一个项目包含其他子项目,并在这个项目里控制版本.

手动安装

```bash
git clone https://github.com/lanthora/tellus.git
make
make install
```

对于 Arch Linux 用户,提供 AUR 方式安装.使用 yay 或者任何你喜欢的 AUR Helper.

```bash
yay -S tellus-git
```

## 写在最后

希望这个项目可以成为对社区有用的玩具.
