[toc]

### k8s基础组件有哪些，什么功能

#### master组件 

##### kube-apiserver 

集群的核心，负责各个功能模块之间的通信。各个模块都通过apiserver将信息存入etcd。

##### kube-controller-manager 

控制器有很多种，通过 apiserver 监控整个集群的状态,确保集群始终处于预期的工作状态。

##### kube-scheduler 

负责调度,将pod调度到最优节点上。

##### etcd

存储kubernetes的元数据,高可用。

#### node组件

##### kubelet

每个节点上运行的代理，确保容器处于运行状态且健康。会定期想master汇报资源使用情况。

### 一个pod创建流程

可通过kubectl或yaml创建pod请求

apiserver接收到请求后,会存储pod数据到etcd

scheduler的watch接口会从apiserver处监听到新建pod的信息，scheduler会根据集群的整体信息，选出最优的node进行调度,会通过apiserver将信息写到etcd里

目标node上的kubelet通过apiserver监听到新建的pod,会去调用Container Runtime Interface (CRI) 创建相应pod.

(kube-controller-manager 会确保pod处于预期的状态)

### 网络选型需要注意什么

pod跨node的通信

网络性能

网络插件成熟度 

网络策略

(最常用的应该是flannel和calico)

### etcd用的什么算法，简单解释一下

raft算法  强一致性 同一时间只能有一个leader,所有的操作都在leader上。

### pod中pending状态，是什么原因产生的，pod出现问题，排查思路

#### pending原因

节点资源不足

不满足 nodeSelector 与 affinity

Node 存在 Pod 没有容忍的污点

kube-scheduler 未正常运行

镜像不存在

#### 排查思路

kubectl describe pod 查看具体信息 

到相应node上查看具体日志

### kubernetes发布策略（4种）

Recreate

RollingUpdate

蓝绿发布

灰度发布(金丝雀发布)

### 手写raft

raft是一种强一致性的算法  同一时间只能有一个leader,所有的操作都在leader上。

最重要的是leader选举和日志复制

### 你们监控用的什么，怎么利用普罗米修斯监控pod信息，k8s状态，如果来设计相关的监控如何落地

使用promethues-opreator ，数据持久化，如果集群过大可以考虑使用thanos高可用。

pod 的metrics信息已经集成到的kubelet,直接用cAdvisor提供的`metrics`接口获取到所有容器相关的性能指标数据。  

自定义相关模板 通过grafana展示。（grafana上有许多优秀的模板）



### 如果利用k8s实现滚动更新，我说的配置文件机制

deployment  strategy  RollingUpdate

### statefulset是怎么实现滚动更新的？

多个pod时，会按照顺序(序列号)滚动更新，每一个 Pod 都正常运行时才会继续处理下一个 Pod。

### 基本就是继续k8s架构问，，遇到的问题，怎么处理

https://llussy.github.io/2020/09/04/kubernetes-troubleshooting/

### kubectl exec实现的原理？

-v=7 可以看到详细信息

首先会读取config文件 ，生成GET/POST请求 去访问apiserver

apiserver会向kubelet发起连接

https://blog.fleeto.us/post/how-kubectl-exec-works/

![kubectl exec](/Users/lisai/go/src/llussy.github.io/images/1737323-20200526120352435-47536857.png)

### 如何实现schedule水平扩展？

scheduler 只能有一个leader  ，一般有三个节点就够。

### 为什么k8s要用申明式？

只需描述期望状态就行,剩下的交给k8s。

### 了解过 endpointslice吗？ 怎么实现的？

### 容器的驱逐时间是？

node notready时，默认驱逐时间是5min，可修改，是由kube-controller-manager的pod-eviction-timeout控制的。

node资源不足时也会发生驱逐，是按照qos等级来驱逐的。



### 节点notready是什么导致的？ notready会发生什么？

#### notready原因

kubelet异常 不能和apiserver通信

网络异常

kubelet version

kernel

#### notready会发生什么

node notready 5min,pod 会驱逐到其他节点。



### api-server到etcd怎么保证事件不丢失？



### sidecar要保证顺序启动怎么保证？几种方式可以做到？

### 有了解过qos吗？ 怎么实现的？

qos三种 *Guaranteed*, *Burstable*, and *Best-Effort*，它们的QoS级别依次递减。



**Guaranteed** 如果Pod中所有Container的所有Resource的`limit`和`request`都相等且不为0，则这个Pod的QoS Class就是Guaranteed。

**Best-Effort** 如果Pod中所有容器的所有Resource的request和limit都没有赋值，则这个Pod的QoS Class就是Best-Effort.

**Burstable** 除了符合Guaranteed和Best-Effort的场景，其他场景的Pod QoS Class都属于Burstable。

**node资源不足时会按qos级别驱逐pod。** 最先驱逐的是Best-Effort ,重要组件一定要设置limit和request.



### 详述kube-proxy原理

监听 API server 中 service 和 endpoint 的变化情况，并通过 iptables 等来为服务配置负载均衡



kube-proxy的作用主要是负责service的实现

service另外一个重要作用是，一个服务后端的Pods可能会随着生存灭亡而发生IP的改变，service的出现，给服务提供了一个固定的IP，而无视后端Endpoint的变化。

**kube-proxy 模式** 

- userspace 已弃用
- iptables 大规模下会有性能问题 且不支持会话保持
- ipvs 

### k8s的pause容器有什么用。是否可以去掉

为pod中的多个容器提供共享网络空间，实现pod里容器间的通信。

主要有两个职责：

1. 是pod里其他容器共享Linux namespace的基础
2. 扮演PID 1的角色，负责处理僵尸进程

### k8s的service和ep是如何关联和相互影响的

service中有selector时才会创建endpoints.  

`kube-proxy` 会监视 Kubernetes 控制节点对 Service 对象和 Endpoints 对象的添加和移除。 

只要服务中的pod集合发生更改，endpoints就会被更新。



### StatefulSets和operator区别

StatefulSet是为了解决有状态服务的。

opertator用来扩展k8s api,自定义crd。