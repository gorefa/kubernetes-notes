[TOC]

### k8s dns

DNS会监控Kubernetes的新的Service并且会为其创建一组DNS记录。如果DNS在整个集群中都被启用的话，那所有的Pod应该能自动对Service进行命名解析。

例如，如果你有一个叫做my-service的Service，其位于"my-ns"的Namespace下，那么对于"my-service.my-ns"就会有一个DNS的记录被创建。位于名为"my-ns" Namespace下的Pod可以通过简单地使用"my-service"进行查找。而位于其他Namespaces的Pod必须把查找名称指定为"my-service.my-ns"。这些命名查找的结果是一个集群的IP地址。

对于命名的端口，Kubernetes也支持DNS SRV (service)记录。如果名为"my-service.my-ns"的Service 有一个协议为TCP的名叫"http"的端口，你可以对"_http._tcp.my-service.my-ns"做一次DNS SRV查询来发现”http”的端口号。

例子：

 ```bash
# 使用nsenter  进入某个pod

# nslookup 
> set type=srv
> server 10.96.0.10
Default server: 10.96.0.10
Address: 10.96.0.10#53
> http.tcp.jenkins.default.svc.cluster.local
Server:         10.96.0.10
Address:        10.96.0.10#53

http.tcp.jenkins.default.svc.cluster.local      service = 0 100 80 jenkins.default.svc.cluster.local.
> https.tcp.jenkins.default.svc.cluster.local
Server:         10.96.0.10
Address:        10.96.0.10#53

https.tcp.jenkins.default.svc.cluster.local     service = 0 100 443 jenkins.default.svc.cluster.local.
> exit
 ```

![image-20200921102200744](/Users/lisai/go/src/llussy.github.io/images/image-20200921102200744.png)

### coredns配置文件介绍

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health
        kubernetes cluster.local in-addr.arpa ip6.arpa {
          pods insecure
          upstream 10.80.40.250 10.80.26.211 10.90.1.250 10.90.1.245 114.114.114.114
          fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        proxy . 10.80.40.250 10.80.26.211 10.90.1.250 10.90.1.245
        cache 60
        reload
        loadbalance
    }
    staff.test.com {
        forward . 172.30.226.253 172.30.226.252
        log
    }
    swoole.nine.test.com {
      forward . 10.80.40.250 10.80.26.211 10.90.1.250 10.90.1.245
      log
    }
```

配置文件各项目的含义
| 名称       | 含义                                                         |
| ---------- | ------------------------------------------------------------ |
| errors     | 错误会被记录到标准输出                                       |
| health     | 可以通过http://localhost:8080/health查看健康状况             |
| kubernetes | 根据服务的IP响应DNS查询请求，kubeadm的Cluster Domain和Service CIDR默认为cluster.local和10.95.0.0/12，可以通过--service-dns-domain和--service-cidr参数配置。 |
| prometheus | 可以通过http://localhost:9153/metrics获取prometheus格式的监控数据 |
| proxy      | 本地无法解析后，向上级地址进行查询，默认使用宿主机的 /etc/resolv.conf 配置 |
| cache      | 缓存时间                                                     |

### yaml

```yaml
# cat coredns.ds.yml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: coredns
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:coredns
rules:
- apiGroups:
  - ""
  resources:
  - endpoints
  - services
  - pods
  - namespaces
  verbs:
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:coredns
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:coredns
subjects:
- kind: ServiceAccount
  name: coredns
  namespace: kube-system
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health
        kubernetes cluster.local in-addr.arpa ip6.arpa {
          pods insecure
          upstream 192.168.1.1 192.168.1.2
          fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        proxy . 192.168.1.1 192.168.1.2
        cache 180
        reload
        loadbalance
    }
    staff.test.com {
        forward . 192.168.1.1 192.168.1.2
        log
    }
---
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: coredns
  namespace: kube-system
  labels:
    k8s-app: kube-dns
    kubernetes.io/name: "CoreDNS"
spec:
  updateStrategy:
    type: RollingUpdate
  selector:
    matchLabels:
      k8s-app: kube-dns
  template:
    metadata:
      labels:
        k8s-app: kube-dns
    spec:
      nodeSelector:
        traefik: "svc"
        #coredns: "dns"
      serviceAccountName: coredns
      tolerations:
      - key: "CriticalAddonsOnly"
        operator: "Exists"
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: coredns
        image: registry.cn-hangzhou.aliyuncs.com/criss/coredns:1.2.0
        imagePullPolicy: IfNotPresent
        args: [ "-conf", "/etc/coredns/Corefile" ]
        volumeMounts:
        - name: config-volume
          mountPath: /etc/coredns
          readOnly: true
        - mountPath: /etc/localtime
          name: localtime
          readOnly: true
        ports:
        - containerPort: 53
          name: dns
          protocol: UDP
        - containerPort: 53
          name: dns-tcp
          protocol: TCP
        - containerPort: 9153
          name: metrics
          protocol: TCP
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            add:
            - NET_BIND_SERVICE
            drop:
            - all
          readOnlyRootFilesystem: true
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 40
          timeoutSeconds: 3
          successThreshold: 1
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 40
          timeoutSeconds: 3
          successThreshold: 1
          failureThreshold: 3
      dnsPolicy: Default
      terminationGracePeriodSeconds: 90
      volumes:
      - name: config-volume
        configMap:
          name: coredns
          items:
          - key: Corefile
            path: Corefile
      - hostPath:
          path: /etc/localtime
          type: ""
        name: localtime
---
apiVersion: v1
kind: Service
metadata:
  name: kube-dns
  namespace: kube-system
  annotations:
    prometheus.io/port: "9153"
    prometheus.io/scrape: "true"
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    kubernetes.io/name: "CoreDNS"
spec:
  selector:
    k8s-app: kube-dns
  clusterIP: 10.96.0.10
  ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
    protocol: TCP
```

### 参考

[CoreDNS](https://www.cnblogs.com/cocowool/p/kubernetes_coredns.html)

[详解 DNS 与 CoreDNS 的实现原理](https://draveness.me/dns-coredns)