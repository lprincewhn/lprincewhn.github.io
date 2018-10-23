---
layout: post
title: ssh的代理和端口转发机制介绍
key: 2018-09-10
tags: ssh tunnel socks5
modify_date: 2018-09-10
---

本文介绍通过ssh建立隧道的三种方式。

<!--more-->

ssh的隧道均通过端口转发来实现，包括三种模式：
- 本地端口转发，使用-L参数
- 远程端口转发，使用-R参数
- 动态端口转发，使用-D参数

MobaXterm作为一个良心的终端工具，在其MobaSSHTunnel菜单项可以帮助我们基于图形化的方式建立上述三种隧道，其提供的向导也能帮助我们更有效的理解和记住上面这3个参数。因此本文借用这些向导中的截图来进行说明。

## 1. 本地端口转发（Local port forwarding）

本地端口转发实现的功能：把ssh服务器能够访问的ip和端口映射到客户端的指定端口，这样在客户端网络内访问客户端的指定端口就能访问到ssh服务器所在网络中的服务。经常使用的场景是客户端自身通过localhost:<指定端口>去访问。

![SSH-Local-Portforwarding.PNG](http://lprincewhn.github.io/assets/images/SSH-Local-Portforwarding.PNG)

```
ssh -L <本地端口>:<目的地址>:<目的端口> core@192.168.56.80
```

参数说明：
- 本地端口为客户端的指定端口，运行了上述命令后，在ssh客户端上运行netstat -lutup可以看到这个端口被侦听。
- 目的地址是相对于ssh服务器而言的，可以是localhost，此时表示ssh服务器本身。

## 2. 远程端口转发（Remote port forwarding）
远程端口转发实现的功能是把ssh客户端能访问到的ip和端口映射到ssh服务器的指定端口，这样在服务器端的网络内访问服务器的指定端口就能访问到ssh客户端算在网络中的服务。

![SSH-Remote-Portforwarding.PNG](http://lprincewhn.github.io/assets/images/SSH-Remote-Portforwarding.PNG)

```
ssh -R <远程端口>:<目的地址>:<目的端口> core@192.168.56.80
```

参数说明：
- 远程端口为服务器的指定端口，运行了上述命令后，在ssh服务器上运行netstat -lutup可以看到这个端口被侦听。
- 目的地址是相对于ssh客户端而言的，可以是localhost，此时表示ssh客户端本身。

## 3. 动态端口转发（Dynamic port forwarding）

动态端口转发实际上是本地端口转发的升级版，除了建立本地端口转发之外，这种模式还在ssh的通信两端启动了socks5代理服务，并且通过本地端口转发机制把两个socks5代理连接在一起，因此当访问本地的socks5服务时实际上也是在访问远程的socks5服务。这样当指定客户端作为socks5代理之后，实际上相当于把客户端放入服务器端所在网络中，能够访问网络中的任意一个服务，而无需为每个服务（ip:端口）都配一次本地端口转发规则。

![SSH-Dynamic-Portforwarding.PNG](http://lprincewhn.github.io/assets/images/SSH-Dynamic-Portforwarding.PNG)

```
ssh -D <本地端口> core@192.168.56.80
```

参数说明：
- 本地端口为客户端的指定端口，即socks5的代理端口。
