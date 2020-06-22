### 启动一个consul单点

```bash
consul agent -server -dev -ui -data-dir /data/consul -client=10.21.8.113 -bind=10.21.8.113 -config-dir /etc/consul.d -enable-script-checks=true
```



### consul中创建service

```bash
# 写入配置文件service
echo '{"service": {"name": "consul_test", "tags": ["exporter"], "port": 9100}}' \
    | sudo tee /etc/consul.d/consul_test.json

# 通过接口创建service
curl -X PUT -d '{"id": "node-10.80.137.155","name": "consul_test","address": "10.80.137.155","port": 9100,"tags": ["exporter"],"checks": [{"http": "http://10.80.137.155:9100/metrics","interval": "35s"}]}' http://10.21.8.113:8500/v1/agent/service/register
```



### prometheus中增加consul配置

```json
- job_name: consul_collect
  metrics_path: /metrics
  scheme: http
  consul_sd_configs:
    - server: 192.168.1.1:8500
      services:
        - test-monitor-loda   
```

