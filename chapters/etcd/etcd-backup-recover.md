### etcd v3 备份集群

恢复集群首先需要来自 etcd 成员的键空间的快照。快速可以是用 `etcdctl snapshot save` 命令从活动成员获取，或者是从 etcd 数据目录复制 `member/snap/db` 文件. 例如，下列命令快照在 `$ENDPOINT` 上服务的键空间到文件 `snapshot.db`:

```bash
ETCDCTL_API=3 ./etcdctl --endpoints 'https://10.21.8.24:2379,https://10.21.8.25:2379,https://10.21.8.26:2379' --cacert /etc/etcd/ssl/etcd-ca.pem --cert /etc/etcd/ssl/etcd.pem --key /etc/etcd/ssl/etcd-key.pem snapshot save /snapshot/$(date +%Y%m%d_%H%M%S)_snapshot.db
```

#### cronjob 备份

```yaml
apiVersion: batch/v1beta1 
kind: CronJob
metadata:
  name: etcd-disaster-recovery
spec:
 schedule: "*/1 * * * *"
 jobTemplate:
  spec:
    template:
      metadata:
       labels:
        app: etcd-disaster-recovery
      spec:
        affinity:
          nodeAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
                  nodeSelectorTerms:
                  - matchExpressions:
                    - key: role
                      operator: In
                      values:
                      - master
        containers:
        - name: etcd
          image: mirrorgooglecontainers/etcd-amd64:3.1.13
          command:
          - sh
          - -c
          - "ETCDCTL_API=3 etcdctl --endpoints 'https://10.21.8.24:2379,https://10.21.8.25:2379,https://10.21.8.26:2379' --cacert /etc/etcd/ssl/etcd-ca.pem --cert /etc/etcd/ssl/etcd.pem --key /etc/etcd/ssl/etcd-key.pem snapshot save /snapshot/$(date +%Y%m%d_%H%M%S)_snapshot.db; \
             echo etcd backup sucess"
          volumeMounts:
            - mountPath: "/snapshot"
              name: snapshot
              subPath: data/etcd-snapshot
            - mountPath: /etc/localtime
              name: lt-config
            - mountPath: /etc/timezone
              name: tz-config
            - mountPath: /etc/etcd/ssl
              name: etcd-certs
        restartPolicy: OnFailure
        volumes:
          - name: snapshot
            persistentVolumeClaim:
              claimName: cron-nas
          - name: lt-config
            hostPath:
              path: /etc/localtime
          - name: tz-config
            hostPath:
              path: /etc/timezone
          - name: etcd-certs
            hostPath:
              path: /etc/etcd/ssl
        hostNetwork: true
```



### etcd v3 恢复集群

为了恢复集群，需要的只是一个简单的快照 "db" 文件。使用 `etcdctl snapshot restore` 的集群恢复创建新的 etcd 数据目录;所有成员应该使用相同的快照恢复。恢复覆盖某些快照元数据(特别是，成员ID和集群ID);成员丢失它之前的标识。这个元数据覆盖防止新的成员不经意间加入已有的集群。因此为了从快照启动集群，恢复必须启动一个新的逻辑集群。

在恢复时快照完整性的检验是可选的。如果快照是通过 `etcdctl snapshot save` 得到的，它将有一个被 `etcdctl snapshot restore` 检查过的完整性hash。如果快照是从数据目录复制而来，没有完整性hash，因此它只能通过使用 `--skip-hash-check` 来恢复。

恢复初始化新集群的新成员，带有新的集群配置，使用 `etcd` 的集群配置标记，但是保存 etcd 键空间的内容。继续上面的例子，下面为一个3成员的集群创建新的 etcd 数据目录(`m1.etcd`, `m2.etcd`, `m3.etcd`):

```bash
# yztestopshadoop63v8yz
ETCDCTL_API=3 ./etcdctl snapshot restore /data/20190228_091014_snapshot.db \
  --name yztestopshadoop63v8yz \
  --initial-cluster yztestopshadoop63v8yz=http://10.21.8.63:2380,yztestopshadoop64v8yz=http://10.21.8.64:2380,yztestopshadoop65v8yz=http://10.21.8.65:2380 \
  --initial-cluster-token etcd-cluster-1 \
  --initial-advertise-peer-urls http://10.21.8.63:2380 

# yztestopshadoop64v8yz
ETCDCTL_API=3 ./etcdctl snapshot restore /data/20190228_091014_snapshot.db \
  --name yztestopshadoop64v8yz \
  --initial-cluster yztestopshadoop63v8yz=http://10.21.8.63:2380,yztestopshadoop64v8yz=http://10.21.8.64:2380,yztestopshadoop65v8yz=http://10.21.8.65:2380 \
  --initial-cluster-token etcd-cluster-1 \
  --initial-advertise-peer-urls http://10.21.8.64:2380 

# yztestopshadoop65v8yz
ETCDCTL_API=3 ./etcdctl snapshot restore /data/20190228_091014_snapshot.db \
  --name yztestopshadoop65v8yz \
  --initial-cluster yztestopshadoop63v8yz=http://10.21.8.63:2380,yztestopshadoop64v8yz=http://10.21.8.64:2380,yztestopshadoop65v8yz=http://10.21.8.65:2380 \
  --initial-cluster-token etcd-cluster-1 \
  --initial-advertise-peer-urls http://10.21.8.65:2380 
```

