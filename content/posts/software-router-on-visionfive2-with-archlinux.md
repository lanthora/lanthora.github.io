---
title: "在 VisionFive 2 上使用 Arch Linux 实现软路由透明代理"
date: 2024-03-24T12:15:43Z
categories: ["计算机网络"]
draft: false
---

## 为什么要整这么抽象的活

以前用树莓派 4B 做软路由器.有线网口作为 WAN 口,其他设备通过 WiFi 接入.由于树莓派 WiFi 信号比较弱,所以放在了卧室.又由于卧室里没有网线,用了另一个迷你主机的无线接入客厅的无线路由器,然后用迷你主机的有线网口直连树莓派的有线网口.最近给迷你主机换了硬盘以后总是有很大的电流声,考虑到这些设备都用灯光,于是决定把这些设备都放到客厅,并想办法让 WiFi 信号可以覆盖整个房间.

最初的方案是,让树莓派的 Wan 口接光猫,原本接光猫的无线路由器作为中继.这个方案用了几天感觉还算可以,不过入户的千兆网被树莓派无线网卡的限制成了百兆.

再后来打算换成常规的有两个有线网口的软路由方案,已经在看买 R2S 还是买 R6S 了.这个时候突然发现 VisionFive2 刚好是双网口.到目前为止 OpenWRT 社区还没有支持 VisionFive 2, 考虑到以前在树莓派上的软路由就是用的 Arch Linux 系统,并且 VisionFive 2 上原本安装的也是 Arch Linux, 所以就有了标题中的想法.

## 在 VisionFive 2 上安装 Arch Linux

