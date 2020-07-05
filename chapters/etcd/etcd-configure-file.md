> **2379端口**	用于与client端交互

> **2380端口**	用于etcd内部交互



```bash
--name
etcd集群中的节点名，这里可以随意，可区分且不重复就行
--listen-peer-urls
监听的用于节点之间通信的url，可监听多个，集群内部将通过这些url进行数据交互(如选举，数据同步等)
--initial-advertise-peer-urls
建议用于节点之间通信的url，节点间将以该值进行通信。
--listen-client-urls
监听的用于客户端通信的url,同样可以监听多个。
--advertise-client-urls
建议使用的客户端通信url,该值用于etcd代理或etcd成员与etcd节点通信。
--initial-cluster-token etcd-cluster-1
节点的token值，设置该值后集群将生成唯一id,并为每个节点也生成唯一id,当使用相同配置文件再启动一个集群时，只要该token值不一样，etcd集群就不会相互影响。
--initial-cluster
也就是集群中所有的initial-advertise-peer-urls 的合集
--initial-cluster-state new
新建集群的标志，初始化状态使用 new，建立之后改此值为 existing
```