**数据目录在/usr/local/etcd-v3.1.13-linux-amd64/yztestopshadoop65v8yz.etcd**

下一步, 用新的数据目录启动 `etcd` :

```bash
./etcd \
  --name yztestopshadoop63v8yz \
  --listen-client-urls http://10.21.8.63:2379 \
  --advertise-client-urls http://10.21.8.63:2379 \
  --listen-peer-urls http://10.21.8.63:2380 &

./etcd \
  --name yztestopshadoop64v8yz \
  --listen-client-urls http://10.21.8.64:2379 \
  --advertise-client-urls http://10.21.8.64:2379 \
  --listen-peer-urls http://10.21.8.64:2380 &

./etcd \
  --name yztestopshadoop65v8yz \
  --listen-client-urls http://10.21.8.65:2379 \
  --advertise-client-urls http://10.21.8.65:2379 \
  --listen-peer-urls http://10.21.8.65:2380 &
```

### 查看恢复集群状态

```bash
# ETCDCTL_API=3 ./etcdctl --endpoints "http://10.21.8.63:2379,http://10.21.8.64:2379,http://10.21.8.65:2379"  member list 
1332d5f408b3aafe, started, yztestopshadoop63v8yz, http://10.21.8.63:2380, http://10.21.8.63:2379
73fd04b3f0619f14, started, yztestopshadoop65v8yz, http://10.21.8.65:2380, http://10.21.8.65:2379
7f9139c640b1b323, started, yztestopshadoop64v8yz, http://10.21.8.64:2380, http://10.21.8.64:2379



# ETCDCTL_API=3 ./etcdctl --endpoints "http://10.21.8.63:2379,http://10.21.8.64:2379,http://10.21.8.65:2379"   endpoint status --write-out=table
+------------------------+------------------+---------+---------+-----------+-----------+------------+
|        ENDPOINT        |        ID        | VERSION | DB SIZE | IS LEADER | RAFT TERM | RAFT INDEX |
+------------------------+------------------+---------+---------+-----------+-----------+------------+
| http://10.21.8.63:2379 | 1332d5f408b3aafe | 3.1.13  | 60 MB   | true      |         9 |         13 |
| http://10.21.8.64:2379 | 7f9139c640b1b323 | 3.1.13  | 60 MB   | false     |         9 |         13 |
| http://10.21.8.65:2379 | 73fd04b3f0619f14 | 3.1.13  | 60 MB   | false     |         9 |         13 |
+------------------------+------------------+---------+---------+-----------+-----------+------------+



# ETCDCTL_API=3 ./etcdctl --endpoints "http://10.21.8.63:2379,http://10.21.8.64:2379,http://10.21.8.65:2379"   get / --prefix --keys-only
/calico/ipam/v2/assignment/ipv4/block/10.244.1.128-26

/calico/ipam/v2/assignment/ipv4/block/10.244.1.192-26

/calico/ipam/v2/assignment/ipv4/block/10.244.163.128-26

/calico/ipam/v2/assignment/ipv4/block/10.244.191.192-26

/calico/ipam/v2/assignment/ipv4/block/10.244.237.192-26
```



**TODO**   使用证书

### 参考

[etcd 导入导出备份](https://www.xgame.io/?p=106)

[etcd集群备份和数据恢复以及优化运维](https://www.maideliang.com/index.php/archives/25/)

[官方文档](https://github.com/etcd-io/etcd/blob/master/Documentation/op-guide/recovery.md)

[etcd管理指南](https://sealyun.com/post/etcd-manage/)

[Etcd集群备份及容灾恢复](https://xigang.github.io/2018/08/18/etcd-back/)

[Etcd v3备份与恢复](https://mp.weixin.qq.com/s?__biz=MzA4MzIwNTc4NQ==&mid=2247484108&idx=1&sn=9852f8a55ea1f494a2c3caeb7a6655b2&chksm=9ffb493aa88cc02c797611cf5a33246f7705287ac47dbd280a637e13c9e197310755d3adc44b&mpshare=1&scene=23&srcid=&sharer_sharetime=1578439555668&sharer_shareid=85ecb9201cba173f8c63eaf47b0c5559#rd)

