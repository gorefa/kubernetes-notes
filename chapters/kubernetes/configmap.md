[TOC]

### 目的

把应用的代码和配置分开，通过配置configmap管理pod，一种统一的集群配置管理方案。
ConfigMap API资源提供了将配置数据注入容器的方式，同时保持容器是不知道Kubernetes的。ConfigMap可以被用来保存单个属性，也可以用来保存整个配置文件或者JSON二进制等对象。

### 基本原理

ConfigMap是存储通用的配置变量的。ConfigMap有点儿像一个统一的配置文件，使用户可以将分布式系统中用于不同模块的环境变量统一到一个对象中管理；而它与配置文件的区别在于它是存在集群的“环境”中的，并且支持K8s集群中所有通用的操作调用方式。

而资源的使用者可以通过ConfigMap来存储这个资源的配置，这样需要访问这个资源的应用就可以同通过ConfigMap来引用这个资源。相当通过创建Configmap封装资源配置。

configmap以**一个或者多个key:value**的形式保存在k8s系统中供应用使用，既可以用于表示一个变量的值（eg.apploglevel:info)，也可以用于表示一个完整配置文件的内容（eg： server.xml=<?xml...>...)
可以通过yaml配置文件或者直接用`kubectl create configmap `命令行的方式来创建 ConfigMap。

### ConfigMap 配置

#### kubectl 命令行方式创建

(1) 从目录中创建
 该目录中已经存在一些配置文件，而且目录中所有的文件都将作为configMap中的数据，一个文件为一个data。
 key的名字为文件的名字，value为文件的内容。
 `kubectl create configmap game-config --from-file=/opt/configmap/file/`

```shell
[root@master file]# ls
game.properties  ui.properties
[root@master file]# kubectl  create configmap game-config --from-file=/opt/configmap/file/
[root@master file]# kubectl  describe configmap game-config
Name:       game-config
Namespace:  default
Labels:     <none>
Annotations:    <none>

Data
====
game.properties:    158 bytes
ui.properties:      83 bytes
```

通过`kubectl get configmap game-config -o yaml` 可以看到value的值。
 (2) 从文件中创建

```shell
[root@master file]# kubectl create configmap game-config2 --from-file=/opt/configmap/file/game.properties --from-file=/opt/configmap/file/ui.properties
[root@master file]# kubectl  describe configmap game-config2
Name:       game-config2
Namespace:  default
Labels:     <none>
Annotations:    <none>

Data
====
game.properties:    158 bytes
ui.properties:      83 bytes
```

#### yaml文件方式创建

例子1

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: special
  namespace: default
data:
  special.how: very
  special.type: charm
```

例子2

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: example-config
data:
  example.property.1: hello
  example.property.2: world
  example.property.file: |-
    property.1=value-1
    property.2=value-2
    property.3=value-3
[root@master yaml]# kubectl  describe configmap example-config
Name:       example-config
Namespace:  default
Labels:     <none>
Annotations:    <none>

Data
====
example.property.1: 5 bytes
example.property.2: 5 bytes
example.property.file:  56 bytes
```

例子3

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cm-serverxml
data:
  key-serverxml: |
    env: prod
    test: true
```

### ConfigMap的使用

ConfigMap供容器使用的典型用法如下：

1.生成为容器内的环境变量。
 2.设置容器启动命令的启动参数（需设置为环境变量）
 3.以Volume的形式挂载为容器内部的文件或者目录

#### 通过环境变量获取configmap

**此方式更新configmap,pod中的内容不会同步更新**

（1）通过环境变量获取定义好的ConfigMap 中的内容。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: appvars
data:
  apploglevel: info
  appdatadir: /var/data
---
apiVersion: v1
kind: Pod
metadata:
  name: cm-test-pod
spec:
  containers:
  - name: cm-test
    image: centos
    imagePullPolicy: IfNotPresent
    command: [ "/usr/sbin/init" ]
    env:
    - name: APPLOGLEVEL
      valueFrom:
        configMapKeyRef:
          name: appvars
          key: apploglevel
    - name: APPDATADIR
      valueFrom:
        configMapKeyRef:
          name: appvars
          key: appdatadir
  restartPolicy: Never
```

