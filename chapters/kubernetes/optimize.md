[toc]



### node维护

**node服务器出现问题，优先保障业务可以先把业务切到其他node节点上去**

可以优先kubelet delete po -n namespaces xxxxpod  --grace-period=120   把业务切走 

 

1.禁止新调度 到node上
 kubectl cordon nodehostname
2.驱逐这个node的容器 过程比较慢
kubectl drain nodehostname --ignore-daemonsets

3.解锁禁止调度。恢复业务
kubectl uncordon nodehostname



### clear cache

宿主机内存不足是可清理定时晚上清理cache

```bash
#!/bin/bash
meminfo=$(cat /proc/meminfo |awk '/MemFree/{printf("%d\n",$2/1000/1000)}')
if [ $meminfo -lt 3 ];then
   sync;sync;sync;
   sleep 1;
   echo 3 > /proc/sys/vm/drop_caches;
   sleep 3;
   echo 1 > /proc/sys/vm/drop_caches;
else
   echo "now free mem $meminfo G";
fi
exit 0;
```

### docker 日志切割

```bash
# cat /etc/docker/daemon.json 
{
  "registry-mirrors": ["https://934du3yi.mirror.aliyuncs.com"],
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "3000m",
    "max-file": "10"
  }
}
```

或修改/etc/sysconfig/docker 配置  [Kubernetes Pod日志太大导致空间问题](https://www.cnblogs.com/ericnie/p/8297738.html)

### 修改pause容器镜像地址

修改/etc/systemd/system/kubelet.service.d/10-kubelet.conf 配置文件

增加：Environment="KUBELET_LIMIT_ARGS=--pod-infra-container-image=docker.test.com/file/pause-amd64:3.1"

docker.test.com/file/pause-amd64:3.1 为内网镜像地址



### Docker Storage Driver To Overlay2

**Note**: If you use OverlayFS, use the `overlay2` driver rather than the `overlay` driver, because it is more efficient in terms of inode utilization. To use the new driver, you need version 4.0 or higher of the Linux kernel, unless you are a Docker EE user on RHEL or CentOS, in which case you need version 3.10.0-514 or higher of the kernel and to follow some extra steps.

```bash
# 更新前
1.禁止新调度 到node上
 kubectl cordon k8snode1
2.驱逐这个node的容器 过程比较慢
kubectl drain k8snode1 --ignore-daemonsets

# 更新内核版本 

# 取消数据盘挂载(fstab)，重启
sed -i 's/^.*data/#&/' /etc/fstab 

# 重新格式化磁盘 加上ftype=1
mkfs.xfs -n ftype=1 /dev/sda9 -f
更新fstab

# 配置daemon.json 支持overlay2

# kubectl uncordon k8snode1

```



### kubelet node 资源预留

**Kubelet Node Allocatable**

- Kubelet Node Allocatable用来为Kube组件和System进程预留资源，从而保证当节点出现满负荷时也能保证Kube和System进程有足够的资源。
- 目前支持cpu, memory, ephemeral-storage三种资源预留。
- Node Capacity是Node的所有硬件资源，kube-reserved是给kube组件预留的资源，system-reserved是给System进程预留的资源， eviction-threshold是kubelet eviction的阈值设定，allocatable才是真正scheduler调度Pod时的参考值（保证Node上所有Pods的request resource不超过Allocatable）。

![img](https://llussy.github.io/images/5b6011d7ab64416875001bd8.png)

**参数**

```bash
1：设置预留系统服务的资源 
--system-reserved=cpu=200m,memory=1G

2：设置预留给k8s组件的资源（kubelet,kube-proxy,dockerd等）
--kube-reserved=cpu=200m,memory=1G
系统内存-system-reserved-kube-reserved 就是可以分配给pod的内存

3：驱逐条件#驱逐条件只对内存硬盘有效，对CPU无效
--eviction-hard=memory.available<500Mi,nodefs.available<1Gi,imagefs.available<100Gi                    

4：最小驱逐
--eviction-minimum-reclaim="memory.available=0Mi,nodefs.available=500Mi,imagefs.available=2Gi"   #暂时不用

5：节点状态更新时间
--node-status-update-frequency =10s    #(默认10S，不要轻易修改。很容器node再read和notread问题) 

6：驱逐等待时间
--eviction-pressure-transition-period=60s
```

**配置**

```bash
--eviction-hard=memory.available<1Gi  #系统内存小于1G,停止K8S新业务(现有业务不受影响)，并驱逐溢出的pod部分，同时发给API信息，不接受新的，创造pod请求
--system-reserved=cpu=500m,memory=1Gi   #系统层预留0.5个核和1G内存
--kube-reserved=cpu=500m,memory=1Gi       #K8S主进程预留0.5个核和1G内存
```

**修改 /etc/systemd/system/kubelet.service.d/10-kubelet.conf**

```toml
[Service]
ExecStartPre=/usr/bin/mkdir -p /sys/fs/cgroup/cpuset/system.slice/kubelet.service      #没有这2行会报错，K8S启动异常切记
ExecStartPre=/usr/bin/mkdir -p /sys/fs/cgroup/hugetlb/system.slice/kubelet.service     #没有这2行会报错，K8S启动异常切记
Environment="KUBELET_KUBECONFIG_ARGS=--kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_SYSTEM_PODS_ARGS=--pod-manifest-path=/etc/kubernetes/manifests --allow-privileged=true"
Environment="KUBELET_NETWORK_ARGS=--network-plugin=cni --cni-conf-dir=/etc/cni/net.d --cni-bin-dir=/opt/cni/bin"
Environment="KUBELET_DNS_ARGS=--cluster-dns=10.96.0.10 --cluster-domain=cluster.local"
Environment="KUBELET_AUTHZ_ARGS=--authorization-mode=Webhook --client-ca-file=/etc/kubernetes/pki/ca.pem"
Environment="KUBELET_CADVISOR_ARGS=--cadvisor-port=0"
Environment="KUBELET_CERTIFICATE_ARGS=--rotate-certificates=true --cert-dir=/var/lib/kubelet/pki"
Environment="KUBELET_EXTRA_ARGS=--node-labels=node-role.kubernetes.io/node='' --logtostderr=true --v=0"
Environment="KUBELET_LIMIT_ARGS=--enforce-node-allocatable=pods,kube-reserved,system-reserved --kube-reserved-cgroup=/system.slice/kubelet.service --system-reserved-cgroup=/system.slice --eviction-hard=memory.available<200Mi --system-reserved=memory=200Mi --kube-reserved=cpu=200m,memory=200Mi --node-status-update-frequency=10s --eviction-pressure-transition-period=60s --pod-infra-container-image=docker.gorefa.com/file/pause-amd64:3.1"
ExecStart=
ExecStart=/usr/local/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_SYSTEM_PODS_ARGS $KUBELET_NETWORK_ARGS $KUBELET_DNS_ARGS $KUBELET_AUTHZ_ARGS $KUBELET_CADVISOR_ARGS $KUBELET_CERTIFICATE_ARGS $KUBELET_EXTRA_ARGS $KUBELET_LIMIT_ARGS

```

**参考**

[kubeernetes节点资源限制](https://www.cnblogs.com/ssss429170331/p/7685163.html)

[从一次集群雪崩看Kubelet资源预留的正确姿势](<https://my.oschina.net/jxcdwangtao/blog/1629059>)

