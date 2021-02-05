[toc]

#### 证书制作

```bash
wget http://10.21.8.20/k8s/etcd/ca-config.json
wget http://10.21.8.20/k8s/etcd/etcd-ca-csr.json
wget http://10.21.8.20/k8s/etcd/etcd-csr.json

export CFSSL_URL="https://pkg.cfssl.org/R1.2"
wget "${CFSSL_URL}/cfssl_linux-amd64" -O /usr/local/bin/cfssl
wget "${CFSSL_URL}/cfssljson_linux-amd64" -O /usr/local/bin/cfssljson
chmod +x /usr/local/bin/cfssl /usr/local/bin/cfssljson

cfssl gencert -initca etcd-ca-csr.json | cfssljson -bare etcd-ca 

cfssl gencert \
  -ca=etcd-ca.pem \
  -ca-key=etcd-ca-key.pem \
  -config=ca-config.json \
  -hostname=127.0.0.1,10.80.7.153,10.80.8.153,10.80.9.153 \
  -profile=kubernetes \
  etcd-csr.json | cfssljson -bare etcd

cfssl gencert \
  -ca=etcd-ca.pem \
  -ca-key=etcd-ca-key.pem \
  -config=ca-config.json \
  -hostname=127.0.0.1,10.80.7.153,10.80.8.153,10.80.9.153 \
  -profile=kubernetes \
  etcd-csr.json | cfssljson -bare etcd
  
rm -rf *.json *.csr
# ls
etcd-ca-key.pem  etcd-ca.pem  etcd-key.pem  etcd.pem

# etcd配置
  ca-file: '/etc/etcd/ssl/etcd-ca.pem'
  cert-file: '/etc/etcd/ssl/etcd.pem'
  key-file: '/etc/etcd/ssl/etcd-key.pem'
  
  
  
# cat ca-config.json
{
    "signing": {
        "default": {
            "expiry": "87600h"
        },
        "profiles": {
            "kubernetes": {
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ],
                "expiry": "87600h"
            }
        }
    }
}

# cat etcd-ca-csr.json
{
    "CN": "etcd",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "Beijing",
            "L": "Beijing",
            "O": "etcd",
            "OU": "Etcd Security"
        }
    ]
}

# cat etcd-csr.json
{
    "CN": "etcd",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "Beijing",
            "L": "Beijing",
            "O": "etcd",
            "OU": "Etcd Security"
        }
    ]
}
```

#### systemd  unit

##### environment

```bash
cat environment.sh
#!/usr/bin/bash

# 集群各机器 IP 数组
export NODE_IPS=(10.21.8.24 10.21.8.25 10.21.8.26)

# 集群各 IP 对应的 主机名数组
export NODE_NAMES=(yztest-ops-k8s24v8-yz yztest-ops-k8s25v8-yz yztest-ops-k8s26v8-yz)

# etcd 集群服务地址列表
export ETCD_ENDPOINTS="https://10.21.8.24:2379,https://10.21.8.25:2379,https://10.21.8.26:2379"

# etcd 集群间通信的 IP 和端口
export ETCD_NODES="yztest-ops-k8s24v8-yz=https://10.21.8.24:2380,yztest-ops-k8s25v8-yz=https://10.21.8.25:2380,yztest-ops-k8s26v8-yz=https://10.21.8.26:2380"

# kube-apiserver 的反向代理(kube-nginx)地址端口
export KUBE_APISERVER="https://127.0.0.1:8443"

# 节点间互联网络接口名称
export IFACE="eth0"

# etcd 数据目录
export ETCD_DATA_DIR="/var/lib/etcd"

# etcd WAL 目录，建议是 SSD 磁盘分区，或者和 ETCD_DATA_DIR 不同的磁盘分区
export ETCD_WAL_DIR="/var/lib/etcd/wal"
```

##### etcd.service.template

