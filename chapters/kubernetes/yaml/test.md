### 创建一个centos测试pod

```yaml
apiVersion: v1  
kind: Pod  
metadata:  
  name: centos
  labels:  
    app: centos 
spec:  
  containers:  
  - name: mycentos
    image: centos
    imagePullPolicy: IfNotPresent
    command: ["/bin/sh"]
    args: ["-c","while true;do echo hello;sleep 1;done"]

# 或者
apiVersion: v1    
kind: Pod    
metadata:    
  name: centos  
  labels:    
    app: centos    
spec:    
  containers:    
  - name: mycentos  
    image: centos  
    imagePullPolicy: IfNotPresent  
    command: ["/bin/sh","-c","while true;do echo hello;sleep 1;done"]
```

