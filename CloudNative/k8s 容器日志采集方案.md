## k8s 容器日志采集方案

### filebeat

简单的采集配置

```yaml
data:
  filebeat.yml: |-
    filebeat.inputs:
    - type: container
      paths:
        - /var/log/containers/*.log
      fields:
        source_name: "dockerlog"
        node: ${NODE_NAME}
      fields_under_root: true
      processors:
        - add_labels:
            labels:
              cluster: ${CLUSTER_NAME}
        - add_kubernetes_metadata:
            in_cluster: true
            host: ${NODE_NAME}
            matchers:
            - logs_path:
                logs_path: "/var/log/containers/"
    output.logstash:
      enabled: true
      hosts: ['${LOGSTASH_HOST:cloud-logstash}:${LOGSTASH_PORT:15050}']
    logging.level: info
    logging.selectors: ["*"]
    logging.to_files: true
    logging.files:
      path: /opt/log/filebeat
      name: filebeat
      rotateeverybytes: 10485760 # = 10MB


```

