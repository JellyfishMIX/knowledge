# Beats



## 简介

Lightweight data shipper

收集数据用。

- filebeat 日志文件

- metricbeat 度量数据

  例如操作系统的 cpu 数据，内存数据，磁盘数据，常用的软件的指标等。

- Packetbeat 网络数据

  抓包分析网络传输情况。

- Winlogbeat Windows 数据

- Heartbeat 健康检查

![image-20201106225743013](https://image-hosting.jellyfishmix.com/20201106225743.png)



## Filebeat

![image-20201106230842870](https://image-hosting.jellyfishmix.com/20201106230842.png)

Prospector:

跟踪检查制定的文件是否更新，有更新则交给 Harvester 收集。

一个 prospector 可以跟踪检查多个指定匹配模式的文件，一个文件对应一个 harvester 实例来收集。

### Filebeat 配置

```yml
filebeat.prospectors:
  - input_type: log
    paths: 
      - /var/log/apache/httpd-*.log
  - input_type: log
    paths:
      - /var/logs/messages
      - /var/log/*.log
```

input_type 目前有两种可选类型：

- log
- stdin、

### Filebeat output 配置简介

- console
- elasticsearch
- logstash
- kafka
- redis
- file

```yaml
output.elasticsearch:
  hosts: ["http://localhost:9200"]
  username: "admin"
  password: "s3cr3t"

output.console:
  # 把 filebeat 的输出，格式化为json后输出
  pretty: true
```

