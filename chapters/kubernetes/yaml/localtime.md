**pod.spec.containers下：**

```yaml
    volumeMounts:
    - mountPath: /etc/localtime
      name: localtime
      readOnly: true
```

**pod.spec.volumes下：**

```yaml
  volumes:
  - hostPath:
      path: /etc/localtime
      type: ""
    name: localtime
```

