[TOC]

#### 常用命令

```bash
kubectl logs mypod --previous  
kubectl delete pod -n kube-system traefik-q2ngc --grace-period=180
kubectl delete pod <pod> --force --grace-period=0 
kubectl exec -it -n monitoring test-f7bdb9769-ctxbs -- env COLUMNS=210 LINES=60 bash
kubectl scale deployments/kubernetes-bootcamp --replicas=3
kubectl rollout status ds/<daemonset-name>  # status
kubectl rollout undo deployments/kubernetes-bootcamp #回退
kubectl  get --raw "/apis/metrics.k8s.io/v1beta1/nodes" | jq
kubectl -v=8 top node
kubectl -v=8 get svc kubernetes
kubectl apply -f ds.yaml --dry-run -o go-template='{{.spec.updateStrategy.type}}{{"\n"}}'

# doc
kubectl api-resources 
kubectl explain statefulsets

journalctl -xef -u kubelet
```

#### 运行测试pod

```bash
kubectl run busytest --rm -it --image=busybox sh
kubectl run dig --rm -it --image=docker.io/azukiapp/dig /bin/sh
kubectl run centos --rm -it --image=centos bash
```

#### label  master

```shell
# label
kubectl get node --show-labels
kubectl label node yztest-ops-k8s24v8-yz role=master
kubectl label node yztest-ops-k8s24v8-yz role-
# spec.template.spec.nodeSelector:  node label位置

kubectl label node hostname node-role.kubernetes.io/node=
# master 充当node
kubectl taint node yztest-ops-k8s24v8-yz node-role.kubernetes.io/master-
# 恢复master角色
kubectl taint node yztest-ops-k8s24v8-yz node-role.kubernetes.io/master="":NoSchedule
```

#### k8s自动补全

```bash
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
```

#### patch

如果一个容器已经在运行，这时需要对一些容器属性进行修改，又不想删除容器，或不方便通过replace的方式进行更新。kubernetes还提供了一种在容器运行时，直接对容器进行修改的方式，就是patch命令。 
如前面创建pod的label是app=nginx-2，如果在运行过程中，需要把其label改为app=nginx-3，这patch命令如下：

```bash
kubectl patch pod rc-nginx-2-kpiqt -p '{"metadata":{"labels":{"app":"nginx-3"}}}'
```



#### rolling-update

`rolling-update`是一个非常重要的命令，对于已经部署并且正在运行的业务，rolling-update提供了不中断业务的更新方式。rolling-update每次起一个新的pod，等新pod完全起来后删除一个旧的pod，然后再起一个新的pod替换旧的pod，直到替换掉所有的pod。

rolling-update需要确保新的版本有不同的name，Version和label，否则会报错 。

```bash
kubectl rolling-update rc-nginx-2 -f rc-nginx.yaml 
```

如果在升级过程中，发现有问题还可以中途停止update，并回滚到前面版本

```bash
kubectl rolling-update rc-nginx-2 —rollback 
```



#### pod 与主机文件传输

```bash
kubectl cp foo-pod:/var/log/foo.log foo.log
kubectl cp localfile foo-pod:/etc/remotefile
```



#### kubeadm

```bash
kubeadm config print init-defaults
kubeadm config view
```

#### node上快速查看pid的容器名字

```bash
podinfo() {
  CID=$(cat /proc/$1/cgroup | awk -F '/' '{print $5}')
  CID=$(echo ${CID:7:10})
  crictl inspect $CID | jq '.status.labels["io.kubernetes.pod.name"]'
}

增加到~/.bashrc


# podinfo 16529
"traefik-6cc546b5-75q9n"

```



#### 参考

[kubelet常用命令](https://blog.csdn.net/xingwangc2014/article/details/51204224)