

### docker安装

#### centos


```bash
# docker官方包下载地址
https://download.docker.com/linux/centos/7/x86_64/stable/Packages/

yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum install docker-ce

mkdir -p /etc/docker/
cat>/etc/docker/daemon.json<<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "registry-mirrors": ["https://fz5yth0r.mirror.aliyuncs.com"],
  "data-root": "/data/docker",
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  }
}
EOF

yum install -y epel-release bash-completion && cp /usr/share/bash-completion/completions/docker /etc/bash_completion.d/
systemctl enable --now docker

```

#### ubuntu
```bash
apt-get remove docker docker-engine docker.io -y 
sudo apt-get update
sudo apt-get install     apt-transport-https     ca-certificates     curl     software-properties-common -y 
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository    "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
$(lsb_release -cs) \
stable"
add-apt-repository ppa:ubuntu-sdk-team/ppa
apt-get update


# apt-cache madison docker-ce
 docker-ce | 18.03.1~ce-0~ubuntu | https://download.docker.com/linux/ubuntu xenial/stable amd64 Packages
 docker-ce | 18.03.0~ce-0~ubuntu | https://download.docker.com/linux/ubuntu xenial/stable amd64 Packages
 docker-ce | 17.12.1~ce-0~ubuntu | https://download.docker.com/linux/ubuntu xenial/stable amd64 Packages
 docker-ce | 17.12.0~ce-0~ubuntu | https://download.docker.com/linux/ubuntu xenial/stable amd64 Packages
 docker-ce | 17.09.1~ce-0~ubuntu | https://download.docker.com/linux/ubuntu xenial/stable amd64 Packages
 docker-ce | 17.09.0~ce-0~ubuntu | https://download.docker.com/linux/ubuntu xenial/stable amd64 Packages
 docker-ce | 17.06.2~ce-0~ubuntu | https://download.docker.com/linux/ubuntu xenial/stable amd64 Packages
 docker-ce | 17.06.1~ce-0~ubuntu | https://download.docker.com/linux/ubuntu xenial/stable amd64 Packages
 docker-ce | 17.06.0~ce-0~ubuntu | https://download.docker.com/linux/ubuntu xenial/stable amd64 Packages
 docker-ce | 17.03.2~ce-0~ubuntu-xenial | https://download.docker.com/linux/ubuntu xenial/stable amd64 Packages
 docker-ce | 17.03.1~ce-0~ubuntu-xenial | https://download.docker.com/linux/ubuntu xenial/stable amd64 Packages
 docker-ce | 17.03.0~ce-0~ubuntu-xenial | https://download.docker.com/linux/ubuntu xenial/stable amd64 Packages
 
 
apt install docker-ce -y
```


#### 自动补全

```bash
# 自动补全
curl -L https://raw.githubusercontent.com/docker/docker/v$(docker version -f "{{.Client.Version}}")/contrib/completion/bash/docker -o /etc/bash_completion.d/docker

vim ~/.bash_profile
# 添加如下内容
[ -f /etc/bash_completion.d ] && . /etc/bash_completion.d
source ~/.bash_profile
```

### docker Storage Driver

#### devicemapper

推荐使用overlay2

```bash
# docker info | grep 'Base Device Size'
WARNING: the devicemapper storage-driver is deprecated, and will be removed in a future release.
WARNING: devicemapper: usage of loopback devices is strongly discouraged for production use.
         Use `--storage-opt dm.thinpooldev` to specify a custom block storage device.
 Base Device Size: 10.74GB
```

devicemapper存储空间默认为10G.

此参数可以修改，`--storage-opt dm.basesize=20G`但是需要删除docker目录下面的所有内容。

#### overlay2

```bash
# cat /etc/docker/daemon.json 
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "registry-mirrors": ["https://fz5yth0r.mirror.aliyuncs.com"],
  "data-root": "/data/docker",
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  }
}
```





### docker 常用命令

**查看容器大小**

```bash
docker ps -as
```

**格式化输出**

```bash
docker inspect --format {{.State.Pid}} b214beb6935d      #格式化，也可使用-f

docker inspect  --format  "{{.NetworkSettings.IPAddress}}" 51d6d51cbe5d  #查看容器IP
```

**删除容器**

```bash
# 删除所有容器
docker rm `docker ps -a -q` 

#删除所有退出的容器
docker rm -v $(docker ps -aq -f status=exited)** 
```

**暂停恢复容器**

```bash
docker pause xxxxxx
docker unpause xxxxxx
```

**导出导入容器**

```
docker export 43710668e83c > ubuntu.tar
cat ubuntu.tar | sudo docker import - test/ubuntu:v1.0
```

#### 导入导出镜像

```bash
docker save centos > /data/iso/centos.tar.gz  #导出镜像 
docker load < /data/iso/centos.tar.gz #导入镜像 
#导出所有镜像
docker save $(docker images --format {{.Repository}}:{{.Tag}}) | gzip -> all.zip
```



**限制**

```
-m 或 --memory：设置内存的使用限额，例如 100M, 2G。--memory-swap：设置 内存+swap 的使用限额。

docker run -m 200M --memory-swap=300M ubuntu

其含义是允许该容器最多使用 200M 的内存和 100M 的 swap。默认情况下，上面两组参数为 -1，即对容器内存和 swap 的使用没有限制。

Docker 可以通过 -c 或 --cpu-shares 设置容器使用 CPU 的权重。如果不指定，默认值为 1024。
```





### docker 端口

**-p** <宿主机端口>:<容器端口>    映射宿主端口和容器端口，换句话说，就是将容器的对应端口服务公开给外界访问。

**-P** 参数表示随机映射，把宿主机的一个随机端口映射到容器对外开放的端口。



### docker 卷

使用-v可以挂载一个本地的目录到容器中作为数据卷。

**-v** <宿主机目录>:<容器目录>  ` docker run -it -v /test:/soft centos /bin/bash`



### docker 前台执行

**Docker 不是虚拟机，容器中的应用都应该以前台执行，而不是像虚拟机、物理机**里面那样，用 upstart/systemd 去启动后台服务，容器内没有后台服务的概念。

一些初学者将  CMD  写为：

**CMD service nginx start 发现容器执行后就立即退出了。**

甚至在容器内去使用 systemctl 命令结果却发现根本执行不了。这就是因为没有搞明白前台、后台的概念，没有区分容器和虚拟机的差异，依旧在以传统虚拟机的角度去理解容器。对于容器而言，其启动程序就是容器应用进程，容器就是为了主进程而存在的，主进程退出，容器就失去了存在的意义，从而退出，其它辅助进程不是它需要关心的东西。

而使用 service nginx start 命令，则是希望 upstart 来以后台守护进程形式启动 nginx 服务。而刚才说了 CMD service nginx start 会被理解为 `CMD ["sh", "-c", "service nginx start"] `，因此主进程实际上是 sh 。那么当service nginx start 命令结束后， sh 也就结束了， sh 作为主进程退出了，自然就会令容器退出。正确的做法是直接执行 nginx 可执行文件，并且要求以前台形式运行。比如：

`CMD ["nginx", "-g", "daemon off;"]`

### 清理docker空间

```bash
docker network prune --filter "until=24h"
docker image prune -a --filter "until=24h"
docker container prune --filter "until=24h"
```

