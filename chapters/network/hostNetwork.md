

> Avoid using `hostNetwork`, for the same reasons as `hostPort`.

​		hostNetwork设置适用于Kubernetes pod。当pod配置为hostNetwork：true时，在此类pod中运行的应用程序可以直接查看启动pod的主机的网络接口。配置为侦听所有网络接口的应用程序，又可以在主机的所有网络接口上访问。以下是使用主机网络的pod的示例定义:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: influxdb
spec:
  hostNetwork: true
  containers:
    - name: influxdb
      image: influxdb
```

您可以使用以下命令启动pod:

```yaml
$ kubectl create -f influxdb-hostnetwork.yml
```

您可以检查InfluxDB应用程序是否正在运行:

```bash
# curl -v 10.21.8.106:8086/ping
* About to connect() to 10.21.8.106 port 8086 (#0)
*   Trying 10.21.8.106...
* Connected to 10.21.8.106 (10.21.8.106) port 8086 (#0)
> GET /ping HTTP/1.1
> User-Agent: curl/7.29.0
> Host: 10.21.8.106:8086
> Accept: */*
> 
< HTTP/1.1 204 No Content
< Content-Type: application/json
< Request-Id: 4ccebe25-45fb-11e9-8002-005056bdf3a2
< X-Influxdb-Build: OSS
< X-Influxdb-Version: 1.7.4
< X-Request-Id: 4ccebe25-45fb-11e9-8002-005056bdf3a2
< Date: Thu, 14 Mar 2019 01:48:43 GMT
< 
* Connection #0 to host 10.21.8.106 left intact
```

当pod 设置hostNetwork: true时候，Pod中的所有容器就直接暴露在宿主机的网络环境中，这时候，Pod的PodIP就是其所在Node的IP。

对于同Deployment下的hostNetwork: true启动的Pod，每个node上只能启动一个。也就是说，Host模式的Pod启动副本数不可以多于“目标node”的数量，“目标node”指的是在启动Pod时选定的node，若未选定（没有指定nodeSelector），“目标node”的数量就是集群中全部的可用的node的数量。当副本数大于“目标node”的数量时，多出来的Pod会一直处于Pending状态，因为schedule已经找不到可以调度的node了。

### 参考

[hostNetwork](https://segmentfault.com/a/1190000016123122)

