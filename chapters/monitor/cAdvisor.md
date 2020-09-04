### cAdvisor

​		cAdvisor是谷歌开源的一个容器监控工具，目前`cAdvisor`集成到了`kubelet`组件内，可以在kube集群中每个启动了kubelet的节点使用cAdvisor来查看该节点的运行数据。因此可以直接用过cAdvisor提供的`metrics`接口获取到所有容器相关的性能指标数据。

​		prometheus获取监控端点的方式有很多，其中就包括k8s，prometheus会通过调用master的apiserver获取到节点信息，然后去调取每个节点的数据。 

​		prometheus作为一个时间序列数据收集，处理，存储的服务，能够监控的对象必须直接或间接提供prometheus认可的数据模型，通过http api的形式暴露出来。我们知道`cAdvisor`支持`prometheus`,同样，包含了cAdivisor的kubelet也支持`prometheus`。每个节点都暴露了供prometheus调用的api。

metrics地址:  http://nodeip:10255/metrics/cadvisor

