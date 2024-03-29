# Table of contents

* [Introduction](README.md)
* [docker](docker/README.md)
  * [docker](chapters/docker/docker.md)
  * [dockerfile](chapters/docker/dockerfile.md)
  * [docker-tls](chapters/docker/dockertls.md)
  * [harbor](chapters/docker/harbor.md)
* [kubernetes install](kubernetes-install/README.md)
  * [kubeadm1.18安装](chapters/install/kubeadm1.18.md)
  * [Page 1](kubernetes-install/page-1/README.md)
    * [Page 2](kubernetes-install/page-1/page-2.md)
* [kubernetes](kubernetes/README.md)
  * [kubernetes component](kubernetes/kubernetes-component.md)
  * [concept](chapters/kubernetes/concept.md)
  * [certs](chapters/kubernetes/certs.md)
  * [相关端口](kubernetes/xiang-guan-duan-kou.md)
  * [相关命令](chapters/kubernetes/command.md)
  * [kube-proxy](chapters/kubernetes/kube-proxy.md)
  * [ipvs](kubernetes/ipvs.md)
  * [configmap](kubernetes/configmap.md)
  * [pod](kubernetes/pod.md)
  * [services](kubernetes/services.md)
  * [deployment](kubernetes/deployment.md)
  * [daemonset](kubernetes/daemonset.md)
  * [statefulset](kubernetes/statefulset.md)
  * [liveness和readiness](chapters/kubernetes/probe.md)
  * [应用创建流程](chapters/kubernetes/app-create-process.md)
  * [kubernetes相关优化](chapters/kubernetes/optimize.md)
* [etcd](etcd/README.md)
  * [etcd安装](chapters/etcd/etcd-install.md)
  * [etcd配置文件](etcd/etcd-pei-zhi-wen-jian.md)
  * [etcd常用命令](chapters/etcd/etcd-cmd.md)
  * [etcd备份恢复](chapters/etcd/etcd-backup-recover.md)
* [montior](montior/README.md)
  * [cAdvisor](chapters/monitor/cAdvisor.md)
  * [promethues](montior/promethues/README.md)
    * [prometheus](chapters/monitor/prometheus/prometheus.md)
    * [query](chapters/monitor/prometheus/query.md)
    * [metrics类型](chapters/monitor/prometheus/metrics.md)
    * [job](montior/promethues/job.md)
    * [pushgateway](chapters/monitor/prometheus/pushgateway.md)
    * [结合consul](chapters/monitor/prometheus/consul.md)
* [log](log/README.md)
  * [elk日志收集](chapters/log/elk.md)
  * [loki](chapters/log/loki.md)
* [storage](storage/README.md)
  * [k8s存储](chapters/storage/k8s-storage.md)
  * [hostPath-emptyDir](chapters/storage/hostPath-emptyDir.md)
  * [ceph-install](chapters/storage/ceph-install.md)
  * [k8s-ceph-storageclass](chapters/storage/k8s-ceph-storageclass.md)
* [CICD](cicd/README.md)
  * [helm3](chapters/CICD/helm.md)
  * [jenkins](chapters/CICD/jenkins.md)
* [ingress](ingress.md)
* [client-go](client-go.md)
* [Troubleshooting](troubleshooting/README.md)
  * [抓包分析](chapters/Troubleshooting/caught-analysis.md)
  * [日常中遇到的问题](chapters/Troubleshooting/kubernetes.md)
* [TODO](todo.md)
