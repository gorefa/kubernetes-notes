### nsenter

快速脚本，需进入进入pod所在node,增加下面函数到~/.bashrc

```bash
  function e() {
      set -eu
      ns=${2-"default"}
      pod=`kubectl -n $ns describe pod $1 | grep -A10 "^Containers:" | grep -Eo 'docker://.*$' | head -n 1 | sed 's/docker:\/\/\(.*\)$/\1/'`
      pid=`docker inspect -f {{.State.Pid}} $pod`
      echo "entering pod netns for $ns/$1"
      cmd="nsenter -n --target $pid"
      echo $cmd
      $cmd
  }
```

进入pod所在netns

```bash
e istio-galley-58c7c7c646-m6568 istio-system
e proxy-5546768954-9rxg6 # 省略 NAMESPACE 默认为 default
tcpdump -i eth0 -w test.pcap port 80
```

原文地址： [Kubernetes 问题定位技巧：容器内抓包](https://imroc.io/posts/kubernetes/capture-packets-in-container/)

