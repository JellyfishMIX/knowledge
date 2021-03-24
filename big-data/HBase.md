# HBase



## HBase Description

1. HBase 是一个分布式的、面向列的开源数据库。
2. HBase 在 Hadoop 之上提供了类似于 Bigtable 的能力。
3. HBase 不同于一般的关系型数据库，它适合非结构化的数据存储。
4. 是 Apache 基金会顶级项目。
5. HBase 基于 Hadoop 的核心 HDFS 系统进行数据存储，类似于 Hive
6. HBase 可以存储超大数据并适合用来进行大数据的实时查询。
7. HBase 建立在 Hadoop 文件系统之上，利用了 Hadoop 文件系统的容错能力。
8. HBase 提供对数据的随机实时读/写访问功能。
9. HBase 内部使用哈希表，并存储索引，可将在 HDFS 中的数据进行快速查找。

### Bigtable

BigTable是一种压缩的、高性能的、高可扩展性的，基于Google文件系统（Google File System，GFS）的数据存储系统，用于存储大规模结构化数据，适用于云端计算。

### HDFS

Hadoop分布式文件系统(*HDFS*)是指被设计成适合运行在通用硬件(commodity hardware)上的分布式文件系统（Distributed File System）。

 

## HBase概念

NameSpace: 可以把 NameSpace 理解为 RDBMS 的"数据库"。

Table: 表名必须是能用在文件路径中的合法文字。

Row: 在表里面，每一行代表一个数据对象，每一行都是以一个行键（Row Key）来进行唯一标识的。