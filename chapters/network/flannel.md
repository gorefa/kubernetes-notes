[toc]

> 原文地址 [flannel跨主网络通信方案（UDP、VXLAN、HOST-GW）详解](https://mp.weixin.qq.com/s/hgVAjl0q9y0osgNfYIO4ng)

### flannel三层网络实现方式

需求描述：`node1（11.101.1.2）`上的`container1（10.244.0.13）`需要跨主机访问`node2（11.101.1.3）`上的`container2（10.244.1.14）`。对于`flannel`是如何实现的呢？

![image-20200730105305924](/Users/lisai/go/src/llussy.github.io/images/image-20200730105305924.png)

#### UDP跨主通信模式

<img src="/Users/lisai/go/src/llussy.github.io/images/image-20200730105520608.png" alt="image-20200730105520608" style="zoom:67%;" />

如上图所示：首先`flannel`会在各个节点上创建路由规则，这些路由规则存储在`etcd`中，跟宿主机节点IP一一对应。

然后在`node1`上，`container1`跨主访问`node2`上的`container2`，因为`container2`的`IP`地址为`10.244.1.14`，根据路由规则，从而进入到`flannel0`设备中。

最后`flannel0`看到`container1`要访问的`IP`地址为`10.244.1.14`的容器，因为`flannel`在`etcd`中存储着子网和宿主机`ip`的对应关系，所以能够找到`10.244.1.14`对应的宿主机`IP`为`11.101.1.3`，进而开始组装`UDP`数据包发送数据到目的主机。当然这个请求得以完成的原因每个节点上都启动着一个`flanneld udp`进程，都监听着`8285`端口，所以`node1`通过`flanneld`进程把数据包发送给`node2`的`flanneld`进程的相应端口即可。

可以看出`flannel UDP`模式提供了一个三层`OverLay`网络，这就好比在不同宿主机的容器上打通了一条隧道，容器不用关心`IP`地址即可直接通信。它首先对发出端的数据包进行`UDP`封装，然后在接收端进行解包，进而把包发送到目的容器地址。但是这里面有一个非常严重的问题，就是`flannel UDP`进程运行在用户态，而数据的交互和传递则在内核态完成，这就造成了为了传递数据，需要内核态和用户态的频繁切换，这个切换过程有一定性能损耗代价，所以`UDP`模式已经废弃。

#### VXLAN跨主通信模式

`VXLAN（Vitrue Extension lan`虚拟可扩展局域网）`Linux`本身支持的一种虚拟可扩展局域网。`VXLAN`完全在内核态完成上述封包和解包的过程。从而通过前面的隧道机制完成`overlay` 覆盖网络。`rfc7348`详细地介绍了`VXLAN`的实现机制。本质上`VXLAN`是一种隧道技术。通过将虚拟网络中的数据帧封装在实际物理网络中的报文中进行传输。具体实现方式为：将虚拟网络的数据帧添加`VXLAN`首部后，封装在物理网络中的`UDP`报文中，然后以传统网络的通信方式传送该`UDP`报文，到达目的主机后，去掉物理网络报文的头部信息以及`VXLAN`首部，将报文交付给目的终端。整个通信过程目的终端不会感知到物理网络的存在。

- VXLAN技术组网过程

  ![img](/Users/lisai/go/src/llussy.github.io/images/vxlan1.png)

  

  图中两台终端T1和T2位于不同的网络中，二者通过路由器来实现互通，通过`VXLAN`可以使得这两台终端在“逻辑上”位于“同一个”链路层网络中而与两台终端直接相连的路由器也在逻辑上构建了一条在虚拟链路中的通道`vxlan tunnel`，这样的路由器我们称之为`vxlan`隧道终端`(VXLAN Tunnel End Point, VTEP)`。在包含`VXLAN`的网络中，`VXLAN`的实现机制仅仅对`VTEP`节点可见。

  

- VXLAN通信原理

  ![img](/Users/lisai/go/src/llussy.github.io/images/vxlan2.png)

  

  `VXLAN`通过将逻辑网络中通信的数据帧封装在物理网络中进行传输，封装和解封装的过程由`VTEP`节点完成。`VXLAN`将逻辑网络中的数据帧添加`VXLAN`首部后，封装在物理网络中的UDP报文中传送，`VXLAN`首部的格式如下：

  ![img](/Users/lisai/go/src/llussy.github.io/images/vxlan3.png)

  `VXLAN`首部由8个字节组成，第1个字节为标志位，其中标志位I设为1表示是一个合法的`VXLAN`首部，其余标志则保留，在传输过程中必须置为0；第2-4字节为保留部分，第`5-7`字节为`VXLAN`标识符，用来表示唯一的一个逻辑网络；第8个字节同样为保留字段，暂未使用。`VXLAN`传输过程中，将逻辑链路网络的数据帧添加VXLAN首部后，依次添加UDP首部，IP首部，以太网帧首部后，在物理网络中传输，数据帧的封装格式可以用下图来描述：

  ![img](/Users/lisai/go/src/llussy.github.io/images/vxlan4.png)

  需要注意的是，外部`UDP`首部的目的端口号为`4789`，该数值为默认`VXLAN`解析程序的端口，外层`IP`首部中的源`IP`和目的`IP`地址均填写通信双方的`VTEP`地址，协议的其余部分和传统网络相同。

- 通信过程

对于处于同一个`VXLAN`的两台虚拟终端，其通信过程可以概括为如下的步骤：

1. 发送方 向 接收方发送数据帧，帧中包含了发送方和接收方的虚拟`MAC`地址。
2. 发送方连接的`VTEP`节点收到了数据帧，通过查找发送方所在的`VXLAN`以及接收方所连接的`VTEP`节点，将该报文添加`VXLAN`首部、外部`UDP`首部、外部IP首部后，发送给目的`VTEP`节点。
3. 报文经过物理网络传输到达目的`VTEP`节点。
4. 目的`VTEP`节点接收到报文后，拆除报文的外部`IP`首部和外部`UDP`首部，检查报文的`VNI`以及内部数据帧的目的`MAC`地址，确认接收方与本`VTEP`节点相连后，拆除`VXLAN`首部，将内部数据帧交付给接收方。
5. 接收方收到数据帧，传输完成。

通过以上的步骤可以看出：`VXLAN`的实现细节以及通信过程对于处于`VXLAN`中的发送方和接收方是不可见的，基于发送方和接收方的视角，其通信过程和二者真实处于同一链路层网络中的情况完全相同。



对于`Kubernetes flannel`也是完全依赖`linux vxlan`实现了`overlay`的跨主网络通信，如下图所示：

![img](/Users/lisai/go/src/llussy.github.io/images/vxlan5.png)从形式和流程上看，这个通信过程和上面基于`UDP`的通信方式是非常类似的，只不过`flannel UDP`进程换成了`VTEP`设备，通过`VTEP`设备完成封包和解包的过程，另外一点这个过程完全在内核中完成。`flannel.1`充当网桥的角色，进行`UDP`数据包的转发。其中`vxlan`的通信过程也是`flannel`网络插件中默认的通信方式，如下图所示：

![img](/Users/lisai/go/src/llussy.github.io/images/vxlan6.png)

巨人的肩膀：

```
https://blog.csdn.net/jsh13417/article/details/80303098
```

Flannel uses either the Kubernetes API or [etcd](https://github.com/coreos/etcd) directly to store the network configuration, the allocated subnets, and any auxiliary data (such as the host's public IP).

flanneld一旦获取子网租约、配置后端后，会将一些信息写入/run/flannel/subnet.env文件。

```bash
$ cat /var/run/flannel/subnet.env
FLANNEL_NETWORK=10.244.0.0/16
FLANNEL_SUBNET=10.244.3.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=true
```

##### 清空vxlan flannel网络

```bash
ifconfig cni0 down
ip link delete cni0
ifconfig flannel.1 down
ip link delete flannel.1
rm -rf /var/lib/cni/
```





#### host-gw跨主通信模式

​	host-gw在传输层走的是tcp。然后在网络层的源IP和目的IP均是容器的IP，虚拟IP。这就决定了二层互联，因为只有交换机是不关注源IP和目的IP。假如两台主机在两个lan中，二层不通，三层通，那么就需要路由器，而路由器是无法识别容器的这些ip。当然也可以配置路由规则，但是显然没有这么做的。

```bash
# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.21.8.254     0.0.0.0         UG    0      0        0 eth0
10.21.8.0       0.0.0.0         255.255.255.0   U     0      0        0 eth0
10.244.1.0      10.21.8.106     255.255.255.0   UG    0      0        0 eth0
10.244.2.0      10.21.8.105     255.255.255.0   UG    0      0        0 eth0
10.244.3.0      10.21.8.25      255.255.255.0   UG    0      0        0 eth0
169.254.0.0     0.0.0.0         255.255.0.0     U     1002   0        0 eth0
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
```



`host-gw`的工作原理就是将每个`flannel`子网转发地址设置成了该子网对应的宿主机的`IP`地址，通过这个过程，容器在通信过程中就减少了封包和解包的性能损耗。如下图所示：

![img](/Users/lisai/go/src/llussy.github.io/images/vxlan7.png)

1. 同`UDP、VXLAN`模式一致，通过容器A的路由表`IP`包到达`cni0`,到达`cni0` 的`IP`包匹配到`node1`中的路由规则`（10.244.1.1/24）`，且网关为`11.101.1.3`，即主机`node2`，所以内核将`IP`包发送给`node2`,`IP`包通过物理网络到达`node2`的`eth0`;
2. 到达`node2`的`eth0` 的`IP`包匹配到`node2`上的路由表`（10.244.0.1/24）`，`IP`包转发给`cni0`, `cni0`将`IP`包转发给连接在`cni0`上的`Container2`。

通过上述这个过程可以看出这台主机的`host`就充当了容器通信路径里的网关，`IP`封装成帧的时候，会使路由表中的下一跳来设置目的`MAC`地址，它会经过二层网络达到宿主机，但同时这个限制也是有问题的，首先要求我们必须保证集群内部所有主机二层网络必须是连通的，然后在大规模集群路由表的动态更新也存在一定压力。采用`host-gw`模式后，`flanneld`的唯一作用就是负责主机上路由表的动态更新。当然这个限制也是有解决方案的，这里不在过多介绍，详细可以了解`Calico`。





[k8s与网络--Flannel解读](https://segmentfault.com/a/1190000016304924)

[Kubernetes网络组件之Flannel策略实践(vxlan、host-gw)](https://blog.51cto.com/14143894/2462379)

