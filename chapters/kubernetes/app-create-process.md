1、客户端提交创建RC请求，可以通过API Server的Restful API，也可以使用kubectl命令行工具。

2、API Server处理用户RC请求，存储Pod数据到etcd。

3、Controller Manager通过API Server的监听资源变化的接口监听到这个RC事件，分析发现集群中没有它所对应的Pod实例，于是根据RC里的Pod模板定义生成一个Pod对象,通过APIServer写入etcd。至此，API Server 创建过程完成，剩下的由 scheduler 和 kubelet 来完成，此时 pod 处于 pending 状态。

4、此事件被Scheduler通过API Server发现，它立即执行一个复杂的调度流程，为这个新Pod选定一个落户的Node(会过滤掉不符合要求的主机，如资源、node标签等)，然后通过API Server讲这一结果写入到etcd中。

5、目标Node上运行的Kubelet进程通过APIServer监测到这个“新生的”Pod，并按照它的定义，启动该Pod。



[k8s创建pod和service的过程](https://www.cnblogs.com/lexiaofei/p/11078532.html)

[Kubernetes中pod创建流程](https://blog.csdn.net/kwame211/article/details/79058350)





1）客户端提交创建请求，可以通过API Server的Restful API，也可以使用kubectl命令行工具。支持的数据类型包括JSON和YAML。

（2）API Server处理用户请求，存储Pod数据到etcd。

（3）调度器通过API Server查看未绑定的Pod。尝试为Pod分配主机。

（4）过滤主机 (调度预选)：调度器用一组规则过滤掉不符合要求的主机。比如Pod指定了所需要的资源量，那么可用资源比Pod需要的资源量少的主机会被过滤掉。

（5）主机打分(调度优选)：对第一步筛选出的符合要求的主机进行打分，在主机打分阶段，调度器会考虑一些整体优化策略，比如把容一个Replication Controller的副本分布到不同的主机上，使用最低负载的主机等。

（6）选择主机：选择打分最高的主机，进行binding操作，结果存储到etcd中。

（7）kubelet根据调度结果执行Pod创建操作： 绑定成功后，scheduler会调用APIServer的API在etcd中创建一个boundpod对象，描述在一个工作节点上绑定运行的所有pod信息。运行在每个工作节点上的kubelet也会定期与etcd同步boundpod信息，一旦发现应该在该工作节点上运行的boundpod对象没有更新，则调用Docker API创建并启动pod内的容器。

