# RabbitMQ



## MQ应用场景

- 异步处理

- 流量削峰

- 日志处理

- 应用解偶



## 安装

使用docker安装

```dockerfile
docker run -d --hostname my-rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:3.8.5-management
```

- 5672

  rabbitmq server的端口

- 15672

  rabbitmq 管理界面的端口



## 客户端界面(guest)

默认用户名和密码

```
username = guest;
password = guest;
```

