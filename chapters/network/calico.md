[TOC]

### Calico简介

[Calico](https://www.projectcalico.org/) 是一个纯三层的数据中心网络方案（不需要 Overlay网络做报文的转发),三层通信模型表示每个容器都通过 IP 直接通信，中间通过路由转发找到对方。在这个过程中，容器所在的节点类似于传统的路由器，提供了路由查找的功能。

Calico与 OpenStack、Kubernetes、AWS、GCE 等 IaaS 和容器平台都有良好的集成。

Calico 在每一个计算节点利用 Linux Kernel 实现了一个高效的 vRouter 来负责数据转发，而每个 vRouter 通过 BGP 协议负责把自己上运行的 workload 的路由信息像整个 Calico 网络内传播——小规模部署可以直接互联，大规模下可通过指定的 BGP route reflector 来完成。 这样保证最终所有的 workload 之间的数据流量都是通过 IP 路由的方式完成互联的。Calico 节点组网可以直接利用数据中心的网络结构（无论是 L2 或者 L3），不需要额外的 NAT，隧道或者 Overlay Network。

此外，Calico 基于 iptables 还提供了丰富而灵活的网络 Policy，保证通过各个节点上的 ACLs 来提供 Workload 的多租户隔离、安全组以及其他可达性限制等功能。

**由于通信时不需要解包和封包，网络性能损耗小，易于排查，且易于水平扩展。**

   要想路由工作能够正常，每个虚拟路由器（容器所在的主机节点）必须有某种方法知道整个集群的路由信息，calico 采用的是 [BGP 路由协议](https://en.wikipedia.org/wiki/Border_Gateway_Protocol)，全称是 `Border Gateway Protocol`。

小规模部署时可以通过BGP client直接互联，大规模下可通过指定的BGP Route Reflector来完成，这样保证所有的数据流量都是通过IP路由的方式完成互联的。

### calico架构

**首先请看calico的架构图**

![pmg](https://llussy.github.io/images/kubernetes/calico-01.png)



calico包括如下重要组件：Felix，etcd，BGP Client，BGP Route Reflector。下面分别说明一下这些组件。

**Felix**：Calico agent  运行在每台node上，为容器设置网络信息：IP路由规则，iptable规则等

**etcd**：分布式键值存储，主要负责网络元数据一致性，确保*Calico*网络状态的准确性*，可以与**kubernetes**共用*；

**BGP Client(BIRD),** 主要负责把 *Felix*写入 *kernel*的路由信息分发到当前 *Calico*网络，确保 *workload*间的通信的有效性；

**BGPRoute Reflector** 大规模部署时使用，摒弃所有节点互联的*mesh*模式，通过一个或者多个 *BGPRoute Reflector* 来完成集中式的路由分发；

### calico原理

如下图所示，描述了从源容器经过源宿主机，经过数据中心的路由，然后到达目的宿主机最后分配到目的容器的过程。

![img](http://llussy.github.io/images/kubernetes/calico-02.png)



整个过程中始终都是根据iptables规则进行路由转发，并没有进行封包，解包的过程，这和flannel比起来效率就会快多了。

### IP Pool

 IP Pool可以使用两种模式：BGP或IPIP。使用IPIP模式时，设置 CALICO_IPV4POOL_IPIP="always"，不使用IPIP模式时，设置为"off"，此时将使用BGP模式。

 IPIP是一种将各Node的路由之间做一个tunnel，再把两个网络连接起来的模式，启用IPIP模式时，Calico将在各Node上创建一个名为"tunl0"的虚拟网络接口。

如果设置CALICO_IPV4POOL_IPIP="off" ，即不使用IPIP模式，则Calico将不会创建tunl0网络接口，路由规则直接使用物理机网卡作为路由器转发。

### pod的对端网卡

```bash
#  ethtool -S eth0
NIC statistics:
     peer_ifindex: 2034

# route -n | grep califcbbba66cc1
10.244.162.155  0.0.0.0         255.255.255.255 UH    0      0        0 califcbbba66cc1


# ip a | grep 2034
2034: califcbbba66cc1@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP

```



### 参考

[calico](https://jimmysong.io/kubernetes-handbook/concepts/calico.html)

[Kubernetes之部署calico网络](<https://blog.51cto.com/newfly/2062210>)

[图解Kubernetes网络](<https://mp.weixin.qq.com/s/NETiIHI7pib1Ws6ltPDtjA>)

[calico网络](https://kubernetes.feisky.xyz/cha-jian-kuo-zhan/network/index-2#calico-de-bu-zu)

[calico install doc](https://docs.projectcalico.org/getting-started/kubernetes/self-managed-onprem/onpremises)

[赞 k8s网络之Calico网络](https://www.cnblogs.com/goldsunshine/p/10701242.html)

