[toc]

### master组件

#### kube-apiserver

apiserver提供了k8s各类资源对象的增删改查及watch等HTTP Rest接口，是整个系统的数据总线和数据中心。

#### kube-scheduler

负责pod的调度,调度到最优的node。

#### kube-controller-manager

控制器，通过 apiserver 监控整个集群的状态,确保集群始终处于预期的工作状态。

#### etcd

存储kubernetes的数据,高可用。

### node组件

#### kubelet 

一个在集群中每个节点上运行的代理。它保证容器都运行在 Pod 中。

kubelet 接收一组通过各类机制提供给它的 PodSpecs，确保这些 PodSpecs 中描述的容器处于运行状态且健康。kubelet 不会管理不是由 Kubernetes 创建的容器。

**kubelet 组件在工作时，采用主动的查询机制，即定期请求 apiserver 获取自己所应当处理的任务，如哪些 pod 分配到了自己身上，从而去处理这些任务；同时 kubelet 自己还会暴露出两个本身 api 的端口，用于将自己本身的私有 api 暴露出去，这两个端口分别是 10250 与 10255；对于 10250 端口，kubelet 会在其上采用 TLS 加密以提供适当的鉴权功能；对于 10255 端口，kubelet 会以只读形式暴露组件本身的私有 api，并且不做鉴权处理**

**总结一下，就是说 kubelet 上实际上有两个地方用到证书，一个是用于与 API server 通讯所用到的证书，另一个是 kubelet 的 10250 私有 api 端口需要用到的证书**

#### kube-proxy

**网络代理,kube-proxy的作用主要是负责service的实现**

- `kube-proxy`运行在node节点，负责为 pod 提供代理功能，会定期从 etcd 获取 service 信息，并根据 service 信息通过修改 iptables 来实现流量转发，将流量转发到要访问的 pod 上(**iptables模式**)。

- `kube-proxy`其实就是管理`service`的访问入口，包括集群内Pod到Service的访问和集群外访问service。
- kube-proxy管理sevice的Endpoints，该service对外暴露一个Virtual IP，也称为Cluster IP, 集群内通过访问这个<Cluster IP>:<Port>就能访问到集群内对应的serivce下的Pod(**如果有多个副本，kube-proxy会实现负载均衡**)。

![image-20190216220054465](https://llussy.github.io/images/kubernetes/kube-proxy.png)

### 参考

[kubernetes组件](https://kubernetes.io/zh/docs/concepts/overview/components/)

[K8S基础概念](https://www.cnblogs.com/menkeyi/p/7134460.html)

[kube-proxy工作原理](https://blog.csdn.net/WaltonWang/article/details/55236300)