---
layout: post
title: 使用vagrant和virtualbox在本地搭建openstack集群
key: 20160107
tags: vagrant virtualbox openstack
modify_date: 2018-04-09
---

本文介绍如何在自己的PC环境搭建一个最简版的openstack。

<!--more-->

## 1. 搭建实验环境

硬件要求：
- 4核cpu
- 至少8G内存，建议16G，（如果使用8G内存需要修改Vagrantfile，将os-ctl1的内存改为5G）

虚拟化软件：
- [Virtual Box 5.6.2](https://www.virtualbox.org/wiki/Downloads)
- [Vagrant 2.0.2](https://www.vagrantup.com/downloads.html)

实验虚机：
- os-ctl1: 192.168.56.15 （控制节点，网络节点，计算节点）
- os-cpu1: 192.168.56.16 （计算节点，供未来使用）


为了区分这两台实验虚机和实验过程中openstack创建的虚机，下文将这两台虚机称为宿主机，而openstack虚机称为实例。

克隆项目并启动上述虚拟机：
``` bash
$ git clone https://github.com/lprincewhn/openstack.git
$ cd openstack
$ vagrant up
$ vagrant ssh os-ctl1
```

## 2. 使用packstack安装openstack

### 2.1 安装packstack

``` bash
$ sudo yum install -y centos-release-openstack-queens
$ sudo yum install -y openstack-packstack
```

### 2.2 快速安装openstack
最简单的安装方式是直接使用--allinone参数安装，但是在virtaulbox环境下，这种安装方式有两个问题：
1. packstack默认使用设置了网关的网口所在网络进行安装，即eth0，但是virtualbox中这个网口仅用于访问外部网络，无法进行虚机间通信，因此需要将ip从10.0.2.15修改为192.168.56.15。
2. packstack默认创建demo租户，如果要该demo正常工作，需要确保相关配置正确，如demo镜像的下载URL，demo所需的网络，为了避免这些配置工作，可以修改配置，使其不创建demo。
```
$ sudo packstack --allinone --gen-answer-file=allinone
$ sudo vi allinone
```
修改生成的配置文件：
1. 将其中的ip 10.0.2.15修改为主机名192.168.56.15
``` bash
$ grep 192.168.56.15 allinone  
CONFIG_CONTROLLER_HOST=192.168.56.15
CONFIG_COMPUTE_HOSTS=192.168.56.15
CONFIG_NETWORK_HOSTS=192.168.56.15
CONFIG_SSL_CERT_SUBJECT_CN=192.168.56.15
CONFIG_SSL_CERT_SUBJECT_MAIL=admin@192.168.56.15
CONFIG_AMQP_HOST=192.168.56.15
CONFIG_MARIADB_HOST=192.168.56.15
CONFIG_REDIS_HOST=192.168.56.15
```
2. 取消安装demo
```
CONFIG_PROVISION_DEMO=n
``` 
开始安装：
``` bash
$ sudo packstack --answer-file=allinone
```

## 3. Neutron网络介绍

安装完毕之后，宿主机通过网口eth0使用NAT访问外部网络，ip是virtualbox自动分配的10.0.2.15/24，而网口eth1用作openstack实例间通信，ip是我们在Vagrant中指定的。
openstack中的虚机通过网桥br-ex访问外部网络，因此我们需要将网络节点的eth0加入到br-ex网桥中。加入网桥后，必须将其ip配到br-ex上，否则外部无法访问（体现在：1. vagrant ssh无法登陆宿主机；2. floating ip无法访问openstack实例，因为网络节点没有floating ip的路由)