```bash
source environment.sh
cat > etcd.service.template <<EOF
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
Type=notify
WorkingDirectory=${ETCD_DATA_DIR}
ExecStart=/opt/k8s/bin/etcd \\   
  --data-dir=${ETCD_DATA_DIR} \\
  --wal-dir=${ETCD_WAL_DIR} \\
  --name=##NODE_NAME## \\
  --cert-file=/etc/etcd/ssl/etcd.pem \\
  --key-file=/etc/etcd/ssl/etcd-key.pem \\
  --trusted-ca-file=/etc/etcd/ssl/etcd-ca.pem \\
  --peer-cert-file=/etc/etcd/ssl/etcd.pem \\
  --peer-key-file=/etc/etcd/ssl/etcd-key.pem \\
  --peer-trusted-ca-file=/etc/etcd/ssl/etcd-ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --listen-peer-urls=https://##NODE_IP##:2380 \\
  --initial-advertise-peer-urls=https://##NODE_IP##:2380 \\
  --listen-client-urls=https://##NODE_IP##:2379,http://127.0.0.1:2379 \\
  --advertise-client-urls=https://##NODE_IP##:2379 \\
  --initial-cluster-token=etcd-cluster-0 \\
  --initial-cluster=${ETCD_NODES} \\
  --initial-cluster-state=new \\
  --auto-compaction-mode=periodic \\
  --auto-compaction-retention=1 \\
  --max-request-bytes=33554432 \\
  --quota-backend-bytes=6442450944 \\
  --heartbeat-interval=250 \\
  --election-timeout=2000
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

- `WorkingDirectory`、`--data-dir`：指定工作目录和数据目录为 `${ETCD_DATA_DIR}`，需在启动服务前创建这个目录；
- `--wal-dir`：指定 wal 目录，为了提高性能，一般使用 SSD 或者和 `--data-dir` 不同的磁盘；
- `--name`：指定节点名称，当 `--initial-cluster-state` 值为 `new` 时，`--name` 的参数值必须位于 `--initial-cluster` 列表中；
- `--cert-file`、`--key-file`：etcd server 与 client 通信时使用的证书和私钥；
- `--trusted-ca-file`：签名 client 证书的 CA 证书，用于验证 client 证书；
- `--peer-cert-file`、`--peer-key-file`：etcd 与 peer 通信使用的证书和私钥；
- `--peer-trusted-ca-file`：签名 peer 证书的 CA 证书，用于验证 peer 证书；

```bash
--name
etcd集群中的节点名，这里可以随意，可区分且不重复就行
--listen-peer-urls
监听的用于节点之间通信的url，可监听多个，集群内部将通过这些url进行数据交互(如选举，数据同步等)
--initial-advertise-peer-urls
建议用于节点之间通信的url，节点间将以该值进行通信。
--listen-client-urls
监听的用于客户端通信的url,同样可以监听多个。
--advertise-client-urls
建议使用的客户端通信url,该值用于etcd代理或etcd成员与etcd节点通信。
--initial-cluster-token etcd-cluster-1
节点的token值，设置该值后集群将生成唯一id,并为每个节点也生成唯一id,当使用相同配置文件再启动一个集群时，只要该token值不一样，etcd集群就不会相互影响。
--initial-cluster
也就是集群中所有的initial-advertise-peer-urls 的合集
--initial-cluster-state new
新建集群的标志，初始化状态使用 new，建立之后改此值为 existing
```



##### 生成三个systemd文件

```bash
for (( i=0; i < 3; i++ ))
  do
    sed -e "s/##NODE_NAME##/${NODE_NAMES[i]}/" -e "s/##NODE_IP##/${NODE_IPS[i]}/" etcd.service.template > etcd-${NODE_IPS[i]}.service 
  done
ls *.service
# 重命名为etcd.service 放入/etc/systemd/system/
systemctl daemon-reload && systemctl enable etcd && systemctl restart etcd
```

##### 启动

```bash
systemctl daemon-reload && systemctl enable etcd && systemctl restart etcd
```



#### 查看集群状态

```bash
ETCDCTL_API=3 etcdctl --endpoints "https://10.80.7.153:2379,https://10.80.8.153:2379,https://10.80.9.153:2379" --cacert  /etc/etcd/ssl/etcd-ca.pem --cert /etc/etcd/ssl/etcd.pem --key /etc/etcd/ssl/etcd-key.pem endpoint health


ETCDCTL_API=3 etcdctl --endpoints "https://10.80.7.153:2379,https://10.80.8.153:2379,https://10.80.9.153:2379" --cacert  /etc/etcd/ssl/etcd-ca.pem --cert /etc/etcd/ssl/etcd.pem --key /etc/etcd/ssl/etcd-key.pem endpoint status -w table


2379端口	用于与client端交互
2380端口	用于etcd内部交互
```

#### 参考

[部署 etcd 集群](https://github.com/opsnull/follow-me-install-kubernetes-cluster/blob/master/04.%E9%83%A8%E7%BD%B2etcd%E9%9B%86%E7%BE%A4.md)

[etcd管理指南](https://sealyun.com/post/etcd-manage/)

