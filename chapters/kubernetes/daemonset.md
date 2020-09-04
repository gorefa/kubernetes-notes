```bash
kubectl rollout status -n kube-system daemonset calico-node 
kubectl rollout  undo -n kube-system daemonset calico-node 
```

