[toc]

### kube-proxy

**网络代理,kube-proxy的作用主要是负责service的实现**

k8s会根据service和pod直接的关系创建endpoint，可以通过kubectl get ep查看。

- `kube-proxy`运行在node节点，负责为 pod 提供代理功能，会定期从 etcd 获取 service 信息，并根据 service 信息通过修改 iptables 来实现流量转发，将流量转发到要访问的 pod 上(**iptables模式**)。

- `kube-proxy`其实就是管理`service`的访问入口，包括集群内Pod到Service的访问和集群外访问service。
- kube-proxy管理sevice的Endpoints，该service对外暴露一个Virtual IP，也称为Cluster IP, 集群内通过访问这个<Cluster IP>:<Port>就能访问到集群内对应的serivce下的Pod(**如果有多个副本，kube-proxy会实现负载均衡**)。

![image-20190216220054465](https://llussy.github.io/images/kubernetes/kube-proxy.png)

### iptables

基于iptables的kube-proxy的主要职责包括两大块：

一块是侦听service更新事件，并更新service相关的iptables规则;

一块是侦听endpoint更新事件，更新endpoint相关的iptables规则（如 KUBE-SVC-链中的规则），然后将包请求转入endpoint对应的Pod。如果某个service尚没有Pod创建，那么针对此service的请求将会被drop掉。

#### Endpoints

当svc中有selector时会自动创建endpoints.

每个 Endpoints 对象的 IP 对应一个 Kubernetes 的内部域名，可以通过这个域名直接访问到具体的 pod。

Endpoint Controller，它会 watch Service 对象、还有 pod 的变化情况，维护对应的 Endpoint 信息。然后在每一个节点上，KubeProxy 根据 Service 和 Endpoint 来维护本地的路由规则。

#### iptables缺点

**1．可扩展性差。**随着service数据达到数千个，其控制面和数据面的性能都会急剧下降。原因在于iptables控制面的接口设计中，每添加一条规则，需要遍历和修改所有的规则，其控制面性能是*O(n²\)*。在数据面，规则是用链表组织的，其性能是*O(n)*

**2．LB调度算法仅支持随机转发。**

**iptables主要是专门用来做主机防火墙的，而不是专长做负载均衡的。**

### ipvs

IPVS 是专门为LB设计的。它用hash table管理service，对service的增删查找都是*O(1)*的时间复杂度。



ipvs模式会创建一个kube-ipvs0的网卡，也会增加route信息。

**作用：**由于 IPVS 的 DNAT 钩子挂在 INPUT 链上，因此必须要让内核识别 VIP 是本机的 IP。这样才会过INPUT 链，要不然就通过OUTPUT链出去了。k8s 通过设置将service cluster ip 绑定到虚拟网卡kube-ipvs0。



[kube-proxy ipvs模式详解](https://blog.csdn.net/qq_36183935/article/details/90734936)

[k8s集群中ipvs负载详解](https://www.jianshu.com/p/89f126b241db?utm_campaign=maleskine&utm_content=note&utm_medium=seo_notes&utm_source=recommendation)

