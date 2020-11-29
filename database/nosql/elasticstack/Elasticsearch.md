# Elasticsearch



## 配置文件

配置文件位于config目录中

- elasticsearch.yml es相关的配置
- jvm.options jvm相关参数
- log4j2.properties 日志配置相关

### elasticsearch.yml

cluster.name 集群名称，以此作为是否同一集群的判断条件

node.name 节点名称，以此作为集群中不同节点的区分条件

network.host/http.port 网络地址和端口，用于http 和 transport服务的使用

path.data 数据存储地址

path.log 日志存储地址

### 参数修改的第二种方式

通过启动参数

e.g.

```
bin/elasticsearch -Ehttp.port=19200
```

### 默认端口

localhost:9200



## Development 与 Production模式

模式判断条件：以transport 的地址是否绑定在localhost为条件

即判断 network.host 是否为本地localhost所对应的ip (e.g. 192.168.0.1)

- Development 模式下在启动时会以 warning 的方式提示配置检查异常。
- Production 模式下在启动时会以 error 的方式提示配置检查异常并退出。



## 本地快速启动集群

通过修改启动参数

- bin/elasticsearch
- bin/elasticsearch -Ehttp.port=8200 -Ehttp.data=node2
- bin/elasticsearch -Ehttp.port=7200 -Ehttp.data=node3

查看同集群下的节点情况：

```
127.0.0.1:8200/_cat/nodes?v
```

查看集群的情况：

```
127.0.0.1:8200/_cluster/stats
```



## 常用术语

- document 

  文档数据。文档是 ES 中记录数据的基本单位，是一系列数据字段的组合，本质上是一个 JSON 对象，类似于 MySQL 数据库中的一行数据，由各个不同数据类型的字段组成。包括常见的字符串类型、整型、日期等，也有一些 ElasticSearch 中特有的类型。

- index 索引

  类似于 mysql 中的 database 的概念。

- type

  索引中的数据类型（类似于 mysql 中的 table 的概念）。在 6.0 之后已经不再使用了，默认所有的 type 都为 _doc 即可，不用做太多的关注。

- field 

  文档的属性。

- document version

  表示文档被修改过几次。

- Query DSL 

  Elasticsearch 查询语法。



## Elasticsearch CRUD （杂乱）

- Create 创建文档（PUT）
- Read 读取文档（GET）
- Update 更新文档（POST /index/type/id/_update）
- Delete 删除文档（DELETE）

##### 【1】Index 的 CRUD

- 创建索引

```
PUT /test_index

# 请求返回内容，表示创建成功
{
  "acknowledged": true,
  "shards_acknowledged": true,
  "index": "test_index"
}
12345678
```

需要注意，ES 中创建索引使用的是 PUT 请求，而不是我们常用的 POST， 使用 POST 请求的话会报错，信息如下，可以看出其支持的方法为 GET、PUT、DELETE 和 HEAD

```
{
  "error": "Incorrect HTTP method for uri [/test_index] and method [POST], allowed: [GET, PUT, DELETE, HEAD]",
  "status": 405
}
1234
```

- 查询索引

```
# 查询 ES 实例中现有的所有索引
GET _cat/indices
# 查询新创建的索引
GET /test_index

# 查询 test_index 的基本信息
{
  "test_index": {
    "aliases": {},
    "mappings": {},
    "settings": {
      "index": {
        "creation_date": "1522159397979",
        "number_of_shards": "5",
        "number_of_replicas": "1",
        "uuid": "tcqyYJXlRQadel-sfqC6ng",
        "version": {
          "created": "6010299"
        },
        "provided_name": "test_index"
      }
    }
  }
}

123456789101112131415161718192021222324252627
```

- 删除索引

```
DELETE demo_index

# 返回内容
{
  "acknowledged": true
}
12345
```

##### 【2】Index 的 CRUD

- 指定 ID 创建文档，使用 PUT

其中的 *doc* 就是之前提到过的 type，在 6.0 之后已经废弃了，全部写成 *doc* 即可

```
PUT /test_index/_doc/1
{
  "username": "es",
  "age": 12
}
12345
```

- 不指定 ID ，使用 POST

此时会随机生成一个 ID

```
POST /test_index/_doc
{
  "username": "elk",
  "age": 12
}
12345
```

- 查询 文档

```
# 指定 ID 查询
GET /test_index/_doc/1
GET /test_index/_doc/_search
123
```

- 删除文档

```
DELETE /test_index/_doc/1
1
```

##### 【3】 批量操作

ES 提供了批量操作的 _bulk 和 _mget API ，允许我们批量的对文档进行 CRUD 的操作，具体使用如下:

首先指明 action 动作和要操作的文档，如果是创建或者修改，后面在跟对应的字段数据。操作的 action 有四种

- index: 创建文档，若存在，则报错
- create: 创建文档，若存在，仍然创建，覆盖已有数据
- delete: 删除文档
- update: 更新文档

```
POST _bulk
# 创建
{"index":{"_index":"test_index","_type":"doc","_id":"3"}}
{"username":"alfred","age":20}
# 创建
{"create":{"_index":"test_index","_type":"doc","_id":"3"}}
{"username":"alfred","age":20}
# 删除
{"delete":{"_index":"test_index","_type":"doc","_id":"1"}}
# 更新
{"update":{"_id":"3","_index":"test_index","_type":"doc"}}
{"doc":{"age":"20"}}


# 批量查询
GET /_mget
{
  "docs": [
    {
      "_index": "test_index",
      "_type": "doc",
      "_id": 1
    },  
    {
      "_index": "test_index",
      "_type": "doc",
      "_id": 3
    }
  ]
}
123456789101112131415161718192021222324252627282930
```

### 根据索引中存储的数据进行查询

query string

```
GET /account/_search?q=John
```

query DSL

```
GET /account/_doc/_search
{
  "query": {
    "term": {
      "name": {
        "value": "John"
      }
    }
  }
}
```



## Elasticsearch Ingest Node

新增的 node 类型

在数据写入 elasticsearch 前对数据进行转换

pipeline API



## 引用/参考

[【ELK笔记】-ES基本概念与 API 操作 - AhriJ邹同学 - CSDN](https://blog.csdn.net/Ahri_J/article/details/79720824)