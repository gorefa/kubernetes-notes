[toc]

### jenkins部署

[主要参考-基于 kubernetes 的动态 jenkins slave](https://www.qikqiak.com/post/kubernetes-jenkins1/)

**rbac**

```yaml
# cat rbac.yaml 
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins2
  namespace: kube-ops

---

kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: jenkins2
rules:
  - apiGroups: ["extensions", "apps"]
    resources: ["deployments"]
    verbs: ["create", "delete", "get", "list", "watch", "patch", "update"]
  - apiGroups: [""]
    resources: ["services"]
    verbs: ["create", "delete", "get", "list", "watch", "patch", "update"]
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["create","delete","get","list","patch","update","watch"]
  - apiGroups: [""]
    resources: ["pods/exec"]
    verbs: ["create","delete","get","list","patch","update","watch"]
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get","list","watch"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get"]

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: jenkins2
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: jenkins2
subjects:
  - kind: ServiceAccount
    name: jenkins2
    namespace: kube-ops
```

**pvc**

```yaml
# cat pvc.yaml 
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: jenkins-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi

# cat kube-ops.yaml 
apiVersion: v1
kind: Secret
metadata:
  name: ceph-secret
  namespace: kube-ops
type: "kubernetes.io/rbd"
data:
  # ceph auth add client.kubernetes mon 'allow r' osd 'allow rwx pool=kubernetes'
  # ceph auth get-key client.kubernetes | base64
  key: QVFCYUtraGNxSjdaRXhBQTlCR1h4NkdxUUxjRU9uZlNoYjZadXc9PQ==
```



**deployment svc**

```bash
# cat jenkins2.yaml 
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins2
  namespace: kube-ops
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins2
  template:
    metadata:
      labels:
        app: jenkins2
    spec:
      terminationGracePeriodSeconds: 10
      serviceAccountName: jenkins2
      containers:
      - name: jenkins
        env:
        - name: JAVA_OPTS
          value: "-Dhudson.model.DownloadService.noSignatureCheck=true"
        image: jenkins/jenkins:lts
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
          name: web
          protocol: TCP
        - containerPort: 50000
          name: agent
          protocol: TCP
        resources:
          limits:
            cpu: 3000m
            memory: 4Gi
          requests:
            cpu: 500m
            memory: 512Mi
        livenessProbe:
          httpGet:
            path: /login
            port: 8080
          initialDelaySeconds: 60
          timeoutSeconds: 5
          failureThreshold: 12
        readinessProbe:
          httpGet:
            path: /login
            port: 8080
          initialDelaySeconds: 60
          timeoutSeconds: 5
          failureThreshold: 12
        volumeMounts:
        - name: jenkinshome
          subPath: jenkins2
          mountPath: /var/jenkins_home
        - name: localtime
          mountPath: /etc/localtime
          readOnly: true
      securityContext:
        fsGroup: 1000
      volumes:
      - name: jenkinshome
        persistentVolumeClaim:
          claimName: jenkins-pvc
      - name: localtime
        hostPath:
          path: /etc/localtime
          type: ""
        
---
apiVersion: v1
kind: Service
metadata:
  name: jenkins2
  namespace: kube-ops
  labels:
    app: jenkins2
spec:
  selector:
    app: jenkins2
  ports:
  - name: web
    port: 8080
    targetPort: web
  - name: agent
    port: 50000
    targetPort: agent
```

**ingress**

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: traefik
  labels:
    app: jenkins
  name: jenkins
  namespace: kube-ops
spec:
  rules:
  - host: jenkins2.llussy.com
    http:
      paths:
      - backend:
          serviceName: jenkins2
          servicePort: web
```



### jenkins源

在 `jenkins --> Manage Jenkins --> Manage Plugins --> Advanced` 的选项中选中更新Update Site。 将更新源重 https://updates.jenkins-ci.org/update-center.json 替换成https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json

**javaopt 需要增加 -Dhudson.model.DownloadService.noSignatureCheck=true**

```bash
# 修改jenkins家目录默认配置文件

cd /var/jenkins_home/updates/

sed -i 's/http:\/\/updates.jenkins-ci.org\/download/https:\/\/mirrors.tuna.tsinghua.edu.cn\/jenkins/g' default.json && sed -i 's/http:\/\/www.google.com/https:\/\/www.baidu.com/g' default.json
```



### 配置jenkins连接k8s

#### 生成PKCS证书

```bash
# k8s master节点操作

# cat jenkins-csr.json
{
  "CN": "cluster-admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "jenkins-admin",
      "OU": "jenkins-admin"
    }
  ]
}



# cfssl gencert -ca=/etc/kubernetes/pki/ca.crt -ca-key=/etc/kubernetes/pki/ca.key --profile=kubernetes jenkins-csr.json | cfssljson -bare jenkins-admin


