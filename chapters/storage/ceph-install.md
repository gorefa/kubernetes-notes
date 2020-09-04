[TOC]

### 架构

**ceph版本: mimic 13.2.6**

```bash
10.90.28.26 k8s-master-26v28-syq  ceph-deploy minitor
10.90.28.10 k8s-master-10v28-syq  minitor
10.90.28.11 k8s-master-11v28-syq  minitor mgr

10.90.28.22 k8s-ceph-22v28-syq osd
10.90.28.23 k8s-ceph-23v28-syq osd
10.90.28.24 k8s-ceph-24v28-syq osd
10.90.28.25 k8s-ceph-25v28-syq osd
```



### 创建 Ceph 部署用户

**ceph-deploy 工具必须以普通用户登录 Ceph 节点，且此用户拥有无密码使用 sudo 的权限，因为它需要在安装软件及配置文件的过程中，不必输入密码。官方建议所有 Ceph 节点上给 ceph-deploy 创建一个特定的用户，而且不要使用 ceph 这个名字。这里为了方便，我们使用 cephd 这个账户作为特定的用户，而且每个节点上都需要创建该账户，并且拥有 sudo 权限。**

```bash
# 在 Ceph 集群各节点进行如下操作
# 创建 ceph 特定用户
useradd -d /home/cephd -m cephd
# 添加 sudo 权限
echo "cephd ALL = (root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/cephd
chmod 0440 /etc/sudoers.d/cephd
```

#### 生成并推送密钥

**在ceph-deply admin-node 节点生成密钥，并推送到其他节点**

admin节点

```bash
sudo su - cephd
ssh-keygen
```

**推送密钥**

```bash
# 在其他节点上
mkdir -p /home/cephd/.ssh
echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDmLMJrWyaQxRZOgyegX6aEdjNMFCqn8Vp6slmHNThCgkxWUvXt1rI3opmCgpOpCoIVVLmfBU3llmKivyfzl1U+xFTi23nGF0VKg4JduerbnjEar/CqrQRibK4/vhCNg05gxXZHORajcQEAloVILGCPva/N/gD4uxUJMNONxRFWz4EG5fuXlFOXLhSeHXUAHiFBOEwUyrD8Hxl1AuTxHVkWkYAv77Jdv1S/i4HnpCP4/UG8uZgA/rPhjKW1SjJjjE1m2LtaGoIIeNCRMz6XT8E+6WqLv23AfSUwGe8fo9a+8ESDtUDhJijxJkfJiLUN2htV0HavMdpngn6/gwVHJivx cephd@k8s-master-26v28-syq" > /home/cephd/.ssh/authorized_keys
chmod 600 /home/cephd/.ssh/authorized_keys
chmod 700 /home/cephd/.ssh
```

#### 配置config

**增加config (600)**

为了方便ssh,在admin节点增加config

```bash
Host 10.90.28.26
  Hostname k8s-master-26v28-syq
  User cephd
Host 10.90.28.10
  Hostname k8s-master-10v28-syq
  User cephd
Host 10.90.28.11
  Hostname k8s-master-11v28-syq
  User cephd
Host 10.90.28.22
  Hostname k8s-ceph-22v28-syq
  User cephd
Host 10.90.28.23
  Hostname k8s-ceph-23v28-syq
  User cephd
Host 10.90.28.24
  Hostname k8s-ceph-24v28-syq
  User cephd
Host 10.90.28.25
  Hostname k8s-ceph-25v28-syq
  User cephd
```

**确保sshd_config,开启以下两项**

```bash
# cat /etc/ssh/sshd_config  以下两项开启
RSAAuthentication yes
PubkeyAuthentication yes
```

### ceph源配置

```bash
#epel
yum -y install epel-release      

# ceph 国内源
网易镜像源 http://mirrors.163.com/ceph
阿里镜像源 http://mirrors.aliyun.com/ceph
中科大镜像源 http://mirrors.ustc.edu.cn/ceph
宝德镜像源 http://mirrors.plcloud.com/ceph

# ceph.repo 需要时可以用 
cat > /etc/yum.repos.d/ceph.repo << EOF
[Ceph]
name=Ceph packages for $basearch
baseurl=http://mirrors.163.com/ceph/rpm-mimic/el7/x86_64
enabled=1
gpgcheck=0
type=rpm-md
gpgkey=https://mirrors.163.com/ceph/keys/release.asc
priority=1
[Ceph-noarch]
name=Ceph noarch packages
baseurl=http://mirrors.163.com/ceph/rpm-mimic/el7/noarch
enabled=1
gpgcheck=0
type=rpm-md
gpgkey=https://mirrors.163.com/ceph/keys/release.asc
priority=1
[ceph-source]
name=Ceph source packages
baseurl=http://mirrors.163.com/ceph/rpm-mimic/el7/SRPMS
enabled=1
gpgcheck=0
type=rpm-md
gpgkey=https://mirrors.163.com/ceph/keys/release.asc
priority=1
EOF

#ceph-deploy install 时，export环境变量
CentOS:
export CEPH_DEPLOY_REPO_URL=http://mirrors.ustc.edu.cn/ceph/rpm-mimic/el7
export CEPH_DEPLOY_GPG_URL=http://mirrors.ustc.edu.cn/ceph/keys/release.asc
#或者 
ceph install node-hostname --repo-url http://mirros.aliyun.com/ceph/rpm-mimic/el7/

# 如果遇见超时问题，可以在node节点上
sudo echo "minrate=1" >> /etc/yum.conf
sudo echo "timeout=30000" >> /etc/yum.conf
#ceph-deploy(admin) 节点安装
[cephd@test-ceph101v234_zq ~]$ sudo yum -y install ceph ceph-radosgw ceph-deploy
[cephd@test-ceph101v234_zq ~]$ sudo yum -y install htop sysstat iotop iftop ntp ntpdate

```

