
- [MergeTree](#mergetree)
  - [主要特点](#主要特点)
- [建表](#建表)
# MergeTree
## 主要特点
**1.** 存储的数据按主键排序。  
**2.** 如果指定了 分区键 的话，可以使用分区。  
在相同数据集和相同结果集的情况下 ClickHouse 中某些带分区的操作会比普通操作更快。查询中指定了分区键时 ClickHouse 会自动截取分区数据。这也有效增加了查询性能。  
**3.** 支持数据副本。  
ReplicatedMergeTree 系列的表提供了数据副本功能。更多信息，请参阅 数据副本 一节。  
**4.** 支持数据采样。

# 建表
```sql
CREATE TABLE [IF NOT EXISTS] [db.]table_name [ON CLUSTER cluster]
(
    name1 [type1] [DEFAULT|MATERIALIZED|ALIAS expr1] [TTL expr1],
    name2 [type2] [DEFAULT|MATERIALIZED|ALIAS expr2] [TTL expr2],
    ...
    INDEX index_name1 expr1 TYPE type1(...) GRANULARITY value1,
    INDEX index_name2 expr2 TYPE type2(...) GRANULARITY value2
) ENGINE = MergeTree()
ORDER BY expr
[PARTITION BY expr]
[PRIMARY KEY expr]
[SAMPLE BY expr]
[TTL expr [DELETE|TO DISK 'xxx'|TO VOLUME 'xxx'], ...]
[SETTINGS name=value, ...]
```