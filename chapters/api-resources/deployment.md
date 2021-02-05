[TOC]

## deployment

Deployment 只负责管理不同版本的 ReplicaSet，由 ReplicaSet 来管理具体的 Pod 副本数，每个 ReplicaSet 对应 Deployment template 的一个版本。

Deployment 创建 ReplicaSet，而 ReplicaSet 创建 Pod。他们的 OwnerRef 其实都对应了其控制器的资源。



deployment主要职责是为了保证pod的数量和健康。

功能：

- Replication Controller全部功能
- 事件和状态查看：可以查看Deployment的升级详细进度和状态。
- 回滚：当升级pod镜像或者相关参数的时候发现问题，可以使用回滚操作回滚到上一个稳定的版本或者指定的版本。
- 版本记录: 每一次对Deployment的操作，都能保存下来，给予后续可能的回滚使用。
- 暂停和启动：对于每一次升级，都能够随时暂停和启动。
- 多种升级方案：Recreate：删除所有已存在的pod,重新创建新的; RollingUpdate：滚动升级，逐步替换的策略，同时滚动升级时，支持更多的附加参数，例如设置最大不可用pod数量，最小升级间隔时间等等。

## deployment的常用命令

### 查看部署状态

```bash
# apply
kubectl apply -f test.yaml --record=true

kubectl rollout status deployments review-demo --namespace=scm
kubectl describe deployments review-demo  --namespace=scm
```

### 升级

```bash
kubectl set image deployment/review-demo review-demo=library/review-demo:0.0.1 --namespace=scm
```

或者

```bash
kubectl edit deployment/review-demo --namespace=scm
```

编辑.spec.template.spec.containers[0].image的值

**修改yaml文件**

使用修改yaml文件的方式  **kubectl apply -f httpd.yaml --record**

`--record` 的作用是将当前命令记录到 revision 记录中，这样我们就可以知道每个 revison 对应的是哪个配置文件，通过 `kubectl rollout history deployment httpd` 查看 revison 历史记录。

### 查看升级状态

```bash
kubectl rollout status deployment  deployment-name
```

### 终止升级

```bash
kubectl rollout pause deployment review-demo --namespace=scm
```

### 继续升级

```bash
kubectl rollout resume deployment review-demo --namespace=scm
```

### 回滚上一版本

```
kubectl rollout undo deployment review-demo --namespace=scm
```

### 查看deployments历史版本

```bash
kubectl rollout history deployments  deployment-name
```

**回滚到指定版本**

```bash
kubectl rollout undo deployment review-demo --to-revision=2 --namespace=scm
```

**详情**

```bash
kubectl describe deployment review-demo  --namespace=scm
```

### K8S重启Deployment的小技巧

```bash
kubectl patch deployment <deployment-name> \
  -p '{"spec":{"template":{"spec":{"containers":[{"name":"<container-name>","env":[{"name":"RESTART_","value":"'$(date +%s)'"}]}]}}}}'


kubectl -n test-k8s-loda patch deployment nginx-test-dpt \
  -p '{"spec":{"template":{"spec":{"containers":[{"name":"nginx-test","env":[{"name":"RESTART_","value":"'$(date +%s)'"}]}]}}}}'
```

基本思路就是给Container添加一个无关紧要的环境变量，这个环境变量的值就是时间戳，而这个时间戳则是每次执行上述命令的系统当前时间。这样一来对于K8S来讲这个Deployment spec就变化了，就可以像[Updating a deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#updating-a-deployment)一样，重启Pod了。



## 几个重要参数说明

### maxSurge与maxUnavailable

maxSurge: 1 表示滚动升级时会先启动1个pod
maxUnavailable: 1 表示滚动升级时允许的最大Unavailable的pod个数
由于replicas为3,则整个升级,pod个数在2-4个之间

### terminationGracePeriodSeconds

k8s将会给应用发送SIGTERM信号，可以用来正确、优雅地关闭应用,默认为30秒。

如果需要更优雅地关闭，则可以使用k8s提供的pre-stop lifecycle hook 的配置声明，将会在发送SIGTERM之前执行。

### livenessProbe与readinessProbe

livenessProbe是kubernetes认为该pod是存活的，不存在则需要kill掉，然后再新启动一个，以达到replicas指定的个数。

readinessProbe是kubernetes认为该pod是启动成功的，这里根据每个应用的特性，自己去判断，可以执行command，也可以进行httpGet。比如对于使用java web服务的应用来说，并不是简单地说tomcat启动成功就可以对外提供服务的，还需要等待spring容器初始化，数据库连接连接上等等。对于spring boot应用，默认的actuator带有/health接口，可以用来进行启动成功的判断。

其中readinessProbe.initialDelaySeconds可以设置为系统完全启动起来所需的最少时间，livenessProbe.initialDelaySeconds可以设置为系统完全启动起来所需的最大时间+若干秒。

> 这几个参数配置好了之后，基本就可以实现近乎无缝地平滑升级了。对于使用服务发现的应用来说，readinessProbe可以去执行命令，去查看是否在服务发现里头应该注册成功了，才算成功。

## doc

- [【分享】几种常见的不停机发布方式](https://icewing.cc/ci-cd-hot-deploment.html)
- [Deployment vs ReplicationController in Kubernetes](https://www.qcloud.com/community/article/695320001482723928)
- [kubernetes-user-guide-deployments](https://kubernetes.io/docs/user-guide/deployments/)
- [Kubernetes用户指南（三）--在生产环境中使用Pod来工作、管理部署](http://blog.csdn.net/qq1010885678/article/details/49156557)
- [Kubernetes livenessProbe shutdown during application startup](http://stackoverflow.com/questions/39331991/kubernetes-livenessprobe-shutdown-during-application-startup)
- [Kubernetes技术研究容器监控监测](http://blog.coocla.org/kubernetes-container-monitor.html)
- [Graceful shutdown of pods with Kubernetes](https://pracucci.com/graceful-shutdown-of-kubernetes-pods.html)
- [k8s deployment重启技巧](https://chanjarster.github.io/post/k8s-restart-deployment/)