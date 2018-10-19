---
layout: post
title: keepalived和lvs搭建负载均衡器
key: 20160720
tags: keepalived lvs LoadBalancer
modify_date: 2016-08-25
---



<!--more-->

## 0. 搭建环境

### 0.1 启动虚拟机

### 0.2 安装http服务用于测试

```
yum install -y httpd
systemctl start httpd && systemctl enable httpd
```

修改/var/www/html/index.html文件，使得每个服务器返回主机名，这样在客户端就可以根据响应的内容知道请求是真正被哪个服务器处理的。
```
# cat > /var/www/html/index.html <<EOF
<h1>$(hostname)<h1>
EOF
```

## 1. 安装ipvsadm

lvs由两部分组成，一部分是linux内核中的ipvs，另外一部分是用于管理的ipvsadm。
安装ipvsadm步骤如下：
```
yum -y install -y ipvsadm
echo 'net.ipv4.ip_forward = 1' >> /etc/sysctl.conf
sysctl -p
touch /etc/sysconfig/ipvsadm
systemctl start ipvsadm && systemctl enable ipvsadm
```

```
# ipvsadm -A -t 192.168.56.40:80 -s rr
# ipvsadm -a -t 192.168.56.40:80 -r 192.168.56.41:80 -g
# ipvsadm -a -t 192.168.56.40:80 -r 192.168.56.42:80 -g
# ipvsadm -a -t 192.168.56.40:80 -r 192.168.56.43:80 -g
# ip addr add 192.168.56.40/24 dev eth1
# ip addr add 192.168.56.40 dev lo
```

yum -y install keepalived
vi /etc/keepalived/keepalived.conf
systemctl start keepalived && systemctl enable keepalived


keepalived包括两个功能：
1. VRRP提供一个虚拟的IP
提供了虚拟IP之后，只实现了高可用，并不具备负载均衡功能，因为虚拟IP始终绑定在某一台服务器上，该服务器会处理所有请求，只要当该服务器不可用时，才会切换到另外一台服务器上。
如果需要实现负载均衡，那这个虚拟IP所在的服务器应该是一个负载均衡器，该负载均衡器是lvs，ha-proxy，nginx等均有转发代理功能的程序，由这些程序负责将请求转发到后端服务器上处理。

2. 对ipvs进行配置提供负载均衡

因此，如果有其他组件如ha-proxy或者nginx提供了负载均衡功能，那可以只启用keepalived的VRRP。

### DR模式
DR的意思是直接路由模式（Direct Routing），顾名思义，DR模式中，Director像一个路由器一样转发数据包，路由器转发的步骤如下：
1. 根据路由表找到下一跳的ip地址和所在网口，即下一跳ip所在同一网段的网口。
2. 使用arp根据ip地址找到下一跳的mac地址。
3. 修改数据包中的mac地址，将源mac地址修改为第1步中找到的网口mac，将目的mac地址修改为下一跳的mac。
4. 将数据包通过第1步中找到的网口转发出去。

而在DR模式中Director选中RealServer之后，Director也将数据包的mac地址修改为与RealServer相连的网口mac，将目的mac地址修改为选中的RealServer网口mac，然后将数据包转发出去。

DR模式下需要把虚拟IP配置到每个RealServer上：
```
# ip addr add 192.168.56.40 dev lo
```
目的在于让RealServer接收地址为虚拟IP的数据包，单就这个目的而言，可以把虚拟ip配置在任意一个网口上，但是除此之外，我们还希望RealServer不要响应对这个IP的arp请求(否则可能会和负载均衡器的对外接口冲突)，因此一般将其配置在lo网口上。另外，可设置arp内核参数抑制arp应答的发送。

直接操作ipvs命令
```
# ipvsadm -A -t 192.168.56.40:80 -s rr
# ipvsadm -a -t 192.168.56.40:80 -r 192.168.56.41:80 -g
# ipvsadm -a -t 192.168.56.40:80 -r 192.168.56.42:80 -g
# ipvsadm -a -t 192.168.56.40:80 -r 192.168.56.43:80 -g
```

在DR模式下，LB收到的请求包和RS收到的请求包基本一模一样，唯一不同的是mac地址。因此当LB和RS部署在同一台服务器上的时候，会出现请求包被不断转发的而不被处理的现象。
lvs提供了基于firewall mark创建virtual-server的机制可以绕过这个问题。
解决思路是通过iptables为符合条件的数据包打上标记，然后lvs根据这个数据包进行转发，而不是简单的根据ip和端口转发。这样可以使得从外部来的数据包（源mac地址和三台主机均不相同）才被转发，而内部转发的数据包（源mac地址是三台主机之一）不被转发。
iptables -t mangle -A PREROUTING -i eth1 -p tcp -m mac ! --mac-source 08:00:27:e1:80:cc -d 192.168.56.40 --dport http -j MARK --set-mark 1234
virtual_server fwmark 1234
