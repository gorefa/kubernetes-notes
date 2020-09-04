[toc]

### etcdctl 常用命令

#### 查看版本

```bash
#v3 
 ETCDCTL_API=3 ./etcdctl --endpoints "https://10.21.8.24:2379,https://10.21.8.25:2379,https://10.21.8.26:2379" --cacert  /etc/etcd/ssl/etcd-ca.pem --cert /etc/etcd/ssl/etcd.pem --key /etc/etcd/ssl/etcd-key.pem version

#v2
./etcdctl --endpoints "https://10.21.8.24:2379,https://10.21.8.25:2379,https://10.21.8.26:2379" --ca-file=/etc/etcd/ssl/etcd-ca.pem --cert-file=/etc/etcd/ssl/etcd.pem --key-file=/etc/etcd/ssl/etcd-key.pem --version
```


#### etcd中成员管理

列出etcd集群中的节点

`etcdctl member list`

```bash
kubectl -n kube-system exec etcd-yztest-ops-k8s24v8-yz -- etcdctl --endpoints="https://10.21.8.24:2379,https://10.21.8.25:2379,https://10.21.8.26:2379" --ca-file=/etc/etcd/ssl/etcd-ca.pem --cert-file=/etc/etcd/ssl/etcd.pem --key-file=/etc/etcd/ssl/etcd-key.pem member list
2018-09-28 09:38:35.619705 I | warning: ignoring ServerName for user-provided CA for backwards compatibility is deprecated
5fea6424f8fd1e54: name=yztest-ops-k8s105v8-yz peerURLs=https://10.21.8.105:2380 clientURLs=https://10.21.8.105:2379 isLeader=false
698e3b06fb49add4: name=yztest-ops-k8s24v8-yz peerURLs=https://10.21.8.24:2380 clientURLs=https://10.21.8.24:2379 isLeader=false
7b3c7735d8a34350: name=yztest-ops-k8s25v8-yz peerURLs=https://10.21.8.25:2380 clientURLs=https://10.21.8.25:2379 isLeader=false
d3824ebb2dadafac: name=yztest-ops-k8s26v8-yz peerURLs=https://10.21.8.26:2380 clientURLs=https://10.21.8.26:2379 isLeader=true
```

```bash
--cert-file value        identify HTTPS client using this SSL certificate file
--key-file value         identify HTTPS client using this SSL key file
--ca-file value          verify certificates of HTTPS-enabled servers using this CA bundle
```

--cert-file 证书文件  /etc/etcd/ssl/etcd.pem  `CERTIFICATE`

--key-file 密钥文件  /etc/etcd/ssl/etcd-key.pem  `RSA PRIVATE KEY`

--ca-file  /etc/etcd/ssl/etcd-ca.pem `CERTIFICATE`

#### 查看etcd健康状态

`etcdctl cluster-health`

#### 查看etcd中的数据

```bash
ETCDCTL_API=3 ./etcdctl --endpoints "https://10.21.8.24:2379,https://10.21.8.25:2379,https://10.21.8.26:2379" --cacert  /etc/etcd/ssl/etcd-ca.pem --cert /etc/etcd/ssl/etcd.pem --key /etc/etcd/ssl/etcd-key.pem get / --prefix --keys-only

./etcdctl --endpoints "https://10.21.8.24:2379,https://10.21.8.25:2379,https://10.21.8.26:2379" --ca-file=/etc/etcd/ssl/etcd-ca.pem --cert-file=/etc/etcd/ssl/etcd.pem --key-file=/etc/etcd/ssl/etcd-key.pem get /


# k8s中查看,下面为kubeadm安装的etcd
kubectl  -n kube-system exec -it etcd-yztest-k8s-master-104v8-yz -- etcdctl --cert /etc/kubernetes/pki/etcd/peer.crt --key /etc/kubernetes/pki/etcd/peer.key --cacert /etc/kubernetes/pki/etcd/ca.crt --endpoints "https://10.21.8.104:2379,https://10.21.8.105:2379,https://10.21.8.106:2379" get / --prefix --keys-only 
```

#### 查看当前leader

```bash
ETCDCTL_API=3 etcdctl --endpoints "https://10.80.7.153:2379,https://10.80.8.153:2379,https://10.80.9.153:2379" --cacert  /etc/etcd/ssl/etcd-ca.pem --cert /etc/etcd/ssl/etcd.pem --key /etc/etcd/ssl/etcd-key.pem endpoint status -w table
```



#### 删除etcd中的数据

使用`kubectl delete pv prometheus-pv`不能删除pv

```bash
# kubectl get pv
NAME                CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS        CLAIM                                STORAGECLASS   REASON    AGE
prometheus-pv       10Gi       RWO            Recycle          Terminating   monitoring/prometheus-alertmanager   nfs                      48d
prometheus-pv-pro   8Gi        RWO            Recycle          Terminating   monitoring/prometheus-server         nfs                      48d
```

**尝试从etcd中直接删除数据。**

```bash
ETCDCTL_API=3 ./etcdctl --endpoints "https://10.21.8.24:2379,https://10.21.8.25:2379,https://10.21.8.26:2379" --cacert  /etc/etcd/ssl/etcd-ca.pem --cert /etc/etcd/ssl/etcd.pem --key /etc/etcd/ssl/etcd-key.pem del /registry/persistentvolumes/prometheus-pv-pro
```

#### snapshot save

```bash
ETCDCTL_API=3 ./etcdctl --endpoints "https://10.21.8.24:2379,https://10.21.8.25:2379,https://10.21.8.26:2379" --cacert  /etc/etcd/ssl/etcd-ca.pem --cert /etc/etcd/ssl/etcd.pem --key /etc/etcd/ssl/etcd-key.pem snapshot save /root/etcd.save
```

#### 压缩数据
```bash
#使用API3
export ETCDCTL_API=3
# 查看告警信息，告警信息一般 memberID:8630161756594109333 alarm:NOSPACE
etcdctl --endpoints=http://127.0.0.1:2379 alarm list

# 获取当前版本
rev=$(etcdctl --endpoints=http://127.0.0.1:2379 endpoint status --write-out="json" | egrep -o '"revision":[0-9]*' | egrep -o '[0-9].*')
# 压缩掉所有旧版本
etcdctl --endpoints=http://127.0.0.1:2379 compact $rev
# 整理多余的空间
etcdctl --endpoints=http://127.0.0.1:2379 defrag
# 取消告警信息
etcdctl --endpoints=http://127.0.0.1:2379 alarm disarm
```

### 参考

[k8s etcd数据管理](https://www.jianshu.com/p/f9f83dd21770)

[etcd无法自动加入etcd集群](https://github.com/k8sp/sextant/issues/333)

