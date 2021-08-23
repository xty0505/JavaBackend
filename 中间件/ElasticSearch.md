## 简介

Elasticsearch（简称ES）是一个分布式、可扩展、实时的搜索与数据分析引擎。底层以来 Lucene，使用 Lucene 做索引与搜索，对外提供 RESTful API。大多数的场景下，查询服务的顺序可概括为：

![image-20210820164034161](../pic/image-20210820164034161.png)

对于模糊查询和全文查询等复杂的查询，Redis 不好创建 key，MySQL 也利用不到索引，因此先通过 ES 查询到主键 ID ，然后走缓存和数据库可以确保数据的一致性和查询速度。

- 与 Mysql 相比，ES对模糊查询有着更好的支持，并且利用缓存为反复出现的查询命令提供加速，提高查询速度。

- 与 Redis 相比，ES在查询方面提供复杂的组合查询语句，在存储方面不会将全部数据写入内存，可以支撑数据规模较大的场景。
- 但是 ES 再创建字段时要预先建立 Mapping，字段类型设置后便无法修改，**不够灵活**。同时写入数据时会自动**创建大量索引**，写入性能相对较低，并且会占用大量存储空间。



ES 的特点：

- 分布式的实时文档存储引擎，每个字段都可以被索引和搜索
- 分布式实时分析搜索引擎，支持各种查询和聚合操作
- 能扩展到上百个服务节点，支持 PB 级别的结构化和非结构化数据



## 基本概念

### 文档结构概念

- Index：可以理解为 MySQL 中的 database
- Mapping：表定义，json
- Documnet：表中的一个文档，MySQL 中一行数据
- Field：字段，MySQL 中的lie

```json
PUT /megacorp/employee/1
{
    "first_name" : "John",
    "last_name" :  "Smith",
    "age" :        25,
    "about" :      "I love to go rock climbing",
    "interests": [ "sports", "music" ]
}
```

### 架构概念

- Node：实例节点
  - Master：管理集群范围内的所有变更，如索引增加、删除，节点增加和删除。保存和更新集群的元数据信息。
  - Data
  - Coordinate
- Shard：ES 将数据存储在多个物理 Lucene 引擎上，这些数据就是分片
  - 主分片
  - 副本
- Segment：本质上就是分片中的一个倒排索引。每秒生成一个 segment，segment 过多时出发合并，并将 segment 标记删除的文档真正删除。
- File



## 架构

一个索引有多个分片，可以存储在多个节点上。

![image-20210820171332268](../pic/image-20210820171332268.png)

ES 中所有节点都是对等的。每个节点都能收到请求并路由到存储相关分片的其他节点上。接收请求的节点负责收集其他节点的查询结果最后返回 Client。

![image-20210820171520116](../pic/image-20210820171520116.png)



### 选举

选举策略为：如果有 Master 则认可该 Master，否则从候选 Master 中选取 id 最小的作为 Master。

选举的时机：

- 启动集群时，守护进程 ping 集群中的节点，使用上述策略选 Master
- Master 挂掉时，后台定时 ping Master，发现挂掉重新选举

选举流程：

- 节点 A 通过多播（默认）或单播 ping 全部节点，如果得到 response 报文内含有响应节点的基本信息和其选举的 Master。当半数以上节点认可一个 Master 时，向该节点发送 join 请求加入集群。当 A 收到 Master 发布的 cluster_state 时表明自己已经加入集群。
- 若不足半数则需要选举：
  1. A 将一直所有节点按照 clusterStateVersion 降序 UID 升序的顺序取第一个节点 B 为自己选举的 Master，向其发送 join 请求并等待一段时间
     - 若收到 B 发布的 cluster_state，则 B 为新 Master
     - 超市或 B 拒绝，开启下一轮选举。
  2. A 选自己当 Master， 等待其他节点 join 自己，当收到半数以上的请求后， 修改 cluster_state 中 master node 为自己，向集群发布消息
- 集群分裂成长时间无法联通的两部分，Master 选举会一直失败，此时 ES 集群挂掉。



容错机制：



