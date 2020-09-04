[TOC]

> 主要参考： [使用rbd-provisioner提供rbd持久化存储](https://jimmysong.io/kubernetes-handbook/practice/rbd-provisioner.html)



### 前提

要有一个k8s集群，一个ceph集群。

在每个k8s node中安装ceph-common。

```bash
# 增加ceph源
cat > /etc/yum.repos.d/ceph.repo << EOF
[Ceph]
name=Ceph packages for $basearch
baseurl=http://mirrors.163.com/ceph/rpm-mimic/el7/x86_64
enabled=1
gpgcheck=0
type=rpm-md
gpgkey=https://mirrors.163.com/ceph/keys/release.asc
priority=1
[Ceph-noarch]
name=Ceph noarch packages
baseurl=http://mirrors.163.com/ceph/rpm-mimic/el7/noarch
enabled=1
gpgcheck=0
type=rpm-md
gpgkey=https://mirrors.163.com/ceph/keys/release.asc
priority=1
[ceph-source]
name=Ceph source packages
baseurl=http://mirrors.163.com/ceph/rpm-mimic/el7/SRPMS
enabled=1
gpgcheck=0
type=rpm-md
gpgkey=https://mirrors.163.com/ceph/keys/release.asc
priority=1
EOF

# install ceph-common
yum install -y ceph-common
```

分发ceph pool 认证文件ceph.client.poolname.keyring复制到k8s node的/etc/ceph/ 目录下。 此步骤在下面。

### 没有storageclass情况

首先在ceph 中创建一个2G 的image测试ceph是否ok

```bash
# rbd -p kubernetes create tmp-image -s 2G --image-feature  layering
# rbd info tmp-image -p kubernetes                                    
rbd image 'tmp-image':
        size 2 GiB in 512 objects
        order 22 (4 MiB objects)
        id: 5b5a86b8b4567
        block_name_prefix: rbd_data.5b5a86b8b4567
        format: 2
        features: layering
        op_features: 
        flags: 
        create_timestamp: Thu Jul 30 09:08:52 2020
```

注：这里有个ceph的坑，在jewel版本下默认format是2，开启了rbd的一些属性，而这些属性有的内核版本是不支持的，会导致map不到device的情况，可以在创建时指定feature（我们就是这样做的）,也可以在ceph配置文件中关闭这些新属性：rbd_default_features = 2。参考[rbd无法map(rbd feature disable)](http://www.zphj1987.com/2016/06/07/rbd%E6%97%A0%E6%B3%95map-rbd-feature-disable/)。

例如：`rbd feature disable pv001-image exclusive-lock, object-map, fast-diff, deep-flatten --pool kube`

创建PV,需要指定ceph mon节点地址，以及对应的pool，image等：

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: test-pv
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce 
  rbd:
    monitors:
      - 10.21.8.57:6789
      - 10.21.8.59:6789
      - 10.21.8.60:6789
    pool: kubernetes
    image: test-image   # 任意名字
    user: admin
    secretRef:
      name: ceph-secret
  persistentVolumeReclaimPolicy: Recycle
```

创建pvc

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
```

查看绑定关系，如果状态是bound那么两者绑定成功。

```bash
# kubectl get pvc
NAME        STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
test-pvc    Bound    test-pv    2Gi        RWO                           2m48s
```

创建一个pod验证：

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-dm
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:alpine
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
          volumeMounts:
            - name: ceph-rbd-volume
              mountPath: "/usr/share/nginx/html"
      volumes:
      - name: ceph-rbd-volume
        persistentVolumeClaim:
          claimName: test-pvc
```

进入该容器，利用dh -h 命令验证是否挂载成功。



没有storageclass的时候，需要手动创建pv pvc,指定名字 大小，当有大量容器要改在存储的时候，会非常麻烦。

下面使用storageclass. 直接声明pvc，会自动创建并绑定pv.

### 部署rbd-provisioner

kube-controller-manager以容器的方式运行。这种方式下，kubernetes在创建使用ceph rbd pv/pvc时没任何问题，但使用dynamic provisioning自动管理存储生命周期时会报错。提示`"rbd: create volume failed, err: failed to create rbd image: executable file not found in $PATH:"`。

```bash
git clone https://github.com/kubernetes-incubator/external-storage.git
cd external-storage/ceph/rbd/deploy
NAMESPACE=kube-system
sed -r -i "s/namespace: [^ ]+/namespace: $NAMESPACE/g" ./rbac/clusterrolebinding.yaml ./rbac/rolebinding.yaml
kubectl -n $NAMESPACE apply -f ./rbac
```

### storageclass

部署完rbd-provisioner，还需要创建StorageClass。创建SC前，我们还需要创建相关用户的secret；

#### secret

**kube-system-ceph-secret.yaml**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ceph-admin-secret
  namespace: kube-system
type: "kubernetes.io/rbd"  
data:
  # ceph auth get-key client.admin | base64
  key: QVFBK0YwaGNSZ3B1R3hBQUkxbEtnZlZ2QmVIT3RaTzRZY2kxR1E9PQ==

--- 
apiVersion: v1
kind: Secret
metadata:
  name: ceph-secret
  namespace: kube-system
type: "kubernetes.io/rbd"
data:
  # ceph auth add client.kubernetes mon 'allow r' osd 'allow rwx pool=kubernetes'
  # ceph auth get-key client.kubernetes | base64
  key: QVFCYUtraGNxSjdaRXhBQTlCR1h4NkdxUUxjRU9uZlNoYjZadXc9PQ==
```

**ceps-secret.yaml**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ceph-secret
  namespace: ceph
type: "kubernetes.io/rbd"
data:
 # ceph auth add client.kubernetes mon 'allow r' osd 'allow rwx pool=kubernetes'
  # ceph auth get-key client.kuberbetes | base64
  key: QVFBeUhFQmNRNFpDRWhBQUcrNlZOTGp4NTE4bitjSGZlRGV2V2c9PQ==
```

创建secret保存`client.admin`和`client.kubernetes`用户的key，`client.admin`和`client.kubernetes`用户的secret可以放在kube-system namespace，但如果其他namespace需要使用ceph rbd的dynamic provisioning功能的话，要在相应的namespace创建secret来保存client.kubernetes用户key信息。**这里在ceph ns里创建了。**

（PS是否需要在ns创建还需测试）

#### StorageClass yaml

**storage.yaml **

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ceph-rbd
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ceph.com/rbd
parameters:
    monitors: 10.21.8.57:6789,10.21.8.59:6789,10.21.8.60:6789
    adminId: admin
    adminSecretName: ceph-admin-secret
    adminSecretNamespace: kube-system
    pool: kubernetes
    userId: kubernetes
    userSecretName: ceph-secret
    fsType: ext4
    imageFormat: "2"
    imageFeatures: "layering"
```

- 其他设置和普通的ceph rbd StorageClass一致，但provisioner需要设置为`ceph.com/rbd`，不是默认的`kubernetes.io/rbd`，这样rbd的请求将由rbd-provisioner来处理；
- 考虑到兼容性，建议尽量关闭rbd image feature，并且kubelet节点的`ceph-common`版本尽量和ceph服务器端保持一致，我的环境都使用的M版本；
- 执行完查看

```bash
# kubectl get storageclasses.storage.k8s.io 
NAME                  PROVISIONER    AGE
ceph-rbd (default)    ceph.com/rbd   3h
managed-nfs-storage   test.cn/nfs    78d
```

#### k8s node分发配置文件

**需要将ceph 的keyring文件分发到k8s所有node,这里是/etc/ceph/ceph.client.kubernetes.keyring**

权限644

```shell
# ceph auth list  查看kubernetes pool的信息
mkdir -p /etc/ceph
cat << EOF > /etc/ceph/ceph.client.kubernetes.keyring
client.kubernetes
        key: AQBaKkhcqJ7ZExAA9BGXx6GqQLcEOnfShb6Zuw==
        caps: [mon] allow r
        caps: [osd] allow rwx pool=kubernetes
EOF
```

### 测试ceph rbd自动分配

**test-pod.yaml**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ceph-pod1
spec:
  containers:
  - name: ceph-busybox
    image: busybox
    command: ["sleep", "60000"]
    volumeMounts:
    - name: ceph-vol1
      mountPath: /usr/share/busybox
      readOnly: false
  volumes:
  - name: ceph-vol1
    persistentVolumeClaim:
      claimName: ceph-claim
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: ceph-claim
spec:
  storageClassName: ceph-rbd  # 声明storageclass
  accessModes:  
    - ReadWriteOnce  #可读可写，但只支持被单个Pod挂载。
  resources:
    requests:
      storage: 2Gi
```

**rbd-pvc.yaml**

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: rbd-pvc1
spec:
  storageClassName: ceph-rbd
  accessModes:
  - ReadWriteOnce  #可读可写，但只支持被单个Pod挂载。
  resources:
    requests:
      storage: 1Gi
```

**测试结果**

```bash
[root@yztest-ops-k8s24v8-yz test-ceph]# kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                                STORAGECLASS   REASON    AGE
altermanager-pv                            10Gi       RWO            Recycle          Bound       monitoring/prometheus-alertmanager   nfs                      12d
etcd-pv                                    8Gi        RWO            Recycle          Bound       default/cron-nas                     nfs                      103d
naftis-pv                                  10Gi       RWO            Retain           Released    naftis/naftis-mysql                  manual                   87d
prometheus-pv-pro                          8Gi        RWO            Recycle          Bound       monitoring/prometheus-server         nfs                      12d
pvc-12faa893-1e0e-11e9-a900-005056bd0b19   2Gi        RWO            Delete           Bound       ceph/ceph-claim                      ceph-rbd                 3h
pvc-1d9d66c9-1e25-11e9-b6b8-005056bd7916   1Gi        RWO            Delete           Bound       ceph/rbd-pvc1                        ceph-rbd                 25m
[root@yztest-ops-k8s24v8-yz test-ceph]# kubectl get pvc
NAME         STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
ceph-claim   Bound     pvc-12faa893-1e0e-11e9-a900-005056bd0b19   2Gi        RWO            ceph-rbd       3h
rbd-pvc1     Bound     pvc-1d9d66c9-1e25-11e9-b6b8-005056bd7916   1Gi        RWO            ceph-rbd       25m
[root@yztest-ops-k8s24v8-yz test-ceph]# rbd -p kubernetes list
kubernetes-dynamic-pvc-17703950-1e0e-11e9-a8ab-b65250c94f1a
kubernetes-dynamic-pvc-1da1909f-1e25-11e9-a8ab-b65250c94f1a
```

**请注意：删除测试 rbd 设备会被删除**

可以在StorageClass中设置删除策略`reclaimPolicy: Retain`

### 遇到的问题

#### map failed exit status 5, rbd output: rbd: sysfs write failed

```shell
 MountVolume.WaitForAttach failed for volume "pvc-12faa893-1e0e-11e9-a900-005056bd0b19" : rbd: map failed exit status 5, rbd output: rbd: sysfs write failed
In some cases useful info is found in syslog - try "dmesg | tail".
```

**dmesg会看到**

```bash
libceph: mon0 10.103.2.24:6789 feature set mismatch, my 106b84a842a42 < server's 40106b84a842a42, missing 400000000000000
libceph: mon0 10.103.2.24:6789 missing required protocol features
```

**目前解决办法**(不确定生产环境执行一下命令会遇到什么问题，慎重)

```bash
$ ceph osd crush tunables legacy
$ ceph osd crush reweight-all
```

>> 其實也可以將 kernel 升級到 4.5 以上解決此問題

使用这种方式ceph会出现报错**crush map has straw_calc_version=0**

```bash
ceph -s 
  cluster:
    id:     e9ffcfbe-976d-450f-91df-7072fc03367a
    health: HEALTH_WARN
            crush map has straw_calc_version=0
 
  services:
    mon: 3 daemons, quorum yztestopshadoop60v8yz,yztestopshadoop61v8yz,yztestopshadoop65v8yz
    mgr: yztestopshadoop60v8yz(active), standbys: yztestopshadoop65v8yz, yztestopshadoop61v8yz
    mds: cephfs-1/1/1 up  {0=yztestopshadoop65v8yz=up:active}
    osd: 5 osds: 5 up, 5 in
    rgw: 2 daemons active
 
  data:
    pools:   12 pools, 400 pgs
    objects: 2.86 k objects, 7.7 GiB
    usage:   28 GiB used, 172 GiB / 200 GiB avail
    pgs:     400 active+clean
```

可通过`ceph osd crush tunables optimal`解决。

#### missing required protocol features

```bash
# 报错信息
[7802779.239433] libceph: mon2 10.21.8.65:6789 feature set mismatch, my 102b84a842a42 < server's 40102b84a842a42, missing 400000000000000
[7802779.239891] libceph: mon2 10.21.8.65:6789 missing required protocol features

# 解决办法
[cephd ~]$ ceph osd crush tunables legacy
adjusted tunables profile to legacy
[cephd ~]$ ceph osd crush reweight-all
reweighted crush hierarchy
```



### 参考

[使用rbd-provisioner提供rbd持久化存储](https://jimmysong.io/kubernetes-handbook/practice/rbd-provisioner.html)

[Kubernetes設定 StorageClass 以 Ceph RBD 為例](https://godleon.github.io/blog/Kubernetes/k8s-Config-StorageClass-with-Ceph-RBD/)

[Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)

[Ceph - Crush Map has Legacy Tunables](http://indrapr.blogspot.com/2016/06/ceph-crush-map-has-legacy-tunables.html)

[使用Ceph RBD为Kubernetes集群提供存储卷](https://tonybai.com/2016/11/07/integrate-kubernetes-with-ceph-rbd/)

[kubernetes 使用ceph rbd作为持久存储](https://zhangchenchen.github.io/2017/11/17/kubernetes-integrate-with-ceph/)