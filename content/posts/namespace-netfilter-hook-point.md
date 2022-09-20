---
title: "命名空间对netfilter的影响"
date: 2021-09-29T21:42:32+08:00
categories: ["操作系统"]
draft: false
---

# 需求

主机内,只允许 TCP 控制报文(TCP报文仅有头部)向外发送,包含数据的 TCP 报文禁止外发.

# 实现

通过 netfilter 对 NF_INET_LOCAL_OUT 进行 hook, 丢弃包含数据的 TCP 报文.

# 测试

在虚拟机进行测试.有两条策略:放行 ssh; 阻断 portainer. 

然而配置策略后两个服务都可以正常访问.~~与没有配置 netfilter 过滤规则效果相同,怀疑是不是从头开始就是错的~~

更换环境再测试,发现表现一切正常.不过这次配置阻断的是 hugo server.

初步认定实现应该是部分正确的,找到两个环境的差异即可.

再次回到出现异常的环境,通过 tcpdump 抓包发现每次访问 portainer 都会有虚拟机内部之间的一次访问.

恍然间意识到 portainer 是运行在 docker 中的服务.

# 解决

问题逐渐清晰, docker 通过内核命名空间实现资源隔离.

盲猜 NF_INET_LOCAL_OUT 这个 hook 点可能跟命名空间有关,于是把 hook 点换成了 NF_INET_POST_ROUTING .

问题解决.

# TODO

虽然解决了上述问题,不过新问题也随之产生:

1. 命名空间对 netfilter hook 点的影响.
2. 命名空间对内核中其他组件的影响

挖个坑,等到必要的时候补上.

