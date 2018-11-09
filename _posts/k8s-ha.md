---
layout: post
title: 在国内使用kubeadm搭建k8s集群
key: 20181031
tags: kubeadm k8s kubernetes coreos
modify_date: 2018-1031
---

k8s是目前最流行的容器编排工具，不过k8s的安装部署一直是国内用户的一大痛点，复杂的部署配置过程使得其学习曲线非常陡峭。而新版本推出的kubeadm工具大大的简化了整个过程，这也是k8s官方推荐的安装工具。可惜，在国内的网络环境下，kubeadm是无法顺利运行的。因此，本文提供了在国内环境下使用kubeadm搭建k8s集群的过程。
本文的步骤基于k8s的官方文档 (https://kubernetes.io/docs/setup) 中的“Bootstrapping Clusters with kubeadm”章节，使用coreos公司的Container Linux作为操作系统进行部署k8s的1.11.2版本。

<!--more-->

## 0. 在3个节点上准备kubeadm运行环境，《》第2，3节。


export PATH=$PATH:/opt/bin

## 1. 准备负载均衡器：
[root@k8s1 ~]# ip addr add 192.168.56.40/24 dev eth1
[root@k8s1 ~]# ip addr add 192.168.56.40 dev lo
[root@k8s1 ~]# yum install -y ipvsadm
[root@k8s1 ~]# ipvsadm -A -t 192.168.56.40:6443 -s rr
[root@k8s1 ~]# ipvsadm -a -t 192.168.56.40:6443 -r 192.168.56.41:6443 -g
[root@k8s1 ~]# ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  192.168.56.40:6443 rr
  -> 192.168.56.41:6443           Route   1      0          0

## 2. 安装第1个控制节点
```
apiVersion: kubeadm.k8s.io/v1alpha3
kind: ClusterConfiguration
kubernetesVersion: v1.12.1
apiServerCertSANs:
- 192.168.56.40
controlPlaneEndpoint: 192.168.56.40:6443
etcd:
    external:
        endpoints:
        - https://192.168.56.41:2379
        - https://192.168.56.42:2379
        - https://192.168.56.43:2379
        caFile: /etc/etcd/pki/rootca.pem
        certFile: /etc/etcd/pki/etcdctl.pem
        keyFile: /etc/etcd/pki/etcdctl-key.pem
networking:
    podSubnet: "10.244.0.0/16"
```
[root@k8s1 ~]# kubeadm init --config kubeadm-config.yaml
[init] using Kubernetes version: v1.11.2
[preflight] running pre-flight checks
        [WARNING FileExisting-socat]: socat not found in system path
[preflight] Some fatal errors occurred:
        [ERROR FileContent--proc-sys-net-bridge-bridge-nf-call-iptables]: /proc/sys/net/bridge/bridge-nf-call-iptables contents are not set to 1
        [ERROR Swap]: running with swap on is not supported. Please disable swap
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`

yum install -y socat
sysctl net.bridge.bridge-nf-call-iptables=1
swapoff -a

## 3. 拷贝证书到其他控制节点
```
[root@k8s1 ~]# scp -rp /etc/kubernetes/pki/ca.{crt,key} k8s2:/etc/kubernetes/pki/
root@k8s2's password:
ca.crt                                                                                100% 1025   974.1KB/s   00:00
ca.key                                                                                100% 1675     1.5MB/s   00:00
[root@k8s1 ~]# scp -rp /etc/kubernetes/pki/sa.{pub,key} k8s2:/etc/kubernetes/pki/
root@k8s2's password:
sa.pub                                                                                100%  451   496.3KB/s   00:00
sa.key                                                                                100% 1679     1.7MB/s   00:00
[root@k8s1 ~]# scp -rp /etc/kubernetes/pki/front-proxy-ca.{crt,key} k8s2:/etc/kubernetes/pki/
root@k8s2's password:
front-proxy-ca.crt                                                                    100% 1038   945.7KB/s   00:00
front-proxy-ca.key                                                                    100% 1675     1.6MB/s   00:00
```
