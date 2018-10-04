---
layout: post
title: Derby
key: 20160720
tags: derby java
modify_date: 2018-07-20
---

Derby是一个开源的纯Java的嵌入式关系型数据库，嵌入式数据库无需额外的数据库服务器，适合轻小量的数据持久化存储，因此在使用Java语言开发的桌面型应用或者Web应用中内部的业务逻辑验证时使用较多，可以理解为Java领域的sqlite。其商业版即为Oracle的Java DB，可见它与Java技术栈的关系。

<!--more-->

## 1 下载地址：

https://db.apache.org/derby/derby_downloads.html

其实derby已经嵌入在jdk中，即使不下载上述软件包，也能够以嵌入模式使用。本文中重要使用上述软件包中的两个命令：
1. ij，derby的命令行客户端。
2. NetworkServerControl，网络模式的启动程序。

## 2 命令行客户端ij

和其他数据库一样，derby也提供了自己的命令行客户端，可执行文件位于bin目录下，可直接启动：
```
$ /d/GreenProgram/db-derby-10.12.1.1-bin/bin/ij
ij version 10.12
ij>
```

## 3 连接数据库

Derby支持两种访问模式：

- 嵌入模式：数据库服务器直接内嵌在客户端的JVM进程中，客户端无需通过网络即可访问服务器。这种模式仅支持单会话，因为数据库被客户端进程锁定，其他进程无法范围。
- 网络模式：和其他数据库服务器类似，运行专门的服务器进程，侦听指定端口（默认为1527），客户端通过tcp连接访问服务器。这种模式支持多会话同时访问。

### 3.1 嵌入模式
使用嵌入模式连接数据库使用以下命令：
connect 'jdbc:derby:<db path>;[create=true;][user;][password;]'

```
ij> connect 'jdbc:derby:demodb;create=true;';
```

### 3.2 网络模式
使用网络模式访问首先需要启动服务器进程:
```
$ /d/GreenProgram/db-derby-10.12.1.1-bin/bin/NetworkServerControl start
Mon May 14 11:06:51 CST 2018 : Security manager installed using the Basic server security policy.
Mon May 14 11:06:56 CST 2018 : Apache Derby Network Server - 10.12.1.1 - (1704137) started and ready to accept connections on port 1527

```
启动服务器进程后，在ij内使用以下命令连接数据库：
connect 'jdbc:derby://<ip:port>/<db name>;[create=true;][user;][password;]'

ij> connect 'jdbc:derby://localhost:1527/demodb';

```
ij> connect 'jdbc:derby://localhost:1527/demodb;';
```

由于之前已经使用嵌入模式创建了demodb，此时无需增加create=true选项。


## 4 常用命令
show tables


## 5. 增加用户

Derby中Schema的概念与普通数据库不同，Derby中的Schema主要实现了命名空间的隔离，将数据库的对象划分在不同的命名空间中，实现隔离。可将其理解为数据库“用户”。当不指定不用户名访问数据库时，默认的scheme是APP。当指定用户名访问数据库时，默认schema即为用户名。

```
ij>CALL SYSCS_UTIL.SYSCS_SET_DATABASE_PROPERTY(‘derby.authentication.provider’,‘BUILTIN’);
ij>CALL SYSCS_UTIL.SYSCS_SET_DATABASE_PROPERTY(‘derby.connection.requireAuthentication’,‘true’);
ij>CALL SYSCS_UTIL.SYSCS_SET_DATABASE_PROPERTY(‘derby.user.demo’,‘123456’);
ij>CALL SYSCS_UTIL.SYSCS_SET_DATABASE_PROPERTY(‘derby.database.fullAccessUsers’,‘demo’);
ij>CALL SYSCS_UTIL.SYSCS_SET_DATABASE_PROPERTY(‘derby.database.defaultConnectionMode’,‘noAccess’);
```
