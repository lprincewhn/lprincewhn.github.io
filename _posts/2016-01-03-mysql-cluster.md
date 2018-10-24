---
layout: post
title: Mysql Cluster 集群搭建
key: 20160103
tags: mysql ha
modify_date: 2016-09-09
---

Mysql集群方案的特点是无需共享存储，其数据通过主从同步来实现各节点的一致性。整个集群方案架构如下图，其中共有3类节点。

![mysql-cluster-architecture.jpg](http://lprincewhn.github.io/assets/images/mysql-cluster-architecture.jpg)

<!--more-->

## 1. 安装包及实验环境

- 虚拟软件：Virtual Box 5.0
- 操作系统：CentOS 7.0
- 虚拟机1：192.168.59.104 （三种节点均运行一个实例）
- 虚拟机2：192.168.59.105 （三种节点均运行一个实例）

下载 [MySQL-Cluster-gpl-7.4.9-1.el7.x86_64.rpm-bundle.tar](https://www.mysql.com/downloads/)

解压后在所有机器上安装以下两个rpm包：
- MySQL-Cluster-client-gpl-7.4.9-1.el7.x86_64.rpm
- MySQL-Cluster-server-gpl-7.4.9-1.el7.x86_64.rpm

3类节点的配置和启动顺序为: [管理节点](#管理节点) -> [数据节点](#数据节点) -> [SQL服务节点](#SQL服务节点)


## 2. 管理节点<span id="管理节点"></span>
管理节点定义了集群的配置信息，如各类节点数目，节点ID，节点地址等。集群启动阶段各个数据和SQL节点都需要连接管理节点，获取配置信息。集群运行时通过这个该节点查看及管理集群。

### 2.1 配置文件

``` bash
$ more /var/lib/mysql-cluster/config.ini
[ndb_mgmd]		# 定义管理节点
nodeid=1
hostname=192.168.59.104
datadir=/var/lib/mysql-cluster/

[ndb_mgmd]
nodeid=2
hostname=192.168.59.105
datadir=/var/lib/mysql-cluster/

[ndbd default]	# ndbd的默认配置信息，数据节点的通用配置写在此处
NoOfReplicas=2	# NoOfReplicas表示数据份数，如果为1，会有数据节点单点故障
DataMemory=80M	# 数据节点使用的共享内存，生产环境中应根据数据量的大小计算出来。
IndexMemory=18M

[ndbd]			#定义数据节点
nodeid=11
hostname=192.168.59.104
datadir=/var/lib/mysql/data

[ndbd]
nodeid=12
hostname=192.168.59.105
datadir=/var/lib/mysql/data

[mysqld]		#定义SQL服务节点
id=21
hostname=192.168.59.104

[mysqld]
nodeid=22
hostname=192.168.59.105

[mysqld]
nodeid=23		#没有定义节点地址的时候，表示这个节点可能来自任何地址。
```

### 2.2 配置及启动
``` bash
$ mkdir -p /var/lib/mysql-cluster/config.ini
$ vim /var/lib/mysql-cluster/config.ini
$ chown -R mysql:mysql /var/lib/mysql-cluster/
$ ndb_mgmd -f /var/lib/mysql-cluster/config.ini --initial #--initial参数为第一次启动时使用
```
如果有多个管理节点，第一次启动时需要把所有管理节点都启动，否则运行ndb_mgm时会报错。

### 2.3 查看集群状态
``` bash
$ ndb_mgm -e show
Connected to Management Server at: 192.168.59.104:1186
Cluster Configuration
---------------------
[ndbd(NDB)]     2 node(s)
id=11 (not connected, accepting connect from 192.168.59.104)
id=12 (not connected, accepting connect from 192.168.59.105)

[ndb_mgmd(MGM)] 2 node(s)
id=1    @192.168.59.104  (mysql-5.6.28 ndb-7.4.9)
id=2 (not connected, accepting connect from 192.168.59.105)

[mysqld(API)]   3 node(s)
id=21 (not connected, accepting connect from 192.168.59.104)
id=22 (not connected, accepting connect from 192.168.59.105)
id=23 (not connected, accepting connect from any host)
```
上图中，由于在管理节点中定义了节点地址，因此nbd_mgmd进程正在等待192.168.59.104上的节点11，21和192.168.59.105上的节点12，22。而没有定义节点地址的SQL服务节点23可以来自任何机器。

## 3. 数据节点<span id="数据节点"></span>
### 3.1 配置文件
数据节点的配置信息检索顺序如下：
``` bash
$ ndbd --help
Default options are read from the following files in the given order:
/etc/my.cnf /etc/mysql/my.cnf /usr/etc/my.cnf ~/.my.cnf
The following groups are read: mysql_cluster ndbd
...
```

``` bash
$ cat /etc/my.cnf
...
[mysql_cluster]
ndb-connectstring=192.168.59.104,192.168.59.105	# 定义管理节点连接信息
...
```
其余配置信息从管理节点中获取。

### 3.2 启动命令
``` bash
$ ndbd --initial  # --initial参数为第一次启动时使用
```

## 4. SQL服务节点<span id="服务节点"></span>
### 4.1 配置文件
``` bash
$ mysqld --verbose --help
...
Default options are read from the following files in the given order:
/etc/my.cnf /etc/mysql/my.cnf /usr/etc/my.cnf ~/.my.cnf
The following groups are read: mysql_cluster mysqld server mysqld-5.6
...
```
``` bash
$ cat /etc/my.cnf
...
[mysqld]
ndbcluster
datadir=/var/lib/mysql/data
socket=/tmp/mysql.sock
port=3307
ndb-connectstring=192.168.59.104,192.168.59.105 # 定义管理节点连接信息
...
```
### 4.2 启动命令
``` bash
$ /usr/share/mysql/mysql.server start
```

所有节点启动完毕后，再检查集群状态：
``` bash
$ ndb_mgm -e show
Connected to Management Server at: 192.168.59.104:1186
Cluster Configuration
---------------------
[ndbd(NDB)]     2 node(s)
id=11   @192.168.59.104  (mysql-5.6.28 ndb-7.4.9, Nodegroup: 0, *)
id=12   @192.168.59.105  (mysql-5.6.28 ndb-7.4.9, Nodegroup: 0)

[ndb_mgmd(MGM)] 2 node(s)
id=1    @192.168.59.104  (mysql-5.6.28 ndb-7.4.9)
id=2    @192.168.59.105  (mysql-5.6.28 ndb-7.4.9)

[mysqld(API)]   3 node(s)
id=21   @192.168.59.104  (mysql-5.6.28 ndb-7.4.9)
id=22   @192.168.59.105  (mysql-5.6.28 ndb-7.4.9)
id=23 (not connected, accepting connect from any host)
```
