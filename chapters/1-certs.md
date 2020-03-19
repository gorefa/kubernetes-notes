[toc]

### openssl

```bash
[ req ]
default_bits = 2048
default_md = sha256
distinguished_name = req_distinguished_name

[req_distinguished_name]

[ v3_ca ]
basicConstraints = critical, CA:TRUE
keyUsage = critical, digitalSignature, keyEncipherment, keyCertSign

[ v3_req_server ]
basicConstraints = CA:FALSE
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth

[ v3_req_client ]
basicConstraints = CA:FALSE
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth

[ v3_req_apiserver ]
basicConstraints = CA:FALSE
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names_cluster

[ v3_req_etcd ]
basicConstraints = CA:FALSE
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth, clientAuth
subjectAltName = @alt_names_etcd

[ alt_names_cluster ]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster.local
DNS.5 = localhost
IP.1 = 10.96.0.1
IP.2 = 127.0.0.1
IP.3 = 10.90.28.220 # VIP
IP.4 = 10.90.28.10  # master
IP.5 = 10.90.28.11  # master
IP.8 = 10.90.28.26  # master

[ alt_names_etcd ]
DNS.1 = localhost
IP.1 = 127.0.0.1
IP.2 = 10.90.28.10 # etcd
IP.3 = 10.90.28.11 # etcd
IP.4 = 10.90.28.26 # etcd
```

cd  /etc/kubernetes/pki

**kubernetes-ca**

```bash
openssl genrsa -out ca.key 2048
openssl req -x509 -new -nodes -key ca.key -config openssl.cnf -subj "/CN=kubernetes-ca" -extensions v3_ca -out ca.crt -days 10000
```

**etcd-ca**

```bash
openssl genrsa -out etcd/ca.key 2048
openssl req -x509 -new -nodes -key etcd/ca.key -config openssl.cnf -subj "/CN=etcd-ca" -extensions v3_ca -out etcd/ca.crt -days 10000
```

**front-proxy-ca**

```bash
openssl genrsa -out front-proxy-ca.key 2048
openssl req -x509 -new -nodes -key front-proxy-ca.key -config openssl.cnf -subj "/CN=kubernetes-ca" -extensions v3_ca -out front-proxy-ca.crt -days 10000
```

**apiserver-etcd-client**

```bash
openssl genrsa -out apiserver-etcd-client.key 2048
openssl req -new -key apiserver-etcd-client.key -subj "/CN=apiserver-etcd-client/O=system:masters" -out apiserver-etcd-client.csr
openssl x509 -in apiserver-etcd-client.csr -req -CA etcd/ca.crt -CAkey etcd/ca.key -CAcreateserial -extensions v3_req_etcd -extfile openssl.cnf -out apiserver-etcd-client.crt -days 10000
```

**kube-etcd**

```bash
openssl genrsa -out etcd/server.key 2048
openssl req -new -key etcd/server.key -subj "/CN=etcd-server" -out etcd/server.csr
openssl x509 -in etcd/server.csr -req -CA etcd/ca.crt -CAkey etcd/ca.key -CAcreateserial -extensions v3_req_etcd -extfile openssl.cnf -out etcd/server.crt -days 10000
```

**kube-etcd-peer**

```bash
openssl genrsa -out etcd/peer.key 2048
openssl req -new -key etcd/peer.key -subj "/CN=etcd-peer" -out etcd/peer.csr
openssl x509 -in etcd/peer.csr -req -CA etcd/ca.crt -CAkey etcd/ca.key -CAcreateserial -extensions v3_req_etcd -extfile openssl.cnf -out etcd/peer.crt -days 10000
```

**kube-etcd-healthcheck-client**

```bash
openssl genrsa -out etcd/healthcheck-client.key 2048
openssl req -new -key etcd/healthcheck-client.key -subj "/CN=etcd-client" -out etcd/healthcheck-client.csr
openssl x509 -in etcd/healthcheck-client.csr -req -CA etcd/ca.crt -CAkey etcd/ca.key -CAcreateserial -extensions v3_req_etcd -extfile openssl.cnf -out etcd/healthcheck-client.crt -days 10000
```

**kube-apiserver**

```bash
openssl genrsa -out apiserver.key 2048
openssl req -new -key apiserver.key -subj "/CN=kube-apiserver" -config openssl.cnf -out apiserver.csr
openssl x509 -req -in apiserver.csr -CA ca.crt -CAkey ca.key -CAcreateserial -days 10000 -extensions v3_req_apiserver -extfile openssl.cnf -out apiserver.crt
```

**apiserver-kubelet-client**

