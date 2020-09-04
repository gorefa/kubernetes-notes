[toc]

### K8s 1.8 新特性——ipvs

#### ipvs与iptables的性能差异

随着服务的数量增长，IPTables 规则则会成倍增长，这样带来的问题是路由延迟带来的服务访问延迟，同时添加或删除一条规则也有较大延迟。不同规模下，kube-proxy添加一条规则所需时间如下所示： 

![img](https://nos.netease.com/cloud-website-bucket/2018071011514703e8c404-8728-4d71-bfa5-ef68bcb0eebf.png)

可以看出当集群中服务数量达到5千个时，路由延迟成倍增加。添加 IPTables 规则的延迟，有多种产生的原因，如：

添加规则不是增量的，而是先把当前所有规则都拷贝出来，再做修改然后再把修改后的规则保存回去，这样一个过程的结果就是 IPTables 在更新一条规则时会把 IPTables 锁住，这样的后果在服务数量达到一定量级的时候，性能基本不可接受：在有5千个服务（4万条规则）时，添加一条规则耗时11分钟；在右2万个服务（16万条规则）时，添加一条规则需要5个小时。

这样的延迟时间，对生产环境是不可以的，那该性能问题有哪些解决方案呢？从根本上解决的话，可以使用 “IP Virtual Server”(IPVS )来替换当前 kube-proxy 中的 IPTables 实现，这样能带来显著的性能提升以及更智能的负载均衡功能如支持权重、支持重试等等。

那什么是 “IP Virtual Server”(IPVS ) 呢？

#### ipvs 简介

k8s 1.8 版本中，社区 SIG Network 增强了 NetworkPolicy API，以支持 Pod 出口流量策略，以及允许策略规则匹配源或目标 CIDR 的匹配条件。这两个增强特性都被设计为 beta 版本。 SIG Network 还专注于改进 kube-proxy，除了当前的 iptables 和 userspace 模式，kube-proxy 还引入了一个 alpha 版本的 IPVS 模式。

作为 Linux Virtual Server(LVS) 项目的一部分，IPVS 是建立于 Netfilter之上的高效四层负载均衡器，支持 TCP 和 UDP 协议，支持3种负载均衡模式：NAT、直接路由（通过 MAC 重写实现二层路由）和IP 隧道。ipvs(IP Virtual Server)安装在LVS(Linux Virtual Server)集群作为负载均衡主节点上，通过虚拟出一个IP地址和端口对外提供服务。客户端通过访问虚拟IP+端口访问该虚拟服务，之后访问请求由负载均衡器调度到后端真实服务器上。

ipvs相当于工作在netfilter中的input链。 

![img](https://nos.netease.com/cloud-website-bucket/2018071011520216fd39ee-0b70-417d-aba6-b3c1428c1ba5.png)

**配置方法**：IPVS 负载均衡模式在 kube-proxy 处于测试阶段还未正式发布，完全兼容当前 Kubernetes 的行为，通过修改 kube-proxy 启动参数，在 mode=userspace 和 mode=iptables 的基础上，增加 mode=IPVS 即可启用该功能。

#### ipvs转发模式

**● DR模式（Direct Routing）**

特点：

<1> 数据包在LB转发过程中，源/目的IP和端口都不会变化。LB只修改数据包的MAC地址为RS的MAC地址

<2> RS须在环回网卡上绑定LB的虚拟机服务IP

<3> RS处理完请求后，响应包直接回给客户端，不再经过LB

缺点：

<1> LB和RS必须位于同一子网

![img](https://nos.netease.com/cloud-website-bucket/201807101152172564a8cf-28f4-41f6-9f0d-ea28d423c0bd.png)



**● NAT模式（Network Address Translation**）

特点：

<1> LB会修改数据包地址：对于请求包，进行DNAT；对于响应包，进行SNAT

<2> 需要将RS的默认网关地址配置为LB的虚拟IP地址

缺点：

<1> LB和RS必须位于同一子网，且客户端和LB不能位于同一子网

![img](https://nos.netease.com/cloud-website-bucket/20180710115232d5cd98ab-1c6b-4e5a-b98c-d07c8feee58f.png)



**● FULLNAT模式**

特点：

<1> LB会对请求包和响应包都做SNAT+DNAT

<2> LB和RS对于组网结构没有要求

<3> LB和RS必须位于同一子网，且客户端和LB不能位于同一子网

![img](https://nos.netease.com/cloud-website-bucket/20180710115247ea8f6e89-6446-4c54-a24f-a748ab0c50dc.png)



**● 三种转发模式性能从高到低：DR > NAT >FULLNAT**

#### ipvs 负载均衡器常用调度算法

**● 轮询（Round Robin）**

LB认为集群内每台RS都是相同的，会轮流进行调度分发。从数据统计上看，RR模式是调度最均衡的。

**● 加权轮询（Weighted Round Robin）**

LB会根据RS上配置的权重，将消息按权重比分发到不同的RS上。可以给性能更好的RS节点配置更高的权重，提升集群整体的性能。

**● 最少连接调度**

LB会根据和集群内每台RS的连接数统计情况，将消息调度到连接数最少的RS节点上。在长连接业务场景下，LC算法对于系统整体负载均衡的情况较好；但是在短连接业务场景下，由于连接会迅速释放，可能会导致消息每次都调度到同一个RS节点，造成严重的负载不均衡。

**● 加权最少连接调度**

最小连接数算法的加权版。

**● 原地址哈希，锁定请求的用户**

根据请求的源IP，作为散列键（Hash Key）从静态分配的散列表中找出对应的服务器。若该服务器是可用的且未超载，将请求发送到该服务器。