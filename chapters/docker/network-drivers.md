[toc]

Docker’s networking subsystem is pluggable, using drivers. Several drivers exist by default, and provide core networking functionality:

- `bridge`: The default network driver. If you don’t specify a driver, this is the type of network you are creating. **Bridge networks are usually used when your applications run in standalone containers that need to communicate.** See [bridge networks](https://docs.docker.com/network/bridge/).
- `host`: For standalone containers, remove network isolation between the container and the Docker host, and use the host’s networking directly. `host` is only available for swarm services on Docker 17.06 and higher. See [use the host network](https://docs.docker.com/network/host/).
- `overlay`: Overlay networks connect multiple Docker daemons together and enable swarm services to communicate with each other. You can also use overlay networks to facilitate communication between a swarm service and a standalone container, or between two standalone containers on different Docker daemons. This strategy removes the need to do OS-level routing between these containers. See [overlay networks](https://docs.docker.com/network/overlay/).
- `macvlan`: Macvlan networks allow you to assign a MAC address to a container, making it appear as a physical device on your network. The Docker daemon routes traffic to containers by their MAC addresses. Using the `macvlan` driver is sometimes the best choice when dealing with legacy applications that expect to be directly connected to the physical network, rather than routed through the Docker host’s network stack. See [Macvlan networks](https://docs.docker.com/network/macvlan/).
- `none`: For this container, disable all networking. Usually used in conjunction with a custom network driver. `none` is not available for swarm services. See [disable container networking](https://docs.docker.com/network/none/).
- [Network plugins](https://docs.docker.com/engine/extend/plugins_services/): You can install and use third-party network plugins with Docker. These plugins are available from [Docker Hub](https://hub.docker.com/search?category=network&q=&type=plugin) or from third-party vendors. See the vendor’s documentation for installing and using a given network plugin.



### docker默认网络模式

docker缺省配置三个网络类型：bridge host none

默认是bridge网络。可以控制在一个bridge网络内的container互相连通，不同bridge的container互相隔离。

#### bridge模式

bridge模式会为每一个容器分配 network namespace、设置ip等。

#### host模式

host模式这个容器将不会获得一个独立的Network Namespace，而是和宿主机共用一个Network Namespace。容器将不会虚拟出自己的网卡，配置自己的IP等，而是使用宿主机的IP和端口。

#### none模式

none模式Docker容器拥有自己的Network Namespace，但是，并不为Docker容器进行任何网络配置。也就是说，这个Docker容器没有网卡、IP、路由等信息。需要我们自己为Docker容器添加网卡、配置IP等。



```bash
# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
d3600f048159        bridge              bridge              local
af06a218f54a        host                host                local
d03cca9affa3        none                null                local

# docker network inspect bridge 详细信息

# brctl show 
bridge name     bridge id               STP enabled     interfaces
docker0         8000.0242137622bc       no
```

### docker跨主机通信网络模式

docker跨主机通信原生使用： overlay 和macvlan



### 参考

[docker的网络模式：bridge](https://www.jianshu.com/p/7dc04fb9b029)

[Docker的四种网络模式](https://blog.csdn.net/huanongying123/article/details/73556634)