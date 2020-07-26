### statefuleset example

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nginx-none
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-none
  serviceName: nginx-none-svc
  template:
    metadata:
      labels:
        app: nginx-none
    spec:
      containers:
      - name: nginx
        image: nginx:1.10
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "1"
            memory: "1024M"
        volumeMounts:
        - mountPath: /etc/localtime
          name: localtime
          readOnly: true
      volumes:
      - hostPath:
          path: /etc/localtime
          type: ""
        name: localtime
---
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-nginx-none
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: StatefulSet
    name: nginx-none
  minReplicas: 3
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      targetAverageUtilization: 50
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-none-svc
  labels:
    app: nginx-none-svc
spec:
  selector:
    app: nginx-none
  type: ClusterIP
  ports:
  - name: nginx
    port: 80  
    protocol: TCP
    targetPort: 80
```