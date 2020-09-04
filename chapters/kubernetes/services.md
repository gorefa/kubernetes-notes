

[toc]

### k8s service

**作用**

service是发现后端pod服务；(服务发现)
是为一组具有相同功能的容器应用提供一个统一的入口地址；（外部路由）
是将请求进行负载分发到后端的各个容器应用上的控制器。 （负载均衡）

**service的访问来源**

访问service的请求来源有两种：k8s集群内部的程序（Pod）和 k8s集群外部的程序。



#### service type

**● ClusterIP**：提供一个集群内部的虚拟IP以供Pod访问（service默认类型)。  iptables模式下无法ping通clusterip,因为在iptables中没有为ping包做转发。telnet是通的。

**● NodePort**:在每个Node上打开一个端口以供外部访问。

Kubernetes将会在每个Node上打开一个端口并且每个Node的端口都是一样的，通过\:NodePort的方式Kubernetes集群外部的程序可以访问Service。

**● LoadBalancer**：通过外部的负载均衡器来访问。



#### headless service

**headless** service 需要将 spec.clusterIP 设置成 None。

因为没有ClusterIP，kube-proxy 并不处理此类服务，因为没有load balancing或 proxy 代理设置，在访问服务的时候回返回后端的全部的Pods IP地址，主要用于开发者自己根据pods进行负载均衡器的开发(设置了selector)。



#### service selector

service通过selector和pod建立关联。

**k8s会根据service关联到pod的podIP信息组合成一个endpoint。**

若service定义中没有selector字段，service被创建时，`endpoint controller`不会自动创建`endpoint`。



#### service 负载策略

service 负载分发策略有两种：

##### RoundRobin

轮询模式，即轮询将请求转发到后端的各个pod上,默认模式。

##### SessionAffinity

ipvs才支持

基于客户端IP地址进行会话保持的模式，第一次客户端访问后端某个pod，之后的请求都转发到这个pod上。

```yaml
spec:
  sessionAffinity: ClientIP
```



### services port 

**port**: service暴露在cluster ip上的端口，**<cluster ip>:port** 是提供给集群内部客户访问service的入口。

**nodePort:** nodePort是kubernetes提供给集群外部客户访问service入口的一种方式（另一种方式是LoadBalancer）

**targetPort:** targetPort是pod上的端口，从port和nodePort上到来的数据最终经过kube-proxy流入到后端pod的targetPort上进入容器。

**service.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    name: app1
  name: app1
  namespace: default
spec:
  type: NodePort
  ports:
  - port: 8080
    targetPort: 8080
    nodePort: 30062
  selector:
    name: app1
```

#### port nodePort targetPort

总的来说，`port`和`nodePort`都是service的端口，前者暴露给集群内客户访问服务，后者暴露给集群外客户访问服务。从这两个端口到来的数据都需要经过反向代理`kube-proxy`流入后端pod的`targetPort`，从而到达pod上的容器内。

[kubernetes port、nodePort、targetPort对比分析](https://blog.csdn.net/xinghun_4/article/details/50492041)

### 服务发现

#### k8s服务发现

虽然Service解决了Pod的服务发现问题，但不提前知道Service的IP，怎么发现service服务呢？

k8s提供了两种方式进行服务发现：

**● 环境变量**： 当创建一个Pod的时候，kubelet会在该Pod中注入集群内所有Service的相关环境变量。需要注意的是，要想一个Pod中注入某个Service的环境变量，则必须Service要先比该Pod创建。这一点，几乎使得这种方式进行服务发现不可用。

```bash
  例如：一个ServiceName为redis-master的Service，对应的ClusterIP:Port为10.0.0.11:6379，则其在pod中对应的环境变量为：

  REDIS_MASTER_SERVICE_HOST=10.0.0.11
  REDIS_MASTER_SERVICE_PORT=6379
  REDIS_MASTER_PORT=tcp://10.0.0.11:6379
  REDIS_MASTER_PORT_6379_TCP=tcp://10.0.0.11:6379
  REDIS_MASTER_PORT_6379_TCP_PROTO=tcp
  REDIS_MASTER_PORT_6379_TCP_PORT=6379
  REDIS_MASTER_PORT_6379_TCP_ADDR=10.0.0.11
