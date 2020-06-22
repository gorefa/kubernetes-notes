

### 自动创建docker tls证书脚本

```bash
#!/bin/bash
# 
# -------------------------------------------------------------
# 自动创建 Docker TLS 证书
# -------------------------------------------------------------

# 以下是配置信息
# --[BEGIN]------------------------------

CODE="dp"
IP="docker服务器ip"
PASSWORD="证书密码"
COUNTRY="CN"
STATE="BEIJING"
CITY="BEIJING"
ORGANIZATION="公司"
ORGANIZATIONAL_UNIT="Dev"
COMMON_NAME="$IP"
EMAIL="邮箱"

# --[END]--

# Generate CA key
openssl genrsa -aes256 -passout "pass:$PASSWORD" -out "ca-key-$CODE.pem" 4096
# Generate CA
openssl req -new -x509 -days 365 -key "ca-key-$CODE.pem" -sha256 -out "ca-$CODE.pem" -passin "pass:$PASSWORD" -subj "/C=$COUNTRY/ST=$STATE/L=$CITY/O=$ORGANIZATION/OU=$ORGANIZATIONAL_UNIT/CN=$COMMON_NAME/emailAddress=$EMAIL"
# Generate Server key
openssl genrsa -out "server-key-$CODE.pem" 4096

# Generate Server Certs.
openssl req -subj "/CN=$COMMON_NAME" -sha256 -new -key "server-key-$CODE.pem" -out server.csr

echo "subjectAltName = IP:$IP,IP:127.0.0.1" >> extfile.cnf
echo "extendedKeyUsage = serverAuth" >> extfile.cnf

openssl x509 -req -days 365 -sha256 -in server.csr -passin "pass:$PASSWORD" -CA "ca-$CODE.pem" -CAkey "ca-key-$CODE.pem" -CAcreateserial -out "server-cert-$CODE.pem" -extfile extfile.cnf


# Generate Client Certs.
rm -f extfile.cnf

openssl genrsa -out "key-$CODE.pem" 4096
openssl req -subj '/CN=client' -new -key "key-$CODE.pem" -out client.csr
echo extendedKeyUsage = clientAuth >> extfile.cnf
openssl x509 -req -days 365 -sha256 -in client.csr -passin "pass:$PASSWORD" -CA "ca-$CODE.pem" -CAkey "ca-key-$CODE.pem" -CAcreateserial -out "cert-$CODE.pem" -extfile extfile.cnf

rm -vf client.csr server.csr

chmod -v 0400 "ca-key-$CODE.pem" "key-$CODE.pem" "server-key-$CODE.pem"
chmod -v 0444 "ca-$CODE.pem" "server-cert-$CODE.pem" "cert-$CODE.pem"

# 打包客户端证书
mkdir -p "tls-client-certs-$CODE"
cp -f "ca-$CODE.pem" "cert-$CODE.pem" "key-$CODE.pem" "tls-client-certs-$CODE/"
cd "tls-client-certs-$CODE"
tar zcf "tls-client-certs-$CODE.tar.gz" *
mv "tls-client-certs-$CODE.tar.gz" ../
cd ..
rm -rf "tls-client-certs-$CODE"

# 拷贝服务端证书
mkdir -p /etc/docker/certs.d
cp "ca-$CODE.pem" "server-cert-$CODE.pem" "server-key-$CODE.pem" /etc/docker/certs.d/
```

对脚本中的变量进行修改后运行，自动会创建好tls证书，服务器的证书在/etc/docker/certs.d/目录下:

![img](https://llussy.github.io/images/cf128d18f1e5494241522354509639ac476.png)

客户端的证书在运行脚本的目录下，同时还自动打好了一个.tar.gz的包，很方便。

![img](https://llussy.github.io/images/a52727e6614faf2c2985912dd992a699ef8.png)



### docker 配置

```bash
# /usr/lib/systemd/system/docker.service
ExecStart=/usr/bin/dockerd --tls --tlscacert=/etc/docker/certs.d/ca-dp.pem --tlscert=/etc/docker/certs.d/server-cert-dp.pem --tlskey=/etc/docker/certs.d/server-key-dp.pem -H tcp://0.0.0.0:2375 -H unix://var/run/docker.sock


# buildkit 开启
dockerd --tls --tlscacert=/etc/docker/certs.d/ca-dp.pem --tlscert=/etc/docker/certs.d/server-cert-dp.pem --tlskey=/etc/docker/certs.d/server-key-dp.pem -H tcp://0.0.0.0:2375 -D --experimental

```

### 客户端配置

复制客户端配置

```bash
export DOCKER_CERT_PATH="/Users/lisai/docker/18.09tls"
export DOCKER_HOST="tcp://192.168.1.1:2375"
export DOCKER_API_VERSION="1.39"

ls /Users/lisai/docker/18.09tls
ca.pem   cert.pem key.pem

```

### 参考

[docker api tls](https://my.oschina.net/ykbj/blog/1920021)