### docker安装

```bash
cd /data/soft/packs
wget http://10.21.8.20/k8s/docker-ce-18.09.2-3.el7.x86_64.rpm
wget http://10.21.8.20/k8s/docker-ce-cli-18.09.2-3.el7.x86_64.rpm
wget http://10.21.8.20/k8s/containerd.io-1.2.2-3.3.el7.x86_64.rpm
yum install *.rpm -y

mkdir /etc/docker
cat > /etc/docker/daemon.json << EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "registry-mirrors": ["https://fz5yth0r.mirror.aliyuncs.com"],
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
systemctl enable docker
systemctl start docker
```

### docker-compose 安装

```bash
# pip install 
export http_proxy=http://10.90.11.11:3128
export https_proxy=http://10.90.11.11:3128
cd /data/soft
yum install wget -y
wget --no-check-certificate https://pypi.python.org/packages/source/s/setuptools/setuptools-12.0.3.tar.gz#md5=f07e4b0f4c1c9368fcd980d888b29a65 
tar -zxvf setuptools-12.0.3.tar.gz
cd setuptools-12.0.3
python setup.py install

cd /data/soft
wget http://10.21.8.20/soft/pip-9.0.1.tar.gz
tar zxvf pip-9.0.1.tar.gz
cd pip-9.0.1
python setup.py install
pip install --upgrade pip 

pip install docker-compose

# docker-compose version
docker-compose version 1.24.0, build 0aa5906
docker-py version: 3.7.2
CPython version: 2.7.5
OpenSSL version: OpenSSL 1.0.1e-fips 11 Feb 2013
```

### golang安装

```bash
yum install golang -y
# go version
go version go1.11.5 linux/amd64
```

### harbor安装

```bash
wget https://storage.googleapis.com/harbor-releases/release-1.7.0/harbor-offline-installer-v1.7.5.tgz
tar zxvf harbor-offline-installer-v1.7.5.tgz
cd harbor
./install


# 默认的帐号密码是admin, Harbor12345
docker-compose ps  #查看compose状态

```

### 参考

[insallation_guide](<https://github.com/goharbor/harbor/blob/master/docs/installation_guide.md>)

[企业级镜像仓库harbor搭建(http/https)及使用](<https://blog.51cto.com/slitobo/2323332>)