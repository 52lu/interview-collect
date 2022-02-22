## 1. mongodb有哪些特点？

- 面向文档的存储：以 JSON 格式的文档保存数据。
- 任何属性都可以建立索引。
- 复制以及高可扩展性。
- 自动分片。
- 丰富的查询功能。
- 快速的即时更新。
- 来自 MongoDB 的专业支持。

## 2. `MongoDB` 支持哪些数据类型?

### 2.1 常见数据类型 

| 类型       | 解析                                                         |
| ---------- | ------------------------------------------------------------ |
| `String`   | 字符串。存储数据常用的数据类型。在 `MongoDB` 中，`UTF-8` 编码的字符串才是合法的 |
| `Integer`  | 整型数值。用于存储数值。根据你所采用的服务器，可分为 32 位或 64 位 |
| `Double`   | 双精度浮点值。用于存储浮点值                                 |
| `Boolean`  | 布尔值。用于存储布尔值（真/假）                              |
| `Arrays`   | 用于将数组或列表或多个值存储为一个键                         |
| `Datetime` | 记录文档修改或添加的具体时间                                 |

### 2.2 MongoDB特有数据类型

| 类型                 | 解析                                                         |
| -------------------- | ------------------------------------------------------------ |
| `ObjectId`           | 用于存储文档 `id`,`ObjectId`是基于分布式主键的实现`MongoDB`分片也可继续使用 |
| `Min/Max Keys`       | 将一个值与 BSON（二进制的 JSON）元素的最低值和最高值相对比   |
| `Code`               | 用于在文档中存储 `JavaScript`代码                            |
| `Regular Expression` | 用于在文档中存储正则表达式                                   |
| `Binary Data`        | 二进制数据。用于存储二进制数据                               |
| `Null`               | 用于创建空值                                                 |
| `Object`             | 用于内嵌文档                                                 |



## 3.什么是集合`Collection`、文档`Document`

- 集合`Collection`位于单独的一个数据库MongoDB 文档`Document`集合，它类似关系型数据库（MYSQL）中的表`Table`。一个集合`Collection`内的多个文档`Document`可以有多个不同的字段。通常情况下，集合`Collection`中的文档`Document`有着相同含义。

- 文档`Document`由key-value构成。文档`Document`是动态模式,这说明同一集合里的文档不需要有相同的字段和结构。类似于关系型数据库中table中的每一条记录。

**关系型数据库术语类比：**

| mongodb            | 关系型数据库 |
| ------------------ | ------------ |
| Database           | Database     |
| Collection         | Table        |
| Document           | Record/Row   |
| Filed              | Column       |
| Embedded Documents | Table join   |

## 4. `MySQL`和`mongodb`的区别

| 形式         | MongoDB                                                      | MySQL                             |
| ------------ | ------------------------------------------------------------ | --------------------------------- |
| 数据库模型   | 非关系型                                                     | 关系型                            |
| 查询语句     | 独特的MongoDB查询方式                                        | 传统SQL语句                       |
| 架构特点     | 副本集以及分片                                               | 常见单点、M-S、MHA、MMM等架构方式 |
| 数据处理方式 | 基于内存，将热数据存在物理内存中，从而达到高速读写           | 不同的引擎拥有自己的特点          |
| 使用场景     | 事件的记录，内容管理或者博客平台等数据大且非结构化数据的场景 | 适用于数据量少且很多结构化数据    |

## 5. 问`mongodb`和`redis`区别以及选择原因

### 5.1 区别

| 形式           | MongoDB                                                      | redis                                                        |
| -------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 内存管理机制   | MongoDB 数据存在内存，由 linux系统 mmap 实现，当内存不够时，只将热点数据放入内存，其他数据存在磁盘 | Redis 数据全部存在内存，定期写入磁盘，当内存不够时，可以选择指定的 LRU 算法删除数据 |
| 支持的数据结构 | MongoDB 数据结构比较单一，但是支持丰富的数据表达，索引       | Redis 支持的数据结构丰富，包括hash、set、list等              |
| 性能           | mongodb依赖内存，TPS较高                                     | Redis依赖内存，TPS非常高。性能上Redis优于MongoDB             |
| 可靠性         | 支持持久化以及复制集增加可靠性                               | Redis依赖快照进行持久化；AOF增强可靠性；增强可靠性的同时，影响访问性能 |
| 数据分析       | mongodb内置数据分析功能（mapreduce）                         | Redis不支持                                                  |
| 事务支持情况   | 只支持单文档事务，需要复杂事务支持的场景暂时不适合           | Redis 事务支持比较弱，只能保证事务中的每个操作连续执行       |
| 集群           | MongoDB 集群技术比较成熟                                     | Redis从3.0开始支持集群                                       |

### 5.2 选择Mongo的原因

- 架构简单
- 没有复杂的连接
- 深度查询能力,`MongoDB`支持动态查询。
- 容易调试
- 容易扩展
- 不需要转化/映射应用对象到数据库对象
- 使用内部内存作为存储工作区,以便更快的存取数据。

