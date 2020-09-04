[toc]

### master组件

#### kube-apiserver

apiserver提供了k8s各类资源对象的增删改查及watch等HTTP Rest接口，是整个系统的数据总线和数据中心。

​		API server作为集群的核心，负责各个功能模块之间的通信。集群中各个模块通过API server将信息存入etcd，当需要获取和操作这些数据时，则通过API server提供的REST接口来实现，从而实现各模块之间的信息交互。

​		常见的一个交互场景是kubelet与API server交互。每个node节点上的kubelet每隔一个时间周期，会调用一次API server的REST接口报告自身状态。API server接受到这些信息，更新至etcd中，此外，kubelet也通过server的watch接口监听pod信息，若监听到新的pod副本被调度绑定到本节点，则执行pod对应的容器创建和启动。如果监听到pod对象被删除，则删除本节点上对应的pod容器。

​		另一个交互场景是kube-controller-manager进程与API server的交互。前者的Node controller模块通过API server提供的watch接口。实时监控Node的信息。

​		还有一个比较重要的交互场景就是kube-scheduler与api server交互，当前者通过server的watch接口监听到新建pod副本的信息后，它会检索所有符合该pod要求的Node列表，开始执行pod调度逻辑，调度成功后将pod绑定到目标节点上。



#### kube-scheduler

负责pod的调度,调度到最优的node。

#### kube-controller-manager

控制器，通过 apiserver 监控整个集群的状态,确保集群始终处于预期的工作状态。

![img](/Users/lisai/Desktop/control-manager.png)

##### Replication Controller

1. 确保集群中有且仅有N个Pod实例，N是RC中定义的Pod副本数量。
2. 通过调整RC中的spec.replicas属性值来实现系统扩容或缩容。
3. 通过改变RC中的Pod模板来实现系统的滚动升级。

##### Node Controller

kubelet在启动时会通过API Server注册自身的节点信息，并定时向API Server汇报状态信息，API Server接收到信息后将信息更新到etcd中。

Node Controller通过API Server实时获取Node的相关信息，实现管理和监控集群中的各个Node节点的相关控制功能。

[Kubernetes核心原理（二）之Controller Manager](https://blog.csdn.net/huwh_/article/details/75675761)

#### etcd

存储kubernetes的数据,高可用。

### node组件

#### kubelet 

一个在集群中每个节点上运行的代理。它保证容器都运行在 Pod 中。

每个kubelet进程都会在API Server上注册节点自身的信息，定期向Master汇报节点资源的使用情况，并通过cAdvisor监控容器和节点资源。

kubelet 接收一组通过各类机制提供给它的 PodSpecs，确保这些 PodSpecs 中描述的容器处于运行状态且健康。kubelet 不会管理不是由 Kubernetes 创建的容器。



**kubelet 组件在工作时，采用主动的查询机制，即定期请求 apiserver 获取自己所应当处理的任务，如哪些 pod 分配到了自己身上，从而去处理这些任务；同时 kubelet 自己还会暴露出两个本身 api 的端口，用于将自己本身的私有 api 暴露出去，这两个端口分别是 10250 与 10255；对于 10250 端口，kubelet 会在其上采用 TLS 加密以提供适当的鉴权功能；对于 10255 端口，kubelet 会以只读形式暴露组件本身的私有 api，并且不做鉴权处理**

**总结一下，就是说 kubelet 上实际上有两个地方用到证书，一个是用于与 API server 通讯所用到的证书，另一个是 kubelet 的 10250 私有 api 端口需要用到的证书**

#### kube-proxy

**网络代理,kube-proxy的作用主要是负责service的实现**

k8s会根据service和pod直接的关系创建endpoint，可以通过kubectl get ep查看。

- `kube-proxy`运行在node节点，负责为 pod 提供代理功能，会定期从 etcd 获取 service 信息，并根据 service 信息通过修改 iptables 来实现流量转发，将流量转发到要访问的 pod 上(**iptables模式**)。

- `kube-proxy`其实就是管理`service`的访问入口，包括集群内Pod到Service的访问和集群外访问service。
- kube-proxy管理sevice的Endpoints，该service对外暴露一个Virtual IP，也称为Cluster IP, 集群内通过访问这个<Cluster IP>:<Port>就能访问到集群内对应的serivce下的Pod(**如果有多个副本，kube-proxy会实现负载均衡**)。

![image-20190216220054465](https://llussy.github.io/images/kubernetes/kube-proxy.png)

### 相关端口

```bash
`10250` --port: kubelet服务监听的端口,api会检测他是否存活
`10248` --healthz-port: 健康检查服务的端口
`10255` --read-only-port: 只读端口，可以不用验证和授权机制，直接访问
`4194` --cadvisor-port: 当前节点 cadvisor 运行的端口
```



### 参考

[kubernetes组件](https://kubernetes.io/zh/docs/concepts/overview/components/)

[K8S基础概念](https://www.cnblogs.com/menkeyi/p/7134460.html)

[kube-proxy工作原理](https://blog.csdn.net/WaltonWang/article/details/55236300)