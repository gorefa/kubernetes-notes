[toc]

### Overlay

  Overlay 在网络技术领域，指的是一种网络架构上`叠加的虚拟化技术模式`，其大体框架是对基础网络不进行大规模修改的条件下，实现应用在网络上的承载，并能与其它网络业务分离，并且以基于IP的基础网络技术为主。Overlay 技术是在现有的物理网络之上构建一个虚拟网络，上层应用只与虚拟网络相关。



#### VXLAN

VXLAN（Virtual eXtensible LAN）技术是当前最为主流的Overlay标准。

VXLAN的核心在于承载于物理网络上的隧道技术，这就意味着要对报文进行封装和解封装，因此需要硬件来加速处理。（在UDP传输层）

在VXLAN网络中，用于建立VXLAN隧道的端点设备称为VTEP（VXLAN Tunneling End Point，VXLAN隧道终结点），封装和解封装在VTEP节点上进行。

VXLAN网络的MAC表与隧道终端的绑定关系要用组播协议传播，而大规格组播协议离不开物理网络设备的支持。



### BGP

​		BGP 是一种**路径矢量路由协议**，用于传输自治系统间的路由信息，BGP 在启动的时候传播整张路由表，以后只传播网络变化的部分 触发更新它采用 TCP 连接传送信息，端口号为 179 在 Internet 上，BGP 需要通告的路由数目极大，由于 TCP 提供了可靠的传送机制，同时 TCP 使用滑动窗口机制，使得 BGP 可以不断地发送分组，而无需像 OSPF 或 EIGRP 那样停止发送并等待确认。

#### 自治系统

​		在互联网中，一个自治系统(AS)是一个有权自主地决定在本系统中应采用何种`路由协议`的小型单位。这个网络单位可以是一个简单的网络也可以是一个由一个或多个普通的`网络管理员`来控制的网络群体，它是一个单独的可管理的网络单元（例如一所大学，一个企业或者一个公司个体）。一个自治系统有时也被称为是一个路由选择域（routing domain）。

​		自治概念的提出实际是将互联网分为两层，一层是自治系统内部网络，可以将它成为第一层的路由，自治系统内部的路由器完成第一层区域的主机之间的分组交换。另外，如果一个自治系统管理内部的路由器，通过主干网路由器连接到主干区域（backbone area），连接自治系统的主干路由器就构成了主干区域，即第二层路由。

#### 互联网路由选择协议

​		互联网将路由选择协议分为两大类：内部网关协议（Interior Gateway Protocol，IGP），外部网关协议（External Gateway Protocol，EGP）。

​	    内部网关协议是在一个自治系统内部使用的路由选择协议，这与互联网中其它自治系统选择什么路由选择协议无关。目前，内部网关协议主要有：路由信息协议（Routing Information Protocol，RIP）和开放最短路径优先（Open Shortest Path First，OSPF）协议。

  	当源主机和目的主机处在不同的自治系统中，并且这两个自治系统使用不同的内部网管协议时，当分组传送到两个自治系统的边界时，就需要使用一种协议将路由信息传递到另一个自治系统中，这时就需要使用外部网关协议。目前，外部网关协议主要是边界网关协议（Border Gateway Protocol，BGP）。

### 好的文章

[Overlay网络](https://blog.csdn.net/zhaihaifei/article/details/74340428)

[图解Kubernetes网络一](https://mp.weixin.qq.com/s/NETiIHI7pib1Ws6ltPDtjA)

[图解Kubernetes网络二](https://mp.weixin.qq.com/s/QZhWxFKZ-RHAvQ_xev-sbg)

