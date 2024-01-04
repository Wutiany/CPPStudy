# MongoDB 学习

# 1 了解篇

## 1.1 MongoDB 特性

* 面向文档存储：基于 JSON/BSON
* 动态 DDL 能力，没有 schema 约束
* 高性能计算，提供内存快速数据查询
* 容易扩展，利用数据分片可以支持海量数据存储（分布式）
* 丰富的功能集（二级索引，聚合等功能）
* 跨平台

## 1.2 基本模型

**与 sql 类似，有对应的概念**

| SQL 概念 | MongoDB 概念 |
| -------- | ------------ |
| database | database     |
| table    | collection   |
| row      | document     |
| column   | field        |

* database 数据库：一个数据库包含**多个集和**
* collection 集和：相当于**表**，存储的是**文档**，每个集和有**多个文档**（行数据）
* document 文档：相当于**行**，灵活的 JSON/BSON 格式的内容，支持嵌套的文档、数组
* field 字段：相当于**列**，字段更加灵活

| SQL 概念    | MongoDB 概念 |
| ----------- | ------------ |
| primary key | _id          |
| foreign key | reference    |
| view        | view         |
| index       | index        |
| join        | $lookup      |
| transaction | transaction  |
| aggregation | aggregation  |

* _id 主键：保证文档的唯一性（应该由客户端生成，服务端也就检测，如果客户端不生成，服务端也会自动生成）
* reference 引用：相当于外键，但是没有约束
* view 视图：与 SQL 的数据基本相似，也只是物化视图
* index 索引：与 SQL 索引相同
* $lookup：聚合操作符，类似于 join 的功能
* transaction 事务：最初支持单文档事务，现在支持单集和多文档
* aggregation 聚合：聚合操作

## 1.3 BSON 数据类型

* 基于 JSON，当时进行了扩展，支持日期等特定数据类型

| Type                    | Number | Alias               |
| ----------------------- | ------ | ------------------- |
| 32-bit integer          | 16     | 整数                |
| 64-bit integer          | 18     | 长整数              |
| Double                  | 1      | 浮点数              |
| Boolean                 | 8      | 布尔值              |
| String                  | 2      | 字符串              |
| Object                  | 3      | 对象                |
| Array                   | 4      | 数组                |
| Date                    | 9      | 日期                |
| Binary data             | 5      | 二进制数据          |
| Objectld                | 7      | 文档ID              |
| Null                    | 10     | 空值                |
| TimestampDecimal128     | 17     | 时间戳              |
| Decimal128              | 19     | 高精度浮点数        |
| Min key                 | -1     | 最小值              |
| Max key                 | 127    | 最大值              |
| Regular Expression      | 11     | 正则表达式          |
| JavaScript              | 13     | Javascript 函数     |
| JavaScript (with scope) | 15     | javascriptWithScope |

## 1.4 分布式 ID

**使用时间戳、机器号、进程号以及随机数保证唯一性**

Object-ID

* 4 byte Unix 时间戳
* 3 byte 机器ID
* 2 byte 进程ID
* 3 byte 计数器（随机初始化）

`_id` 由**客户端生**成，可以降低服务端的负载，客户端不生成，服务端也会生成

## 1.5 索引

**索引也是基于 B+ 树**

**索引特性**

* unique：表示索引的唯一性
* expireAfterSeconds：设置 TTL
* sparse：稀疏索引，仅索引非空字段文档
* partialFilterExpression：条件是索引，满足条件的文档才进行索引

**索引分类**

* 哈希索引：分片键会使用 Hash 索引
* 地理空间索引：地理空间查询，距离某地多少公里的内容
* 文本索引：全文快速索引
* 模糊索引：基于匹配规则的灵活索引

**索引评估、调优**

* 与SQL类似，使用 explain 查看计划树

## 1.6 集群部署

* 水平扩展
* 副本集备份

**架构**

* 数据分片：存储分片数据的是 chunk，一个分片（shared）有一个或多个 chunk（GFS）
  * 范围分片
  * 哈希分片
* 配置服务器：管理元数据以及分片的路由规则
* 查询路由，通过配置服务器去查找数据指定的分片

**负载均衡**

* 全预分配：提前分配好 shared 中 chunk 的数量
* 非预分配：。集群均衡器检测 chun 的状态，进行数据迁移

**应用高可用**

通过连接多个 Mongos 实现高可用

**副本集**

基于 raft，主从备份，log 为 oplog

# 2 安装篇











# 3 应用篇



