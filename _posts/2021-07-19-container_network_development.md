---
layout: post
title: 容器网络发展
subtitle: ""
catalog: true
hide: true
tags:
- k8s

---

[toc]

## 1. 容器网络基础

### 1.1 network namespace

Network Namespace （以下简称netns）是Linux内核提供的一项实现网络隔离的功能，它能隔离多个不同的网络空间，并且各自拥有独立的网络协议栈，这其中便包括了网络接口（网卡），路由表，iptables规则等。例如大名鼎鼎的docker便是基于netns实现的网络隔离。


![100%](https://segmentfault.com/img/remote/1460000024474310)



### 1.2 bridge

网桥是一个二层的虚拟网络设备，把若干个网络接口“连接”起来，使得网口之间的报文可以转发。网桥能够解析收发的报文，读取目标的Mac地址信息，和自己的Mac地址表结合，来决策报文转发的目标网口。为了实现这些功能，网桥会学习源Mac地址。在转发报文时，网桥只需要向特定的端口转发，从而避免不必要的网络交互。如果它遇到了一个自己从未学过的地址，就无法知道这个报文应该向哪个网口转发，就将报文广播给除了报文来源之外的所有网口。


![100%](https://segmentfault.com/img/remote/1460000024474312)



### 1.3 arp

ARP（Address Resolution Protocol）即地址解析协议， 用于实现从 IP 地址到 MAC 地址的映射，即询问目标IP对应的MAC地址。



![80%](http://cache.yisu.com/upload/admin/customer_case_img/2019-06-20/1561028184.jpg)

 

linux查看arp缓存

```bash
# ip neigh show
172.20.0.17 dev v-h28edb3920 lladdr ee:7a:dd:24:73:ee REACHABLE
172.17.0.2 dev docker0 lladdr 02:42:ac:11:00:02 STALE
```



### 1.4 veth

Veth Pair设备的特点是：它被创建出来后，总是以两张虚拟网卡（Veth Peer）的形式出现。并且，其中一个网卡发出的数据包，可以直接出现在另一张“网卡”上，哪怕这两张网卡在不同的Network Namespace中。

 


![100%](https://segmentfault.com/img/remote/1460000024474313)

 

### 1.5 iptables/netfilter

netfilter/iptables（简称为iptables）组成Linux平台下的包过滤防火墙，与大多数的Linux软件一样，这个包过滤防火墙是免费的，它可以代替昂贵的商业防火墙解决方案，完成封包过滤、封包重定向和网络地址转换（NAT）等功能。

 

iptables 5张表5条链

5张表:

- filter: 过滤某个链上的数据包
- nat: 网络地址转换，转换数据包的源地址和目的地址
- mangle: 修改数据包的IP头部信息
- raw: iptables本身是有状态的，对数据包有链接追踪(/proc/net/nf_conntrack可看到)，raw 可以用来消除这种追踪机制
- security: 数据包上使用selinux

 

5条链:

- PREROUTING: 数据包进入内核网络模块之后, 获得路由之前(可进行DNAT)
- INPUT: 数据包被决定路由到本机之后, 一般处理本地进程的数据包, 目的地址是本机
- FORWARD: 数据包被决定路由到其他主机之后, 一般处理转发到其它主机/网络命名空间的数据包
- OUTPUT: 离开本机的数据包进入内核网络模块之后, 一般处理本地进程的输出数据包, 源地址是本机
- POSTROUTING: 对于离开本机或者FORWARD的数据包, 当数据包被发送到网卡之前(可进行SNAT)

 

![Fit](http://www.iceyao.com.cn/img/posts/2020-02-19/1.png)

 

### 1.6 tc

tc是Linux内核提供的流量限速、整形和策略控制机制。它以`qdisc-class-filter`的树形结构来实现对流量的分层控制 ：

tc由`qdisc`、`fitler`和`class`三部分组成：

- `qdisc`通过队列将数据包缓存起来，用来控制网络收发的速度
- `class`用来表示控制策略
- `filter`用来将数据包划分到具体的控制策略中

 

网络流量的控制通常发生在输出网卡处，Linux内核中由TC(Traffic Control)实现。TC是利用队列规定建立处理数据包的队列，并定义队列中的数据包被发送的方式，从而实现流量控制。基本原理：

![inline](https://www.pianshen.com/images/877/4168e0283f247d275fd8b864219b014d.png)

 

### 1.7 路由

#### 1.7.1 静态路由

静态路由是一种路由的方式，路由项由手动配置，而非动态决定。

![inline](https://pic1.xuehuaimg.com/proxy/csdn/https://img-blog.csdnimg.cn/20190806171213339.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1BhcmhvaWE=,size_16,color_FFFFFF,t_70)

 

linux系统上的路由表

```bash
[root@lianbang-xuexi-server24 ~]# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         172.16.87.1     0.0.0.0         UG    100    0        0 ens160
10.12.0.2       0.0.0.0         255.255.255.255 UH    0      0        0 v-hd976c297e
10.12.1.0       10.12.1.0       255.255.255.0   UG    0      0        0 flannel.1
```



发到127.0.0.1的包怎么没在`route -n`的路由表体现？

```bash
[root@lianbang-xuexi-server24 ~]# ip route show table local
local 10.12.0.0 dev flannel.1 proto kernel scope host src 10.12.0.0
broadcast 127.0.0.0 dev lo proto kernel scope link src 127.0.0.1
local 127.0.0.0/8 dev lo proto kernel scope host src 127.0.0.1
local 127.0.0.1 dev lo proto kernel scope host src 127.0.0.1
broadcast 127.255.255.255 dev lo proto kernel scope link src 127.0.0.1
broadcast 172.16.87.0 dev ens160 proto kernel scope link src 172.16.87.24
local 172.16.87.24 dev ens160 proto kernel scope host src 172.16.87.24
broadcast 172.16.87.255 dev ens160 proto kernel scope link src 172.16.87.24
broadcast 172.17.0.0 dev docker0 proto kernel scope link src 172.17.0.1
local 172.17.0.1 dev docker0 proto kernel scope host src 172.17.0.1
broadcast 172.17.255.255 dev docker0 proto kernel scope link src 172.17.0.1
```



#### 1.7.2 动态路由

##### 1.7.2.1 RIP

RIP(Routing Information Protocol,[路由信息协议](https://baike.baidu.com/item/路由信息协议/2707187)）是一种[内部网关协议](https://baike.baidu.com/item/内部网关协议/167192)（IGP），是一种[动态路由选择](https://baike.baidu.com/item/动态路由选择/1250467)协议，用于自治系统（AS）内的路由信息的传递。RIP协议基于距离矢量算法（DistanceVectorAlgorithms），使用“跳数”(即metric)来衡量到达目标地址的路由距离。这种协议的[路由器](https://baike.baidu.com/item/路由器/108294)只关心自己周围的[世界](https://baike.baidu.com/item/世界/24458)，只与自己相邻的路由器交换信息，范围限制在15跳(15度)之内，再远，它就不关心了。

 

![100%](https://s4.51cto.com/images/blog/201806/03/506fdf4f4e4f6cade2dbfd55308f89de.jpg?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)

 

##### 1.7.2.2 OSPF

开放式最短路径优先（Open Shortest Path First，OSPF）是广泛使用的一种动态路由协议，它属于链路状态路由协议，具有路由变化收敛速度快、无路由环路、支持变长子网掩码（VLSM）和汇总、层次区域划分等优点。

![inline](https://img2020.cnblogs.com/blog/2050139/202007/2050139-20200701193908006-1290678158.png)

 

##### 1.7.2.3 BGP

BGP（边界网关协议）协议主要用于互联网AS（自治系统）之间的互联，BGP的最主要功能在于控制路由的传播和选择最好的路由。

![inline](https://img-blog.csdnimg.cn/20191025114515745.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N0YXJ0ZXJfX19fXw==,size_16,color_FFFFFF,t_70)

 

|         区别          |                OSPF协议                |               BGP协议                |
| :      -: | :            : | :           -: |
|       网关协议        |              内部网关协议              |             外部网关协议             |
| 消除生成树协议（STP） |                   是                   |                  是                  |
|         配置          |                  简单                  |                 复杂                 |
|       收敛时间        |                   短                   |                  长                  |
|       网络结构        |            分层网络拓扑设计            |             网状拓扑设计             |
|     所需设备资源      |        大量内存与密集型CPU资源         | 其路由表的大小决定了其所需的设备资源 |
|       网络规模        | 主要是用于较小规模的网络，可以集中管理 |     通常用于大型网络，例如互联网     |
|         功能          |     最快路由路径优先于最短路由路径     |           确定最佳路由路径           |
|      使用的算法       |              Dijkstra算法              |             最佳路径算法             |
|         协议          |            互联网协议（IP）            |         传输控制协议（TCP）          |

 

#### 1.7.3 策略路由

基于策略的路由比传统路由在功能上更强大，使用更灵活，它使网络管理员不仅能够根据目的地址而且能够根据报文大小，应用或IP源地址等属性来选择转发路径。

```bash
ip rule命令：

Usage: ip rule [ list | add | del ] SELECTOR ACTION （add 添加；del 删除； llist 列表）
 
SELECTOR := [ from PREFIX 数据包源地址] [ to PREFIX 数据包目的地址] [ tos TOS 服务类型]
            [ dev STRING 物理接口] [ pref NUMBER ] [fwmark MARK iptables 标签]
 
ACTION := [ table TABLE_ID 指定所使用的路由表] [ nat ADDRESS 网络地址转换]
          [ prohibit 丢弃该表| reject 拒绝该包| unreachable 丢弃该包]
 
[ flowid CLASSID ]
 
TABLE_ID := [ local | main | default | new | NUMBER ]
```

规则指向路由表，多个规则可以引用一个路由表



```bash
#把源地址为1.2.3.4的数据报的源地址转换为11.22.33.44，并通过表1进行路由
ip rule add from 1.2.3.4 nat 11.22.33.44 table 1 prio 320
```



查看所有策略路由

```bash
# ip rule show 
0:      from all lookup local 
32766:  from all lookup main 
32767:  from all lookup default 
```



查看所有路由表

```bash
[root@lianbang-xuexi-server24 ~]# cat /etc/iproute2/rt_tables
#
# reserved values
#
255	local
254	main
253	default
0	unspec
#
# local
#
#1	inr.ruhep
```



### 1.8 tunnel

Overlay网络是建立在已有物理网络上的虚拟网络，具有独立的控制和转发平面，对于连接到Overlay的终端设备（例如服务器）来说，物理网络是透明的



#### 1.8.1 gre

gre: 通用路由封装（Generic Routing Encapsulation），定义了在任意网络层协议上封装其他网络层协议的机制，所以对于 IPv4 和 IPv6 都适用

![GRE、PPTP、L2TP隧道协议对比介绍](http://www.m2mlib.com/uploads/article/20171106/595f09271b77def22f78d65debed2c8c.png)

 

gre的缺陷：

（1）隧道数量问题

gre是一种点对点实现。所有节点之间都会建立 GRE Tunnel。节点不多的时候，没什么问题。

（2）扩大的广播域

GRE 不支持組播，一个网络（同一个 GRE Tunnel ID）中的一个虚拟机发出一个广播包后，gre会将其广播到所有与该节点有隧道连接的节点

（3）GRE 封裝的IP包的过滤和负载均衡问题

目前还是有很多的防火墙和三层网络设备无法解析 GRE Header，因此它們无法对GRE 封裝包做合适的过滤和负载均衡

 

#### 1.8.2 vxlan

VXLAN（Virtual eXtensible Local Area Network）或许是目前最热门的网络虚拟化技术。网络虚拟化是指在一套物理网络设备上虚拟出多个二层网络。VXLAN由RFC7348定义，这是2014年定稿的一个协议，VXLAN协议将Ethernet帧封装在UDP内，再加上8个字节的VXLAN header，用来标识不同的二层网络。



![What Is VXLAN - Huawei](https://download.huawei.com/mdl/image/download?uuid=7cbc9dea85ca497a8dc89f31d44ccd83)

vxlan通过组播的方式来定位节点

 

#### 1.8.3 ipip

ipip: 普通的 IPIP 隧道，就是在报文的基础上再封装成一个 IPv4 报文

![network-ipip-2.jpg](https://i.loli.net/2020/01/29/95RPciVgDqtSmNb.jpg)

ipip vs vxlan ?

 

## 2. docker网络

Docker 网络架构由 3 个主要部分构成：CNM、Libnetwork 和驱动。

- CNM 是设计标准。在 CNM 中，规定了 Docker 网络架构的基础组成要素。
- Libnetwork 是 CNM 的具体实现，并且被 Docker 采用，Libnetwork 通过 Go 语言编写，并实现了 CNM 中列举的核心组件。
- 驱动通过实现特定网络拓扑的方式来拓展该模型的能力



![inline](http://c.biancheng.net/uploads/allimg/190418/4-1Z41Q55633c8.gif)

 

CNM 定义了 3 个基本要素：沙盒（Sandbox）、终端（Endpoint）和网络（Network）。

- 沙盒是一个独立的网络栈。其中包括以太网接口、端口、路由表以及 DNS 配置。
- 终端就是虚拟网络接口。就像普通网络接口一样，终端主要职责是负责创建连接。在 CNM 中，终端负责将沙盒连接到网络。
- 网络是 802.1d 网桥（类似大家熟知的交换机）的软件实现。因此，网络就是需要交互的终端的集合，并且终端之间相互独立

![CNM](http://c.biancheng.net/uploads/allimg/190418/4-1Z41Q55GJ60.gif)

![控制层、管理层与数据层的关系](http://c.biancheng.net/uploads/allimg/190418/4-1Z41Q60231L7.gif)



### 2.1 单机通信



#### 2.1.1 bridge模式



![img](https://img2018.cnblogs.com/blog/1254238/201811/1254238-20181120223907788-1986792101.png)



#### 2.1.2 container模式

docker在创建容器的时候会指定使用已经存在的容器的网卡设备作为新建容器的网卡设备。这中模式需要注意，由于是多个容器共用同一个eth0

![img](https://img2018.cnblogs.com/blog/1254238/201811/1254238-20181120230818639-1413560441.png)



#### 2.1.3 host模式

![img](https://img2018.cnblogs.com/blog/1254238/201811/1254238-20181120231410520-860628530.png)



#### 2.1.4 none模式

这种模式下，容器无法与外界通信，只能使用容器内部的回环(127.0.0.1)在容器内部通信。

![img](https://img2018.cnblogs.com/blog/1254238/201811/1254238-20181120234353025-1330295419.png)



### 2.2 跨主机通信



![图片](http://mmbiz.qpic.cn/mmbiz_png/Hia4HVYXRicqHpH8y0uwPo7xIHs4UP99pxFCYRJFib59Gqbvn3gP3J3EHCjqhOWaSI4VNmqMwoE38Mrqn5Zat2icwQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





## 3. k8s网络



### 3.1 service

![Kubernetes NodePort vs LoadBalancer vs Ingress? When should I use what? |  by Sandeep Dinesh | Google Cloud - Community | Medium](https://miro.medium.com/max/1400/1*CdyUtG-8CfGu2oFC5s0KwA.png)

service的更新是由k8s哪个组件负责？



####  3.1.1 userspace模式

![Services overview diagram for userspace proxy](https://d33wubrfki0l68.cloudfront.net/e351b830334b8622a700a8da6568cb081c464a9b/13020/images/docs/services-userspace-overview.svg)

什么场景用到了这种模式？



#### 3.1.2 iptables模式

 ![Services overview diagram for iptables proxy](https://d33wubrfki0l68.cloudfront.net/27b2978647a8d7bdc2a96b213f0c0d3242ef9ce0/e8c9b/images/docs/services-iptables-overview.svg)



#### 3.1.3 ipvs模式

![Services overview diagram for IPVS proxy](https://d33wubrfki0l68.cloudfront.net/2d3d2b521cf7f9ff83238218dac1c019c270b1ed/9ac5c/images/docs/services-ipvs-overview.svg)

ipvs vs iptables?



### 3.2 ingress



![iT 邦幫忙::一起幫忙解決難題，拯救IT 人的一天](https://miro.medium.com/max/3970/1*KIVa4hUVZxg-8Ncabo8pdg.png)

从外部访问ingress，流量是从ingress -> service -> pod吗？



#### 3.2.1 ingress controller对比



![img](https://pic2.zhimg.com/80/v2-e11d1a3fef7f0bab2ce64a8f721360bd_1440w.jpg)



### 3.3 cni

CNI（Container Network Interface）是 CNCF 旗下的一个项目，由一组用于配置 Linux 容器的网络接口的规范和库组成，同时还包含了一些插件。CNI 仅关心容器创建时的网络分配，和当容器被删除时释放网络资源。

https://github.com/containernetworking/plugins/tree/master/plugins

![img](https://feisky.gitbooks.io/sdn/content/container/cni/Chart_Container-Network-Interface-Drivers.png)



![1.png](https://ucc.alicdn.com/pic/developer-ecology/eb89cb969a514b2e9b2bdcf6273f31f9.png)



#### 3.3.1 Flannel

Flannel是CoreOS团队针对Kubernetes设计的一个网络规划服务



flannel整体架构：

<a href="https://ibb.co/DfCr885"><img src="https://i.ibb.co/4RT1YYP/image.png" alt="image" border="0" /></a>



##### 3.3.1.1 udp

多出了flanneld的处理过程，这个过程，使用了flannel0这个TUN设备，仅在发出IP包的过程中就要经过了三次用户态到内核态的数据拷贝（linux的上下文切换代价比较大），所以性能非常差

![img](https://img2018.cnblogs.com/blog/662544/201910/662544-20191022103006305-198547277.png)



##### 3.3.1.2 host-gw

![img](https://img2018.cnblogs.com/blog/662544/201910/662544-20191022103224393-1472403318.png)

节点处于同一个大二层网络



##### 3.3.1.2 vxlan

![img](https://img2018.cnblogs.com/blog/662544/201910/662544-20191022103037567-147724494.png)

vxlan模式的directrouting模式，相当于vxlan+host-gw的结合体



##### 3.3.1.3 ipip

性能优于纯vxlan模式，ipip的报头比vxlan报头小





##### 3.3.1.4 ipsec

加密pod间通信



####  3.3.2 Calico

[Calico](https://www.projectcalico.org/) 是一个纯三层的数据中心网络方案（不需要 Overlay），并且与 OpenStack、Kubernetes、AWS、GCE 等 IaaS 和容器平台都有良好的集成。

Calico 在每一个计算节点利用 Linux Kernel 实现了一个高效的 vRouter 来负责数据转发，而每个 vRouter 通过 BGP 协议负责把自己上运行的 workload 的路由信息像整个 Calico 网络内传播——小规模部署可以直接互联，大规模下可通过指定的 BGP route reflector 来完成。 这样保证最终所有的 workload 之间的数据流量都是通过 IP 路由的方式完成互联的。Calico 节点组网可以直接利用数据中心的网络结构（无论是 L2 或者 L3），不需要额外的 NAT，隧道或者 Overlay Network。

此外，Calico 基于 iptables 还提供了丰富而灵活的网络 Policy，保证通过各个节点上的 ACLs 来提供 Workload 的多租户隔离、安全组以及其他可达性限制等功能。

![img](https://feisky.gitbooks.io/kubernetes/content/network/calico/calico.png)

Calico 主要由 Felix、etcd、BGP client 以及 BGP Route Reflector 组成

1. Felix，Calico Agent，跑在每台需要运行 Workload 的节点上，主要负责配置路由及 ACLs 等信息来确保 Endpoint 的连通状态；
2. etcd，分布式键值存储，主要负责网络元数据一致性，确保 Calico 网络状态的准确性；
3. BGP Client（BIRD）, 主要负责把 Felix 写入 Kernel 的路由信息分发到当前 Calico 网络，确保 Workload 间的通信的有效性；
4. BGP Route Reflector（BIRD），大规模部署时使用，摒弃所有节点互联的 mesh 模式，通过一个或者多个 BGP Route Reflector 来完成集中式的路由分发。
5. calico/calico-ipam，主要用作 Kubernetes 的 CNI 插件



#### 3.3.3 Antrea

[Antrea](https://github.com/vmware-tanzu/antrea) 项目是一个开源的联网解决方案，旨在成为 Kubernetes 原生的网络解决方案。它利用 Open vSwitch 作为网络数据平面。 Open vSwitch 是一个高性能可编程的虚拟交换机，支持 Linux 和 Windows 平台。 Open vSwitch 使 Antrea 能够以高性能和高效的方式实现 Kubernetes 的网络策略。 借助 Open vSwitch 可编程的特性，Antrea 能够在 Open vSwitch 之上实现广泛的联网、安全功能和服务。

<img src="https://github.com/antrea-io/antrea/blob/main/docs/assets/antrea_overview.svg.png?raw=true" alt="antrea_overview.svg.png" style="zoom:67%;" />

https://github.com/antrea-io/antrea/blob/main/docs/design/architecture.md





#### 3.3.4 AWS VPC CNI

[AWS VPC CNI](https://github.com/aws/amazon-vpc-cni-k8s) 为 Kubernetes 集群提供了集成的 AWS 虚拟私有云（VPC）网络。该 CNI 插件提供了高吞吐量和可用性，低延迟以及最小的网络抖动。 此外，用户可以使用现有的 AWS VPC 网络和安全最佳实践来构建 Kubernetes 集群。 这包括使用 VPC 流日志、VPC 路由策略和安全组进行网络流量隔离的功能。

使用该 CNI 插件，可使 Kubernetes Pod 拥有与在 VPC 网络上相同的 IP 地址。 CNI 将 AWS 弹性网络接口（ENI）分配给每个 Kubernetes 节点，并将每个 ENI 的辅助 IP 范围用于该节点上的 Pod 。 CNI 包含用于 ENI 和 IP 地址的预分配的控件，以便加快 Pod 的启动时间，并且能够支持多达 2000 个节点的大型集群。

CNI 可以与 [用于执行网络策略的 Calico](https://docs.aws.amazon.com/eks/latest/userguide/calico.html) 一起运行。





#### 3.3.5 Azure CNI

[Azure CNI](https://docs.microsoft.com/en-us/azure/virtual-network/container-networking-overview) 是一个[开源插件](https://github.com/Azure/azure-container-networking/blob/master/docs/cni.md)， 将 Kubernetes Pods 和 Azure 虚拟网络（也称为 VNet）集成在一起，可提供与 VM 相当的网络性能。 Pod 可以通过 Express Route 或者 站点到站点的 VPN 来连接到对等的 VNet ， 也可以从这些网络来直接访问 Pod。Pod 可以访问受服务端点或者受保护链接的 Azure 服务，比如存储和 SQL。 你可以使用 VNet 安全策略和路由来筛选 Pod 流量。 该插件通过利用在 Kubernetes 节点的网络接口上预分配的辅助 IP 池将 VNet 分配给 Pod 。





#### 3.3.3 Culium

[Cilium ](https://github.com/cilium/cilium)是一个基于 eBPF 和 XDP 的高性能容器网络方案，其主要功能特性包括

- 安全上，支持 L3/L4/L7 安全策略，这些策略按照使用方法又可以分为
  - 基于身份的安全策略（security identity）
  - 基于 CIDR 的安全策略
  - 基于标签的安全策略
- 网络上，支持三层平面网络（flat layer 3 network），如
  - 覆盖网络（Overlay），包括 VXLAN 和 Geneve 等
  - Linux 路由网络，包括原生的 Linux 路由和云服务商的高级网络路由等
- 提供基于 BPF 的负载均衡
- 提供便利的监控和排错能力

![https://cdn.jsdelivr.net/gh/cilium/cilium@master/Documentation/images/cilium_overview.png](https://camo.githubusercontent.com/714c5d777b0025dda66b46f14e28badc01e3e3360ef264be204f54846a7c9573/68747470733a2f2f63646e2e6a7364656c6976722e6e65742f67682f63696c69756d2f63696c69756d406d61737465722f446f63756d656e746174696f6e2f696d616765732f63696c69756d5f6f766572766965772e706e67)

##### 3.3.3.1 eBPF

eBPF（extended Berkeley Packet Filter）起源于BPF，它提供了内核的数据包过滤机制。BPF的基本思想是对用户提供两种SOCKET选项：`SO_ATTACH_FILTER`和`SO_ATTACH_BPF`，允许用户在sokcet上添加自定义的filter，只有满足该filter指定条件的数据包才会上发到用户空间。`SO_ATTACH_FILTER`插入的是cBPF代码，`SO_ATTACH_BPF`插入的是eBPF代码。eBPF是对cBPF的增强，目前用户端的tcpdump等程序还是用的cBPF版本，其加载到内核中后会被内核自动的转变为eBPF。Linux 3.15 开始引入 eBPF。其扩充了 BPF 的功能，丰富了指令集。它在内核提供了一个虚拟机，用户态将过滤规则以虚拟机指令的形式传递到内核，由内核根据这些指令来过滤网络数据包。

![img](https://feisky.gitbooks.io/sdn/content/container/cilium/bpf.png)



##### 3.3.3.2 XDP

XDP（eXpress Data Path）为Linux内核提供了高性能、可编程的网络数据路径。由于网络包在还未进入网络协议栈之前就处理，它给Linux网络带来了巨大的性能提升。XDP 看起来跟 DPDK 比较像，但它比 DPDK 有更多的优点，如

- 无需第三方代码库和许可
- 同时支持轮询式和中断式网络
- 无需分配大页
- 无需专用的CPU
- 无需定义新的安全网络模型

当然，XDP的性能提升是有代价的，它牺牲了通用型和公平性：（1）不提供缓存队列（qdisc），TX设备太慢时直接丢包，因而不要在RX比TX快的设备上使用XDP；（2）XDP程序是专用的，不具备网络协议栈的通用性。

https://github.com/cilium/cilium





#### 3.3.4 cni-ipvlan-vpc-k8s

[cni-ipvlan-vpc-k8s](https://github.com/lyft/cni-ipvlan-vpc-k8s) 包含了一组 CNI 和 IPAM 插件来提供一个简单的、本地主机、低延迟、高吞吐量 以及通过使用 Amazon 弹性网络接口（ENI）并使用 Linux 内核的 IPv2 驱动程序 以 L2 模式将 AWS 管理的 IP 绑定到 Pod 中， 在 Amazon Virtual Private Cloud（VPC）环境中为 Kubernetes 兼容的网络堆栈。

这些插件旨在直接在 VPC 中进行配置和部署，Kubelets 先启动， 然后根据需要进行自我配置和扩展它们的 IP 使用率，而无需经常建议复杂的管理 覆盖网络、BGP、禁用源/目标检查或调整 VPC 路由表以向每个主机提供每个实例子网的 复杂性（每个 VPC 限制为50-100个条目）。 简而言之，cni-ipvlan-vpc-k8s 大大降低了在 AWS 中大规模部署 Kubernetes 所需的网络复杂性。



#### 3.3.5 Contiv

[Contiv](https://github.com/contiv/netplugin) 为各种使用情况提供了一个可配置网络（使用了 BGP 的本地 L3， 使用 vxlan 、经典 L2 或 Cisco-SDN/ACI 的覆盖网络）。 [Contiv](https://contiv.io/) 是完全开源的。

- 为同一主机上的容器提供不重叠网络的多租户环境
- SDN 应用程序和与 SDN 解决方案的互操作性
- 与非容器环境的互操作性和到物理网络的切换
- 与容器相关的策略/ACL/QoS
- 多播或多目标相关应用程序
- 与现有 IPAM 工具集成以迁移客户
- 处理 NIC 的加速功能（SRIOV/Offload/等等）

https://github.com/contiv/netplugin



#### 3.3.6 kube-ovn

[Kube-OVN](https://github.com/alauda/kube-ovn) 是一个基于 OVN 的用于企业的 Kubernetes 网络架构。 借助于 OVN/OVS ，它提供了一些高级覆盖网络功能，例如子网、QoS、静态 IP 分配、流量镜像、网关、 基于 openflow 的网络策略和服务代理。

![topology](https://github.com/kubeovn/kube-ovn/raw/master/docs/ovn-network-topology.png)

多网卡模式(借助multus实现)

![topology](https://github.com/kubeovn/kube-ovn/raw/master/docs/multi-nic.png)





#### 3.3.7 kube-router

[Kube-router](https://github.com/cloudnativelabs/kube-router) 是 Kubernetes 的专用网络解决方案， 旨在提供高性能和易操作性。 Kube-router 提供了一个基于 Linux [LVS/IPVS](https://www.linuxvirtualserver.org/software/ipvs.html) 的服务代理、一个基于 Linux 内核转发的无覆盖 Pod-to-Pod 网络解决方案和基于 iptables/ipset 的网络策略执行器。

![Arch](https://github.com/cloudnativelabs/kube-router/raw/master/docs/img/kube-router-arch.png)



#### 3.3.8 Romana

[Romana](https://romana.io/) 是一个开源网络和安全自动化解决方案。 它可以让你在没有覆盖网络的情况下部署 Kubernetes。 Romana 支持 Kubernetes [网络策略](https://kubernetes.io/zh/docs/concepts/services-networking/network-policies/)， 来提供跨网络命名空间的隔离。可以运行BGP或OSPF





#### 3.3.9 Weave Net

Weave Net是一个多主机容器网络方案，支持去中心化的控制平面，各个host上的wRouter间通过建立Full Mesh的TCP链接，并通过Gossip协议来同步控制信息。这种方式省去了集中式的K/V Store，能够在一定程度上减低部署的复杂性，Weave将其称为“data centric”

数据平面上，Weave通过UDP封装实现L2 Overlay，封装支持两种模式：

- 运行在user space的sleeve mode：通过pcap设备在Linux bridge上截获数据包并由wRouter完成UDP封装，支持对L2 traffic进行加密，还支持Partial Connection，但是性能损失明显。
- 运行在kernal space的 fastpath mode：即通过OVS的odp封装VxLAN并完成转发，wRouter不直接参与转发，而是通过下发odp 流表的方式控制转发，这种方式可以明显地提升吞吐量，但是不支持加密等高级功能。

![Weave-net always renders one node unhealthy in the cluster · Issue #3243 ·  weaveworks/weave · GitHub](https://user-images.githubusercontent.com/11160747/36148997-cfd44de8-10df-11e8-94b9-6d68db7b3a24.png)



#### 3.3.10 Canal

[Canal](https://github.com/tigera/canal) 是 Flannel 和 Calico 联合发布的一个统一网络插件，提供 CNI 网络插件，并支持 network policy。

![Canal Phase 1 Diagram.png](https://github.com/projectcalico/canal/blob/master/Canal%20Phase%201%20Diagram.png?raw=true)



#### 3.3.11 danm(多cni)

[DANM](https://github.com/nokia/danm) 是一个针对在 Kubernetes 集群中运行的电信工作负载的网络解决方案。 它由以下几个组件构成：

- 能够配置具有高级功能的 IPVLAN 接口的 CNI 插件
- 一个内置的 IPAM 模块，能够管理多个、群集内的、不连续的 L3 网络，并按请求提供动态、静态或无 IP 分配方案
- CNI 元插件能够通过自己的 CNI 或通过将任务授权给其他任何流行的 CNI 解决方案（例如 SRI-OV 或 Flannel）来实现将多个网络接口连接到容器
- Kubernetes 控制器能够集中管理所有 Kubernetes 主机的 VxLAN 和 VLAN 接口
- 另一个 Kubernetes 控制器扩展了 Kubernetes 的基于服务的服务发现概念，以在 Pod 的所有网络接口上工作

通过这个工具集，DANM 可以提供多个分离的网络接口，可以为 Pod 使用不同的网络后端和高级 IPAM 功能。

https://github.com/nokia/danm





#### 3.3.12 multus(多cni)

[Multus](https://github.com/Intel-Corp/multus-cni) 是一个多 CNI 插件， 使用 Kubernetes 中基于 CRD 的网络对象来支持实现 Kubernetes 多网络系统。

Multus 支持所有[参考插件](https://github.com/containernetworking/plugins)（比如： [Flannel](https://github.com/containernetworking/plugins/tree/master/plugins/meta/flannel)、 [DHCP](https://github.com/containernetworking/plugins/tree/master/plugins/ipam/dhcp)、 [Macvlan](https://github.com/containernetworking/plugins/tree/master/plugins/main/macvlan) ） 来实现 CNI 规范和第三方插件（比如： [Calico](https://github.com/projectcalico/cni-plugin)、 [Weave](https://github.com/weaveworks/weave)、 [Cilium](https://github.com/cilium/cilium)、 [Contiv](https://github.com/contiv/netplugin)）。 除此之外， Multus 还支持 [SRIOV](https://github.com/hustcat/sriov-cni)、 [DPDK](https://github.com/Intel-Corp/sriov-cni)、 [OVS-DPDK & VPP](https://github.com/intel/vhost-user-net-plugin) 的工作负载， 以及 Kubernetes 中基于云的本机应用程序和基于 NFV 的应用程序。

![multus-pod-image](https://github.com/k8snetworkplumbingwg/multus-cni/raw/master/docs/images/multus-pod-image.svg)

缺少一个cluster-wide ipam（https://github.com/k8snetworkplumbingwg/whereabouts）







#### 3.3.13 cni-genie(多cni)

[CNI-Genie](https://github.com/Huawei-PaaS/CNI-Genie) 是一个 CNI 插件， 可以让 Kubernetes 在运行时使用不同的[网络模型](https://kubernetes.io/zh/docs/concepts/cluster-administration/networking/#the-kubernetes-network-model)的 [实现同时被访问](https://github.com/Huawei-PaaS/CNI-Genie/blob/master/docs/multiple-cni-plugins/README.md#what-cni-genie-feature-1-multiple-cni-plugins-enables)。 这包括以 [CNI 插件](https://github.com/containernetworking/cni#3rd-party-plugins)运行的任何实现，比如 [Flannel](https://github.com/coreos/flannel#flannel)、 [Calico](https://docs.projectcalico.org/)、 [Romana](https://romana.io/)、 [Weave-net](https://www.weave.works/products/weave-net/)。

CNI-Genie 还支持[将多个 IP 地址分配给 Pod](https://github.com/Huawei-PaaS/CNI-Genie/blob/master/docs/multiple-ips/README.md#feature-2-extension-cni-genie-multi-ip-addresses-per-pod)， 每个都来自不同的 CNI 插件。

![image](https://github.com/huawei-cloudnative/CNI-Genie/raw/master/docs/multiple-cni-plugins/what-cni-genie.png)

https://github.com/huawei-cloudnative/CNI-Genie





#### 3.3.14 galaxy(多cni)

Galaxy 是一个 Kubernetes 网络项目，旨在为 Pod 提供覆盖和高性能的底层网络。并且它还实现了浮动IP（或弹性IP），即pod的IP即使由于节点崩溃而浮动到另一个节点上也不会改变，这有利于运行有状态的集合应用程序。

目前，它由三个组件组成 - Galaxy、CNI 插件和 Galaxy IPAM。 Galaxy 是一个运行在每个 kubelet 节点上的守护进程，它调用不同类型的 CNI 插件来为 pod 设置所需的网络。 Galaxy IPAM 是一个 Kubernetes 调度程序插件，可用作浮动 IP 配置和分配管理器。

Galaxy 与 CNI 规范兼容，可以通过安装 CNI 二进制文件和更新将任何 CNI 插件与它集成.

![How galaxy-ipam works](https://github.com/tkestack/galaxy/raw/master/doc/image/galaxy-ipam.png)



![How galaxy ipam allocates IP according to network typology](https://github.com/tkestack/galaxy/raw/master/doc/image/galaxy-ipam-scheduling-process.png)

https://github.com/tkestack/galaxy







### 3.4 network policy

随着微服务的流行，越来越多的云服务平台需要大量模块之间的网络调用。Kubernetes 在 1.3 引入了 Network Policy，Network Policy 提供了基于策略的网络控制，用于隔离应用并减少攻击面。它使用标签选择器模拟传统的分段网络，并通过策略控制它们之间的流量以及来自外部的流量。

在使用 Network Policy 时，需要注意

- v1.6 以及以前的版本需要在 kube-apiserver 中开启 `extensions/v1beta1/networkpolicies`
- v1.7 版本 Network Policy 已经 GA，API 版本为 `networking.k8s.io/v1`
- v1.8 版本新增 **Egress** 和 **IPBlock** 的支持
- 网络插件要支持 Network Policy，如 Calico、Romana、Weave Net 和 trireme 等





### 3.5 性能对比

![Kubernetes 网络插件在超过10Gbit/s 下的基准测试结果| CloudNative 架构](http://team.jiunile.com/images/k8s/cni_18.png)

https://zhuanlan.zhihu.com/p/230774496





## 4. 容器网络的本质

network namespace + 网络设备 + 网络协议的组合



```bash
node1 192.168.104.111
node2 192.168.104.128
            
        +        -+               +        -+
        |                         |               |                         |
        |      +    +     |               |      +    +     |
        |      |            |     |               |      |            |     |
        |      |  3.3.3.4   |     |               |      |   3.3.3.3  |     |
        |      +    +     |               |      +    +     |
        |            |eth0        |               |            |eth0        |
        |            |            |               |            |            |
        |            +veth1       |               |            +veth1       |
        |      +    +     |               |      +    +     |
        |      |            |     |               |      |            |     |
        |      |    br1     |     |               |      |    br1     |     |
        |      + --+  +     |               |      + --+  +     |
        |            |            |               |            |            |
        |            |            |               |            |            |
        |      + --+  +     |               |      + --+  +     |
        |      |   vxlan100 |     |               |      |   vxlan100 |     |
        |      |            |     |               |      |            |     |
        |      + --+  +     |               |      + --+  +     |
        |            |            |               |            |            |
        |            |            |               |            |            |
node2   |            | eth0       |               |            | eth0       | node1
        +        -+               +        -+
                     | 192.168.104.128          192.168.104.111|
                     |                                         |
                     |                                         |
         +   --+             --+     +
```



node1

```bash
sysctl -w net.ipv4.ip_forward=1
#创建命名空间
ip netns add test1
#创建veth设备
ip link add veth1 type veth peer name eth0 netns test1
ip netns exec test1 ip link set eth0 up
ip netns exec test1 ip link set lo up
#命名空间里的eth0配置ip
ip netns exec test1 ip addr add 3.3.3.3/24 dev eth0
ip link set up dev veth1
#创建br1 bridge
ip link add br1 type bridge
ip link set br1 up
#veth1加入br1 bridge
ip link set veth1 master br1
#指定了nolearning来禁用源地址学习, 通过ip -d a可以看到设备属性
ip link add vxlan100 type vxlan id 100 dstport 4789 local 192.168.104.111 nolearning
#vxlan100加入br1 bridge
ip link set vxlan100 master br1
ip link set up vxlan100
#添加的是对端的mac地址和ip
bridge fdb append 86:a7:37:e5:4f:e0 dev vxlan100 dst 192.168.104.128
#添加vxlan设备arp代答
ip neighbor add 3.3.3.4 lladdr 86:a7:37:e5:4f:e0 dev vxlan100
```



node2

```bash
sysctl -w net.ipv4.ip_forward=1
#创建命名空间
ip netns add test1
#创建veth设备
ip link add veth1 type veth peer name eth0 netns test1
ip netns exec test1 ip link set eth0 up
ip netns exec test1 ip link set lo up
ip netns exec test1 ip addr add 3.3.3.4/24 dev eth0
ip link set up dev veth1
ip link add br1 type bridge
ip link set br1 up
ip link set veth1 master br1 
ip link add vxlan100 type vxlan id 100 dstport 4789 local 192.168.104.128 nolearning
ip link set vxlan100 master br1
ip link set up vxlan100
bridge fdb append ea:da:ea:f7:13:be dev vxlan100 dst 192.168.104.111
#添加vxlan设备arp代答
ip neighbor add 3.3.3.3 lladdr ea:da:ea:f7:13:be dev vxlan100
```





## 5. 当前现状

### 5.1 内部

<img src="https://i.ibb.co/0KVnLzt/9b535e06-3ee6-4819-9ba2-a19a45840f1e.png" alt="9b535e06-3ee6-4819-9ba2-a19a45840f1e" border="0">





## 6. 我们的选择会是什么?

考虑的维度？

- ToB变化的场景 - 多插件模式
- 网络链路加密 - (pod-pod、node-node、pod-service)
- 可控



## 7. 容器网络未来展望

1. **容器网络方案方面**，考虑将SDN、微服务与容器网络相结合，实现更加高效便捷的一体化管理
2. **容器网络安全方面，**考虑研究容器网络的安全管理模式
3. **容器网络可观测性方面**



>从谷歌分享的网络排错案例中可以看到，即便是作为全球三大公有云平台之一，也需要执行大量的工具和命令来逐步定位网络问题的根源。并且在很多时候，还需要客户自己来重现问题，才能拿到第一手的调试数据。这一方面说明了网络问题的复杂性，另一方面也说明在网络排错、调试、监控等方面还有很多可以改进的空间。如果这些问题解决的出色，相应的网络方案很可能就会一跃而起，成为新的流行技术。
>
>在这方面，Cilium 可以说是一个典型代表。Cilium 不仅需要较新的内核，还会通过 eBPF 机制在内核中注入包括观测、安全、过滤等在内的一系列网络机制。但这并没有阻止 Cilium 的流行。很多其他的网络方案也在借鉴 Cilium 的原理，借助 eBPF 加速网络性能，并实现更透明的网络观测机制。





## 8. 定位问题小工具

### 8.1 nsenter 

nsenter命令是一个可以在指定进程的命令空间下运行指定程序的命令, 可以进入该容器的网络命名空间

```bash
function e() {
    set -eu
    ns=${2-"default"}
    pod=`kubectl -n $ns describe pod $1 | grep -Eo 'docker://.*$' | head -n 1 | sed 's/docker:\/\/\(.*\)$/\1/'`
    pid=`docker inspect -f {{.State.Pid}} $pod`
    echo "enter pod netns successfully for $ns/$1"
    nsenter -n --target $pid
}
```

```bash
# e pv-local-pod
entering pod netns for default/pv-local-pod
nsenter -n --target 1991
[root@lianbang-xuexi-server24 ~]#
[root@lianbang-xuexi-server24 ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
103: eth0@if104: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP
    link/ether 8e:c7:e3:77:20:ce brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.12.0.49/32 brd 10.12.0.49 scope global eth0
       valid_lft forever preferred_lft forever
```



### 8.2 tcpdump

**tcpdump** 是一个Unix下一个功能强大的网络抓包工具，它允许用户拦截和显示发送或收到过网络连接到该计算机的TCP/IP和其他数据包，可用于大多数的类Unix系统：包括Linux、Solaris、BSD、Mac OS X、HP-UX和AIX 等等。一般使用tcpdump+wireshark组合的方式，tcpdump是命令行方式，wireshark是gui方式

![6.12. Time Display Formats And Time References](https://www.wireshark.org/docs/wsug_html_chunked/wsug_graphics/ws-time-reference.png)

![img](http://www.iceyao.com.cn/img/posts/2020-12-12/tcp-ip.png)

![img](http://www.iceyao.com.cn/img/posts/2020-12-12/tcp-ip-status.png)

 

### 8.3 ping

Ping是工作在[ TCP/IP](https://baike.baidu.com/item/ TCP%2FIP/214077)网络体系结构中[应用层](https://baike.baidu.com/item/应用层/16412033)的一个服务命令， 主要是向特定的目的主机发送[ ICMP](https://baike.baidu.com/item/ ICMP/572452)（Internet Control Message Protocol 因特网报文控制协议）[Echo](https://baike.baidu.com/item/Echo/35157) 请求报文，测试目的站是否可达及了解其有关状态

 

### 8.4 telnet

telnet是远程连接服务，它工作于在tcp/ip协议的应用层。检测tcp服务

 

### 8.5 nc

netcat是网络工具中的瑞士军刀，它能通过TCP和UDP在网络中读写数据。不仅可以检测tcp/udp服务，而且还可以变成tcp/udp的server

 

## 9. Q&A









