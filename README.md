
- [MergeTree](#mergetree)
  - [主要特点](#主要特点)
- [建表](#建表)
- [案例分析](#案例分析)
  - [建表](#建表-1)
  - [目录](#目录)
    - [在宿主机通过ls查看相关目录](#在宿主机通过ls查看相关目录)
    - [在docker中查看数据的相关目录](#在docker中查看数据的相关目录)
    - [通过system.tables表查看相关信息](#通过systemtables表查看相关信息)
    - [通过zookeeper查看任务相关信息](#通过zookeeper查看任务相关信息)
    - [通过system.parts查看相关信息](#通过systemparts查看相关信息)
    - [添加索引](#添加索引)
    - [分析分区目录](#分析分区目录)
    - [分析数据文件](#分析数据文件)
      - [查看checksums.txt](#查看checksumstxt)
      - [partition.dat](#partitiondat)
      - [columns.txt](#columnstxt)
      - [count.txt](#counttxt)
      - [default_compression_codec.txt](#default_compression_codectxt)
      - [primary.idx](#primaryidx)
        - [结论](#结论)
      - [{column}.mrk2](#columnmrk2)
      - [{column}.bin](#columnbin)
      - [minmax_LOORDERDATE.idx](#minmax_loorderdateidx)
      - [skp_idx_idx_1.idx 需要补充](#skp_idx_idx_1idx-需要补充)
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
# 案例分析
## 建表
```sql
CREATE TABLE lineorder on cluster test_cluster_three_shards
(
    LOORDERKEY             UInt32,
    LOLINENUMBER           UInt8,
    LOCUSTKEY              UInt32,
    LOPARTKEY              UInt32,
    LOSUPPKEY              UInt32,
    LOORDERDATE            Date,
    LOORDERPRIORITY        LowCardinality(String),
    LOSHIPPRIORITY         UInt8,
    LOQUANTITY             UInt8,
    LOEXTENDEDPRICE        UInt32,
    LOORDTOTALPRICE        UInt32,
    LODISCOUNT             UInt8,
    LOREVENUE              UInt32,
    LOSUPPLYCOST           UInt32,
    LOTAX                  UInt8,
    LOCOMMITDATE           Date,
    LOSHIPMODE             LowCardinality(String)
)
ENGINE = MergeTree PARTITION BY toYear(LOORDERDATE) ORDER BY (LOORDERDATE, LOORDERKEY);

CREATE TABLE ssb.lineorder_all  on cluster test_cluster_three_shards AS ssb.lineorder 
ENGINE = Distributed(test_cluster_three_shards, ssb, lineorder, rand());
```
## 目录
### 在宿主机通过ls查看相关目录
```sh
#程序、数据目录
ll /var/lib/clickhouse/
drwxr-x--- 17 clickhouse clickhouse 4096 Sep 25 05:22 ./
drwxr-xr-x  1 root       root       4096 Jan 23  2022 ../
drwxr-x---  2 clickhouse clickhouse 4096 Sep 25 03:53 access/
drwxr-x---  6 clickhouse clickhouse 4096 Sep 25 05:24 data/
drwxr-x---  2 clickhouse clickhouse 4096 Sep 25 03:49 default/
drwxr-x---  2 clickhouse clickhouse 4096 Sep 25 03:53 dictionaries_lib/
drwxr-x---  2 clickhouse clickhouse 4096 Sep 25 03:53 flags/
drwxr-xr-x  2 clickhouse clickhouse 4096 Sep 25 03:53 format_schemas/
drwxr-x---  4 clickhouse clickhouse 4096 Sep 25 05:24 metadata/
drwxr-x---  2 clickhouse clickhouse 4096 Sep 25 06:48 metadata_dropped/
drwxr-x---  2 clickhouse clickhouse 4096 Sep 25 03:53 preprocessed_configs/
-rw-r-----  1 clickhouse clickhouse   55 Sep 25 05:22 status
drwxr-x--- 34 clickhouse clickhouse 4096 Sep 25 06:40 store/
drwxr-x---  2 clickhouse clickhouse 4096 Sep 25 03:49 system/
drwxr-xr-x  2 clickhouse clickhouse 4096 Oct  4 00:16 tmp/
drwxr-x---  2 clickhouse clickhouse 4096 Sep 25 03:53 user_defined/
drwxr-xr-x  2 clickhouse clickhouse 4096 Sep 25 03:53 user_files/
drwxr-x---  2 clickhouse clickhouse 4096 Sep 25 03:53 user_scripts/
-rw-r-----  1 clickhouse clickhouse   36 Sep 25 03:53 uuid
#配置
ll /etc/clickhouse-server/*
-rw-rw-rw- 1 root root 56860 Sep 25 05:20 /etc/clickhouse-server/config.xml
-rw-rw-rw- 1 root root  6248 Jan 22  2022 /etc/clickhouse-server/users.xml
/etc/clickhouse-server/config.d:
-rw-rw-r-- 1 root root  314 Jan 22  2022 docker_related_config.xml
/etc/clickhouse-server/users.d:
#日志
ll /var/log/clickhouse-server/
-rw-r----- 1 clickhouse clickhouse    147792 Sep 27 03:05 clickhouse-server.err.log
-rw-r----- 1 clickhouse clickhouse 707454923 Oct  4 02:34 clickhouse-server.log
```
### 在docker中查看数据的相关目录
```sh
root@aa3b704485e4:/var/lib/clickhouse/store/898/898b7dba-2be4-4419-ab62-92c40909dfe3/1993_2_36_1# ll /var/lib/clickhouse/data/ssb/
total 24
drwxr-x--- 2 clickhouse clickhouse 4096 Oct  4 06:54 ./
drwxr-x--- 6 clickhouse clickhouse 4096 Oct  4 05:41 ../
lrwxrwxrwx 1 clickhouse clickhouse   67 Oct  4 06:44 customer -> /var/lib/clickhouse/store/784/784ed9b0-ad56-4c4a-a9fc-8e2b3d15044b//
lrwxrwxrwx 1 clickhouse clickhouse   67 Oct  4 06:44 customer_all -> /var/lib/clickhouse/store/4fa/4fa25c80-deff-4436-b4d3-b98d8f2cc2d0//
lrwxrwxrwx 1 clickhouse clickhouse   67 Oct  4 06:54 lineorder -> /var/lib/clickhouse/store/898/898b7dba-2be4-4419-ab62-92c40909dfe3//
lrwxrwxrwx 1 clickhouse clickhouse   67 Oct  4 06:54 lineorder_all -> /var/lib/clickhouse/store/9b7/9b79ebfd-2634-496b-a059-68cce08b75e0//
root@aa3b704485e4:/var/lib/clickhouse/store/898/898b7dba-2be4-4419-ab62-92c40909dfe3/1993_2_36_1# ll /var/lib/clickhouse/data/ssb/lineorder/
total 44
drwxr-x--- 10 clickhouse clickhouse 4096 Oct  4 07:03 ./
drwxr-x---  3 clickhouse clickhouse 4096 Oct  4 06:54 ../
drwxr-x---  2 clickhouse clickhouse 4096 Oct  4 06:55 1992_3_39_1/
drwxr-x---  2 clickhouse clickhouse 4096 Oct  4 06:55 1993_2_36_1/
drwxr-x---  2 clickhouse clickhouse 4096 Oct  4 06:55 1994_5_41_1/
drwxr-x---  2 clickhouse clickhouse 4096 Oct  4 06:55 1995_4_38_1/
drwxr-x---  2 clickhouse clickhouse 4096 Oct  4 06:55 1996_1_37_1/
drwxr-x---  2 clickhouse clickhouse 4096 Oct  4 06:55 1997_7_40_1/
drwxr-x---  2 clickhouse clickhouse 4096 Oct  4 06:55 1998_6_42_1/
drwxr-x---  2 clickhouse clickhouse 4096 Oct  4 06:54 detached/
-rw-r-----  1 clickhouse clickhouse    1 Oct  4 06:54 format_version.txt
root@aa3b704485e4:/var/lib/clickhouse/store/898/898b7dba-2be4-4419-ab62-92c40909dfe3/1993_2_36_1# ll /var/lib/clickhouse/data/ssb/lineorder_all/
total 16
drwxr-x--- 4 clickhouse clickhouse 4096 Oct  4 06:55 ./
drwxr-x--- 3 clickhouse clickhouse 4096 Oct  4 06:54 ../
drwxr-x--- 3 clickhouse clickhouse 4096 Oct  4 06:55 shard2_replica1/
drwxr-x--- 3 clickhouse clickhouse 4096 Oct  4 06:55 shard3_replica1/
root@aa3b704485e4:/var/lib/clickhouse/store/898/898b7dba-2be4-4419-ab62-92c40909dfe3/1993_2_36_1# ll /var/lib/clickhouse/data/ssb/lineorder_all/shard2_replica1/
total 12
drwxr-x--- 3 clickhouse clickhouse 4096 Oct  4 06:55 ./
drwxr-x--- 4 clickhouse clickhouse 4096 Oct  4 06:55 ../
drwxr-x--- 2 clickhouse clickhouse 4096 Oct  4 06:55 tmp/
root@aa3b704485e4:/var/lib/clickhouse/store/898/898b7dba-2be4-4419-ab62-92c40909dfe3/1993_2_36_1# ll /var/lib/clickhouse/data/ssb/lineorder_all/shard2_replica1/tmp/
total 8
drwxr-x--- 2 clickhouse clickhouse 4096 Oct  4 06:55 ./
drwxr-x--- 3 clickhouse clickhouse 4096 Oct  4 06:55 ../
root@aa3b704485e4:/var/lib/clickhouse/store/898/898b7dba-2be4-4419-ab62-92c40909dfe3/1993_2_36_1# 
```
### 通过system.tables表查看相关信息
```sql
select * from system.tables where database='ssb' limit 1\G

SELECT *
FROM tables
WHERE database = 'ssb'
LIMIT 1

Query id: 1724f9ca-b3a7-46df-86d3-af4e353c695c

Row 1:
──────
database:                      ssb
name:                          customer
uuid:                          a6befd5a-2707-4860-871b-ca21d056ad52
engine:                        MergeTree
is_temporary:                  0
data_paths:                    ['/var/lib/clickhouse/store/a6b/a6befd5a-2707-4860-871b-ca21d056ad52/']
metadata_path:                 /var/lib/clickhouse/store/8c6/8c621997-4ae3-4ef3-a7f6-1c4f63db48c1/customer.sql
metadata_modification_time:    2022-09-25 13:26:30
dependencies_database:         []
dependencies_table:            []
create_table_query:            CREATE TABLE ssb.customer (`CCUSTKEY` UInt32, `CNAME` String, `CADDRESS` String, `CCITY` LowCardinality(String), `CNATION` LowCardinality(String), `CREGION` LowCardinality(String), `CPHONE` String, `CMKTSEGMENT` LowCardinality(String)) ENGINE = MergeTree ORDER BY CCUSTKEY SETTINGS index_granularity = 8192
engine_full:                   MergeTree ORDER BY CCUSTKEY SETTINGS index_granularity = 8192
as_select:                     
partition_key:                 
sorting_key:                   CCUSTKEY
primary_key:                   CCUSTKEY
sampling_key:                  
storage_policy:                default
total_rows:                    99817
total_bytes:                   4068748
lifetime_rows:                 ᴺᵁᴸᴸ
lifetime_bytes:                ᴺᵁᴸᴸ
comment:                       
has_own_data:                  1
loading_dependencies_database: []
loading_dependencies_table:    []
loading_dependent_database:    []
loading_dependent_table:       []

1 rows in set. Elapsed: 0.005 sec. 
```
### 通过zookeeper查看任务相关信息
```sh
docker exec -it zk01 bash
/apache-zookeeper-3.8.0-bin/bin/zkCli.sh

[zk: localhost:2181(CONNECTED) 2] ls /clickhouse
[task_queue]
[zk: localhost:2181(CONNECTED) 3] ls /clickhouse/task_queue
[ddl]
[zk: localhost:2181(CONNECTED) 4] ls /clickhouse/task_queue/ddl/query-0000000000/active
[]
[zk: localhost:2181(CONNECTED) 5] ls /clickhouse/task_queue/ddl/query-0000000000/finished
[10%2E206%2E13%2E5:9000, 10%2E206%2E13%2E6:9000, 10%2E206%2E13%2E7:9000]
[zk: localhost:2181(CONNECTED) 6] get /clickhouse/task_queue/ddl/query-0000000000/finished/10%2E206%2E13%2E5:9000
0

[zk: localhost:2181(CONNECTED) 7] get /clickhouse/task_queue/ddl/query-0000000000
version: 1
query: CREATE TABLE ssb.lineorder UUID \'e8d1b074-b9c6-48ff-9fb3-9845b46cf045\' ON CLUSTER test_cluster_three_shards (`LOORDERKEY` UInt32, `LOLINENUMBER` UInt8, `LOCUSTKEY` UInt32, `LOPARTKEY` UInt32, `LOSUPPKEY` UInt32, `LOORDERDATE` Date, `LOORDERPRIORITY` LowCardinality(String), `LOSHIPPRIORITY` UInt8, `LOQUANTITY` UInt8, `LOEXTENDEDPRICE` UInt32, `LOORDTOTALPRICE` UInt32, `LODISCOUNT` UInt8, `LOREVENUE` UInt32, `LOSUPPLYCOST` UInt32, `LOTAX` UInt8, `LOCOMMITDATE` Date, `LOSHIPMODE` LowCardinality(String)) ENGINE = MergeTree PARTITION BY toYear(LOORDERDATE) ORDER BY (LOORDERDATE, LOORDERKEY)
hosts: ['10%2E206%2E13%2E5:9000','10%2E206%2E13%2E6:9000','10%2E206%2E13%2E7:9000']
initiator: aa3b704485e4:9000

[zk: localhost:2181(CONNECTED) 8] 
```
### 通过system.parts查看相关信息
```sql
select * FROM system.parts where database='ssb' and table='lineorder' and partition='1993'\G

SELECT *
FROM system.parts
WHERE (database = 'ssb') AND (table = 'lineorder') AND (partition = '1993')

Query id: d002daa0-cfe8-4bb8-a37a-a866384b58d9

Row 1:
──────
partition:                             1993
name:                                  1993_2_159_2
uuid:                                  00000000-0000-0000-0000-000000000000
part_type:                             Wide
active:                                0
marks:                                 157
rows:                                  1275685
bytes_on_disk:                         42809766
data_compressed_bytes:                 42737219
data_uncompressed_bytes:               54859565
marks_bytes:                           71592
secondary_indices_compressed_bytes:    0
secondary_indices_uncompressed_bytes:  0
secondary_indices_marks_bytes:         0
modification_time:                     2022-09-25 13:31:58
remove_time:                           2022-10-04 11:38:38
refcount:                              1
min_date:                              1993-01-01
max_date:                              1993-12-31
min_time:                              1970-01-01 08:00:00
max_time:                              1970-01-01 08:00:00
partition_id:                          1993
min_block_number:                      2
max_block_number:                      159
level:                                 2
data_version:                          2
primary_key_bytes_in_memory:           942
primary_key_bytes_in_memory_allocated: 8192
is_frozen:                             0
database:                              ssb
table:                                 lineorder
engine:                                MergeTree
disk_name:                             default
path:                                  /var/lib/clickhouse/store/af9/af98bc2c-75e2-4780-bd26-8898bcae0cab/1993_2_159_2/
hash_of_all_files:                     2fb4e43dac6e56398eb83ab6bd531993
hash_of_uncompressed_files:            ab2593f43a7f1a0af665431268a11b6b
uncompressed_hash_of_compressed_files: 85336dec6921a7bc3d1b97d23062c8fc
delete_ttl_info_min:                   1970-01-01 08:00:00
delete_ttl_info_max:                   1970-01-01 08:00:00
move_ttl_info.expression:              []
move_ttl_info.min:                     []
move_ttl_info.max:                     []
default_compression_codec:             LZ4
recompression_ttl_info.expression:     []
recompression_ttl_info.min:            []
recompression_ttl_info.max:            []
group_by_ttl_info.expression:          []
group_by_ttl_info.min:                 []
group_by_ttl_info.max:                 []
rows_where_ttl_info.expression:        []
rows_where_ttl_info.min:               []
rows_where_ttl_info.max:               []
projections:                           []

Row 2:
──────
partition:                             1993
name:                                  1993_2_382_3
uuid:                                  00000000-0000-0000-0000-000000000000
part_type:                             Wide
active:                                1
marks:                                 372
rows:                                  3037236
bytes_on_disk:                         101933278
data_compressed_bytes:                 101761401
data_uncompressed_bytes:               130613138
marks_bytes:                           169632
secondary_indices_compressed_bytes:    0
secondary_indices_uncompressed_bytes:  0
secondary_indices_marks_bytes:         0
modification_time:                     2022-10-04 11:38:38
remove_time:                           1970-01-01 08:00:00
refcount:                              1
min_date:                              1993-01-01
max_date:                              1993-12-31
min_time:                              1970-01-01 08:00:00
max_time:                              1970-01-01 08:00:00
partition_id:                          1993
min_block_number:                      2
max_block_number:                      382
level:                                 3
data_version:                          2
primary_key_bytes_in_memory:           2232
primary_key_bytes_in_memory_allocated: 8192
is_frozen:                             0
database:                              ssb
table:                                 lineorder
engine:                                MergeTree
disk_name:                             default
path:                                  /var/lib/clickhouse/store/af9/af98bc2c-75e2-4780-bd26-8898bcae0cab/1993_2_382_3/
hash_of_all_files:                     2dd90eb0f48c2cf3dad4c12d4bde93ae
hash_of_uncompressed_files:            70ea192e8d633434f266ac59cf6a4d66
uncompressed_hash_of_compressed_files: 06274722cc582972df1bba6859c787c4
delete_ttl_info_min:                   1970-01-01 08:00:00
delete_ttl_info_max:                   1970-01-01 08:00:00
move_ttl_info.expression:              []
move_ttl_info.min:                     []
move_ttl_info.max:                     []
default_compression_codec:             LZ4
recompression_ttl_info.expression:     []
recompression_ttl_info.min:            []
recompression_ttl_info.max:            []
group_by_ttl_info.expression:          []
group_by_ttl_info.min:                 []
group_by_ttl_info.max:                 []
rows_where_ttl_info.expression:        []
rows_where_ttl_info.min:               []
rows_where_ttl_info.max:               []
projections:                           []

Row 3:
──────
partition:                             1993
name:                                  1993_166_317_2
uuid:                                  00000000-0000-0000-0000-000000000000
part_type:                             Wide
active:                                0
marks:                                 156
rows:                                  1267921
bytes_on_disk:                         42558986
data_compressed_bytes:                 42486901
data_uncompressed_bytes:               54525681
marks_bytes:                           71136
secondary_indices_compressed_bytes:    0
secondary_indices_uncompressed_bytes:  0
secondary_indices_marks_bytes:         0
modification_time:                     2022-09-25 13:32:29
remove_time:                           2022-10-04 11:38:38
refcount:                              1
min_date:                              1993-01-01
max_date:                              1993-12-31
min_time:                              1970-01-01 08:00:00
max_time:                              1970-01-01 08:00:00
partition_id:                          1993
min_block_number:                      166
max_block_number:                      317
level:                                 2
data_version:                          166
primary_key_bytes_in_memory:           936
primary_key_bytes_in_memory_allocated: 8192
is_frozen:                             0
database:                              ssb
table:                                 lineorder
engine:                                MergeTree
disk_name:                             default
path:                                  /var/lib/clickhouse/store/af9/af98bc2c-75e2-4780-bd26-8898bcae0cab/1993_166_317_2/
hash_of_all_files:                     9d657cfe2ea8b48a430123c54e3fc823
hash_of_uncompressed_files:            7f2ce566baaaa9f1cfbd237d0c9a3cea
uncompressed_hash_of_compressed_files: fd4c1de0efc4116c5c232535b9b1bbb6
delete_ttl_info_min:                   1970-01-01 08:00:00
delete_ttl_info_max:                   1970-01-01 08:00:00
move_ttl_info.expression:              []
move_ttl_info.min:                     []
move_ttl_info.max:                     []
default_compression_codec:             LZ4
recompression_ttl_info.expression:     []
recompression_ttl_info.min:            []
recompression_ttl_info.max:            []
group_by_ttl_info.expression:          []
group_by_ttl_info.min:                 []
group_by_ttl_info.max:                 []
rows_where_ttl_info.expression:        []
rows_where_ttl_info.min:               []
rows_where_ttl_info.max:               []
projections:                           []

Row 4:
──────
partition:                             1993
name:                                  1993_328_359_1
uuid:                                  00000000-0000-0000-0000-000000000000
part_type:                             Wide
active:                                0
marks:                                 42
rows:                                  332232
bytes_on_disk:                         11151995
data_compressed_bytes:                 11132579
data_uncompressed_bytes:               14287406
marks_bytes:                           19152
secondary_indices_compressed_bytes:    0
secondary_indices_uncompressed_bytes:  0
secondary_indices_marks_bytes:         0
modification_time:                     2022-09-25 13:32:33
remove_time:                           2022-10-04 11:38:38
refcount:                              1
min_date:                              1993-01-01
max_date:                              1993-12-31
min_time:                              1970-01-01 08:00:00
max_time:                              1970-01-01 08:00:00
partition_id:                          1993
min_block_number:                      328
max_block_number:                      359
level:                                 1
data_version:                          328
primary_key_bytes_in_memory:           252
primary_key_bytes_in_memory_allocated: 8192
is_frozen:                             0
database:                              ssb
table:                                 lineorder
engine:                                MergeTree
disk_name:                             default
path:                                  /var/lib/clickhouse/store/af9/af98bc2c-75e2-4780-bd26-8898bcae0cab/1993_328_359_1/
hash_of_all_files:                     5f356097f75f2fbd95f745b69d23e75f
hash_of_uncompressed_files:            ab26dce0fc4e303cf03fdf843aa59544
uncompressed_hash_of_compressed_files: 2b8ffaf03afa257151a52403d62e0a46
delete_ttl_info_min:                   1970-01-01 08:00:00
delete_ttl_info_max:                   1970-01-01 08:00:00
move_ttl_info.expression:              []
move_ttl_info.min:                     []
move_ttl_info.max:                     []
default_compression_codec:             LZ4
recompression_ttl_info.expression:     []
recompression_ttl_info.min:            []
recompression_ttl_info.max:            []
group_by_ttl_info.expression:          []
group_by_ttl_info.min:                 []
group_by_ttl_info.max:                 []
rows_where_ttl_info.expression:        []
rows_where_ttl_info.min:               []
rows_where_ttl_info.max:               []
projections:                           []

Row 5:
──────
partition:                             1993
name:                                  1993_370_382_1
uuid:                                  00000000-0000-0000-0000-000000000000
part_type:                             Compact
active:                                0
marks:                                 21
rows:                                  161398
bytes_on_disk:                         5520774
data_compressed_bytes:                 5514756
data_uncompressed_bytes:               6943114
marks_bytes:                           5880
secondary_indices_compressed_bytes:    0
secondary_indices_uncompressed_bytes:  0
secondary_indices_marks_bytes:         0
modification_time:                     2022-09-28 12:15:06
remove_time:                           2022-10-04 11:38:38
refcount:                              1
min_date:                              1993-01-01
max_date:                              1993-12-31
min_time:                              1970-01-01 08:00:00
max_time:                              1970-01-01 08:00:00
partition_id:                          1993
min_block_number:                      370
max_block_number:                      382
level:                                 1
data_version:                          370
primary_key_bytes_in_memory:           126
primary_key_bytes_in_memory_allocated: 8192
is_frozen:                             0
database:                              ssb
table:                                 lineorder
engine:                                MergeTree
disk_name:                             default
path:                                  /var/lib/clickhouse/store/af9/af98bc2c-75e2-4780-bd26-8898bcae0cab/1993_370_382_1/
hash_of_all_files:                     734bf1af5ea7b11f0541345b5ad1b688
hash_of_uncompressed_files:            1cee1a8edc2d2cae7113d49839fe18dd
uncompressed_hash_of_compressed_files: 16e0122188eeb2f36653af5b13b27460
delete_ttl_info_min:                   1970-01-01 08:00:00
delete_ttl_info_max:                   1970-01-01 08:00:00
move_ttl_info.expression:              []
move_ttl_info.min:                     []
move_ttl_info.max:                     []
default_compression_codec:             LZ4
recompression_ttl_info.expression:     []
recompression_ttl_info.min:            []
recompression_ttl_info.max:            []
group_by_ttl_info.expression:          []
group_by_ttl_info.min:                 []
group_by_ttl_info.max:                 []
rows_where_ttl_info.expression:        []
rows_where_ttl_info.min:               []
rows_where_ttl_info.max:               []
projections:                           []

5 rows in set. Elapsed: 0.005 sec. 
```
或者简化结果
```sql
select partition,name,active,database,table,engine,path FROM system.parts where database='ssb' and table='lineorder' and partition='1993'

SELECT
    partition,
    name,
    active,
    database,
    table,
    engine,
    path
FROM system.parts
WHERE (database = 'ssb') AND (table = 'lineorder') AND (partition = '1993')

Query id: 96a80625-27dd-48b2-b541-7f06c70ff2d5

┌─partition─┬─name─────────┬─active─┬─database─┬─table─────┬─engine────┬─path─────────────────────────────────────────────────────────────────────────────┐
│ 1993      │ 1993_2_2_0   │      0 │ ssb      │ lineorder │ MergeTree │ /var/lib/clickhouse/store/898/898b7dba-2be4-4419-ab62-92c40909dfe3/1993_2_2_0/   │
│ 1993      │ 1993_2_36_1  │      1 │ ssb      │ lineorder │ MergeTree │ /var/lib/clickhouse/store/898/898b7dba-2be4-4419-ab62-92c40909dfe3/1993_2_36_1/  │
│ 1993      │ 1993_13_13_0 │      0 │ ssb      │ lineorder │ MergeTree │ /var/lib/clickhouse/store/898/898b7dba-2be4-4419-ab62-92c40909dfe3/1993_13_13_0/ │
│ 1993      │ 1993_17_17_0 │      0 │ ssb      │ lineorder │ MergeTree │ /var/lib/clickhouse/store/898/898b7dba-2be4-4419-ab62-92c40909dfe3/1993_17_17_0/ │
│ 1993      │ 1993_24_24_0 │      0 │ ssb      │ lineorder │ MergeTree │ /var/lib/clickhouse/store/898/898b7dba-2be4-4419-ab62-92c40909dfe3/1993_24_24_0/ │
│ 1993      │ 1993_31_31_0 │      0 │ ssb      │ lineorder │ MergeTree │ /var/lib/clickhouse/store/898/898b7dba-2be4-4419-ab62-92c40909dfe3/1993_31_31_0/ │
│ 1993      │ 1993_36_36_0 │      0 │ ssb      │ lineorder │ MergeTree │ /var/lib/clickhouse/store/898/898b7dba-2be4-4419-ab62-92c40909dfe3/1993_36_36_0/ │
└───────────┴──────────────┴────────┴──────────┴───────────┴───────────┴──────────────────────────────────────────────────────────────────────────────────┘

7 rows in set. Elapsed: 0.002 sec. 
```
### 添加索引
```sql
alter table ssb.lineorder on cluster test_cluster_three_shards add index idx_1 (LOORDTOTALPRICE) TYPE set(0) GRANULARITY 1
alter table ssb.lineorder on cluster test_cluster_three_shards drop index idx_1

alter table ssb.customer on cluster test_cluster_three_shards add index idx_1 (CNAME) TYPE ngrambf_v1(3, 256, 2, 0) GRANULARITY 1
alter table ssb.customer on cluster test_cluster_three_shards drop index idx_1
```
### 分析分区目录
```sh
#查看1993分区有哪些目录
/var/lib/clickhouse/store/898/898b7dba-2be4-4419-ab62-92c40909dfe3# ll | grep 1993
drwxr-x---  2 clickhouse clickhouse 4096 Oct  4 06:55 1993_13_13_0/
drwxr-x---  2 clickhouse clickhouse 4096 Oct  4 06:55 1993_17_17_0/
drwxr-x---  2 clickhouse clickhouse 4096 Oct  4 06:55 1993_2_2_0/
drwxr-x---  2 clickhouse clickhouse 4096 Oct  4 06:55 1993_2_36_1/
drwxr-x---  2 clickhouse clickhouse 4096 Oct  4 06:55 1993_24_24_0/
drwxr-x---  2 clickhouse clickhouse 4096 Oct  4 06:55 1993_31_31_0/
drwxr-x---  2 clickhouse clickhouse 4096 Oct  4 06:55 1993_36_36_0/


#查看每个1993分区目录有哪些文件
ll 1993*
1993_13_13_0:
total 1996
drwxr-x---  2 clickhouse clickhouse    4096 Oct  4 06:55 ./
drwxr-x--- 52 clickhouse clickhouse    4096 Oct  4 06:55 ../
-rw-r-----  1 clickhouse clickhouse     346 Oct  4 06:55 checksums.txt
-rw-r-----  1 clickhouse clickhouse     418 Oct  4 06:55 columns.txt
-rw-r-----  1 clickhouse clickhouse       5 Oct  4 06:55 count.txt
-rw-r-----  1 clickhouse clickhouse 1868041 Oct  4 06:55 data.bin
-rw-r-----  1 clickhouse clickhouse    2240 Oct  4 06:55 data.mrk3
-rw-r-----  1 clickhouse clickhouse      10 Oct  4 06:55 default_compression_codec.txt
-rw-r-----  1 clickhouse clickhouse       4 Oct  4 06:55 minmax_LOORDERDATE.idx
-rw-r-----  1 clickhouse clickhouse       2 Oct  4 06:55 partition.dat
-rw-r-----  1 clickhouse clickhouse      48 Oct  4 06:55 primary.idx
-rw-r-----  1 clickhouse clickhouse  124533 Oct  4 06:55 skp_idx_idx_1.idx
-rw-r-----  1 clickhouse clickhouse     168 Oct  4 06:55 skp_idx_idx_1.mrk3

1993_17_17_0:
total 2004
drwxr-x---  2 clickhouse clickhouse    4096 Oct  4 06:55 ./
drwxr-x--- 52 clickhouse clickhouse    4096 Oct  4 06:55 ../
-rw-r-----  1 clickhouse clickhouse     346 Oct  4 06:55 checksums.txt
-rw-r-----  1 clickhouse clickhouse     418 Oct  4 06:55 columns.txt
-rw-r-----  1 clickhouse clickhouse       5 Oct  4 06:55 count.txt
-rw-r-----  1 clickhouse clickhouse 1878056 Oct  4 06:55 data.bin
-rw-r-----  1 clickhouse clickhouse    2240 Oct  4 06:55 data.mrk3
-rw-r-----  1 clickhouse clickhouse      10 Oct  4 06:55 default_compression_codec.txt
-rw-r-----  1 clickhouse clickhouse       4 Oct  4 06:55 minmax_LOORDERDATE.idx
-rw-r-----  1 clickhouse clickhouse       2 Oct  4 06:55 partition.dat
-rw-r-----  1 clickhouse clickhouse      48 Oct  4 06:55 primary.idx
-rw-r-----  1 clickhouse clickhouse  124892 Oct  4 06:55 skp_idx_idx_1.idx
-rw-r-----  1 clickhouse clickhouse     168 Oct  4 06:55 skp_idx_idx_1.mrk3

1993_2_2_0:
total 2004
drwxr-x---  2 clickhouse clickhouse    4096 Oct  4 06:55 ./
drwxr-x--- 52 clickhouse clickhouse    4096 Oct  4 06:55 ../
-rw-r-----  1 clickhouse clickhouse     346 Oct  4 06:55 checksums.txt
-rw-r-----  1 clickhouse clickhouse     418 Oct  4 06:55 columns.txt
-rw-r-----  1 clickhouse clickhouse       5 Oct  4 06:55 count.txt
-rw-r-----  1 clickhouse clickhouse 1879330 Oct  4 06:55 data.bin
-rw-r-----  1 clickhouse clickhouse    2240 Oct  4 06:55 data.mrk3
-rw-r-----  1 clickhouse clickhouse      10 Oct  4 06:55 default_compression_codec.txt
-rw-r-----  1 clickhouse clickhouse       4 Oct  4 06:55 minmax_LOORDERDATE.idx
-rw-r-----  1 clickhouse clickhouse       2 Oct  4 06:55 partition.dat
-rw-r-----  1 clickhouse clickhouse      48 Oct  4 06:55 primary.idx
-rw-r-----  1 clickhouse clickhouse  124820 Oct  4 06:55 skp_idx_idx_1.idx
-rw-r-----  1 clickhouse clickhouse     168 Oct  4 06:55 skp_idx_idx_1.mrk3

1993_2_36_1:
total 10388
drwxr-x---  2 clickhouse clickhouse    4096 Oct  4 06:55 ./
drwxr-x--- 52 clickhouse clickhouse    4096 Oct  4 06:55 ../
-rw-r-----  1 clickhouse clickhouse    1720 Oct  4 06:55 checksums.txt
-rw-r-----  1 clickhouse clickhouse     418 Oct  4 06:55 columns.txt
-rw-r-----  1 clickhouse clickhouse       6 Oct  4 06:55 count.txt
-rw-r-----  1 clickhouse clickhouse      10 Oct  4 06:55 default_compression_codec.txt
-rw-r-----  1 clickhouse clickhouse  451720 Oct  4 06:55 LOCOMMITDATE.bin
-rw-r-----  1 clickhouse clickhouse     912 Oct  4 06:55 LOCOMMITDATE.mrk2
-rw-r-----  1 clickhouse clickhouse  728167 Oct  4 06:55 LOCUSTKEY.bin
-rw-r-----  1 clickhouse clickhouse     912 Oct  4 06:55 LOCUSTKEY.mrk2
-rw-r-----  1 clickhouse clickhouse  245617 Oct  4 06:55 LODISCOUNT.bin
-rw-r-----  1 clickhouse clickhouse     912 Oct  4 06:55 LODISCOUNT.mrk2
-rw-r-----  1 clickhouse clickhouse 1216288 Oct  4 06:55 LOEXTENDEDPRICE.bin
-rw-r-----  1 clickhouse clickhouse     912 Oct  4 06:55 LOEXTENDEDPRICE.mrk2
-rw-r-----  1 clickhouse clickhouse  213739 Oct  4 06:55 LOLINENUMBER.bin
-rw-r-----  1 clickhouse clickhouse     912 Oct  4 06:55 LOLINENUMBER.mrk2
-rw-r-----  1 clickhouse clickhouse    4352 Oct  4 06:55 LOORDERDATE.bin
-rw-r-----  1 clickhouse clickhouse     912 Oct  4 06:55 LOORDERDATE.mrk2
-rw-r-----  1 clickhouse clickhouse  773066 Oct  4 06:55 LOORDERKEY.bin
-rw-r-----  1 clickhouse clickhouse     912 Oct  4 06:55 LOORDERKEY.mrk2
-rw-r-----  1 clickhouse clickhouse  172595 Oct  4 06:55 LOORDERPRIORITY.bin
-rw-r-----  1 clickhouse clickhouse      85 Oct  4 06:55 LOORDERPRIORITY.dict.bin
-rw-r-----  1 clickhouse clickhouse     912 Oct  4 06:55 LOORDERPRIORITY.dict.mrk2
-rw-r-----  1 clickhouse clickhouse     912 Oct  4 06:55 LOORDERPRIORITY.mrk2
-rw-r-----  1 clickhouse clickhouse  895735 Oct  4 06:55 LOORDTOTALPRICE.bin
-rw-r-----  1 clickhouse clickhouse     912 Oct  4 06:55 LOORDTOTALPRICE.mrk2
-rw-r-----  1 clickhouse clickhouse 1213133 Oct  4 06:55 LOPARTKEY.bin
-rw-r-----  1 clickhouse clickhouse     912 Oct  4 06:55 LOPARTKEY.mrk2
-rw-r-----  1 clickhouse clickhouse  304079 Oct  4 06:55 LOQUANTITY.bin
-rw-r-----  1 clickhouse clickhouse     912 Oct  4 06:55 LOQUANTITY.mrk2
-rw-r-----  1 clickhouse clickhouse 1216297 Oct  4 06:55 LOREVENUE.bin
-rw-r-----  1 clickhouse clickhouse     912 Oct  4 06:55 LOREVENUE.mrk2
-rw-r-----  1 clickhouse clickhouse  228055 Oct  4 06:55 LOSHIPMODE.bin
-rw-r-----  1 clickhouse clickhouse      75 Oct  4 06:55 LOSHIPMODE.dict.bin
-rw-r-----  1 clickhouse clickhouse     912 Oct  4 06:55 LOSHIPMODE.dict.mrk2
-rw-r-----  1 clickhouse clickhouse     912 Oct  4 06:55 LOSHIPMODE.mrk2
-rw-r-----  1 clickhouse clickhouse    1363 Oct  4 06:55 LOSHIPPRIORITY.bin
-rw-r-----  1 clickhouse clickhouse     912 Oct  4 06:55 LOSHIPPRIORITY.mrk2
-rw-r-----  1 clickhouse clickhouse  861165 Oct  4 06:55 LOSUPPKEY.bin
-rw-r-----  1 clickhouse clickhouse     912 Oct  4 06:55 LOSUPPKEY.mrk2
-rw-r-----  1 clickhouse clickhouse 1050596 Oct  4 06:55 LOSUPPLYCOST.bin
-rw-r-----  1 clickhouse clickhouse     912 Oct  4 06:55 LOSUPPLYCOST.mrk2
-rw-r-----  1 clickhouse clickhouse  230011 Oct  4 06:55 LOTAX.bin
-rw-r-----  1 clickhouse clickhouse     912 Oct  4 06:55 LOTAX.mrk2
-rw-r-----  1 clickhouse clickhouse       4 Oct  4 06:55 minmax_LOORDERDATE.idx
-rw-r-----  1 clickhouse clickhouse       2 Oct  4 06:55 partition.dat
-rw-r-----  1 clickhouse clickhouse     228 Oct  4 06:55 primary.idx
-rw-r-----  1 clickhouse clickhouse  666940 Oct  4 06:55 skp_idx_idx_1.idx
-rw-r-----  1 clickhouse clickhouse     888 Oct  4 06:55 skp_idx_idx_1.mrk2

1993_24_24_0:
total 2000
drwxr-x---  2 clickhouse clickhouse    4096 Oct  4 06:55 ./
drwxr-x--- 52 clickhouse clickhouse    4096 Oct  4 06:55 ../
-rw-r-----  1 clickhouse clickhouse     346 Oct  4 06:55 checksums.txt
-rw-r-----  1 clickhouse clickhouse     418 Oct  4 06:55 columns.txt
-rw-r-----  1 clickhouse clickhouse       5 Oct  4 06:55 count.txt
-rw-r-----  1 clickhouse clickhouse 1875891 Oct  4 06:55 data.bin
-rw-r-----  1 clickhouse clickhouse    2240 Oct  4 06:55 data.mrk3
-rw-r-----  1 clickhouse clickhouse      10 Oct  4 06:55 default_compression_codec.txt
-rw-r-----  1 clickhouse clickhouse       4 Oct  4 06:55 minmax_LOORDERDATE.idx
-rw-r-----  1 clickhouse clickhouse       2 Oct  4 06:55 partition.dat
-rw-r-----  1 clickhouse clickhouse      48 Oct  4 06:55 primary.idx
-rw-r-----  1 clickhouse clickhouse  125061 Oct  4 06:55 skp_idx_idx_1.idx
-rw-r-----  1 clickhouse clickhouse     168 Oct  4 06:55 skp_idx_idx_1.mrk3

1993_31_31_0:
total 2012
drwxr-x---  2 clickhouse clickhouse    4096 Oct  4 06:55 ./
drwxr-x--- 52 clickhouse clickhouse    4096 Oct  4 06:55 ../
-rw-r-----  1 clickhouse clickhouse     346 Oct  4 06:55 checksums.txt
-rw-r-----  1 clickhouse clickhouse     418 Oct  4 06:55 columns.txt
-rw-r-----  1 clickhouse clickhouse       5 Oct  4 06:55 count.txt
-rw-r-----  1 clickhouse clickhouse 1884782 Oct  4 06:55 data.bin
-rw-r-----  1 clickhouse clickhouse    2240 Oct  4 06:55 data.mrk3
-rw-r-----  1 clickhouse clickhouse      10 Oct  4 06:55 default_compression_codec.txt
-rw-r-----  1 clickhouse clickhouse       4 Oct  4 06:55 minmax_LOORDERDATE.idx
-rw-r-----  1 clickhouse clickhouse       2 Oct  4 06:55 partition.dat
-rw-r-----  1 clickhouse clickhouse      48 Oct  4 06:55 primary.idx
-rw-r-----  1 clickhouse clickhouse  125563 Oct  4 06:55 skp_idx_idx_1.idx
-rw-r-----  1 clickhouse clickhouse     168 Oct  4 06:55 skp_idx_idx_1.mrk3

1993_36_36_0:
total 708
drwxr-x---  2 clickhouse clickhouse   4096 Oct  4 06:55 ./
drwxr-x--- 52 clickhouse clickhouse   4096 Oct  4 06:55 ../
-rw-r-----  1 clickhouse clickhouse    344 Oct  4 06:55 checksums.txt
-rw-r-----  1 clickhouse clickhouse    418 Oct  4 06:55 columns.txt
-rw-r-----  1 clickhouse clickhouse      5 Oct  4 06:55 count.txt
-rw-r-----  1 clickhouse clickhouse 634044 Oct  4 06:55 data.bin
-rw-r-----  1 clickhouse clickhouse    840 Oct  4 06:55 data.mrk3
-rw-r-----  1 clickhouse clickhouse     10 Oct  4 06:55 default_compression_codec.txt
-rw-r-----  1 clickhouse clickhouse      4 Oct  4 06:55 minmax_LOORDERDATE.idx
-rw-r-----  1 clickhouse clickhouse      2 Oct  4 06:55 partition.dat
-rw-r-----  1 clickhouse clickhouse     18 Oct  4 06:55 primary.idx
-rw-r-----  1 clickhouse clickhouse  42053 Oct  4 06:55 skp_idx_idx_1.idx
-rw-r-----  1 clickhouse clickhouse     48 Oct  4 06:55 skp_idx_idx_1.mrk3
```
### 分析数据文件
#### 查看checksums.txt
```sh
od -An -c checksums.txt
   c   h   e   c   k   s   u   m   s       f   o   r   m   a   t
       v   e   r   s   i   o   n   :       4  \n 223   w 243   u
 234   :   e 003   &   % 326 315 233  \f 263 336 202 214 006  \0
  \0 265  \a  \0  \0 371   +   , 020   L   O   C   O   M   M   I
   T   D   A   T   E   .   b   i   n 210 311 033   8 343 241   ~
 213 301 313   C   o   Q   / 216 226   x   i  \v 001 330 372   $
 345 346   J 247 005  \v 232 217 255   V   > 303 034 317 373   Y
 021   8  \0 360 022   m   r   k   2 220  \a 257   u   3 354 204
 034 255   ] 252 030 243 223 246   ^   k   F  \0  \r   L   O   C
   U   S   T   K   E   Y   Z  \0 366 031 347 270   ,   S   Y 247
 254 016 365 365 276 364 232 026   o   y   8   .   C 001 260 365
   I 214 215 032 362 271 374 311 203 357 334   g   I 320   ` 305
 237 016   5  \0 002   W  \0 360  \r   .   < 036   *   W 247 365
   t 311 217   >   3 203   B 220 306  \0 016   L   O   D   I   S
   C   O   U   N   T   X  \0 367 031 361 376 016   L 263 313   "
 333   S   x 300 254   " 246 267 243   2   \ 212 001 254 275 022
 205 377 265   D   i 351 354   g   e   q   J  \r   M 220 212 344
 017   6  \0 002   Y  \0 361 021   p   * 220   >  \v   ^   k 210
 240 345 004   6   % 277 270   :  \0 023   L   O   E   X   T   E
   N   D   E   D   P   R   I   C 020 001 360 004 240 236   J   S
   { 210 231 344 247 003   K 235 304 204   E 002   8   # 374 266
  \0 374 002 251 303   c   @ 242   f   2   } 237 211   A 324 371
   ]   "   2 024   ;  \0 002   c  \0 360 017 017 212 177   w   #
   G 323   ? 341   [ 037 331   a   B 243 234  \0 020   L   O   L
   I   N   E   N   U   M   B   E   R 276  \0 360 004 353 205  \r
   c 331   U 243   3   v  \f 377 327   L 334 206   )   u 272   I
 276  \0 371 002 226   L 003   i 036 200       z   D 223   \   :
 265 320  \t 201 021   8  \0 002   ]  \0 364  \n 370 327 301   7
 216 232   *   ( 305 324 216   .   K 031   e 355  \0 017   L   O
   O   R   D   E   R 314 001 360 003 200   " 001 212 264 272 210
 356 262   + 343 320   O 005  \t   E 316 354 313 001 370 002   7
 351 275   4 305   E 340   t 242   - 275   D 375 253 322 261 020
   6  \0 002   Z  \0 360 001 206   9 232 346 274   . 375 312   (
   <   %   A   , 335 262 225   s 001 001   Z  \0 003 313 001 360
 004 312 227   /   !   ^ 026 341   <   { 356 307 270   S   <   ~
   H   T   V   ` 025 001 364 001 253   L 006   3 305   l   D  \0
   - 204 256 337   [   *   v 357 220  \0  \0   6  \0 002   Y  \0
 360 001 323 304 026   [ 213   I   N 300   D 266 250 350 262 341
 317   {   s 001 001   Y  \0   q   P   R   I   O   R   I   T   )
 002 374 031 263 304  \n   a   3  \f 373 333   J 330 346 325   %
  \r 241   B  \f   ;   G 001 374 301 022 300 312 211 001       t
 022   Y   "   3   # 020 351   X 217 250 030   ;  \0   @   d   i
   c   t   S 001 377 025   U   0 337 255 371 021   -   | 212 234
   y  \f 222 345   |   ! 021 001   @   g 235 210 365   / 224 364
 025   Y   " 366   \   J   E 307 322 031   <  \0 002 002 244  \0
 374 003  \n   6   u   k   B 207   v   {   G   W 265   < 272   B
 237   T  \0 024   -  \0 002   (  \0 363 001     023 346   ;   d
 320 312 346 321   X 330   ,   L 231 264 250 314  \0   U   T   O
   T   A   L   ? 002 360 004 367 325   6   J 314 202 344 364   e
   y 215   x   M 177 023   9   v 366 351   * 001 362 001 346   j
   O   (   4   e 276 347   ~   r 031 207   b 313 255   k   c  \0
  \a   ;  \0 002   c  \0 360 001 027 257 035   6  \t   q 240 353
  \n 347 337   + 353 224 242   A   R 003   4   P   A   R   R 003
 360 004 315 205   J   &   a 334 234   s 371 031 277 264   i   m
 307 003   u   d   F   ]  \0 366 002 301 237 321 251   6 371 230
 257 362 271 340 005 264   P 017   [ 016   5  \0 002   W  \0 363
  \n   V 221 337 347 300 270 032   q 227 224   \  \v 305   +   <
   o  \0 016   L   O   Q   U   A   N   T 201 001 360 004 317 307
 022 243       ;   8   Q  \r   5 212 345 227       |   ' 223 202
   3 224 002 367 002   i 343   , 242   1   0 251 304   6 240   4
   (   ` 251 037 304 017   6  \0 002   Y  \0 360 001   E 257   c
 005 214   4   } 256   G   ^ 002   A 201 024   v   s 260  \0   a
   R   E   V   E   N   U   L 003 360 004 251 236   J 260 335 340
 362 215   ,   -   : 347 036 207 212 225   Q  \n 346 260  \0 366
 002 371   H  \a   & 321   s 243  \f   \   N 271   z 376 024   %
   L 016   5  \0 002   W  \0 360 001 352   >   , 234 310 227 304
 265 235 303   .   : 203   %     217 260  \0   q   S   H   I   P
   M   O   D   X  \0 360 004 327 365  \r 335   \   }   w   u   \
 221 326 350   6 251 273 344   B 263   A   1 002 367 002 236 256
 346 213 301 021   {   ?   +   ` 346   / 264 227 020 324 023   6
  \0 004   , 002 374 025   K 370   I 344   G 037 002 356 321   ]
   G 250   C   h 352 373   & 001   6 206   ;   b 222  \n 306 332
   G  \v 303 261 334 301   % 342 201 024   7  \0 002 225  \0 360
 001   U 263 346  \f 220 324 037   G 326 274   /   B   | 306   +
   %   ~ 003 005 225  \0 002   #  \0 362 003 024   8 231 363   =
 320   R   f   ` 036   7   : 230 335 207   s  \0 022   #  \0  \b
 355 002 360 003 323  \n 307 324 230 377   4   s   y 031 254 244
   c 230   )   V 377   C   k 001 363 001   H 312   R   N   | 234
   T 324   1   8   L  \f   Q 202   >   K 273  \0 005   9  \0 002
   `  \0 363  \t   b 330 300   C 346 251   5  \n   X  \a 340 321
   a 232   1   n  \0  \r   L   O   S   U   P   P   q 005 360 004
 355 307   4   .   t   s 226   \ 251 020   E   L 301   . 333   J
 213 275   q   o 001 366 002   1 250   k 035 251  \t 346 366 323
   \   z 261 206 200 351 345 016   5  \0 002   W  \0 362 003 233
 303   0   \   5 035 210   N 376 312 376 215 232   u 344 267  \0
 020   "  \0   Q   L   Y   C   O   S   s 005 360 004 344 217   @
   @ 212 020 313 311 205 203   s   q 237   S 032 232   5   \ 335
   Z  \0 371 002   t 247 212 322 302   Y   B 032 225   h 327  \t
 224 002 231 244 021   8  \0 002   ]  \0 360  \b   9 241   ]   .
 241   ; 326 326   s 220   N 322 363 350 344   2  \0  \t   L   O
   T   A   X 270 003 360 004 373 204 016 333   & 037 021   6 200
  \f 002   | 264 237   ;   A 216 241  \n  \f 001 362 002   O 306
   _   r 030 227   ?   y 314 315 206   | 366 376   P 017  \n   1
  \0 002   O  \0 363   & 322  \r 324 252 030  \n  \f 277 233 373
   i   B   4   n 225 367  \0  \t   c   o   u   n   t   .   t   x
   t 006 261 213   d 313 273 354 222   % 275   ?   [ 323   G 355
   .  \b  \0 026   m   i   n   m   a   x   _ 320 003 001   ' 005
 361   M   i   d   x 004   [ 232 257 355 241 335   p 300 350   w
 035 032 261 300 031 254  \0  \r   p   a   r   t   i   t   i   o
   n   .   d   a   t 002   6  \f   +  \0   6 277   D 350   2   8
  \r 215 251 252 251   l  \0  \v   p   r   i   m   a   r   y   .
   i   d   x 344 001   T 336 270   :   3   ! 350 302 375 314   b
 314 271 347 032 017  \0 021   s   k   p   _   i   d   x 004  \0
 020   1   %  \0 372 031 274 332   (   j   ^   @ 306 271   d 250
 204 255 266 224 255 036   -   0 017 001 240 304   ( 275 325 035
 353 267 321   z   s   > 017 025   7 322 330 226 345 022   9  \0
  \0 343  \0 360 004 370 006 355 267 306 024 204 247 324 361 311
 276   E 300 252   z 272   9  \0
```
#### partition.dat
```sh
od -An -l partition.dat
                 1993
```
#### columns.txt
```sh
cat columns.txt
columns format version: 1
17 columns:
`LOORDERKEY` UInt32
`LOLINENUMBER` UInt8
`LOCUSTKEY` UInt32
`LOPARTKEY` UInt32
`LOSUPPKEY` UInt32
`LOORDERDATE` Date
`LOORDERPRIORITY` LowCardinality(String)
`LOSHIPPRIORITY` UInt8
`LOQUANTITY` UInt8
`LOEXTENDEDPRICE` UInt32
`LOORDTOTALPRICE` UInt32
`LODISCOUNT` UInt8
`LOREVENUE` UInt32
`LOSUPPLYCOST` UInt32
`LOTAX` UInt8
`LOCOMMITDATE` Date
`LOSHIPMODE` LowCardinality(String)
```
#### count.txt
```sh
cat count.txt
302764
```
#### default_compression_codec.txt
```sh
cat default_compression_codec.txt
CODEC(LZ4)
```
#### primary.idx
```sh
od -Ax -tx2 -w6  primary.idx
000000 20d1 3b81 0000
000006 20da e8a3 0043
00000c 20e4 8de0 0045
000012 20ee a3e6 003c
000018 20f8 6863 0036
00001e 2102 f3e6 001e
000024 210c 6c23 001c
00002a 2115 00e6 0056
000030 211f b805 0053
000036 2129 9827 004f
00003c 2133 e942 0052
000042 213d cbe0 0043
000048 2147 bee4 002f
00004e 2151 91c1 0039
000054 215b a706 0039
00005a 2165 8a84 0032
000060 216f 21c5 0031
000066 2179 51a5 002c
00006c 2183 fcc0 0006
000072 218c f706 0041
000078 2196 f266 0038
00007e 21a0 7c84 0021
000084 21aa c380 001b
00008a 21b4 d107 0028
000090 21be 3320 0010
000096 21c7 c544 004c
00009c 21d1 bbe4 0035
0000a2 21db 2b01 0036
0000a8 21e5 73e4 002f
0000ae 21ee 6466 0059
0000b4 21f9 4f01 0005
0000ba 2202 fd66 0058
0000c0 220c 5767 003e
0000c6 2216 f5a3 0026
0000cc 2220 dbe1 002f
0000d2 222a f465 003d
0000d8 2234 d406 0035
0000de 223d 15e3 005b
0000e4

hexdump -C primary.idx
00000000  d1 20 81 3b 00 00 da 20  a3 e8 43 00 e4 20 e0 8d  |. .;... ..C.. ..|
00000010  45 00 ee 20 e6 a3 3c 00  f8 20 63 68 36 00 02 21  |E.. ..<.. ch6..!|
00000020  e6 f3 1e 00 0c 21 23 6c  1c 00 15 21 e6 00 56 00  |.....!#l...!..V.|
00000030  1f 21 05 b8 53 00 29 21  27 98 4f 00 33 21 42 e9  |.!..S.)!'.O.3!B.|
00000040  52 00 3d 21 e0 cb 43 00  47 21 e4 be 2f 00 51 21  |R.=!..C.G!../.Q!|
00000050  c1 91 39 00 5b 21 06 a7  39 00 65 21 84 8a 32 00  |..9.[!..9.e!..2.|
00000060  6f 21 c5 21 31 00 79 21  a5 51 2c 00 83 21 c0 fc  |o!.!1.y!.Q,..!..|
00000070  06 00 8c 21 06 f7 41 00  96 21 66 f2 38 00 a0 21  |...!..A..!f.8..!|
00000080  84 7c 21 00 aa 21 80 c3  1b 00 b4 21 07 d1 28 00  |.|!..!.....!..(.|
00000090  be 21 20 33 10 00 c7 21  44 c5 4c 00 d1 21 e4 bb  |.! 3...!D.L..!..|
000000a0  35 00 db 21 01 2b 36 00  e5 21 e4 73 2f 00 ee 21  |5..!.+6..!.s/..!|
000000b0  66 64 59 00 f9 21 01 4f  05 00 02 22 66 fd 58 00  |fdY..!.O..."f.X.|
000000c0  0c 22 67 57 3e 00 16 22  a3 f5 26 00 20 22 e1 db  |."gW>.."..&. "..|
000000d0  2f 00 2a 22 65 f4 3d 00  34 22 06 d4 35 00 3d 22  |/.*"e.=.4"..5.="|
000000e0  e3 15 5b 00                                       |..[.|
000000e4


select count(1) from ssb.lineorder where toYear(LOORDERDATE)=1993

select 302764/8192
36.95849609375
```
##### 结论
od -Ax -tx2 -w6  primary.idx查看结果是38行，但302764/8192只有37块，两者差1，因为最后一行有标记。
#### {column}.mrk2
一个{column}.bin文件有1至多个数据压缩块组成，mark2数据标记文件格式比较固定，primary.idx文件中的每个索引在此文件中都有一个对应的Mark，有三列：
- Offset in compressed file，8 Bytes，代表该标记指向的压缩数据块在bin文件中的偏移量。
- Offset in decompressed block，8 Bytes，代表该标记指向的数据在解压数据块中的偏移量。
- Rows count，8 Bytes，行数，通常情况下其等于index_granularity。
```sh
od -Ad -tu8 -w24  LOORDERKEY.mrk2
0000000                    0                    0                 8192
0000024                    0                32768                 8192
0000048                41931                    0                 8192
0000072                41931                32768                 8192
0000096                83974                    0                 8192
0000120                83974                32768                 8192
0000144               125395                    0                 8192
0000168               125395                32768                 8192
0000192               167085                    0                 8192
0000216               167085                32768                 8192
0000240               209177                    0                 8192
0000264               209177                32768                 8192
0000288               251011                    0                 8192
0000312               251011                32768                 8192
0000336               292869                    0                 8192
0000360               292869                32768                 8192
0000384               334687                    0                 8192
0000408               334687                32768                 8192
0000432               376564                    0                 8192
0000456               376564                32768                 8192
0000480               418340                    0                 8192
0000504               418340                32768                 8192
0000528               460377                    0                 8192
0000552               460377                32768                 8192
0000576               502324                    0                 8192
0000600               502324                32768                 8192
0000624               544173                    0                 8192
0000648               544173                32768                 8192
0000672               586007                    0                 8192
0000696               586007                32768                 8192
0000720               627829                    0                 8192
0000744               627829                32768                 8192
0000768               669290                    0                 8192
0000792               669290                32768                 8192
0000816               711039                    0                 8192
0000840               711039                32768                 8192
0000864               752966                    0                 7852
0000888               752966                31408                    0
0000912
```
#### {column}.bin
- 第一行16个字节是该文件的checksum值
- 第二行（以Id.bin为例）  
1、第一个字节是0x82，是默认的LZ4算法  
2、第2个到第5个字节是压缩后的数据块的大小，这里是小端模式，Int32占4个字节, 倒着就是 00 00 00 1c = 28  
3、第6个字节到第9个字节是压缩前的数据块大小，同理00 00 00 18=24  
4、与 clickhouse-compressor --stat < Id.bin得到的结果一致
```sh
clickhouse-compressor --stat LOORDERKEY.bin
65536   41915
65536   42027
65536   41405
65536   41674
65536   42076
65536   41818
65536   41842
65536   41802
65536   41861
65536   41760
65536   42021
65536   41931
65536   41833
65536   41818
65536   41806
65536   41445
65536   41733
65536   41911
31408   20084


hexdump -C LOORDERKEY.bin  | more
00000000  6b a2 42 d4 a4 6e dd cd  59 bf f7 d4 5c 68 79 13  |k.B..n..Y...\hy.|
00000010  82 bb a3 00 00 00 00 01  00 44 81 3b 00 00 04 00  |.........D.;....|
00000020  62 46 4f 00 00 04 6c 04  00 26 e1 89 04 00 71 05  |bFO...l..&....q.|
00000030  b6 00 00 24 18 01 04 00  62 45 19 01 00 45 66 04  |...$....bE...Ef.|
00000040  00 22 c1 6b 04 00 22 25  93 04 00 22 62 f0 04 00  |.".k.."%..."b...|
00000050  22 00 f5 04 00 a2 07 0b  02 00 60 55 02 00 24 e3  |".........`U..$.|
00000060  04 00 31 c5 4e 03 04 00  62 a4 69 03 00 44 87 04  |..1.N...b.i..D..|
00000070  00 a2 82 c0 03 00 47 d8  03 00 a6 db 04 00 22 85  |......G.......".|
00000080  dd 04 00 a2 a3 de 03 00  47 f1 03 00 23 fc 04 00  |........G...#...|
00000090  62 e0 2c 04 00 e1 3f 04  00 66 20 41 04 00 04 57  |b.,...?..f A...W|
000000a0  04 00 22 e5 59 04 00 22  65 86 04 00 26 62 95 04  |..".Y.."e...&b..|
000000b0  00 2e c5 fc 04 00 31 27  20 05 04 00 22 25 71 04  |......1' ..."%q.|
000000c0  00 22 82 f1 04 00 a2 a4  11 06 00 03 18 06 00 27  |.".............'|
000000d0  44 04 00 22 47 67 04 00  26 a3 69 04 00 f6 03 c7  |D.."Gg..&.i.....|
000000e0  9f 06 00 61 cc 06 00 83  d5 06 00 44 0a 07 00 46  |...a.......D...F|
000000f0  14 04 00 66 44 5a 07 00  c1 86 04 00 66 c0 ca 07  |...fDZ......f...|
00000100  00 24 d2 04 00 71 63 fb  07 00 c3 04 08 04 00 22  |.$...qc........"|
00000110  06 7a 04 00 71 a7 c8 08  00 63 73 09 04 00 23 41  |.z..q....cs...#A|
00000120  83 04 00 16 bf 04 00 22  c2 e5 04 00 3d 84 00 0a  |......."....=...|
00000130  04 00 aa c6 33 0a 00 e4  45 0a 00 c7 56 04 00 66  |....3...E...V..f|
00000140  45 65 0a 00 02 6a 04 00  22 a0 c6 04 00 b1 87 cd  |Ee...j..".......|
00000150  0a 00 a0 f9 0a 00 81 31  0b 04 00 62 e7 4e 0b 00  |.......1...b.N..|
00000160  e3 85 04 00 26 40 d0 04  00 22 22 ed 04 00 3d c3  |....&@...""...=.|
00000170  31 0c 04 00 22 82 5a 04  00 f2 03 65 a6 0c 00 23  |1...".Z....e...#|
00000180  c2 0c 00 64 1b 0d 00 45  2b 0d 00 62 45 04 00 66  |...d...E+..bE..f|
00000190  03 bc 0d 00 c6 d8 04 00  71 60 11 0e 00 a3 22 0f  |........q`....".|
000001a0  04 00 22 c0 89 04 00 22  c4 ba 04 00 a6 a6 19 10  |.."...."........|
000001b0  00 a1 2d 10 00 c5 4e 04  00 66 a0 7b 10 00 02 7f  |..-...N..f.{....|
000001c0  04 00 2e 26 d0 04 00 a2  a0 31 11 00 06 32 11 00  |...&.....1...2..|
000001d0  03 3a 04 00 2e 02 82 04  00 26 81 e4 04 00 a2 a2  |.:.......&......|
000001e0  3d 12 00 86 67 12 00 a6  ba 04 00 26 c0 de 04 00  |=...g......&....|
```
#### minmax_LOORDERDATE.idx
```sh
hexdump -C minmax_LOORDERDATE.idx
00000000  d1 20 3d 22                                       |. ="|
00000004

#验证上面的数值
0x20d1=8401  
0x223d=8765

┌─addDays(toDate('1970-01-01'), 8401)─┐
│                          1993-01-01 │
└─────────────────────────────────────┘

┌─addDays(toDate('1970-01-01'), 8765)─┐
│                          1993-12-31 │
└─────────────────────────────────────┘
```
#### skp_idx_idx_1.idx 需要补充