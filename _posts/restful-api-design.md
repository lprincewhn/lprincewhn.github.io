---
layout: post
title: Restful Api设计原则
key: 20160720
tags: restful api
modify_date: 2016-08-25
---

SNMP是最常用的网络管理协议。

<!--more-->

1. 资源扁平化，
/zoos/<zoo id>/annimals 是一个层次化的api，权限控制较为简单。
/animals?zooid=<zoo id> 是一个扁平化的api，查询更加灵活。

2. 隐式资源
filter
后台任务，如批量操作
TCC事务

3. 声明式的api和命令式的api
声明式的api更容易为用户理解，但是背后的实现难度较大。

4. api的设计粒度
api分层：资源层，简单固定
         业务层，复杂多变，一般对应边缘服务
