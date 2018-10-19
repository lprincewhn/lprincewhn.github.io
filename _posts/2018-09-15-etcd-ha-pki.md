---
layout: post
title: 通过etcd集群搭建了解pki安全认证
key: 20180915
tags: etcd pki
modify_date: 2018-9-15
---

本文使用3台CentOS7虚拟机搭建etcd集群，该集群是k8s高可用集群的基础。并且基于搭建过程解释了etcd中的pki机制，了解pki机制在集群安全中的一般工作模式。

<!--more-->

## 0. 实验环境

创建3台CentOS7虚拟机，主机名为k8s1-k8s3，并分配ip 192.168.56.41-43，可通过以下vagrant项目快速启动：
```
$ git clone https://github.com/lprincewhn/k8s-ha.git
$ cd k8s-ha
$ vagrant up
```

## 1. 启动etcd非安全集群

### 1. 安装并启动etcd

在3个节点上安装etcd：
```
# yum install -y etcd
# systemctl start etcd && systemctl enable etcd
```

使用etcdctl访问etcd并检查其状态验证启动成功。
```
# etcdctl cluster-health
member 8e9e05c52164694d is healthy: got healthy result from http://localhost:2379
```

### 2. 修改配置启动集群

上述3个节点上的etcd并未形成集群，配置集群需修改配置文件/etc/etcd/etcd.conf的的以下参数，以k8s1节点为例：
```
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"          # 数据存储目录，可使用默认值
ETCD_LISTEN_PEER_URLS="http://192.168.56.41:2380"   # 用于集群环境中侦听来自于其他节点的请求
ETCD_LISTEN_CLIENT_URLS="http://192.168.56.41:2379,http://127.0.0.1:2379"   # 服务URL，默认只允许本地访问，
ETCD_NAME="k8s1"                                    # 节点名称，默认为default，在集群环境中有多个节点，因此需要修改
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.56.41:2380"  # 广播给集群其他节点访问的地址
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.56.41:2379"        # 广播给客户端访问的地址
ETCD_INITIAL_CLUSTER="k8s1=http://192.168.56.41:2380,k8s2=http://192.168.56.42:2380,k8s3=http://192.168.56.43:2380" # 集群及其URL列表
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"   # 集群相互认证时使用的token
ETCD_INITIAL_CLUSTER_STATE="new"            
ETCD_STRICT_RECONFIG_CHECK="true"
```

删除已有实例，然后重新启动etcd。
```
# systemctl stop etcd
# rm -rf /var/lib/etcd/default.etcd
# systemctl start etcd
```

可以在任意一个节点上使用etcdctl验证集群状态:
```
# etcdctl cluster-health
member 6bf749bb3ab16843 is healthy: got healthy result from http://192.168.56.43:2379
member 7c1dfc5e13a8008a is healthy: got healthy result from http://192.168.56.42:2379
member c920522ba9a75e17 is healthy: got healthy result from http://192.168.56.41:2379
cluster is healthy
```

etcdctl默认访问的服务端点是http://127.0.0.1:2379, 因此此时访问的是本地的etcd，但是etcd会返回整个集群的状态。如果之前在配置文件中的ETCD_LISTEN_CLIENT_URLS不包含http://127.0.0.1:2379, 或者需要访问其他节点上的etcd，可以通过--endpoints参数指定服务端点。
```
[root@k8s1 ~]# etcdctl --endpoints="http://192.168.56.41:2379" cluster-health
member 6bf749bb3ab16843 is healthy: got healthy result from http://192.168.56.43:2379
member 7c1dfc5e13a8008a is healthy: got healthy result from http://192.168.56.42:2379
member c920522ba9a75e17 is healthy: got healthy result from http://192.168.56.41:2379
cluster is healthy
[root@k8s1 ~]# etcdctl --endpoints="http://192.168.56.42:2379" cluster-health
member 6bf749bb3ab16843 is healthy: got healthy result from http://192.168.56.43:2379
member 7c1dfc5e13a8008a is healthy: got healthy result from http://192.168.56.42:2379
member c920522ba9a75e17 is healthy: got healthy result from http://192.168.56.41:2379
cluster is healthy
[root@k8s1 ~]# etcdctl --endpoints="http://192.168.56.43:2379" cluster-health
member 6bf749bb3ab16843 is healthy: got healthy result from http://192.168.56.43:2379
member 7c1dfc5e13a8008a is healthy: got healthy result from http://192.168.56.42:2379
member c920522ba9a75e17 is healthy: got healthy result from http://192.168.56.41:2379
cluster is healthy
```

## 3. 集群通信场景

