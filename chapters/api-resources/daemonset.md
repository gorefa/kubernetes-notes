```bash
kubectl rollout status -n kube-system daemonset calico-node 
kubectl rollout  undo -n kube-system daemonset calico-node 
```

### 更新策略

**RollingUpdate**

**OnDelete** 

OnDelete 其实也是一个很好的更新策略，就是模板更新之后，pod 不会有任何变化，需要我们手动控制。我们去删除某一个节点对应的 pod，它就会重建，不删除的话它就不会重建，这样的话对于一些我们需要手动控制的特殊需求也会有特别好的作用。

