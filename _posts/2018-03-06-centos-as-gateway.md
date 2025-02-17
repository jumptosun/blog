---
layout: post
title:  "centos做网关配置"
date:   2018-03-06
categories: centos
---

## 1. 网卡配置

```
[root@localhost ~]# cat  /etc/sysconfig/network
NETWORKING=yes
HOSTNAME=beijing.verycloud.cn

[root@localhost ~]# cat /etc/sysconfig/network-scripts/ifcfg-enp1s0f0
BOOTPROTO=static
ONBOOT=yes
NM_CONTROLLED=yes
IPADDR=119.2.12.178
NETMASK=255.255.255.248
GATEWAY=119.2.12.177
DNS1=219.141.136.10
DNS2=219.141.141.10

[root@localhost ~]# cat /etc/sysconfig/network-scripts/ifcfg-enp1s0f1 
BOOTPROTO=static
ONBOOT=yes
NM_CONTROLLED=YES
IPADDR=192.168.100.178
NETMASK=255.255.255.0
```

## 2. 转发开启，Nat配置

```
[root@localhost sysctl.d]# cat /etc/sysctl.d/99-sysctl.conf 
net.ipv4.ip_forward=1

# Nat转换, iptable安装启动
    iptables -F  //清除
    iptables -t nat -A POSTROUTING -o enp1s0f0 -j MASQUERADE // 内网到外网的NAT
    iptables -t filter -A FORWARD -i enp1s0f0 -o enp1s0f1 -j ACCEPT
    iptables -t filter -A FORWARD -i enp1s0f1 -o enp1s0f0 -j ACCEPT
```

## 3. 路由设置

```
# Centos 7
[root@localhost sysctl.d]# cat /etc/sysconfig/network-scripts/route-enp1s0f0
119.2.12.176/29	via	0.0.0.0	dev	enp1s0f0
0.0.0.0/0	via	119.2.12.177	dev	enp1s0f0

[root@localhost sysctl.d]# cat /etc/sysconfig/network-scripts/route-enp1s0f1
192.168.0.0/16   via	192.168.100.1	dev	enp1s0f1
192.168.100.0/24	via	0.0.0.0	dev	enp1s0f

# Centos 6
[root@very-route ~]# cat /etc/rc.local
route add -net 192.168.0.0/16 gw 192.168.100.1 metric 1

[root@very-route ~]# cat /etc/sysconfig/network
NETWORKING=yes
HOSTNAME=very-route

```

## 4. 配置 DNS

```
[root@very-route ~]# cat /etc/resolv.conf
nameserver 202.106.0.20
nameserver 114.114.114.114
```

## 5. 安装配置denyhost
