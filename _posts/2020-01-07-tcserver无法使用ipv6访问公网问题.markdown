---
layout: post
title:  "tcserver无法使用ipv6访问公网问题"
date:   2020-01-07 15:37:36 +0530
description: 网络问题排查案例分析
categories: NetworkIssue NetworkTroubleshooting
---

_vxlan环境下ipv6部署遇到的问题_

### 问题描述

tc机房开启ipv6的服务器能够正常生成ipv6 global ip，但是无法ping通公网。

![图1](https://raw.githubusercontent.com/NetprogDong/image_repo/master/image_blog/B3D4EC31-FF95-4824-BEF0-51912D263C37.png "图1")

### 网络拓扑

![图2](https://raw.githubusercontent.com/NetprogDong/image_repo/master/image_blog/995284BB-AEE3-4D2D-8AE9-884BC32DE4C3.png "图2")

### 排查过程

1.首先查看tcserver的ipv6路由表项

```
[zhaodong3@localhost ~]$ ip -6 route
default via fe80::1 dev eth0  proto ra  metric 1024  expires 1391sec hoplimit 64
default via fe80::1 dev eth1  proto ra  metric 1024  expires 1257sec hoplimit 64
```

可以看到，服务器通过eth0和eth1两个网卡都收到了ra报文，并生成了两条等价默认路由分别指到eth0和eth1。又由于eth1上联的是internal内网，内网与外网分别在两个不同的instance实例中，所以访问公网的报文一旦选择从eth1出口发出，必然不能访问公网。

2.关闭交换机对应的internal vsi的ra报文发送功能

```
interface vsi xxxxx
  ipv6 nd ra halt
```

再次查看服务器ipv6路由表项

```
[zhaodong3@localhost ~]$ ip -6 route
default via fe80::1 dev eth0  proto ra  metric 1024  expires 1391sec hoplimit 64
```

表项正常，当前默认路由只走external接口eth0。

3.做ping测试

发现仍然无法ping通对端，在对端服务器抓包能抓到icmp request和相对应的icmp reply发出。

抓包现象表明，源端的报文能够正常发出并且已经到达对端，对端也正常做了回包，说明报文丢在了远端回土城机房的方向。

4.查看br路由表

由于土城机房的ipv6地址早已在土城ISP做宣告（ISP静态代播），因此报文丢弃在运营商的可能性不大。

查看br路由表，发现br上没有tcserver的明细128位路由，但是64位网段路由学习正常，下一跳递归到leaf9（vxlan分布式网关环境，所有leaf都会发布服务器网段路由，br根据bgp选路原则优选leaf9）。

```
<br3>dis ipv routing-table vpn-instance external 2400:89c0:1012:51::122:1 128
Summary count : 5
Destination: 2000::/3                                    Protocol  : BGP4+
NextHop    : 2400:89C0:1010::1:7:13B                     Preference: 200
Interface  : XGE1/0/43.200                               Cost      : 0
Destination: 2000::/3                                    Protocol  : BGP4+
NextHop    : 2400:89C0:1010::1:7:14B                     Preference: 200
Interface  : XGE1/0/44.200                               Cost      : 0
Destination: 2400:89C0:1012::/48                         Protocol  : BGP4+
NextHop    : 2400:89C0:1010::1:7:13B                     Preference: 200
Interface  : XGE1/0/43.200                               Cost      : 0
Destination: 2400:89C0:1012::/48                         Protocol  : BGP4+
NextHop    : 2400:89C0:1010::1:7:14B                     Preference: 200
Interface  : XGE1/0/44.200                               Cost      : 0
Destination: 2400:89C0:1012:51::/64                      Protocol  : BGP4+
NextHop    : ::FFFF:10.101.1.9                           Preference: 200
Interface  : Vsi12000                                    Cost      : 0
```

查看tcserver直连leaf12的ipv6 neighbor表项，发现学不到tcserver的nd信息。

因为上述原因，导致leaf12无法将tcserver的nd表项转化为evpn路由，进而导致br上学不到tcserver明细路由。

5.分析leaf12学不到tcserver ipv6 neighbor表项的原因

按照正常流程，公网回包到达br后，br通过/64大段路由会将报文经过vxlan封装后发给leaf9，leaf9将报文解封装后查instance路由表，主动向目的ip所在vsi内广播ns消息查找tcserver的nd表项（广播范围包括leaf9本地和所有包含该vsi的远端leaf），广播的ns经过vxlan tunnel到达leaf14，leaf14再将报文广播到本地vsi接口。此时，tcserver收到ns报文后，回复na报文给leaf14，leaf14就应该有tcserver的nd表项了。

接下来按照正常的推理过程，在tcserver上抓包，并没有抓到相关的ns消息。

6.分析ns报文为什么没发到tcserver

1）首先，如拓扑图中标示，在[1]端口入方向流统，查看是否能够收到远端发回的icmp reply，结果是能够抓到，证明br已经正常将reply转发到leaf9。

2）在leaf12的[2]端口入方向流统，查看是否能够收到leaf9发来的ns消息，结果是能抓到，证明leaf9已经正常广播泛洪ns消息。

3）在leaf12与服务器直连的[3]端口出方向流统，查看是否发送了ns消息，结果是并没有抓到，证明leaf12并没有将ns报文泛洪到本地其他端口，问题出在leaf12。

4）尝试在leaf12的vsi接口上开启ipv6 nd抑制功能。

```
vsi xxxxx
  ipv6 nd suppression enable
```

开启该功能后，ns能够正确发送，tcserver收到ns并回复na后，leaf就能够正常学习到tcserver的nd表项，问题解决。

### 故障原因

排查过程6.2中leaf12收到leaf9发的ns消息报文的源mac地址使用leaf9 vsi接口下配置的mac地址，由于分布式网关的部署模式，leaf12的vsi接口配置相同的mac地址。

```
interface Vsi-interface12127
  description VXLAN_12127_Gateway
  ip binding vpn-instance external
  ip address x.x.x.x 255.255.255.0
  mac-address 0000-0001-2127
  ipv6 address FE80::1 link-local
  ipv6 address 2400:89C0:1012:51::x/64
  undo ipv6 nd ra halt
  distributed-gateway local
```

然而，H3C交换机对收到的arp/ns消息会首先在硬件层面做一个合法性判断，判断的内容为收到的arp/ns报文的源mac是否与自己的mac相同，如果相同，认为不合法，直接将报文丢弃。leaf12收到leaf9发来的ns后检查源mac与自身mac相同，于是丢弃，从而造成了tcserver无法收到ns消息，不能触发na答复，进而导致leaf12学不到tcserver的nd信息。

在开启ipv6 nd抑制功能后，nd学习过程上送软件层面处理，软件层面无该合法性检查，于是故障恢复。在询问H3C研发确认开启ipv6 nd抑制无风险后，全网开启该功能来规避该问题。

### 解决方案

当前vxlan网络环境中:

1. 在internal vsi中关闭ipv6功能或将ipv6发送ra消息功能关闭。
2. 在external vsi中开启ipv6 nd泛洪抑制功能。
