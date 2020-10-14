# Tomcat command



## Tomcat启动/停止

### 启动

```bash
cd ./bin
./startup.sh
```

.为tomcat根目录

### 停止

```bash
cd ./bin
./shutdown.sh
```

.为tomcat根目录



## 查看Tomcat是否正在运行

1. 方式一：

   ```bash
   ps -ef | grep java
   ```

2. 方式二：

   ```bash
   lsof -i:8080
   ```

   8080为tomcat开放的端口，查看端口是否正在被监听。