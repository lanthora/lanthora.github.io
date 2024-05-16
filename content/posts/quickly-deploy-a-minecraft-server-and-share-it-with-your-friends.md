---
title: "快速部署能够与朋友共享的 Minecraft 服务器"
date: 2024-05-16T10:18:30Z
categories: ["游戏"]
draft: true
---

# Docker Compose

创建两个服务,一个用来启动 Minecraft, 另一个用来组建网络.并持久化存储这两个服务产生的数据.

```yaml
services:
  minecraft:
    image: itzg/minecraft-server
    hostname: Minecraft
    environment:
      EULA: "TRUE"
    tty: true
    stdin_open: true
    restart: always
    volumes:
      - ./minecraft-data:/data
  candy:
    image: lanthora/candy
    network_mode: "service:minecraft"
    cap_add:
      - NET_ADMIN
    devices:
      - "/dev/net/tun:/dev/net/tun"
    restart: always
    volumes:
      - ./candy-data:/var/lib/candy
```