集群服务中的通信一般包括两种场景：
- 对外提供服务的通信，发生在集群外部的客户端和集群某个节点之间，etcd默认端口为2379。
- 集群内部的通信，发生在集群内部的任意两个节点之间，etcd的默认端口为2380。

后面的章节将针对这两部分通信逐步在etcd集群中应用pki机制，加强集群安全。

## 4. 创建RootCA

### 4.1 安装pki证书管理工具cfssl：
```
# curl -s -L -o /bin/cfssl https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
# curl -s -L -o /bin/cfssljson https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
# curl -s -L -o /bin/cfssl-certinfo https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
# chmod +x /bin/cfssl*
```
使用openssl同样可以完整PKI的证书配置，而cfssl是一个轻量级的工具，其命令使用起来相对简单。

### 4.2 PKI配置以及生成根CA证书

对于上面描述的两种场景，通信过程中的证书作用是不同的：

- 服务器与客户端之间的通信，这种情况下服务器的证书仅用于服务器认证，客户端证书仅用于客户端认证
- 服务器间的通信，这种情况下每个etcd既是服务器也是客户端，因此其证书既要用于服务器认证，也要用于客户端认证。

创建PKI配置文件：
```
# mkdir /etc/etcd/pki
# cd /etc/etcd/pki
# cfssl print-defaults config > ca-config.json
# vi ca-config.json
```

基于上述两种场景修改生成的配置文件ca-config.json，在其中定义3个profile：
- server，作为服务器与客户端通信时的服务器证书，其配置如下：
```
# cat ca-config.json
...
            "server": {
                "expiry": "8760h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth"
                ]
            },
...
```
- client，作为服务器与客户端通信时的客户端证书。
```
# cat ca-config.json
...
            "client": {
                "expiry": "8760h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "client auth"
                ]
            },
...
```
- peer，作为服务器间通信时用的证书，既认证服务器也认证客户端。
```
# cat ca-config.json
...
            "peer": {
                "expiry": "8760h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ]
            },
...
```

实验集群有3个节点，需要配置的所有证书总结如下：
- 根CA证书1个, 每个节点都保存，只保存证书即可。
- 服务器server证书3个, 每个节点保存自己的证书和私钥。
- 服务器peer证书1个, 为了节省篇幅，本实验中每个节点都使用同样的peer证书，每个服务器均保存证书和私钥。
- 客户端证书1个, 本实验环境中仅供etcdctl使用，因此在运行etcdctl的主机上保存证书和私钥即可。生产环境中每个访问etcd的客户端都应该有自己的客户端证书和私钥。

### 4.3 创建RootCA证书

准备创建证书请求文件所需配置，其中应该包含CA的标识，所在主机，证书加密算法等信息。
```
# cfssl print-defaults csr > rootca-csr.json
# vi rootca-csr.json
```
修改后内容如下，其中重要的是CN和hosts字段，CN是通信实体的标识，而由于CA证书不表示任何一台服务器，因此此处无需hosts字段。
names可根据实际情况添加作为辅助信息，key指明了加密算法。除非遇上某些特殊算法不被支持，否则没必要修改。

```
# cat rootca-csr.json
{
    "CN": "ETCD Root CA",
    "key": {
        "algo": "ecdsa",
        "size": 256
    },
    "names": [
        {
            "C": "US",
            "L": "CA",
            "ST": "San Francisco"
        }
    ]
}
```
运行以下指令生成3个文件:
- rootca.csr, 证书请求文件，实际上是cfssl的一个中间文件，有上述配置文件生成，然后根据其中的内容生成证书。
- rootca.pem, 证书文件，即上面提到的根CA证书，需要保存到所有服务器和客户端上，用于验证通信过程中交互的证书。
- rootca-key.pem, 私钥文件，可用于验证rootca.pem, 日常通信不会使用，应该离线保存。
```
# cfssl gencert -initca rootca-csr.json | cfssljson -bare rootca
# ls rootca*
rootca.csr  rootca-csr.json  rootca-key.pem  rootca.pem
```

可使用openssl指令查看上述生成的证书信息：
```
# openssl x509 -in rootca.pem -text -noout
```

## 5. 服务器端验证

### 5.1 创建服务器证书

和创建CA证书类似，准备创建证书请求文件所需配置:
```
# cfssl print-defaults csr > etcd1-csr.json
# vi etcd1.json
```

