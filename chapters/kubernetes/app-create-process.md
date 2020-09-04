1、客户端提交创建RC请求，可以通过API Server的Restful API，也可以使用kubectl命令行工具。

2、API Server处理用户RC请求，存储Pod数据到etcd。

3、Controller Manager通过API Server的监听资源变化的接口监听到这个RC事件，分析发现集群中没有它所对应的Pod实例，于是根据RC里的Pod模板定义生成一个Pod对象,通过APIServer写入etcd。至此，API Server 创建过程完成，剩下的由 scheduler 和 kubelet 来完成，此时 pod 处于 pending 状态。

4、此事件被Scheduler通过API Server发现，它立即执行一个复杂的调度流程，为这个新Pod选定一个落户的Node(会过滤掉不符合要求的主机，如资源、node标签等)，然后通过API Server讲这一结果写入到etcd中。

5、目标Node上运行的Kubelet进程通过APIServer监测到这个“新生的”Pod，并按照它的定义，启动该Pod。



[k8s创建pod和service的过程](https://www.cnblogs.com/lexiaofei/p/11078532.html)

[Kubernetes中pod创建流程](https://blog.csdn.net/kwame211/article/details/79058350)

