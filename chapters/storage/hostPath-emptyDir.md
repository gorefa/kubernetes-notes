**hostPath** Volume 的作用是将 Docker Host 文件系统中已经存在的目录 mount 给 Pod 的容器。

**mountPath**  容器里的挂载点

**emptyDir** emptyDir 是最基础的 Volume 类型。正如其名字所示，一个 emptyDir Volume 是 Host 上的一个空目录。

`emptyDir Volume` 对于容器来说是持久的，对于 Pod 则不是。当 Pod 从节点删除时，Volume 的内容也会被删除。但如果只是容器被销毁而 Pod 还在，则 Volume 不受影响。也就是说：**emptyDir Volume 的生命周期与 Pod 一致。**



```yaml
        volumeMounts:
        - name: config
          mountPath: /etc/filebeat.yml
          readOnly: true
          subPath: filebeat.yml
        - name: inputs
          mountPath: /usr/share/filebeat/inputs.d
          readOnly: true
        - name: data
          mountPath: /usr/share/filebeat/data
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - mountPath: /etc/localtime
          name: localtime
          readOnly: true
      volumes:
      - name: config
        configMap:
          defaultMode: 0600
          name: filebeat-config
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: inputs
        configMap:
          defaultMode: 0600
          name: filebeat-inputs
      - hostPath:
          path: /etc/localtime
          type: ""
        name: localtime
      # We set an `emptyDir` here to ensure the manifest will deploy correctly.
      # It's recommended to change this to a `hostPath` folder, to ensure internal data
      # files survive pod changes (ie: version upgrade)
#      - name: data
#        emptyDir: {}
      - name: data
        hostPath:
          path: /data/site
```

