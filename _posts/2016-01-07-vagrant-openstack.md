---
layout: post
title: 使用vagrant和virtualbox搭建openstack集群
key: 20160107
tags: vagrant virtualbox openstack
modify_date: 2018-04-09
---

本文介绍如何在自己的PC环境搭建一个openstack实验环境。

<!--more-->

## 1. 实验环境

硬件要求：
- 4核cpu
- 至少8G内存，建议16G，（如果使用8G内存需要修改Vagrantfile，将os-ctl1的内存改为5G，os-cpu1的内存改为1G）
- 虚拟化软件：
    - [Virtual Box 5.6.2](https://www.virtualbox.org/wiki/Downloads)
    - [Vagrant 2.0.2](https://www.vagrantup.com/downloads.html)
- 实验虚机：
    - os-ctl1: 192.168.56.15 （控制节点，网络节点，计算节点，除非针对该节点的特定用户，否则下文将其称为Allinone节点），默认配置2个cpu，8G内存。
    - os-cpu1: 192.168.56.16 （计算节点），默认配置2个cpu，1G内存。
    - os-net1: 192.168.56.17 （网络节点），默认配置1个cpu，512M内存。

    为了区分实验用的虚机和实验过程中openstack创建的虚机，下文将这两台虚机称为宿主机，而openstack虚机称为nova实例。

- 安装和管理网络：192.168.56.0/24，该网络为VirtualBox的Host-Only网络，支持物理机和VirtualBox虚机间的互相访问。
- Openstack版本：Queens

## 2. 克隆项目并启动上述虚拟机

```
$ git clone https://github.com/lprincewhn/openstack.git
$ cd openstack
$ vagrant up
```
虚拟机启动完毕后可使用以下用户登陆：
- root/vagrant
- vagrant/vagrantup

## 3. 安装openstack

使用root用户登陆os-ctl1，使用packstack来部署openstack。

**Step 1 安装Packstack**
```
# yum install -y centos-release-openstack-queens
# yum install -y openstack-packstack
```

Packstack最简单的安装方式是直接使用--allinone参数，但是在virtaulbox环境下，这种安装方式有两个问题：
1. Packstack默认使用设置了网关的网口所在网络进行安装，即eth0，但是virtualbox中这个网口仅用于访问外部网络，无法进行虚机间通信，因此需要将ip从10.0.2.15修改为192.168.56.15。
2. Packstack默认创建demo租户，如果要该demo正常工作，需要确保相关配置正确，如demo镜像的下载URL，demo所需的网络，为了避免这些配置工作，可以修改配置，使其不创建demo。

**Step 2 生成packsatack answer file**
```
# packstack --allinone --gen-answer-file=allinone
```

**Step 3 修改生成的配置文件**
1. 将其中的ip 10.0.2.15修改为主机名192.168.56.15
```
# vi allinone
# grep 192.168.56.15 allinone  
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
# vi allinone
# grep CONFIG_PROVISION_DEMO allinone
CONFIG_PROVISION_DEMO=n
```

**Step 4开始安装**
```
# packstack --answer-file=allinone
```

## 4. 打开Openstack Dashboard
在安装用户的所在目录可以找到admin用户的密码，如：
```
# cat keystonerc_admin
unset OS_SERVICE_TOKEN
    export OS_USERNAME=admin
    export OS_PASSWORD='4be84115c3624a26'
    export OS_AUTH_URL=http://192.168.56.15:5000/v3
    export PS1='[\u@\h \W(keystone_admin)]\$ '

export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_IDENTITY_API_VERSION=3
```

然后通过URL在浏览器中打开dashboard：http://192.168.56.15。

![openstack-dashboard.PNG](http://o7gg8x7fi.bkt.clouddn.com/openstack-dashboard.PNG)

## 5. 增加计算节点

有一些Openstack操作必须有多个计算节点才能进行，如实例迁移，跨宿主机的网络通信等，因此本节将新增一个计算节点os-cpu1。

拷贝一份之前的packstack answer file，然后修改其中的两个参数：
- CONFIG_COMPUTE_HOSTS，增加新的计算节点IP 192.168.56.16。
- EXCLUDE_SERVERS，为了避免影响已安装的节点os-ctl1，将其IP加入该参数，使得packstack不会对该节点做任何操作。

```
# cp allinone addcpu
# diff allinone addcpu  
86c86
< EXCLUDE_SERVERS=
---
> EXCLUDE_SERVERS=192.168.56.15
97c97
< CONFIG_COMPUTE_HOSTS=192.168.56.15
---
> CONFIG_COMPUTE_HOSTS=192.168.56.15,192.168.56.16
# packstack --answer-file=addcpu
```

这样安装第一次必定会失败，因为把现有节点加入EXCLUDE_SERVERS后，packstack在安装过程不会对其进行任何修改，即不会添加新增节点访问现有节点的iptables规则。即新增的os-cpu1无法访问控制节点上的mysql，rabbitmq等服务，因此os-cpu1安装完成后其服务无法正常启动。

针对这个问题，需要手动添加一些规则到Allinone节点的iptables中。
通过比较两个节点上已经安装的规则，可以找到需要添加的规则如下：

```
# iptables -I INPUT 2 -s 192.168.56.16/32 -p tcp -m multiport --dports 5671,5672 -m comment --comment "001 amqp incoming amqp_192.168.56.16" -j ACCEPT #访问控制节点上的rabbitmq-server
# iptables -I INPUT 2 -s 192.168.56.16/32 -p tcp -m multiport --dports 3260 -m comment --comment "001 cinder incoming cinder_192.168.56.16" -j ACCEPT #访问控制节点上的cinder服务
# iptables -I INPUT 2 -s 192.168.56.16/32 -p tcp -m multiport --dports 3306 -m comment --comment "001 mariadb incoming mariadb_192.168.56.16" -j ACCEPT #访问控制节点上的mariadb
# iptables -I INPUT 2 -s 192.168.56.16/32 -p udp -m multiport --dports 4789 -m comment --comment "001 neutron tunnel port incoming neutron_tunnel_192.168.56.15_192.168.56.16" -j ACCEPT #访问网络节点上的neutron tunel
# iptables -I INPUT 2 -s 192.168.56.16/32 -p tcp -m multiport --dports 16509,49152:49215 -m comment --comment "001 nova qemu migration incoming nova_qemu_migration_192.168.56.15_192.168.56.16" -j ACCEPT #计算节点间进行实例迁移时互相访问
# service iptables save  # 修改完毕后必须保存
```

重新运行packstack：
```
# packstack --answer-file=addcpu
```

安装完成后在Dashboard上已经可以看到两个计算节点：
![add-compute.PNG](http://o7gg8x7fi.bkt.clouddn.com/add-compute.PNG)

从上面防火墙规则可以看出，如果按照packstack安装时默认的防火墙规则配置方法，每增加一个计算节点，都要在启动所有节点（包括控制，网络，计算节点）上增加针对新节点IP的防火墙规则，配置复杂度随着集群规模的扩大以平方速度上升。针对这个问题，一个可行的解决方案是：
使用packstack安装完每个节点后，都手动需修改这个节点的防火墙规则，将其针对具体节点的ACCEPT规则合并修改为针对整个管理网络（即本文中的192.168.56.0/24）的规则，如下：

- 控制节点
```
# iptables -I INPUT 2 -s 192.168.56.0/24 -p tcp -m multiport --dports 5671,5672 -m comment --comment "001 amqp incoming amqp_192.168.56.0" -j ACCEPT #访问控制节点上的rabbitmq-server
# iptables -I INPUT 2 -s 192.168.56.0/24 -p tcp -m multiport --dports 3260 -m comment --comment "001 cinder incoming cinder_192.168.56.0" -j ACCEPT #访问控制节点上的cinder服务
# iptables -I INPUT 2 -s 192.168.56.0/24 -p tcp -m multiport --dports 3306 -m comment --comment "001 mariadb incoming mariadb_192.168.56.0" -j ACCEPT #访问控制节点上的mariadb
```

- 网络节点
```
# iptables -I INPUT 2 -s 192.168.56.0/24 -p udp -m multiport --dports 4789 -m comment --comment "001 neutron tunnel port incoming neutron_tunnel_192.168.56.0" -j ACCEPT #网络节点和计算节点两两间的neutron tunel
```

- 计算节点
```
# iptables -I INPUT 2 -s 192.168.56.0/24 -p udp -m multiport --dports 4789 -m comment --comment "001 neutron tunnel port incoming neutron_tunnel_192.168.56.0" -j ACCEPT #网络节点和计算节点两两间的neutron tunel
# iptables -I INPUT 2 -s 192.168.56.0/24 -p tcp -m multiport --dports 16509,49152:49215 -m comment --comment "001 nova qemu migration incoming nova_qemu_migration_192.168.56.0" -j ACCEPT #计算节点间进行实例迁移时互相访问
```

## 5. 增加网络节点

拷贝一份之前的packstack answer file，然后修改其中的两个参数：
- CONFIG_NETWORK_HOSTS，增加新的计算节点IP 192.168.56.17。
- EXCLUDE_SERVERS，为了避免影响已安装的节点os-ctl1和os-cpu1，将其IP加入该参数，使得packstack不会对这些节点做任何操作。
```
# cp addcpu addnet
# diff addcpu addnet         
86c86
< EXCLUDE_SERVERS=192.168.56.15
---
> EXCLUDE_SERVERS=192.168.56.15,192.168.56.16
101c101
< CONFIG_NETWORK_HOSTS=192.168.56.15
---
> CONFIG_NETWORK_HOSTS=192.168.56.15,192.168.56.17
# packstack --answer-file=addnet
```

如果之前已经把现有的两个节点的防火墙策略修改为针对网络放通，则此时安装就无需再次修改，可以直接成功。

## 6. 删除计算节点和网络节点
Openstack中还没有提供删除节点的接口，需要到数据库中手动清除。