```bash
openssl genrsa -out  apiserver-kubelet-client.key 2048
openssl req -new -key apiserver-kubelet-client.key -subj "/CN=apiserver-kubelet-client/O=system:masters" -out apiserver-kubelet-client.csr
openssl x509 -req -in apiserver-kubelet-client.csr -CA ca.crt -CAkey ca.key -CAcreateserial -days 10000 -extensions v3_req_client -extfile openssl.cnf -out apiserver-kubelet-client.crt
```

**front-proxy-client**

```bash
openssl genrsa -out  front-proxy-client.key 2048
openssl req -new -key front-proxy-client.key -subj "/CN=front-proxy-client" -out front-proxy-client.csr
openssl x509 -req -in front-proxy-client.csr -CA front-proxy-ca.crt -CAkey front-proxy-ca.key -CAcreateserial -days 10000 -extensions v3_req_client -extfile openssl.cnf -out front-proxy-client.crt
```

**kube-scheduler**

```bash
openssl genrsa -out  kube-scheduler.key 2048
openssl req -new -key kube-scheduler.key -subj "/CN=system:kube-scheduler" -out kube-scheduler.csr
openssl x509 -req -in kube-scheduler.csr -CA ca.crt -CAkey ca.key -CAcreateserial -days 10000 -extensions v3_req_client -extfile openssl.cnf -out kube-scheduler.crt
```

**sa.pub sa.key**

```bash
openssl genrsa -out  sa.key 2048
openssl ecparam -name secp521r1 -genkey -noout -out sa.key
openssl ec -in sa.key -outform PEM -pubout -out sa.pub
openssl req -new -sha256 -key sa.key -subj "/CN=system:kube-controller-manager" -out sa.csr
openssl x509 -req -in sa.csr -CA ca.crt -CAkey ca.key -CAcreateserial -days 10000 -extensions v3_req_client -extfile openssl.cnf -out sa.crt
```

**admin**

```bash
openssl genrsa -out  admin.key 2048
openssl req -new -key admin.key -subj "/CN=kubernetes-admin/O=system:masters" -out admin.csr
openssl x509 -req -in admin.csr -CA ca.crt -CAkey ca.key -CAcreateserial -days 10000 -extensions v3_req_client -extfile openssl.cnf -out admin.crt
```

清理 csr srl(csr只要key不变那每次生成都是一样的,所以可以删除,如果后期根据ca重新生成证书来添加ip的话可以此处不删除)

```bash
find . -name "*.csr" -o -name "*.srl"|xargs  rm -f
```



**kube-controller-manager**

```bash
CLUSTER_NAME="kubernetes"
KUBE_USER="system:kube-controller-manager"
KUBE_CERT="sa"
KUBE_CONFIG="controller-manager.kubeconfig"

# 设置集群参数
kubectl config set-cluster ${CLUSTER_NAME} \
  --certificate-authority=/etc/kubernetes/pki/ca.crt \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=/etc/kubernetes/${KUBE_CONFIG}

# 设置客户端认证参数
kubectl config set-credentials ${KUBE_USER} \
  --client-certificate=/etc/kubernetes/pki/${KUBE_CERT}.crt \
  --client-key=/etc/kubernetes/pki/${KUBE_CERT}.key \
  --embed-certs=true \
  --kubeconfig=/etc/kubernetes/${KUBE_CONFIG}

# 设置上下文参数
kubectl config set-context ${KUBE_USER}@${CLUSTER_NAME} \
  --cluster=${CLUSTER_NAME} \
  --user=${KUBE_USER} \
  --kubeconfig=/etc/kubernetes/${KUBE_CONFIG}

# 设置当前使用的上下文
kubectl config use-context ${KUBE_USER}@${CLUSTER_NAME} --kubeconfig=/etc/kubernetes/${KUBE_CONFIG}

# 查看生成的配置文件
kubectl config view --kubeconfig=/etc/kubernetes/${KUBE_CONFIG}
```

**kube-scheduler**

```bash
CLUSTER_NAME="kubernetes"
KUBE_USER="system:kube-scheduler"
KUBE_CERT="kube-scheduler"
KUBE_CONFIG="scheduler.kubeconfig"

# 设置集群参数
kubectl config set-cluster ${CLUSTER_NAME} \
  --certificate-authority=/etc/kubernetes/pki/ca.crt \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=/etc/kubernetes/${KUBE_CONFIG}

# 设置客户端认证参数
kubectl config set-credentials ${KUBE_USER} \
  --client-certificate=/etc/kubernetes/pki/${KUBE_CERT}.crt \
  --client-key=/etc/kubernetes/pki/${KUBE_CERT}.key \
  --embed-certs=true \
  --kubeconfig=/etc/kubernetes/${KUBE_CONFIG}

# 设置上下文参数
kubectl config set-context ${KUBE_USER}@${CLUSTER_NAME} \
  --cluster=${CLUSTER_NAME} \
  --user=${KUBE_USER} \
  --kubeconfig=/etc/kubernetes/${KUBE_CONFIG}

# 设置当前使用的上下文
kubectl config use-context ${KUBE_USER}@${CLUSTER_NAME} --kubeconfig=/etc/kubernetes/${KUBE_CONFIG}

# 查看生成的配置文件
kubectl config view --kubeconfig=/etc/kubernetes/${KUBE_CONFIG}
```