## 6. 如何执行事务/加锁

`mongodb`没有使用传统的锁或者复杂的带回滚的事务,因为它设计的宗旨是轻量、快速以及可预计的高性能。

可以把它类比成`mysql mylsam`的自动提交模式.通过精简对事务的支持,性能得到了提升,特别是在一个可能会穿过多个服务器的系统里.

## 7. 更新操作会立刻fsync到磁盘?

不会，磁盘写操作默认是延迟执行的。写操作可能在两三秒(默认在60秒内)后到达磁盘。例如，如果一秒内数据库收到一千个对一个对象递增的操作，仅刷新磁盘一次。(注意，尽管fsync选项在命令行和经过getLastError_old是有效的)

## 8. MongoDB索引

### 8.1 索引类型有哪些？

- 单字段索引(`Single Field Indexes`)

- 复合索引(`Compound Indexes`)

- 多键索引(`Multikey Indexes`)

- 全文索引(`text Indexes`)

- Hash 索引(`Hash Indexes`)

- 通配符索引(`Wildcard Index`)

- 2dsphere索引(`2dsphere Indexes`)

### 8.2 创建索引语法

```sh
# 语法
db.collection.createIndex(keys, options)
# 单字段索引 - 示例
db.col.createIndex({"title":1})
# 联合索引 - 示例
db.col.createIndex({"title":1,"description":-1})
# 通配符索引 - 示例
db.col.createIndex( { "userMetadata.$**" : 1 } )
# 支持查询方式
# db.userData.find({ "userMetadata.likes" : "dogs" })
# db.userData.find({ "userMetadata.dislikes" : "pickles" })
###
```

> **索引值,1 : 指定按升序创建索引，-1: 按降序来创建索引**

**可选参数(options)列表如下：**

| Parameter          | Type          | Description                                                  |
| :----------------- | :------------ | :----------------------------------------------------------- |
| background         | Boolean       | 建索引过程会阻塞其它数据库操作，background可指定以后台方式创建索引，即增加 "background" 可选参数。 "background" 默认值为**false**。 |
| unique             | Boolean       | 建立的索引是否唯一。指定为true创建唯一索引。默认值为**false**. |
| name               | string        | 索引的名称。如果未指定，MongoDB的通过连接索引的字段名和排序顺序生成一个索引名称。 |
| dropDups           | Boolean       | **3.0+版本已废弃。**在建立唯一索引时是否删除重复记录,指定 true 创建唯一索引。默认值为 **false**. |
| sparse             | Boolean       | 对文档中不存在的字段数据不启用索引；这个参数需要特别注意，如果设置为true的话，在索引字段中不会查询出不包含对应字段的文档.。默认值为 **false**. |
| expireAfterSeconds | integer       | 指定一个以秒为单位的数值，完成 TTL设定，设定集合的生存时间。 |
| v                  | index version | 索引的版本号。默认的索引版本取决于mongod创建索引时运行的版本。 |
| weights            | document      | 索引权重值，数值在 1 到 99,999 之间，表示该索引相对于其他索引字段的得分权重。 |
| default_language   | string        | 对于文本索引，该参数决定了停用词及词干和词器的规则的列表。 默认为英语 |
| language_override  | string        | 对于文本索引，该参数指定了包含在文档中的字段名，语言覆盖默认的language，默认值为 language. |

### 8.3 索引操作

**1. 查看集合索引**

```
db.col.getIndexes()
```

**2. 查看集合索引大小**

```
db.col.totalIndexSize()
```

**3. 删除集合所有索引**

```
db.col.dropIndexes()
```

**4. 删除集合指定索引**

```
db.col.dropIndex("索引名称")
```

### 8.4 `MongoDB`在A:{B,C}上建立索引，查询A:{B,C}和A:{C,B}都会使用索引吗？

 由于`MongoDB`索引使用`B-tree`树原理，只会在A:{B,C}上使用索引



## 9. MongoDB分片

### 9.1 **`monogodb` 中的分片`sharding`**

分片`sharding`是将数据水平切分到不同的物理节点。当应用数据越来越大的时候，数据量也会越来越大。当数据量增长时，单台机器有可能无法存储数据或可接受的读取写入吞吐量。利用分片技术可以添加更多的机器来应对数据量增加 以及读写操作的要求。

### 9.2 分片(`Shard`)和复制(`replication`)是怎样工作的?

每一个分片(`shard`)是一个分区数据的逻辑集合。分片可能由单一服务器或者集群组成，我们推荐为每一个分片(`shard`)使用集群。

## 10. MongoDB复制集

### 10.1 **`MongoDB`副本集实现高可用的原理**

 `MongoDB` 使用了其复制(`Replica Set`)方案，实现自动容错机制为高可用提供了基础。目前，`MongoDB` 支持两种复制模式：

- `Master` / `Slave` ，主从复制，角色包括 `Master` 和 `Slave` 。
- `Replica Set` ，复制集复制，角色包括 `Primary` 和 `Secondary` 以及 `Arbiter` 。(**生产环境必选**)