**通过kubectl logs cm-test-pod 可以看到pod曾经执行过的结果.**

```shell
[root@master ~]# kubectl  logs  cm-test-pod
APPDATADIR=/var/data
APPLOGLEVEL=info
```

#### 通过volume挂载获取configmap

**此方式更新configmap，pod会自动更新，但是延迟比较长，10-20s左右.**

(2) 通过volume挂载的方式将configmap中的内容挂载为容器内部的文件或目录.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cm-appconfigfiles
data:
  game.properties: |
    enemy.types=aliens,monsters
    player.maximum-lives=5
  user-interface.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true
---
apiVersion: v1
kind: Pod
metadata:
  name: cm-test-volume-pod
spec:
  containers:
  - name: cm-test-volume
    image: centos
    imagePullPolicy: IfNotPresent
    command: [ "/usr/sbin/init" ]
    volumeMounts:
    - name: config-test
      mountPath: /configfiles
  volumes:
  - name: config-test
    configMap:
      name: cm-appconfigfiles
      items:
      - key: "game.properties"
        path: "game.properties"
      - key: "user-interface.properties"
        path: "user-interface.properties"
        

# kubectl exec -it cm-test-volume-pod -- ls /configfiles/ 
game.properties  user-interface.properties

# kubectl exec -it cm-test-volume-pod -- cat /configfiles/user-interface.properties 
color.good=purple
color.bad=yellow
allow.textmode=true
```

#### 命令行参数使用configmap变量
(3).设置容器启动命令的启动参数

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: var
  namespace: default
data:
  special.how: very
  special.type: charm
apiVersion: v1
kind: Pod
metadata:
  name: var-test
spec:
  containers:
    - name: test-container
      image: busybox
      command: [ "/bin/sh", "-c", "echo $(SPECIAL_LEVEL_KEY) $(SPECIAL_TYPE_KEY)" ]
      env:
        - name: SPECIAL_LEVEL_KEY
          valueFrom:
            configMapKeyRef:
              name: var
              key: special.how
        - name: SPECIAL_TYPE_KEY
          valueFrom:
            configMapKeyRef:
              name: var
              key: special.type
  restartPolicy: Never                  
[root@master var]# kubectl  logs var-test
very charm
```

### ConfigMap修改

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: configmap-demo
  namespace: default
data:
  index.html: |
      Hello Everyone
---
apiVersion: v1
kind: ReplicationController
metadata:
  name: configmap-demo2
spec:
  template:
    metadata:
      labels:
        app: configmap-demo2
    spec:
      containers:
        - name: configmap-demo2
          image: nginx
          ports:
            - containerPort: 80
          volumeMounts:
          - name: config-volume
            mountPath: /usr/share/nginx/html/
      volumes:
        - name: config-volume
          configMap:
            name: configmap-demo
[root@master change]# curl http://172.25.2.2:80
Hello Everyone  
```

修改

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: configmap-demo
  namespace: default
data:
  index.html: |
      Hello World!
[root@master change]# kubectl  replace -f k8s-configmap.yaml
[root@master change]# curl http://172.25.2.2:80
Hello World! 
```

修改configmap会有延时，要过一段时间容器的配置才会发生变化。

### 参考
[https://kubernetes.io/docs/user-guide/configmap/](https://link.jianshu.com?t=https://kubernetes.io/docs/user-guide/configmap/)
[https://segmentfault.com/a/1190000004940306](https://link.jianshu.com?t=https://segmentfault.com/a/1190000004940306)
[https://github.com/thrawn01/configmap-microservice-demo](https://link.jianshu.com?t=https://github.com/thrawn01/configmap-microservice-demo)
[configmap](https://www.jianshu.com/p/cf3e2218f283)
[configmap热更新](https://jimmysong.io/kubernetes-handbook/concepts/configmap-hot-update.html)