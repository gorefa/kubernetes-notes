
[toc]

日志流： **filebeat --> kafka--> logstash-->es**

k8s应用日志输出到标准输出，最好是json格式日志。

### filebeat

**关键配置**

1. **output.kafka** `topic: 'k8s-%{[kubernetes.namespace]}'`  按照ns打入topic
2. filebeat 传输meta信息落盘。 **pod** `/usr/share/filebeat/data` 目录 挂载到宿主机`/data/filebeat`目录

#### yaml
```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeat-config
  namespace: kube-system
  labels:
    k8s-app: filebeat
data:
  filebeat.yml: |-
    filebeat.config:
      inputs:
        path: ${path.config}/inputs.d/*.yml
        reload.enabled: false
      modules:
        path: ${path.config}/modules.d/*.yml
        reload.enabled: false

      filebeat.prospectors:
      - input_type: log
        scan_frequency: 10s
        harvester_buffer_size: 32768
        max_bytes: 10485760
        #enabled: true
        encoding: utf-8
        #tail_files: true
        #ignore_older: 7d
        fields_under_root: true
        json.keys_under_root: true
        json.add_error_key: true
        json.message_key: log
      - decode_json_fields:
        fields: ["log"]
        process_array: false
        max_depth: 3
        target: ""
        overwrite_keys: false
        paths:
        - /var/lib/docker/containers/*/*-json.log
    output.kafka:
      enabled: true
      hosts: ["kafka01:9092","kafka02:9092","kafka03:9092","kafka04:9092"]
      topic: 'kubernetes-%{[kubernetes.namespace]}'
      worker: 4
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeat-inputs
  namespace: kube-system
  labels:
    k8s-app: filebeat
data:
  kubernetes.yml: |-
    - type: docker
      containers.ids:
      - "*"
      processors:
        - add_kubernetes_metadata:
            in_cluster: true
---
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: filebeat
  namespace: kube-system
  labels:
    k8s-app: filebeat
spec:
  template:
    metadata:
      labels:
        k8s-app: filebeat
    spec:
      serviceAccountName: filebeat
      terminationGracePeriodSeconds: 30
      containers:
      - name: filebeat
        image: elastic/filebeat:6.3.1
        args: [
          "-c", "/etc/filebeat.yml",
          "-e",
        ]
        securityContext:
          runAsUser: 0
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 100Mi
        volumeMounts:
        - name: config
          mountPath: /etc/filebeat.yml
          readOnly: true
          subPath: filebeat.yml
        - name: inputs
          mountPath: /usr/share/filebeat/inputs.d
          readOnly: true
        - name: data
          mountPath: /usr/share/filebeat/data
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - mountPath: /etc/localtime
          name: localtime
          readOnly: true
      volumes:
      - name: config
        configMap:
          defaultMode: 0600
          name: filebeat-config
      - name: varlibdockercontainers
        hostPath:
          path: /data/docker/containers
      - name: inputs
        configMap:
          defaultMode: 0600
          name: filebeat-inputs
      - hostPath:
          path: /etc/localtime
          type: ""
        name: localtime
      # We set an `emptyDir` here to ensure the manifest will deploy correctly.
      # It's recommended to change this to a `hostPath` folder, to ensure internal data
      # files survive pod changes (ie: version upgrade)
      #- name: data
      #  emptyDir: {}
      - name: data
        hostPath:
          path: /data/site
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: filebeat
subjects:
- kind: ServiceAccount
  name: filebeat
  namespace: kube-system
roleRef:
  kind: ClusterRole
  name: filebeat
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: filebeat
  labels:
    k8s-app: filebeat
rules:
- apiGroups: [""] # "" indicates the core API group
  resources:
  - namespaces
  - pods
  verbs:
  - get
  - watch
  - list
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: filebeat
  namespace: kube-system
  labels:
    k8s-app: filebeat

```
### kafka

正常kafka集群即可，只做日志的话，可以将副本数设置为1。

### logstash

消费kafka数据，打入不同es的index （按namespace）

#### 配置文件
```bash
input {
  stdin {
    codec=>json {
        charset => "UTF-8"
        }
  }
  kafka {
        bootstrap_servers  => ["kafka01:9092,kafka02:9092,kafka03:9092,kafka04:9092"]
        topics_pattern => "kubernetes-(?!kube-system|log).*"
        group_id => "kubernetes-consumer"
        auto_offset_reset => "latest"
        decorate_events => true
        consumer_threads => 4
  }
}
filter {
    json {
        source => "message"
        remove_field => [ "_id","input_type","@version","prospector","beat","[input][type]","[host][name]","[kubernetes][container][name]","[kubernetes][labels][pod-template-hash]","fields","offset","sort","[_type]","[source]","[kubernetes][labels][controller-revision-hash]","[kubernetes][labels][pod-template-generation]"]
    }
    json {
        source => "message"
    }


    date {
        match => ["timestamp","ISO8601"]
        timezone => "Asia/Shanghai"
        target => "@timestamp"
    }
}

output {
  elasticsearch {
    hosts => ["es:9200"]
    index => "kubernetes-%{[kubernetes_namespace]}-%{+YYYY.MM.dd}"
    template_overwrite => true
    user => "elastic"
    password => "y16a5zzjKIM"
    }
}

```
**output关键配置**

**index => "kubernetes-%{[kubernetes][namespace]}-%{+YYYY.MM.dd}"**

### es

**动态调试索引模板。**
```bash
curl -u elastic:y16a5zzjKIM -XPUT 'http://es:9200/_template/log-k8s-template?pretty' -H 'Content-Type: application/json' -d'
{
    "template" : "log-k8s*",
    "order": 100,
    "settings" : {
        "index.number_of_shards" : 15,
        "number_of_replicas" : 1,
        "routing.allocation.total_shards_per_node": 3,
        "index.routing.allocation.exclude._ip" : "192.168.*.1,192.168.5.2",
        "refresh_interval" : "120s"
    }
}'
```

### 相关资料

[k8s日志收集实战](https://juejin.im/post/5b6eaef96fb9a04fa25a0d37)

[ELK实时日志分析平台环境部署--完整记录](https://www.cnblogs.com/kevingrace/p/5919021.html)

[ctsdb对接ELK生态组件及grafana](https://cloud.tencent.com/developer/article/1146988)