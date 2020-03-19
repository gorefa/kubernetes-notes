[toc]

### endpoint

k8s会根据service和pod直接的关系创建endpoint，可以通过kubectl get ep查看

### pause

**功能**：**实现pod间容器通信。**

k8s在启动容器的时候会先启动一个pause容器，这个容器就是实现这个功能的。pause容器做为Pod的网络接入点，Pod中其他的容器会使用**容器映射模式**启动并接入到这个pause容器。属于同一个Pod的所有容器共享网络的namespace。

kubernetes中的pause容器主要为每个业务容器提供以下功能：

- 在pod中担任Linux命名空间共享的基础；
- 启用pid命名空间，开启init进程。



### 参考

[Kubernetes中的Pause容器究竟是什么?](https://jimmysong.io/posts/what-is-a-pause-container/)