```

**● DNS**：可以通过cluster add-on的方式轻松的创建KubeDNS来对集群内的Service进行服务发现————这也是k8s官方强烈推荐的方式,k8s会修改容器的/etc/resolv.conf配置。

#### k8s服务发现原理

**● endpoint**

endpoint是k8s集群中的一个资源对象，存储在etcd中，用来记录一个service对应的所有pod的访问地址。

service配置selector，endpoint controller才会自动创建对应的endpoint对象；否则，不会生成endpoint对象.

**● endpoint controller**

endpoint controller是k8s集群控制器的其中一个组件，其功能如下：

```
负责生成和维护所有endpoint对象的控制器
负责监听service和对应pod的变化
监听到service被删除，则删除和该service同名的endpoint对象
监听到新的service被创建，则根据新建service信息获取相关pod列表，然后创建对应endpoint对象
监听到service被更新，则根据更新后的service信息获取相关pod列表，然后更新对应endpoint对象
监听到pod事件，则更新对应的service的endpoint对象，将podIp记录到endpoint中
```

### 负载均衡

#### kube-proxy

kube-proxy负责service的实现，即实现了k8s内部从pod到service和外部从node port到service的访问。

kube-proxy默认采用iptables的方式配置负载均衡，基于iptables的kube-proxy的主要职责包括两大块：

一块是侦听service更新事件，并更新service相关的iptables规则。

一块是侦听endpoint更新事件，更新endpoint相关的iptables规则（如 KUBE-SVC-链中的规则），然后将包请求转入endpoint对应的Pod。如果某个service尚没有Pod创建，那么针对此service的请求将会被drop掉。

<img src="/Users/lisai/go/src/llussy.github.io/images/201807101148435b1b5557-71f8-46b5-b84c-01d532a833a4.png" alt="img" style="zoom:50%;" />

#### kube-proxy iptables

kube-proxy监听service和endpoint的变化，将需要新增的规则添加到iptables中。

kube-proxy只是作为controller，而不是server，真正服务的是内核的netfilter，体现在用户态则是iptables。

kube-proxy的iptables方式也支持RoundRobin（默认模式）和SessionAffinity负载分发策略。

`kubernetes只操作了filter和nat表。`

Filter：在该表中，一个基本原则是只过滤数据包而不修改他们。filter table的优势是小而快，可以hook到input，output和forward。这意味着针对任何给定的数据包，只有可能有一个地方可以过滤它。

NAT：此表的主要作用是在PREROUTING和POSTROUNTING的钩子中，修改目标地址和原地址。与filter表稍有不同的是，该表中只有新连接的第一个包会被修改，修改的结果会自动apply到同一连接的后续包中。

kube-proxy对iptables的链进行了扩充，自定义了**KUBE-SERVICES，KUBE-NODEPORTS，KUBE-POSTROUTING，KUBE-MARK-MASQ和KUBE-MARK-DROP**五个链，并主要通过为KUBE-SERVICES chain增加rule来配制traffic routing 规则。我们可以看下自定义的这几个链的作用：

```bash
KUBE-MARK-DROP - [0:0] /*对于未能匹配到跳转规则的traffic set mark 0x8000，有此标记的数据包会在filter表drop掉*/
KUBE-MARK-MASQ - [0:0] /*对于符合条件的包 set mark 0x4000, 有此标记的数据包会在KUBE-POSTROUTING chain中统一做MASQUERADE*/
KUBE-NODEPORTS - [0:0] /*针对通过nodeport访问的package做的操作*/
KUBE-POSTROUTING - [0:0]
KUBE-SERVICES - [0:0] /*操作跳转规则的主要chain*/
```

同时，kube-proxy也为默认的prerouting、output和postrouting chain增加规则，使得数据包可以跳转至k8s自定义的chain，规则如下：

```bash
 -A PREROUTING -m comment --comment "kubernetes service portals" -j KUBE-SERVICES
 -A OUTPUT -m comment --comment "kubernetes service portals" -j KUBE-SERVICES
 -A POSTROUTING -m comment --comment "kubernetes postrouting rules" -j KUBE-POSTROUTING
