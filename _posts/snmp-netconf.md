---
layout: post
title: 从snmp到netconf
key: 20160720
tags: snmp netconf
modify_date: 2016-08-25
---

SNMP是最常用的网络管理协议。

<!--more-->

## 1 下载地址：

按照snmp软件包net-snmp net-snmp-utils
其中net-snmp包含了snmp服务端软件，安装后系统新增snmpd和snmptrapd服务，其中snmpd侦听161端口，用于接收snmp查询请求，配置文件/etc/snmp/snmpd.conf， snmptrapd侦听162端口，用于接收snmp trap，配置文件/etc/snmp/snmptrapd.conf。
因此，一般而言，snmpd运行在被管理对象上，和snmptrapd运行在管理服务器上。

net-snmp-utils包含了snmp客户端软件，常用工具有：
snmpwalk
snmptrap
snmptranslate


[root@snmp ~]# cat lognotify
#!/bin/bash
read host
read ip
vars=
while read oid val
do
  if [ "$vars" = "" ]
  then
    vars="$oid = $val"
  else
    vars="$vars, $oid = $val"
fi
done
echo trap: $host $ip $vars >/var/tmp/snmptrap.out

snmpd.conf 增加以下行：
view    systemview    included   .1.3.6.1.4

snmptrapd.conf 反注释或增加以下行：
authCommunity   log,execute,net public
traphandle SNMPv2-MIB::coldStart    /usr/bin/bin/my_great_script cold
traphandle default /root/lognotify


netconf: (rfc6241)[https://tools.ietf.org/html/rfc6241]
netconf on ssh: rfc6242
