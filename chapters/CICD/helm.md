

helm3没有了tiller,使用起来方便了不少。

[helm2 blog](https://llussy.github.io/2019/01/16/helm/)

### helm3 install

```bash
wget https://get.helm.sh/helm-v3.2.4-linux-amd64.tar.gz
tar zxvf helm-v3.2.4-linux-amd64.tar.gz 
cd linux-amd64
mv helm /usr/local/bin/


source <(helm completion bash)



# 增加国内源

helm repo add stable https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
# helm repo list
NAME    URL                                                   
stable  https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts

helm repo update
helm search repo jenkins


```

### 