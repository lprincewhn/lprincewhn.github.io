---
layout: post
title: Linux HA 集群原理和配置-04
key: 20160721
tags: ha linux dlm gfs2
modify_date: 2018-03-20
---

本文介绍在Linux HA集群中的分布式文件系统GFS2。

<!--more-->

安装dlm和gfs2-utils
 yum install -y dlm gfs2-utils


创建dlm资源：
pcs resource create dlm ocf:pacemaker:controld op monitor interval=60s
pcs resource clone dlm clone-max=3 clone-node-max=1

dlm的创建依赖于stonith设备，必须启动stonith并创建设备后dlm资源才能创建成功。

创建文件系统：
[root@ha-host2 ~]# mkfs.gfs2 -p lock_dlm -j 3 -t linuxha:sharegfs /dev/sde
It appears to contain an existing filesystem (gfs2)
This will destroy any data on /dev/sde
Are you sure you want to proceed? [y/n] y
Discarding device contents (may take a while on large devices): Done
Adding journals: Done
Building resource groups: Done
Creating quota file: Done
Writing superblock and syncing: Done
Device:                    /dev/sde
Block size:                4096
Device size:               1.00 GB (262144 blocks)
Filesystem size:           1.00 GB (262143 blocks)
Journals:                  3
Resource groups:           6
Locking protocol:          "lock_dlm"
Lock table:                "linuxha:sharegfs"
UUID:                      e1e17673-ac4b-4fc2-9f6b-546dc3539bb4

 [root@ha-host1 ~]# systemctl status dlm
 ● dlm.service - dlm control daemon
    Loaded: loaded (/usr/lib/systemd/system/dlm.service; enabled; vendor preset: disabled)
    Active: active (running) since Fri 2018-05-04 08:16:50 UTC; 1min 13s ago
   Process: 10583 ExecStartPre=/sbin/modprobe dlm (code=exited, status=0/SUCCESS)
  Main PID: 10585 (dlm_controld)
    CGroup: /system.slice/dlm.service
            └─10585 /usr/sbin/dlm_controld --foreground

 May 04 08:18:01 ha-host1 dlm_controld[10585]: 4663 fence result 2 pid 10729 ...s
 May 04 08:18:01 ha-host1 dlm_controld[10585]: 4663 fence status 2 receive 23...3
 May 04 08:18:02 ha-host1 dlm_controld[10585]: 4664 fence status 2 receive 23...4
 May 04 08:18:02 ha-host1 dlm_controld[10585]: 4664 fence request 2 pid 10775...h
 May 04 08:18:02 ha-host1 dlm_stonith[10775]: stonith_api_time: Found 42 entr...d
 May 04 08:18:02 ha-host1 dlm_stonith[10775]: stonith_api_time: Node 2/(null)...8
 May 04 08:18:02 ha-host1 stonith-api[10775]: stonith_api_kick: Could not kic...)
 May 04 08:18:02 ha-host1 dlm_stonith[10775]: kick_helper error -19 nodeid 2
 May 04 08:18:03 ha-host1 dlm_controld[10585]: 4665 fence result 2 pid 10775 ...s
 May 04 08:18:03 ha-host1 dlm_controld[10585]: 4665 fence status 2 receive 23...5
 Hint: Some lines were ellipsized, use -l to show in full.
 [root@ha-host1 ~]#

 dlm服务必须有stonith支持