**admin(kubectl)**

```bash
CLUSTER_NAME="kubernetes"
KUBE_USER="kubernetes-admin"
KUBE_CERT="admin"
KUBE_CONFIG="admin.kubeconfig"

# 设置集群参数
kubectl config set-cluster ${CLUSTER_NAME} \
  --certificate-authority=/etc/kubernetes/pki/ca.crt \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=/etc/kubernetes/${KUBE_CONFIG}

# 设置客户端认证参数
kubectl config set-credentials ${KUBE_USER} \
  --client-certificate=/etc/kubernetes/pki/${KUBE_CERT}.crt \
  --client-key=/etc/kubernetes/pki/${KUBE_CERT}.key \
  --embed-certs=true \
  --kubeconfig=/etc/kubernetes/${KUBE_CONFIG}

# 设置上下文参数
kubectl config set-context ${KUBE_USER}@${CLUSTER_NAME} \
  --cluster=${CLUSTER_NAME} \
  --user=${KUBE_USER} \
  --kubeconfig=/etc/kubernetes/${KUBE_CONFIG}

# 设置当前使用的上下文
kubectl config use-context ${KUBE_USER}@${CLUSTER_NAME} --kubeconfig=/etc/kubernetes/${KUBE_CONFIG}

# 查看生成的配置文件
kubectl config view --kubeconfig=/etc/kubernetes/${KUBE_CONFIG}
```



分发到 kubeconfig 及证书其他 master 节点

```bash
for NODE in "${!otherMaster[@]}"; do
    echo "--- $NODE ${otherMaster[$NODE]} ---"
    scp -r /etc/kubernetes ${otherMaster[$NODE]}:/etc
done
```

### Authorize kubelet to create CSR

```bash
kubectl create clusterrolebinding kubeadm:kubelet-bootstrap \
        --clusterrole system:node-bootstrapper --group system:bootstrappers
        
# or
cat <<EOF | kubectl apply -f -
# enable bootstrapping nodes to create CSR
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubeadm:kubelet-bootstrap
subjects:
- kind: Group
  name: system:bootstrappers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: system:node-bootstrapper
  apiGroup: rbac.authorization.k8s.io
EOF
```

### Approval

In order to approve CSRs, you need to tell the controller-manager that it is acceptable to approve them. This is done by granting RBAC permissions to the correct group.

There are two distinct sets of permissions:

- `nodeclient`: If a node is creating a new certificate for a node, then it does not have a certificate yet. It is authenticating using one of the tokens listed above, and thus is part of the group `system:bootstrappers`.
- `selfnodeclient`: If a node is renewing its certificate, then it already has a certificate (by definition), which it uses continuously to authenticate as part of the group `system:nodes`.

To enable the kubelet to request and receive a new certificate, create a `ClusterRoleBinding` that binds the group in which the bootstrapping node is a member `system:bootstrappers` to the `ClusterRole` that grants it permission, `system:certificates.k8s.io:certificatesigningrequests:nodeclient`:

```yml
cat <<EOF | kubectl apply -f -
# Approve all CSRs for the group "system:bootstrappers"
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: auto-approve-csrs-for-group
subjects:
- kind: Group
  name: system:bootstrappers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: system:certificates.k8s.io:certificatesigningrequests:nodeclient
  apiGroup: rbac.authorization.k8s.io
  
```

To enable the kubelet to renew its own client certificate, create a `ClusterRoleBinding` that binds the group in which the fully functioning node is a member `system:nodes` to the `ClusterRole` that grants it permission, `system:certificates.k8s.io:certificatesigningrequests:selfnodeclient`:

```yml
cat <<EOF | kubectl apply -f -
# Approve renewal CSRs for the group "system:nodes"
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: auto-approve-renewals-for-nodes
subjects:
- kind: Group
  name: system:nodes
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: system:certificates.k8s.io:certificatesigningrequests:selfnodeclient
  apiGroup: rbac.authorization.k8s.io
```



### 参考

[张馆长-二进制部署kuberneters v1.13.12 HA可选](https://zhangguanzhang.github.io/2019/03/03/kubernetes-1-13-4/#建立集群CA-keys-与Certificates)

https://kubernetes.io/docs/setup/best-practices/certificates/

https://kubernetes.io/fr/docs/concepts/cluster-administration/certificates/