修改其中CN和hosts，与创建根CA证书不同之处在于服务器证书需要指定hosts，表明该证书只能被192.168.56.41这个主机使用。如果需要使用域名，主机名或者其他ip访问etcd，则此处需要将所有可能用到的域名，主机名，IP加进去。反过来说，如果这里配置了多个节点的IP，则表示这个证书可以在多台主机上使用，也就是，可以整个集群公用一个证书。
```
{
    "CN": "ETCD on k8s1",
    "hosts": [
        "192.168.56.41"
    ],
    "key": {
        "algo": "ecdsa",
        "size": 256
    },
    "names": [
        {
            "C": "US",
            "L": "CA",
            "ST": "San Francisco"
        }
    ]
}
```
运行以下指令生成证书及其私钥：
```
# cfssl gencert -ca=rootca.pem -ca-key=rootca-key.pem -config=ca-config.json -profile=server etcd1-csr.json | cfssljson -bare etcd1
```
参数说明如下:
- -ca: 签发证书的CA证书
- -ca-key: 签发证书的CA私钥，事实上，签发证书正是使用CA私钥对证书进行签名的过程。
- config: 指定配置文件
- profile: 指定配置文件中的profile，参考上面的描述，其中可定义证书的有效期，用途等。

如果是用root用户生成的证书文件，由于私钥文件生成后的权限为rw-------，其他用户不可读，因此需要将其属主修改为etcd。
```
# chown -R etcd:etcd /etc/etcd/pki/
```

### 5.2 修改etcd配置并重启

修改配置文件，让该节点上的etcd服务在https端口，需要修改的参数如下：
```
ETCD_LISTEN_CLIENT_URLS="https://192.168.56.41:2379,http://127.0.0.1:2379"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.56.41:2379"
ETCD_CERT_FILE="/etc/etcd/pki/etcd1.pem"
ETCD_KEY_FILE="/etc/etcd/pki/etcd1-key.pem"
```

然后重启etcd，此时如果在etcdctl指令中不指定--ca-file参数，结果会提示 https://192.168.56.41:2379 访问失败，因为其证书是不受信任的。
```
# etcdctl cluster-health
member 6bf749bb3ab16843 is healthy: got healthy result from http://192.168.56.43:2379
member 7c1dfc5e13a8008a is healthy: got healthy result from http://192.168.56.42:2379
failed to check the health of member c920522ba9a75e17 on https://192.168.56.41:2379: Get https://192.168.56.41:2379/health: x509: certificate signed by unknown authority
member c920522ba9a75e17 is unreachable: [https://192.168.56.41:2379] are all unreachable
cluster is healthy
```
**注意：ETCD_LISTEN_CLIENT_URLS中包含了http://127.0.0.1:2379, 因此直接指定该地址可以访问etcd，但是ETCD_ADVERTISE_CLIENT_URLS中不包含http://127.0.0.1:2379, 因此etcd在给客户端广播集群节点的地址时，只会广播https://192.168.56.41:2379, etcdctl紧接着用这个地址去查询集群健康状态时，则由于证书不受信任无法访问。**

加上--ca-file参数指定用于校验的CA证书，即根CA证书后，访问正常。
```
# etcdctl --ca-file /etc/etcd/pki/rootca.pem cluster-health
member 6bf749bb3ab16843 is healthy: got healthy result from http://192.168.56.43:2379
member 7c1dfc5e13a8008a is healthy: got healthy result from http://192.168.56.42:2379
member c920522ba9a75e17 is healthy: got healthy result from https://192.168.56.41:2379
cluster is healthy
```

上面输出可以看到，仅有1个节点启动了https。对其余两个节点重复本节操作即可。出于对rootca的安全考虑，服务器证书的生成操作在一台服务器上完成，生成后将其拷贝到相应节点即可。配置并重启完所有节点后，应该可以看到所有节点的侦听URL均为https。
```
# etcdctl --ca-file /etc/etcd/pki/rootca.pem cluster-health
member 6bf749bb3ab16843 is healthy: got healthy result from https://192.168.56.43:2379
member 7c1dfc5e13a8008a is healthy: got healthy result from https://192.168.56.42:2379
member c920522ba9a75e17 is healthy: got healthy result from https://192.168.56.41:2379
cluster is healthy
```

## 6. 客户端验证

### 6.1 修改etcd配置并重启

启动客户端认证需要修改以下参数：
```
ETCD_CLIENT_CERT_AUTH="true"
ETCD_TRUSTED_CA_FILE="/etc/etcd/pki/rootca.pem"
```

重启etcd服务：
```
# systemctl restart etcd
```