### 创建集群

```bash
# 创建执行目录
$ mkdir ~/my-cluster && cd ~/my-cluster

#初始化monitor
ceph-deploy new k8s-master-26v28-syq k8s-master-10v28-syq k8s-master-11v28-syq

#Install Ceph packages,这不会把yum换替换为官方的,ps -ef看到只有yum -y install ceph ceph-radosgw
ceph-deploy install k8s-master-26v28-syq k8s-master-10v28-syq k8s-master-11v28-syq

# Deploy the initial monitor(s) and gather the keys
# mon create-initial会根据ceph.conf进行创建mon，判断monitor都创建成功后，会进行keyring的收集，这些keyring在后续创建其他成员的时候要用到
ceph-deploy mon create-initial

#将集群的admin.keyring分发给指定的节点
ceph-deploy admin k8s-master-26v28-syq k8s-master-10v28-syq k8s-master-11v28-syq
```

### 增加osd节点

**osd节点上**

```bash
# 创建 ceph 特定用户
useradd -d /home/cephd -m cephd
# 添加 sudo 权限
echo "cephd ALL = (root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/cephd
chmod 0440 /etc/sudoers.d/cephd

# 推送admin的密钥，见上面步骤
```

**admin节点**

```bash
# 创建osd
ceph-deploy osd create --data /dev/sdb k8s-ceph-22v28-syq
ceph-deploy osd create --data /dev/sdc k8s-ceph-22v28-syq

ceph-deploy osd create --data /dev/sdb k8s-ceph-23v28-syq
ceph-deploy osd create --data /dev/sdc k8s-ceph-23v28-syq

ceph-deploy osd create --data /dev/sdb k8s-ceph-24v28-syq
ceph-deploy osd create --data /dev/sdc k8s-ceph-24v28-syq

ceph-deploy osd create --data /dev/sdb k8s-ceph-25v28-syq
ceph-deploy osd create --data /dev/sdc k8s-ceph-25v28-syq
```

### mgr

创建mgr

```bash
ceph-deploy mgr create k8s-master-11v28-syq
```

#### dashborad

```bash
sudo ceph dashboard create-self-signed-cert

mkdir dashboard-ssl
cd dashboard-ssl/

sudo openssl req -new -nodes -x509 \
-subj "/O=IT/CN=ceph-mgr-dashboard" -days 3650 \
-keyout dashboard.key -out dashboard.crt -extensions v3_ca

sudo ceph config-key set mgr mgr/dashboard/crt -i dashboard.crt
sudo ceph config-key set mgr mgr/dashboard/key -i dashboard.key

sudo ceph config set mgr mgr/dashboard/server_addr 10.90.28.11
sudo ceph config set mgr mgr/dashboard/server_port 8080

sudo ceph mgr module disable dashboard
sudo ceph mgr module enable dashboard 
sudo ceph dashboard set-login-credentials admin admin@2019
sudo ceph mgr services  

```

#### prometheus

```bash
sudo ceph mgr module enable prometheus
sudo ceph config set mgr mgr/prometheus/server_addr 10.90.28.11
```

### 遇到的问题

#### error: GPT headers found

```bash
[k8s-ceph-22v28-syq][WARNIN] ceph-volume lvm create: error: GPT headers found, they must be removed on: /dev/sdb
fdisk -l 会看到：
WARNING: fdisk GPT support is currently new, and therefore in an experimental phase. Use at your own discretion.


# 需要手动清理掉硬盘的分区表
dd if=/dev/zero of=/dev/sdx bs=512K count=1
```

#### HEALTH_WARN application not enabled on 1 pool(s)

```bash
# ceph health detail
HEALTH_WARN application not enabled on 1 pool(s)
POOL_APP_NOT_ENABLED application not enabled on 1 pool(s)
    application not enabled on pool 'kube'
    use 'ceph osd pool application enable <pool-name> <app-name>', where <app-name> is 'cephfs', 'rbd', 'rgw', or freeform for custom applications.
# ceph osd pool application enable kube rbd
enabled application 'rbd' on pool 'kube'
# ceph health
HEALTH_OK
```

#### AttributeError: _init_cffi_1_0_external_module

```bash
启动dashboard有报错 
It should be python-cffi >= 1.4.
```



### 参考

[Ceph安装配置手册](https://sysit.cn/blog/post/sysit/Ceph安装配置手册)

[ceph-rpm](http://download.ceph.com/rpm-mimic/el7/x86_64/)