# ll -h
total 16K
-rw-r--r-- 1 root root 1.1K Aug  4 09:21 jenkins-admin.csr
-rw------- 1 root root 1.7K Aug  4 09:21 jenkins-admin-key.pem
-rw-r--r-- 1 root root 1.3K Aug  4 09:21 jenkins-admin.pem
-rw-r--r-- 1 root root  243 Aug  4 09:19 jenkins-csr.json



我们可以通过openssl来转换成pkc格式

openssl pkcs12 -export -out ./jenkins-admin.pfx -inkey ./jenkins-admin-key.pem -in ./jenkins-admin.pem -passout pass:secret


# clusterrole  这里使用admin权限，线上需权衡
cat clusterrole.yaml 
kind: ClusterRoleBinding
metadata:
  name: jenkins-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin      # 要绑定的权限
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: jenkins-admin  # 与上面的O对应

```

#### 配置连接k8s

点击 Manage Jenkins —> Configure System —> (拖到最下方)Add a new cloud —> 选择 Kubernetes，然后填写 Kubernetes 和 Jenkins 配置信息。

![image-20200804093257672](/Users/lisai/go/src/llussy.github.io/images/image-20200804093257672.png)

Kubernetes server certificate key 为  /etc/kubernetes/pki/ca.crt内容。

Credentials 为上面生成的。参考链接：https://www.cnblogs.com/xiao987334176/p/11338827.html

#### 配置pod模板

jenkins:jnlp dockerfile 

```dockerfile
FROM jenkins/jnlp-slave

ENV DOCKER_VERSION=18.09.6-ce DOCKER_COMPOSE_VERSION=1.24.0 KUBECTL_VERSION=v1.18.5

USER root


RUN apt-get update && \
    apt-get -y install apt-transport-https \
    ca-certificates \
    curl \
    gnupg2 \
    software-properties-common && \
    curl -fsSL https://download.docker.com/linux/$(. /etc/os-release; echo "$ID")/gpg > /tmp/dkey; apt-key add /tmp/dkey && \
    add-apt-repository \
    "deb [arch=amd64] https://download.docker.com/linux/$(. /etc/os-release; echo "$ID") \
    $(lsb_release -cs) \
    stable" && \
    apt-get update && \
    apt-get -y install docker-ce


RUN curl -L https://github.com/docker/compose/releases/download/${DOCKER_COMPOSE_VERSION}/docker-compose-Linux-x86_64 -o /usr/local/bin/docker-compose \
    && chmod +x /usr/local/bin/docker-compose

RUN curl -L https://storage.googleapis.com/kubernetes-release/release/${KUBECTL_VERSION}/bin/linux/amd64/kubectl -o /usr/local/bin/kubectl \
    && chmod +x /usr/local/bin/kubectl

ENTRYPOINT ["jenkins-slave"]
```



![image-20200806133004992](/Users/lisai/go/src/llussy.github.io/images/image-20200806133004992.png)

**挂载相应目录**

![image-20200806133033783](/Users/lisai/go/src/llussy.github.io/images/image-20200806133033783.png)

**add sa**

![image-20200806133053103](/Users/lisai/go/src/llussy.github.io/images/image-20200806133053103.png)

### sample Freestyle project

**shell**

```bash
echo "测试 Kubernetes 动态生成 jenkins slave"
echo "==============docker in docker==========="
docker info

echo "=============kubectl============="
kubectl get pods --all-namespaces

echo "=============kubectl version============="
kubectl version  
```



### k8s pipeline

```bash
node('testjnlp') {
    stage('Clone') {
        echo "1.Clone Stage"
        git url: "https://github.com/llussy/Dockerfile.git"
        sh "git checkout nginx-prometheus-metrics"
        script {
            build_tag = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
        }
    }
    stage('Test') {
      echo "2.Test Stage"
    }
    stage('Build') {
        echo "3.Build Docker Image Stage"
        sh "docker build -t anfeng/nginx-pro:${build_tag} ."
    }
    stage('Push') {
        echo "4.Push Docker Image Stage"
        withCredentials([usernamePassword(credentialsId: 'dockerHub', passwordVariable: 'dockerHubPassword', usernameVariable: 'dockerHubUser')]) {
            sh "docker login -u ${dockerHubUser} -p ${dockerHubPassword}"
            sh "docker push anfeng/nginx-pro:${build_tag}"
        }
    }
    stage('Deploy') {
        echo "5. Deploy Stage"
        def userInput = input(
            id: 'userInput',
            message: 'Choose a deploy environment',
            parameters: [
                [
                    $class: 'ChoiceParameterDefinition',
                    choices: "Dev\nQA\nProd",
                    name: 'Env'
                ]
            ]
        )
        echo "This is a deploy step to ${userInput}"
        sh "sed -i 's/<BUILD_TAG>/${build_tag}/' k8s.yaml"
        if (userInput == "Dev") {
            // deploy dev stuff
        } else if (userInput == "QA"){
            // deploy qa stuff
        } else {
            // deploy prod stuff
        }
        sh "kubectl apply -f k8s.yaml"
    }
}
```

