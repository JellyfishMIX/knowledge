# Elasticsearch CRUD



## 查询文档 API

搜索所有文档，使用 _search，如下：

```
GET /test_index/doc/_search

or

GET /test_index/doc/_search
{
	"query": {
		"term": {
			"_id": "1"
		}
	}
}
```

![image-20201206095103886](https://image-hosting.jellyfishmix.com/20201206095103.png)



## 批量创建 document API

es 允许一次创建多个文档，从而减少网络传输开销，提升写入速率。

- endpoint 为 _bulk，如下：

  ![image-20201206095609316](https://image-hosting.jellyfishmix.com/20201206095609.png)

## 批量查询 document API

es 允许一次查询多个文档

- endpoint 为 _mget，如下：

  ![image-20201206100230554](https://image-hosting.jellyfishmix.com/20201206100230.png)



## Analyze API

elasticsearch 提供了一个测试分词的 API 接口，方便验证分词效果，endpoint 是 _analyze

- 可以直接指定 analyzer 进行测试。

- 可以直接指定索引中的字段进行测试。

  ![image-20201211105608212](https://image-hosting.jellyfishmix.com/20201211105608.png)

- 可以自定义分词器进行测试。

  ![image-20201211115647530](https://image-hosting.jellyfishmix.com/20201211115647.png)

#### 自带分词器

es 自带分词器如下：

- Standard
- Simple
- Whitespace
- Stop
- Keyword
- Pattern
- Language

#### Standard Analyzer

- 默认分词器
- 其组成如图，特性为：
  - 按词切分，支持多语言
  - 小写处理

![image-20201211121933006](https://image-hosting.jellyfishmix.com/20201211122002.png)