```

如果service类型为nodePort，（从LB转发至node的数据包均属此类）那么将KUBE-NODEPORTS链中每个目的地址是NODE节点端口的数据包导入这个“KUBE-SVC-”链：

```bash
 -A KUBE-SERVICES -m comment --comment "kubernetes service nodeports; NOTE: this must be the last rule in this chain" -m addrtype --dst-type LOCAL -j KUBE-NODEPORTS
 -A KUBE-NODEPORTS -p tcp -m comment --comment "default/es1:http" -m tcp --dport 32135 -j KUBE-MARK-MASQ
 -A KUBE-NODEPORTS -p tcp -m comment --comment "default/es1:http" -m tcp --dport 32135 -j KUBE-SVC-LAS23QA33HXV7KBL
```

Iptables chain支持嵌套并因为依据不同的匹配条件可支持多种分支，比较难用标准的流程图来体现调用关系，建单抽象为下图： 

![img](/Users/lisai/go/src/llussy.github.io/images/201807101150064cc9ece6-146a-47d0-b91d-979db2f73b6c.png)

举个例子，在k8s集群中创建了一个名为my-service的服务，其中：

service vip:10.11.97.177

对应的后端两副本pod ip: 10.244.1.10、10.244.2.10

容器端口为:80      服务端口为:80

则kube-proxy为该service生成的iptables规则主要有以下几条：

```bash
 -A KUBE-SERVICES -d 10.11.97.177/32 -p tcp -m comment --comment "default/my-service: cluster IP" -m tcp --dport 80 -j KUBE-SVC-BEPXDJBUHFCSYIC3

 -A KUBE-SVC-BEPXDJBUHFCSYIC3 -m comment --comment “default/my-service:” -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-U4UWLP4OR3LOJBXU  //50%的概率轮询后端pod
 -A KUBE-SVC-BEPXDJBUHFCSYIC3 -m comment --comment "default/my-service:" -j KUBE-SEP-QHRWSLKOO5YUPI7O

 -A KUBE-SEP-U4UWLP4OR3LOJBXU -s 10.244.1.10/32 -m comment --comment "default/my-service:" -j KUBE-MARK-MASQ
 -A KUBE-SEP-U4UWLP4OR3LOJBXU -p tcp -m comment --comment "default/my-service:" -m tcp -j DNAT --to-destination 10.244.1.10:80

 -A KUBE-SEP-QHRWSLKOO5YUPI7O -s 10.244.2.10/32 -m comment --comment "default/my-service:" -j KUBE-MARK-MASQ
 -A KUBE-SEP-QHRWSLKOO5YUPI7O -p tcp -m comment --comment "default/my-service:" -m tcp -j DNAT --to-destination 10.244.2.10:80
```

kube-proxy通过循环的方式创建后端endpoint的转发，概率是通过probability后的1.0/float64(n-i)计算出来的，譬如有两个的场景，那么将会是一个0.5和1也就是第一个是50%概率第二个是100%概率，如果是三个的话类似，33%、50%、100%。

#### kube-proxy iptables的性能缺陷

k8s集群创建大规模服务时，会产生很多iptables规则，非增量式更新会引入一定的时延。iptables规则成倍增长，也会导致路由延迟带来访问延迟。大规模场景下，k8s 控制器和负载均衡都面临这挑战。例如，若集群中有N个节点，每个节点每秒有M个pod被创建，则控制器每秒需要创建N*M个endpoints，需要增加的iptables则是N*M的数倍。

### 参考

[浅谈 kubernetes service 那些事（上篇）](http://link.zhihu.com/?target=https%3A//sq.163yun.com/blog/article/174981072898940928)

 [浅谈 kubernetes service 那些事 （下篇）](http://link.zhihu.com/?target=https%3A//sq.163yun.com/blog/article/174982095700942848)

[k8s服务发现和负载均衡](http://senlinzhan.github.io/2017/12/10/k8s-service/)

