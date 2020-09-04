[TOC]

### 前提准备（所有node）

#### 升级内核

```bash
mkdir -p /data/soft/packs
cd /data/soft/packs
wget http://10.21.8.20/kernel/kernel-ml-4.20.2-1.el7.elrepo.x86_64.rpm
yum install -y kernel-ml-4.20.2-1.el7.elrepo.x86_64.rpm
grub2-editenv list
cat /boot/grub2/grub.cfg |grep menuentry


grub2-set-default  "CentOS Linux (4.20.2-1.el7.elrepo.x86_64) 7 (Core)"
grub2-editenv list
```



#### 关闭selinux iptables

#### 关闭swap

```bash
swapoff -a

cat << EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
```

#### docker 安装

```bash
# 开启磁盘ftype
# 先注释掉data盘然后重启格式化
mkfs.xfs -n ftype=1 /dev/sda9 -f
mount /dev/sda9 /data/
xfs_info /dev/sda9
cat << EOF >> /etc/fstab
/dev/sda9                                 /data                   xfs     defaults        0 0
EOF

mkdir -p /data/soft/packs
cd /data/soft/packs
wget http://10.21.8.20/k8s/docker1809/containerd.io-1.2.6-3.3.el7.x86_64.rpm
wget http://10.21.8.20/k8s/docker1809/docker-ce-18.09.8-3.el7.x86_64.rpm
wget http://10.21.8.20/k8s/docker1809/docker-ce-cli-18.09.8-3.el7.x86_64.rpm
yum install ./*.rpm -y
mkdir -p /etc/docker/
cat>/etc/docker/daemon.json<<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "registry-mirrors": ["https://fz5yth0r.mirror.aliyuncs.com"],
  "data-root": "/data/docker",
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  }
}
EOF
yum install -y epel-release bash-completion && cp /usr/share/bash-completion/completions/docker /etc/bash_completion.d/
systemctl enable --now docker
```

#### kuadm kubelet安装下载

##### 使用阿里云镜像

```bash
##aliyun 镜像
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

##安装：注意版本  node节点不需要安装kubectl
yum install kubelet kubeadm kubectl

systemctl enable kubelet      
```

##### 使用google镜像下载

```bash
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF

yum install --downloadonly --downloaddir=./ kubelet kubeadm kubectl kubernetes-cni

# 安装
yum install cri-tools-1.13.0-0.x86_64.rpm  kubectl-1.16.0-0.x86_64.rpm kubelet-1.16.0-0.x86_64.rpm kubernetes-cni-0.7.5-0.x86_64.rpm kubeadm-1.16.0-0.x86_64.rpm
```

#### 启动ipvs

(**Notes**: use `nf_conntrack` instead of `nf_conntrack_ipv4` for Linux kernel 4.19 and later)

```bash
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack
EOF
chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack

yum install ipset ipvsadm -y
```

https://kubernetes.io/blog/2018/07/09/ipvs-based-in-cluster-load-balancing-deep-dive/#run-kube-proxy-in-ipvs-mode

### kubeadm 单节点

```bash
#  导出默认配置
kubeadm config print init-defaults > kubeadm.yaml

apiVersion: kubeadm.k8s.io/v1beta2
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 10.21.8.104
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  name: yztest-k8s-master-104v8-yz
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.aliyuncs.com/k8sxio
kind: ClusterConfiguration
kubernetesVersion: v1.18.5
networking:
  dnsDomain: cluster.local
  podSubnet: 10.244.0.0/16
  serviceSubnet: 10.96.0.0/12
scheduler: {}
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs


kubeadm init --config kubeadm.yaml 

# 查看token
kubeadm token list

# 查看 discovery-token-ca-cert-hash
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \
   openssl dgst -sha256 -hex | sed 's/^.* //'
      

# 查看配置
kubectl -n kube-system get cm kubeadm-config -oyaml
```



### kubeadm 部署集群

```bash
kubeadm init --config kubeadm.yaml  --upload-certs
kubectl taint nodes --all node-role.kubernetes.io/master-  # master允许调度


The --upload-certs flag is used to upload the certificates that should be shared across all the control-plane instances to the cluster. If instead, you prefer to copy certs across control-plane nodes manually or using automation tools, please remove this flag and refer to Manual certificate distribution section below.

主要增加  controlPlaneEndpoint: "k8s-master.llussy.com:6443"
k8s-master.llussy.com 此域名可以使用负载或者dns指到三台master,推荐负载

# cat kubeadm.yaml 
apiVersion: kubeadm.k8s.io/v1beta2
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 10.21.8.104
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  name: yztest-k8s-master-104v8-yz
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.aliyuncs.com/k8sxio
controlPlaneEndpoint: "k8s-master.llussy.com:6443"
kind: ClusterConfiguration
kubernetesVersion: v1.18.5
networking:
  dnsDomain: cluster.local
  podSubnet: 10.244.0.0/16
  serviceSubnet: 10.96.0.0/12
scheduler: {}
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs

```



#### 其他master加入集群

```bash
kubeadm join k8s-master.llussy.com:6443 --token vu6530.4hvg9ee2h7eok3yc     --discovery-token-ca-cert-hash sha256:617e8fcc0fccc4798c7857a0ec646137115c598a4913aa1efb538e8913dcd5f7     --control-plane --certificate-key f4c47dfd3568d12e8d6a6b38580e08326ce218fe2c06bb8bfbd55d6a789e6120
```

#### calico

```bash
kubectl apply -f https://docs.projectcalico.org/v3.8/manifests/calico.yaml
# 或者
kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml
kubectl create -f https://docs.projectcalico.org/manifests/custom-resources.yaml

```

