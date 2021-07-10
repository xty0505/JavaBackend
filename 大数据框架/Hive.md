# Hive

## 架构

- MetaStore：存储Hive表和HDFS中文件的映射关系
- Client：接受用户输入以及返回结果
  - CLI（hive shell）
  - JDBC（Java API）
- Driver：用于将 HQL 转换为 MR 程序

![Image](C:\Users\aasus\AppData\Local\Temp\Image.png)

## MySQL 与 Hive 的区别

- MySQL 是一个关系型数据库，提供了诸如事务等一系列特性；而 Hive 是一个数据仓库，提供了类 SQL 的查询语言 HQL，能够将查询逻辑转换为 MR 程序从而对底层数据文件进行处理，因此适用于读多写少的分析场景。
- MySQL 不同的存储引擎定义了自己的数据格式；Hive 没有固定的格式，用户可以指定，默认的文件格式有三种：TextFile、Sequence、RCFile。
- MySQL 在数据规模没有超过数据库处理能力时查询延迟较低；Hive 执行延迟高，由于其底层 MR 延迟较高的原因，适用于离线的大数据分析处理。
- MySQL 拥有索引，而Hive 没有索引。

## Hive 的表类型

- 内部表：内部表对应的数据文件存储在 HDFS 中 hive.metastore.warehouse.dir 指定的目录下，删除内部表对应数据文件也会删除。
- 外部表：外部表对应的数据文件存放在 HDFS 中的其他目录下，导入时不会移到指定目录下，同时删除表只会删除元数据而不会删除实际数据文件。
- 分区表：根据数据列将数据文件分为多个区，每个区在该表目录下会创建一个目录，该分区就存储在这个目录
- 分桶表：对指定列计算 hash，根据hash值切分数据，每个桶对应一个文件。

## Hive 数据存储格式

### TextFile

Hive默认格式，数据不做压缩，磁盘开销大，数据解析开销大。
可结合 Gzip、Bzip2、Snappy 等使用（系统自动检查，执行查询时自动解压），但使用这种方式，hive 不会对数据进行切分，从而无法对数据进行并行操作。

### Sequence

SequenceFile 是 Hadoop API 提供的一种二进制文件，将数据以 kv 形式序列化到文件中。

### RCFile

RCFILE 是一种行列存储相结合的存储方式。首先，其将数据按行分块，保证同一个 record 在一个块上，避免读一个记录需要读取多个 block。其次，块数据列式存储，有利于数据压缩和快速的列存取。

![image-20210312163858892](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210312163858892.png)

## Hive 导出数据

sqoop