发现即使指定了--ca-file参数，节点1又连不上了。
```
# etcdctl --ca-file /etc/etcd/pki/rootca.pem cluster-health
member 6bf749bb3ab16843 is healthy: got healthy result from https://192.168.56.43:2379
member 7c1dfc5e13a8008a is healthy: got healthy result from https://192.168.56.42:2379
failed to check the health of member c920522ba9a75e17 on https://192.168.56.41:2379: Get https://192.168.56.41:2379/health: remote error: tls: bad certificate
member c920522ba9a75e17 is unreachable: [https://192.168.56.41:2379] are all unreachable
cluster is healthy
```
这次的错误是证书错误，这是因为客户端没有提供任何证书。

### 6.2 创建客户端证书

创建客户端证书请求文件所需配置:
```
# cfssl print-defaults csr > etcdctl-csr.json
# vi etcdctl.json
```
修改后内容如下，etcdctl可能运行在多台节点上，因此此处不指定可以使用该证书的主机名列表。
```
# cat etcdctl.json
{
    "CN": "ETCDCTL",
    "key": {
        "algo": "ecdsa",
        "size": 256
    },
    "names": [
        {
            "C": "US",
            "L": "CA",
            "ST": "San Francisco"
        }
    ]
}

```
运行以下指令生成证书及其私钥：
```
# cfssl gencert -ca=rootca.pem -ca-key=rootca-key.pem -config=ca-config.json -profile=client etcdctl-csr.json | cfssljson -bare etcdctl
```
然后在etcdctl命令行中指定上面生成的证书和私钥，才能成功访问节点。
```
# etcdctl --ca-file /etc/etcd/pki/rootca.pem --cert-file /etc/etcd/pki/etcdctl.pem --key-file /etc/etcd/pki/etcdctl-key.pem cluster-health
member 6bf749bb3ab16843 is healthy: got healthy result from https://192.168.56.43:2379
member 7c1dfc5e13a8008a is healthy: got healthy result from https://192.168.56.42:2379
member c920522ba9a75e17 is healthy: got healthy result from https://192.168.56.41:2379
cluster is healthy
```

### 6.3 修改其他节点的客户端认证配置

把根CA证书拷贝到集群的所有节点当中：
```
# scp /etc/etcd/pki/rootca.pem k8s2:/etc/etcd/pki/rootca.pem
# scp /etc/etcd/pki/rootca.pem k8s3:/etc/etcd/pki/rootca.pem
```
然后参考6.1修改配置重启服务。

## 7. 集群间通信peer认证

为3个节点创建peer证书请求文件所需配置，为了节省篇幅，这里给3个节点配同样的peer证书因此把3个节点的ip都放到hosts列表中，但这不是一种好的实现方式。

考虑集群扩容的情况，每次增加节点都要在hosts列表中新增主机，重新生成证书，重新配置到集群现有节点上，对现有节点产生影响。
```
# cfssl print-defaults csr > etcd-peer-csr.json
# vi etcd-peer.json
```
修改后内容如下
```
# cat etcd-peer.json
{
    "CN": "ETCD Peer",
    "hosts": [
        "192.168.56.41",
        "192.168.56.42",
        "192.168.56.43"
    ],
    "key": {
        "algo": "ecdsa",
        "size": 256
    },
    "names": [
        {
            "C": "US",
            "L": "CA",
            "ST": "San Francisco"
        }
    ]
}
```

创建服务器peer证书, 并拷贝到其他节点上：
```
# cfssl gencert -ca=rootca.pem -ca-key=rootca-key.pem -config=ca-config.json -profile=peer etcd-peer-csr.json | cfssljson -bare etcd-peer
# scp /etc/etcd/pki/etcd-peer* k8s2:/etc/etcd/pki/
# scp /etc/etcd/pki/etcd-peer* k8s3:/etc/etcd/pki/
```

在所有节点上修改etcd配置文件，将peer的url修改为https，配置相关证书，涉及参数如下：
```
ETCD_LISTEN_PEER_URLS="https://192.168.56.41:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.56.41:2380"
ETCD_INITIAL_CLUSTER="k8s1=https://192.168.56.41:2380,k8s2=https://192.168.56.42:2380,k8s3=https://192.168.56.43:2380"
ETCD_PEER_CERT_FILE="/etc/etcd/pki/etcd1-peer.pem"
ETCD_PEER_KEY_FILE="/etc/etcd/pki/etcd1-peer-key.pem"
ETCD_PEER_CLIENT_CERT_AUTH="true"
ETCD_PEER_TRUSTED_CA_FILE="/etc/etcd/pki/rootca.pem"
```

**注意：集群通信所需的信息包括证均存储在数据目录中，因此修改集群配置后，必须重建集群才能生效，这需要删除原来的数据目录，使得数据丢失，因此集群配置应该在第一次部署时配好，不适合在日常维护中频繁修改**