#### flannel

```bash
# 清空calico环境  在所有node

rm -f /etc/cni/net.d/calico-kubeconfig
rm -f /etc/cni/net.d/10-calico.conflist

wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

# host-gw 只能在二层通信
sed 's/vxlan/host-gw/' -i kube-flannel.yml

https://prefetch.net/blog/2018/02/20/getting-the-flannel-host-gw-working-with-kubernetes/
```



#### 查看etcd中的数据

```bash
# kubectl  get node 
NAME                         STATUS   ROLES    AGE   VERSION
yztest-k8s-master-104v8-yz   Ready    master   87m   v1.18.5
yztest-k8s-master-105v8-yz   Ready    master   71m   v1.18.5
yztest-k8s-master-106v8-yz   Ready    master   76m   v1.18.5
# kubectl  cluster-info 
Kubernetes master is running at https://k8s-master.llussy.com:6443
KubeDNS is running at https://k8s-master.llussy.com:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

# kubectl  -n kube-system exec -it etcd-yztest-k8s-master-104v8-yz --  etcdctl --cert /etc/kubernetes/pki/etcd/peer.crt --key /etc/kubernetes/pki/etcd/peer.key --cacert /etc/kubernetes/pki/etcd/ca.crt --endpoints "https://10.21.8.104:2379,https://10.21.8.105:2379,https://10.21.8.106:2379" endpoint status -w table
+--------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|         ENDPOINT         |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+--------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| https://10.21.8.104:2379 | 6a8bac49cdb19366 |   3.4.3 |  6.9 MB |      true |      false |         4 |      17220 |              17220 |        |
| https://10.21.8.105:2379 | 312474627f846ab5 |   3.4.3 |  6.9 MB |     false |      false |         4 |      17220 |              17220 |        |
| https://10.21.8.106:2379 | a7254796a89272af |   3.4.3 |  6.9 MB |     false |      false |         4 |      17220 |              17220 |        |
+--------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+

# kubectl  -n kube-system exec -it etcd-yztest-k8s-master-104v8-yz --  etcdctl --cert /etc/kubernetes/pki/etcd/peer.crt --key /etc/kubernetes/pki/etcd/peer.key --cacert /etc/kubernetes/pki/etcd/ca.crt --endpoints "https://10.21.8.104:2379,https://10.21.8.105:2379,https://10.21.8.106:2379" member list -w table
+------------------+---------+----------------------------+--------------------------+--------------------------+------------+
|        ID        | STATUS  |            NAME            |        PEER ADDRS        |       CLIENT ADDRS       | IS LEARNER |
+------------------+---------+----------------------------+--------------------------+--------------------------+------------+
| 312474627f846ab5 | started | yztest-k8s-master-105v8-yz | https://10.21.8.105:2380 | https://10.21.8.105:2379 |      false |
| 6a8bac49cdb19366 | started | yztest-k8s-master-104v8-yz | https://10.21.8.104:2380 | https://10.21.8.104:2379 |      false |
| a7254796a89272af | started | yztest-k8s-master-106v8-yz | https://10.21.8.106:2380 | https://10.21.8.106:2379 |      false |
+------------------+---------+----------------------------+--------------------------+--------------------------+------------+

#kubectl  -n kube-system exec -it etcd-yztest-k8s-master-104v8-yz -- etcdctl --cert /etc/kubernetes/pki/etcd/peer.crt --key /etc/kubernetes/pki/etcd/peer.key --cacert /etc/kubernetes/pki/etcd/ca.crt --endpoints "https://10.21.8.104:2379,https://10.21.8.105:2379,https://10.21.8.106:2379" get / --prefix --keys-only  | grep flann

docker run --rm -it \
--net host \
-v /etc/kubernetes:/etc/kubernetes quay.io/coreos/etcd:v3.4.3 etcdctl \
--cert-file /etc/kubernetes/pki/etcd/peer.crt \
--key-file /etc/kubernetes/pki/etcd/peer.key \
--ca-file /etc/kubernetes/pki/etcd/ca.crt \
--endpoints "https://10.21.8.104:2379,https://10.21.8.105:2379,https://10.21.8.106:2379" cluster-health
```

### 环境清空

```bash
# 之前二进制方式部署的 环境清空
systemctl stop kubelet
docker ps -qa |xargs docker stop
docker ps -qa |xargs docker rm
yum remove docker-ce -y
rm -rf /var/lib/docker
rm -rf /usr/local/bin/kubelet
rm -rf /usr/local/bin/kubectl
rm -rf /opt/cni/bin
rm -rf /etc/etcd
rm -rf /etc/kubernetes
rm -rf /etc/haproxy
rm -rf /usr/lib/python2.7/site-packages/ansible/plugins/connection/kubectl.py
rm -rf /etc/sysctl.d/kubernetes.conf
rm -rf /usr/libexec/kubernetes
rm -rf /var/log/pods
rm -rf /var/lib/kubelet
rm -rf /var/log/kubernetes
rm -rf /var/log/containers
rm -rf /home/k8s/kubernetes.conf
rm -rf /etc/systemd/system/multi-user.target.wants/kubelet.service
rm -rf /etc/systemd/system/kubelet.service.d
rm -rf /usr/lib/systemd/system/kubelet.service
rm -rf /usr/lib/python2.7/site-packages/ansible/plugins/connection/kubectl.py*
rm -rf /usr/lib/python2.7/site-packages/sos/plugins/kubernetes.py*
yum install  docker-ce -y
systemctl start docker 
```

