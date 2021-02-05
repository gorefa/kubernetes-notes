[toc]

**namespace 是 Linux 内核用来隔离内核资源的方式。**为虚拟化而生。

PID,IPC,Network等系统资源不再是全局性的，而是属于某个特定的Namespace。每个namespace下的资源对于其他namespace下的资源都是透明的。

Linux Namespaces机制为实现基于容器的虚拟化技术提供了很好的基础，LXC（Linux containers）就是利用这一特性实现了资源的隔离。不同container内的进程属于不同的Namespace，彼此透明，互不干扰。

​	

​		Linux namespace 实现了 6 项资源隔离，基本上涵盖了一个小型操作系统的运行要素，包括主机名、用户权限、文件系统、网络、进程号、进程间通信。Linux Namespace 是操作系统虚拟化技术（e.g. 容器）的底层实现支撑。

![image-20201010112229079](/Users/lisai/go/src/llussy.github.io/images/image-20201010112229079.png)

### network namespace

network namespace 是实现网络虚拟化的重要功能，它能创建多个隔离的网络空间，它们有独自的网络栈信息。不管是虚拟机还是容器，运行的时候仿佛自己就在独立的网络中。

network namespace 之间通信通过 "veth pair"。

veth pair 可以实现两个 network namespace 之间的通信，但是当多个 namespace 需要通信的时候，就无能为力了。 讲到多个网络设备通信，我们首先想到的交换机和路由器。因为这里要考虑的只是同个网络，所以只用到交换机的功能。linux 也提供了虚拟交换机的功能。

[network namespace 简介](https://blog.csdn.net/u012707739/article/details/78163354)

[Linux 操作系统原理 — Namespace 资源隔离](https://blog.csdn.net/Jmilk/article/details/108897853)

