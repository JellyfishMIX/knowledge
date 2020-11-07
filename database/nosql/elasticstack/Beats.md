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

### Filebeat output 配置

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
  # 把 filebeat 的输出，格式化为 json 后输出
  pretty: true
```

### Filebeat filter 配置

input 时处理

- include_lines

  达到某些条件时，读入此行。

- exclude_lines

  达到某些条件时，不读入此行。

- exclude_files

  当文件名符合某些条件时，不读入此文件。

output 前处理 -- Processor

- drop_event

  读取到此条数据，符合某个条件，不输出。

- drop_fields

  读取到此条字段，符合某个条件，不输出。

- Decode_json_fields

  对此条数据中，符合 json 格式的字段做一个解析。

- Include_fields

  加入一些字段。（只取数据中的某些字段）

```yaml
processors:
  - drop_event:
    when:
      regexp:
        message: "^DBG:"

# decode_json_fields
# {"outer": "value", "inner": "{\"data\":\"value\"}"}
# {"outer": "value", "inner": "{"data":"value"}"}
processors:
  - decode_json_fields:
    fields: ["inner"]
```

### 数据处理

filebeat 本身缺乏对数据的转换能力，可使用 elasticsearch ingest node 在数据写入 elasticsearch 前，对数据进行转换。

### Filebeat Module

- 对社区常见需求进行配置封装增加易用性
  - nginx
  - apache
  - mysql

- 封装内容
  - filebeat.yml 配置
  - ingest node pipeline 配置
  - kibana dashboard

- 最佳实践参考，参考官方的模版进行定制化配置。

### 文件目录

- filebeat 可执行文件
- data 存储 filebeat 解析日志过程中的，存储日志中读取到的位置
- filebeat.yml filebeat配置
- module filebeat module功能

### 日志收集

```bash
head -n 2 ~/Downloads/nginx_logs/nginx.log | ./filebeat -e -c nginx.yml
```



## Packetbeat

实时抓取网络包

自动解析应用层协议

- ICMP (v4 and v6)
- DNS
- HTTP
- Mysql
- Redis
- ...

类似于Wireshark

### 解析http协议

```
# 指定监听的网卡
packetbeat.interfaces.device:lo0
# 指定监听的端口号
packetbeat.protocols.http: ports: [200]
# 是否记录完整的http body
send_request: true
# 记录哪些格式的 http body
include_body_for: ["application/json", "x-www-form-urlencoded"]
output.console:
  # 把 packetbeat 的输出，格式化为 json 后输出
  pretty: true
```

### Packetbeat 运行

```bash
sudo ./packetbeat -e -c es.yml -strict.perms=false
```

sudo: 在网络层抓包需要root权限

-strict.perms=false: packetbeat在运行时会检查 es.yml 配置文件的权限，使用 false 取消权限检查