参考[这个帖子](https://forum.rvspace.org/t/arch-linux-image-for-visionfive-2/1459)完全够用.我在很久以前就安装了系统,并持续滚动更新,可能最新的安装方式已经有差异,相信手上能拿到板子的人都能装好系统.

## 软路由

### 网卡

网卡配置部分参考[这篇文章](https://nyac.at/posts/archlinux-router).

`WAN: /etc/systemd/network/end0.network`

```ini
[Match]
Name=end0

[Network]
DHCP=yes
DNSSEC=no
DNS=8.8.8.8
```

`LAN: /etc/systemd/network/end1.network`

```ini
[Match]
Name=end1

[Link]
Multicast=yes

[Network]
Address=10.22.33.1/24
MulticastDNS=yes
IPMasquerade=both
```

不要忘了配置服务开机启动

```bash
systemctl status systemd-networkd
```

### DNS

用 SmartDNS 实现国内外域名分别查询不同的 DNS 服务器.国内域名来源是 `dnsmasq-china-list`.

```bash
git clone https://github.com/felixonmars/dnsmasq-china-list.git

# 让国内的域名都添加到 smartdns 的 china 组
make SERVER=china smartdns
cp *smartdns.conf /etc/smartdns/
```

最终 SmartDNS 的配置文件效果

`/etc/smartdns/smartdns.conf`

```txt
conf-file accelerated-domains.china.smartdns.conf

bind 10.22.33.1:53
cache-size 4096

cache-persist yes
cache-file /var/cache/smartdns/smartdns.cache

prefetch-domain yes

serve-expired yes

log-level info
log-file /run/log/smartdns.log
log-size 128k
log-num 2

# china 组的域名能且仅能通过阿里云的 DNS 的服务查
server-tls 223.5.5.5 -group china -exclude-default-group

# 其他域名通过另外的 DNS 服务器
server-tls 8.8.8.8
server-https https://query.esni.tech/doh-public
server-https https://cloudflare-dns.com/dns-query
```

```bash
systemctl enable smartdns
```

PS: 敏感的小伙伴可能已经发现,这里用的其他 DNS 服务器可能不一定能访问,这个问题在后面解决.

### DHCP

`/etc/dhcpd.conf`

```txt
option domain-name-servers 10.22.33.1;
option subnet-mask 255.255.255.0;
option routers 10.22.33.1;
subnet 10.22.33.0 netmask 255.255.255.0 {
    range 10.22.33.100 10.22.33.250;
}
```

创建能够给指定网卡开 dhcpd 的 systemd service

```bash
cp /usr/lib/systemd/system/dhcpd4.service /etc/systemd/system/dhcpd4@.service
vim /etc/systemd/system/dhcpd4@.service
```

```diff
  [Service]
  Type=forking
- ExecStart=/usr/bin/dhcpd -4 -q -cf /etc/dhcpd.conf -pf /run/dhcpd4/dhcpd.pid
+ ExecStart=/usr/bin/dhcpd -4 -q -cf /etc/dhcpd.conf -pf /run/dhcpd4/dhcpd.pid %I
```

给 `LAN` 口开服务

```bash
systemctl enable dhcpd4@end1
```

### 内核转发

```bash
echo net.ipv4.ip_forward=1 > /etc/sysctl.d/30-ipforward.conf
```

目前完成了软路由的配置, `WAN` 口接光猫, `LAN` 口接 `AP` 的 `WAN` 口,应该就能正常访问国内的网站了.

## 透明代理

为了给游戏加速,需要同时支持 TCP 和 UDP 的透明代理.我选择的方案是 Shadowsocks 的 Redir. 但是由于 Shadowsocks 报文会被屏蔽,所以在此之上使用自己开发的支持 UDP 通信 VPN. 让 UDP 信道传输游戏里的 UDP 报文.避免了其他方案可能带来的时延.并且隐藏 Shadowsocks UDP 报文的特征.

### Candy VPN

这篇文章重点不是讲 VPN 的用法,使用方法可以参考[文档](https://github.com/lanthora/candy), 最终的效果是要让代理服务器与软路由在同一个网络下,可以让 VPN 加密 Shadowsocks 的 UDP 流量.

### ShadowSocks

Arch Linux 的仓库里有 shadowsocks-rust, 使用方法就不做赘述了. TCP 流量可以使用 shadowsocks-v2ray-plugin 加密.

### IPSet

前面的 SmartDNS 已经完成了域名的分流,但是还需要 IP 地址的分流.实现方案是 IPSet + IPTables.

以前写了一个把 [apnic](http://ftp.apnic.net/apnic/stats/apnic/delegated-apnic-latest) 格式转换成 CIDR 格式的工具 [iana-tools](https://github.com/lanthora/iana-tools). 用这个工具可以提取出所有中国大陆的 IP 地址,并写入 IPSet 的配置文件

```bash
#!/bin/bash

echo "Downloading apnic..."
wget --quiet -P /tmp/ http://ftp.apnic.net/apnic/stats/apnic/delegated-apnic-latest

echo "Updating ipset..."
iana-tools                                                                  \
  -file="/tmp/delegated-apnic-latest"                                       \
  -cc="CN"                                                                  \
  -type="ipv4"                                                              \
  -headers="create china hash:net family inet hashsize 2048 maxelem 65536"  \
  -prefix="add china " > /etc/ipset.conf

echo "Cleaning up..."
rm /tmp/delegated-apnic-latest*
```

启用 IPSet 服务

```bash
systemctl enable ipset
```

### IPTables

配置中的 x.x.x.x 替换为 ShadowSocks 服务器的真实地址.以下命令会清空当前的 iptables 配置,并用新的配置更新 iptables 配置文件.

```bash
#!/bin/bash

cd "$(dirname "$0")"

echo "Updating ipstables config"

# clean iptables
iptables-save | awk '/^[*]/ { print $1 } 
                     /^:[A-Z]+ [^-]/ { print $1 " ACCEPT" ; }
                     /COMMIT/ { print $0; }' | iptables-restore

# TCP
iptables -t nat -N SS-TCP
iptables -t nat -A SS-TCP -d x.x.x.x -j RETURN
iptables -t nat -A SS-TCP -d 0.0.0.0/8 -j RETURN
iptables -t nat -A SS-TCP -d 10.0.0.0/8 -j RETURN
iptables -t nat -A SS-TCP -d 127.0.0.0/8 -j RETURN
iptables -t nat -A SS-TCP -d 169.254.0.0/16 -j RETURN
iptables -t nat -A SS-TCP -d 172.16.0.0/12 -j RETURN
iptables -t nat -A SS-TCP -d 192.168.0.0/16 -j RETURN
iptables -t nat -A SS-TCP -d 224.0.0.0/4 -j RETURN
iptables -t nat -A SS-TCP -d 240.0.0.0/4 -j RETURN
iptables -t nat -A SS-TCP -d 8.8.8.8 -j RETURN
iptables -t nat -A SS-TCP -m set --match-set china dst -j RETURN
iptables -t nat -A SS-TCP -p tcp -j REDIRECT --to-ports 1080
iptables -t nat -A PREROUTING -p tcp -j SS-TCP
iptables -t nat -A OUTPUT -p tcp -j SS-TCP

# UDP
iptables -t mangle -N SS-UDP
iptables -t mangle -A SS-UDP -d x.x.x.x -j RETURN
iptables -t mangle -A SS-UDP -d 0.0.0.0/8 -j RETURN
iptables -t mangle -A SS-UDP -d 10.0.0.0/8 -j RETURN
iptables -t mangle -A SS-UDP -d 127.0.0.0/8 -j RETURN
iptables -t mangle -A SS-UDP -d 169.254.0.0/16 -j RETURN
iptables -t mangle -A SS-UDP -d 172.16.0.0/12 -j RETURN
iptables -t mangle -A SS-UDP -d 192.168.0.0/16 -j RETURN
iptables -t mangle -A SS-UDP -d 224.0.0.0/4 -j RETURN
iptables -t mangle -A SS-UDP -d 240.0.0.0/4 -j RETURN
iptables -t mangle -A SS-UDP -m set --match-set china dst -j RETURN
iptables -t mangle -A SS-UDP -p udp -j TPROXY --tproxy-mark 0x1/0x1 --on-ip 127.0.0.1 --on-port 1080
iptables -t mangle -A PREROUTING -p udp -j SS-UDP

# save iptables
iptables-save -f /etc/iptables/iptables.rules

echo "Update Successful"
```

启用 IPTables 服务

```bash
systemctl enable iptables
```

### TPROXY

上面的 UDP 转发使用了 TPROXY, 因此需要添加路由和规则.

```bash
#!/bin/bash
/usr/bin/ip route add local 0.0.0.0/0 dev lo table 100
/usr/bin/ip rule add fwmark 0x1/0x1 lookup 100
```

这样配置的规则在重启后会丢失,因此我用 crontab @reboot 规则,在开启启动时执行,不写 systemd service 是因为我懒.如果你也用了 crontab, 不要忘了开对应的服务.

启用 IPTables 服务

```bash
systemctl enable cronie
```

## 完结

此时重启软路由,应该一切正常了吧.
