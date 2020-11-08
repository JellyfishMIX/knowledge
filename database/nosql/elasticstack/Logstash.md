# Logstash



## 简介

Logstash is an open source, server-side data processing pipeline that ingests data from a multitude of sources simultaneously, transforms it, and then sends it to your favorite "stash".



## 处理流程

### Input

- file
- redis
- beats
- kafka

### Filter

- grok
  - 基于正则 表达式提供了丰富可重用的模式（pattern）。
  - 基于此可以将非结构化数据做结构化处理。
- mutate
  - 对结构化后的数据进行增删改查。

- drop
- date
  - 将字符串类型的时间字段转化为时间戳类型，方便后续数据处理。

### Output

- stdout
- elasticsearch
- redis
- kafka

### e.g.

filter 配置 Grok 示例

```
55.3.244.1 GET /index.html 15824 0.043

%{IP:client} %{WORD:method} %{URIPATHPARAM:request} %{NUMBER:bytes} %{NUMBER:duration}

{
  "client": "55.3.244.1",
  "method": "GET",
  "request": "index.html",
  "bytes": 15824,
  "duration": 0.043
}
```



## Input 和 Output 配置

input { file { path => "/tmp/abc.log" } }

output { stdout { codec => rubydebug } }



## 默认端口

localhost:5044



## 使用

```bash
head -n 2 nginx_logs | ./logstash -f nginx_logstash.conf
```

