- [工具、数据文件准备](#工具数据文件准备)
- [表、数据准备](#表数据准备)
- [数据导入](#数据导入)
- [多表Join查询](#多表join查询)
- [单表查询](#单表查询)
  - [生成分布式大宽表](#生成分布式大宽表)
- [并发测试](#并发测试)
  - [benchmark并发测试](#benchmark并发测试)
# 工具、数据文件准备
```sh
#下载代码：
git clone https://github.com/vadimtk/ssb-dbgen.git
 
#编译生成数据：
cd ssb-dbgen
make 
 
./dbgen -s 10 -T c
./dbgen -s 10 -T l
./dbgen -s 10 -T p
./dbgen -s 10 -T s
./dbgen -s 10 -T d
 
 
#说明：
c--customer.tbl
p--part.tbl
s--supplier.tbl
d--date.tbl
l--lineorder.tbl
上面的表数据可以用如下的命令一次性生成：(for all SSBM tables)
dbgen -s 10 -T a
```
# 表、数据准备
```sql
clickhouse-client -h 10.206.13.5 --port 9000

create database ssb;
use ssb;

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

CREATE TABLE customer on cluster test_cluster_three_shards
(
        CCUSTKEY       UInt32,
        CNAME          String,
        CADDRESS       String,
        CCITY          LowCardinality(String),
        CNATION        LowCardinality(String),
        CREGION        LowCardinality(String),
        CPHONE         String,
        CMKTSEGMENT    LowCardinality(String)
)
ENGINE = MergeTree ORDER BY (CCUSTKEY);
CREATE TABLE ssb.customer_all  on cluster test_cluster_three_shards AS ssb.customer 
ENGINE = Distributed(test_cluster_three_shards, ssb, customer, rand());

CREATE TABLE supplier on cluster test_cluster_three_shards
(
        SSUPPKEY       UInt32,
        SNAME          String,
        SADDRESS       String,
        SCITY          LowCardinality(String),
        SNATION        LowCardinality(String),
        SREGION        LowCardinality(String),
        SPHONE         String
)
ENGINE = MergeTree ORDER BY SSUPPKEY;
CREATE TABLE ssb.supplier_all  on cluster test_cluster_three_shards AS ssb.supplier 
ENGINE = Distributed(test_cluster_three_shards, ssb, supplier, rand());

CREATE TABLE part on cluster test_cluster_three_shards
(
        PPARTKEY       UInt32,
        PNAME          String,
        PMFGR          LowCardinality(String),
        PCATEGORY      LowCardinality(String),
        PBRAND         LowCardinality(String),
        PCOLOR         LowCardinality(String),
        PTYPE          LowCardinality(String),
        PSIZE          UInt8,
        PCONTAINER     LowCardinality(String)
)
ENGINE = MergeTree ORDER BY PPARTKEY;
CREATE TABLE ssb.part_all  on cluster test_cluster_three_shards AS ssb.part 
ENGINE = Distributed(test_cluster_three_shards, ssb, part, rand());

CREATE TABLE date on cluster test_cluster_three_shards
(
        DDATEKEY             UInt32,
        DDATE                String,
        DDAYOFWEEK           String,
        DMONTH               String,
        DYEAR                UInt32,
        DYEARMONTHNUM        UInt32,
        DYEARMONTH           String,
        DDAYNUMINWEEK        UInt32,
        DDAYNUMINMONTH       UInt32,
        DDAYNUMINYEAR        UInt32,
        DMONTHNUMINYEAR      UInt32,
        DWEEKNUMINYEAR       UInt32,
        DSELLINGSEASON       String,
        DLASTDAYINWEEKFL     UInt32,
        DLASTDAYINMONTHFL    UInt32,
        DHOLIDAYFL           UInt32,
        DWEEKDAYFL           UInt32
) 
ENGINE = MergeTree ORDER BY DDATEKEY;
CREATE TABLE ssb.date_all  on cluster test_cluster_three_shards AS ssb.date 
ENGINE = Distributed(test_cluster_three_shards, ssb, date, rand());
```
# 数据导入
```sh
ll -h /data/ssb-dbgen/*.tbl
-rw-r--r-- 1 root root  32M Sep 23 10:51 /data/ssb-dbgen/customer.tbl
-rw-r--r-- 1 root root 270K Sep 23 10:56 /data/ssb-dbgen/date.tbl
-rw-r--r-- 1 root root 6.5G Sep 23 10:53 /data/ssb-dbgen/lineorder.tbl
-rw-r--r-- 1 root root  77M Sep 23 10:55 /data/ssb-dbgen/part.tbl
-rw-r--r-- 1 root root 1.9M Sep 23 10:56 /data/ssb-dbgen/supplier.tbl

clickhouse-client -h 10.206.13.5 --port 9000
clickhouse client -h 10.206.13.5 --time --query "INSERT INTO ssb.customer_all FORMAT CSV" < /data/ssb-dbgen/customer.tbl
clickhouse client -h 10.206.13.5 --time --query "INSERT INTO ssb.part_all FORMAT CSV" < /data/ssb-dbgen/part.tbl
clickhouse client -h 10.206.13.5 --time --query "INSERT INTO ssb.supplier_all FORMAT CSV" < /data/ssb-dbgen/supplier.tbl
clickhouse client -h 10.206.13.5 --time --query "INSERT INTO ssb.lineorder_all FORMAT CSV" < /data/ssb-dbgen/lineorder.tbl
clickhouse client -h 10.206.13.5 --time --query "INSERT INTO ssb.date_all FORMAT CSV" < /data/ssb-dbgen/date.tbl

#查看系统并发：
show settings like 'max_threads';
```
# 多表Join查询
- Q1.1
```sql
select sum(LOREVENUE) as revenue
from lineorder_all 
   global join date_all on toYYYYMMDD(LOORDERDATE) = DDATEKEY
where DYEAR = 1993 and LODISCOUNT between 1 and 3 and LOQUANTITY < 25;
```
- Q1.2
```sql
select sum(LOREVENUE) as revenue
from lineorder_all 
   global join date_all on toYYYYMMDD(LOORDERDATE) = DDATEKEY
where DYEARMONTHNUM = 199401 and LODISCOUNT between 4 and 6 and LOQUANTITY between 26 and 35;
```
- Q1.3
```sql
select sum(LOREVENUE) as revenue
from lineorder_all 
   global join date_all on toYYYYMMDD(LOORDERDATE) = DDATEKEY
where DWEEKNUMINYEAR = 6 and DYEAR = 1994 and LODISCOUNT between 5 and 7 and LOQUANTITY between 26 and 35;
```
- Q2.1
```sql
SELECT
    sum(LOREVENUE) AS lorevenue,
    DYEAR,
    PBRAND
FROM lineorder_all
GLOBAL INNER JOIN date_all ON toYYYYMMDD(LOORDERDATE) = DDATEKEY
GLOBAL INNER JOIN part_all ON LOPARTKEY = PPARTKEY
GLOBAL INNER JOIN supplier_all ON LOSUPPKEY = SSUPPKEY
WHERE (PCATEGORY = 'MFGR#12') AND (SREGION = 'AMERICA')
GROUP BY
    DYEAR,
    PBRAND
ORDER BY
    DYEAR ASC,
    PBRAND ASC
```
- Q2.2
```sql
SELECT
    sum(LOREVENUE) AS lorevenue,
    DYEAR,
    PBRAND
FROM lineorder_all
GLOBAL INNER JOIN date_all ON toYYYYMMDD(LOORDERDATE) = DDATEKEY
GLOBAL INNER JOIN part_all ON LOPARTKEY = PPARTKEY
GLOBAL INNER JOIN supplier_all ON LOSUPPKEY = SSUPPKEY
WHERE ((PBRAND >= 'MFGR#2221') AND (PBRAND <= 'MFGR#2228')) AND (SREGION = 'ASIA')
GROUP BY
    DYEAR,
    PBRAND
ORDER BY
    DYEAR ASC,
    PBRAND ASC
```
- Q2.3
```sql
SELECT
    sum(LOREVENUE) AS lorevenue,
    DYEAR,
    PBRAND
FROM lineorder_all
GLOBAL INNER JOIN date_all ON toYYYYMMDD(LOORDERDATE) = DDATEKEY
GLOBAL INNER JOIN part_all ON LOPARTKEY = PPARTKEY
GLOBAL INNER JOIN supplier_all ON LOSUPPKEY = SSUPPKEY
WHERE (PBRAND = 'MFGR#2239') AND (SREGION = 'EUROPE')
GROUP BY
    DYEAR,
    PBRAND
ORDER BY
    DYEAR ASC,
    PBRAND ASC
```
- Q3.1
```sql
select CNATION, SNATION, DYEAR, sum(LOREVENUE) as lorevenue
from lineorder_all
   global join date_all on toYYYYMMDD(LOORDERDATE) = DDATEKEY
   global join customer_all on LOCUSTKEY = CCUSTKEY
   global join supplier_all on LOSUPPKEY = SSUPPKEY
where CREGION = 'ASIA' and SREGION = 'ASIA'and DYEAR >= 1992 and DYEAR <= 1997
group by CNATION, SNATION, DYEAR
order by DYEAR asc, lorevenue desc;
```
- Q3.2
```sql
select CCITY, SCITY, DYEAR, sum(LOREVENUE) as lorevenue
from lineorder_all
   global join date_all on toYYYYMMDD(LOORDERDATE) = DDATEKEY
   global join customer_all on LOCUSTKEY = CCUSTKEY
   global join supplier_all on LOSUPPKEY = SSUPPKEY
where CNATION = 'UNITED STATES' and SNATION = 'UNITED STATES'
and DYEAR >= 1992 and DYEAR <= 1997
group by CCITY, SCITY, DYEAR
order by DYEAR asc, lorevenue desc;
```
- Q3.3
```sql
select CCITY, SCITY, DYEAR, sum(LOREVENUE) as lorevenue
from lineorder_all
	   global join date_all on toYYYYMMDD(LOORDERDATE) = DDATEKEY
	   global join customer_all on LOCUSTKEY = CCUSTKEY
	   global join supplier_all on LOSUPPKEY = SSUPPKEY
where (CCITY='UNITED KI1' or CCITY='UNITED KI5')
and (SCITY='UNITED KI1' or SCITY='UNITED KI5')
and DYEAR >= 1992 and DYEAR <= 1997
group by CCITY, SCITY, DYEAR
order by DYEAR asc, lorevenue desc;
```
- Q3.4
```sql
select CCITY, SCITY, DYEAR, sum(LOREVENUE) as lorevenue
from lineorder_all
	   global join date_all on toYYYYMMDD(LOORDERDATE) = DDATEKEY
	   global join customer_all on LOCUSTKEY = CCUSTKEY
	   global join supplier_all on LOSUPPKEY = SSUPPKEY
where (CCITY='UNITED KI1' or CCITY='UNITED KI5')
and (SCITY='UNITED KI1' or SCITY='UNITED KI5')
and DYEARMONTH = 'Dec1997'
group by CCITY, SCITY, DYEAR
order by DYEAR asc, lorevenue desc;
```
- Q4.1
```sql
SELECT
    DYEAR,
    CNATION,
    sum(LOREVENUE) - sum(LOSUPPLYCOST) AS profit
FROM lineorder_all
GLOBAL INNER JOIN date_all ON toYYYYMMDD(LOORDERDATE) = DDATEKEY
GLOBAL INNER JOIN customer_all ON LOCUSTKEY = CCUSTKEY
GLOBAL INNER JOIN supplier_all ON LOSUPPKEY = SSUPPKEY
GLOBAL INNER JOIN part_all ON LOPARTKEY = PPARTKEY
WHERE (CREGION = 'AMERICA') AND (SREGION = 'AMERICA') AND ((PMFGR = 'MFGR#1') OR (PMFGR = 'MFGR#2'))
GROUP BY
    DYEAR,
    CNATION
ORDER BY
    DYEAR ASC,
    CNATION ASC
```
- Q4.2
```sql
SELECT
    DYEAR,
    SNATION,
    PCATEGORY,
    sum(LOREVENUE) - sum(LOSUPPLYCOST) AS profit
FROM lineorder_all
GLOBAL INNER JOIN date_all ON toYYYYMMDD(LOORDERDATE) = DDATEKEY
GLOBAL INNER JOIN customer_all ON LOCUSTKEY = CCUSTKEY
GLOBAL INNER JOIN supplier_all ON LOSUPPKEY = SSUPPKEY
GLOBAL INNER JOIN part_all ON LOPARTKEY = PPARTKEY
WHERE (CREGION = 'AMERICA') AND (SREGION = 'AMERICA') AND ((DYEAR = 1997) OR (DYEAR = 1998)) AND ((PMFGR = 'MFGR#1') OR (PMFGR = 'MFGR#2'))
GROUP BY
    DYEAR,
    SNATION,
    PCATEGORY
ORDER BY
    DYEAR ASC,
    SNATION ASC,
    PCATEGORY ASC
```
- Q4.3
```sql
SELECT
    DYEAR,
    SCITY,
    PBRAND,
    sum(LOREVENUE) - sum(LOSUPPLYCOST) AS profit
FROM lineorder_all
GLOBAL INNER JOIN date_all ON toYYYYMMDD(LOORDERDATE) = DDATEKEY
GLOBAL INNER JOIN customer_all ON LOCUSTKEY = CCUSTKEY
GLOBAL INNER JOIN supplier_all ON LOSUPPKEY = SSUPPKEY
GLOBAL INNER JOIN part_all ON LOPARTKEY = PPARTKEY
WHERE (CREGION = 'AMERICA') AND (SNATION = 'UNITED STATES') AND ((DYEAR = 1997) OR (DYEAR = 1998)) AND (PCATEGORY = 'MFGR#14')
GROUP BY
    DYEAR,
    SCITY,
    PBRAND
ORDER BY
    DYEAR ASC,
    SCITY ASC,
    PBRAND ASC	
```
# 单表查询
## 生成分布式大宽表
```sql
SET max_memory_usage = 20000000000;
set distributed_product_mode = 'global';
CREATE TABLE ssb.flat_table
ENGINE = MergeTree
PARTITION BY toYear(LOORDERDATE)
ORDER BY (LOORDERDATE, LOORDERKEY) AS
SELECT
    l.LOORDERKEY AS LOORDERKEY,
    l.LOLINENUMBER AS LOLINENUMBER,
    l.LOCUSTKEY AS LOCUSTKEY,
    l.LOPARTKEY AS LOPARTKEY,
    l.LOSUPPKEY AS LOSUPPKEY,
    l.LOORDERDATE AS LOORDERDATE,
    l.LOORDERPRIORITY AS LOORDERPRIORITY,
    l.LOSHIPPRIORITY AS LOSHIPPRIORITY,
    l.LOQUANTITY AS LOQUANTITY,
    l.LOEXTENDEDPRICE AS LOEXTENDEDPRICE,
    l.LOORDTOTALPRICE AS LOORDTOTALPRICE,
    l.LODISCOUNT AS LODISCOUNT,
    l.LOREVENUE AS LOREVENUE,
    l.LOSUPPLYCOST AS LOSUPPLYCOST,
    l.LOTAX AS LOTAX,
    l.LOCOMMITDATE AS LOCOMMITDATE,
    l.LOSHIPMODE AS LOSHIPMODE,
    c.CNAME AS CNAME,
    c.CADDRESS AS CADDRESS,
    c.CCITY AS CCITY,
    c.CNATION AS CNATION,
    c.CREGION AS CREGION,
    c.CPHONE AS CPHONE,
    c.CMKTSEGMENT AS CMKTSEGMENT,
    s.SNAME AS SNAME,
    s.SADDRESS AS SADDRESS,
    s.SCITY AS SCITY,
    s.SNATION AS SNATION,
    s.SREGION AS SREGION,
    s.SPHONE AS SPHONE,
    p.PNAME AS PNAME,
    p.PMFGR AS PMFGR,
    p.PCATEGORY AS PCATEGORY,
    p.PBRAND AS PBRAND,
    p.PCOLOR AS PCOLOR,
    p.PTYPE AS PTYPE,
    p.PSIZE AS PSIZE,
    p.PCONTAINER AS PCONTAINER
FROM ssb.lineorder_all AS l
INNER JOIN ssb.customer_all AS c ON c.CCUSTKEY = l.LOCUSTKEY
INNER JOIN ssb.supplier_all AS s ON s.SSUPPKEY = l.LOSUPPKEY
INNER JOIN ssb.part_all AS p ON p.PPARTKEY = l.LOPARTKEY;

show create table flat_table;

CREATE TABLE ssb.lineorder_flat on cluster test_cluster_three_shards
(
  `LOORDERKEY` UInt32,
  `LOLINENUMBER` UInt8,
  `LOCUSTKEY` UInt32,
  `LOPARTKEY` UInt32,
  `LOSUPPKEY` UInt32,
  `LOORDERDATE` Date,
  `LOORDERPRIORITY` LowCardinality(String),
  `LOSHIPPRIORITY` UInt8,
  `LOQUANTITY` UInt8,
  `LOEXTENDEDPRICE` UInt32,
  `LOORDTOTALPRICE` UInt32,
  `LODISCOUNT` UInt8,
  `LOREVENUE` UInt32,
  `LOSUPPLYCOST` UInt32,
  `LOTAX` UInt8,
  `LOCOMMITDATE` Date,
  `LOSHIPMODE` LowCardinality(String),
  `CNAME` String,
  `CADDRESS` String,
  `CCITY` LowCardinality(String),
  `CNATION` LowCardinality(String),
  `CREGION` LowCardinality(String),
  `CPHONE` String,
  `CMKTSEGMENT` LowCardinality(String),
  `SNAME` String,
  `SADDRESS` String,
  `SCITY` LowCardinality(String),
  `SNATION` LowCardinality(String),
  `SREGION` LowCardinality(String),
  `SPHONE` String,
  `PNAME` String,
  `PMFGR` LowCardinality(String),
  `PCATEGORY` LowCardinality(String),
  `PBRAND` LowCardinality(String),
  `PCOLOR` LowCardinality(String),
  `PTYPE` LowCardinality(String),
  `PSIZE` UInt8,
  `PCONTAINER` LowCardinality(String)
)
ENGINE = MergeTree
PARTITION BY toYear(LOORDERDATE)
ORDER BY (LOORDERDATE, LOORDERKEY)
SETTINGS index_granularity = 8192;
CREATE TABLE ssb.lineorder_flat_all  on cluster test_cluster_three_shards AS ssb.lineorder_flat 
ENGINE = Distributed(test_cluster_three_shards, ssb, lineorder_flat, rand());
insert into lineorder_flat_all select * from flat_table;
```
- Q1.1
```sql
SELECT sum(LOEXTENDEDPRICE * LODISCOUNT) AS revenue
FROM lineorder_flat_all
WHERE (toYear(LOORDERDATE) = 1993) AND ((LODISCOUNT >= 1) AND (LODISCOUNT <= 3)) AND (LOQUANTITY < 25);
```
- Q1.2
```sql
SELECT sum(LOEXTENDEDPRICE * LODISCOUNT) AS revenue
FROM lineorder_flat_all
WHERE (toYYYYMM(LOORDERDATE) = 199401) AND ((LODISCOUNT >= 4) AND (LODISCOUNT <= 6)) AND ((LOQUANTITY >= 26) AND (LOQUANTITY <= 35));
```
- Q1.3
```sql
SELECT sum(LOEXTENDEDPRICE * LODISCOUNT) AS revenue
FROM lineorder_flat_all
WHERE (toISOWeek(LOORDERDATE) = 6) AND (toYear(LOORDERDATE) = 1994) AND ((LODISCOUNT >= 5) AND (LODISCOUNT <= 7)) AND ((LOQUANTITY >= 26) AND (LOQUANTITY <= 35));
```
- Q2.1
```sql
SELECT
  sum(LOREVENUE),
  toYear(LOORDERDATE) AS year,
  PBRAND
FROM lineorder_flat_all
WHERE (PCATEGORY = 'MFGR#12') AND (SREGION = 'AMERICA')
GROUP BY
  year,
  PBRAND
ORDER BY
  year ASC,
  PBRAND ASC
```
- Q2.2
```sql
SELECT
  sum(LOREVENUE),
  toYear(LOORDERDATE) AS year,
  PBRAND
FROM lineorder_flat_all
WHERE (PBRAND >= 'MFGR#2221') AND (PBRAND <= 'MFGR#2228') AND (SREGION = 'ASIA')
GROUP BY
  year,
  PBRAND
ORDER BY
  year ASC,
  PBRAND ASC
```
- Q2.3
```sql
SELECT
  sum(LOREVENUE),
  toYear(LOORDERDATE) AS year,
  PBRAND
FROM lineorder_flat_all
WHERE (PBRAND = 'MFGR#2239') AND (SREGION = 'EUROPE')
GROUP BY
  year,
  PBRAND
ORDER BY
  year ASC,
  PBRAND ASC
```
- Q3.1
```sql
SELECT
  CNATION,
  SNATION,
  toYear(LOORDERDATE) AS year,
  sum(LOREVENUE) AS revenue
FROM lineorder_flat_all
WHERE (CREGION = 'ASIA') AND (SREGION = 'ASIA') AND (year >= 1992) AND (year <= 1997)
GROUP BY
  CNATION,
  SNATION,
  year
ORDER BY
  year ASC,
  revenue DESC
```
- Q3.2
```sql
SELECT
  CCITY,
  SCITY,
  toYear(LOORDERDATE) AS year,
  sum(LOREVENUE) AS revenue
FROM lineorder_flat_all
WHERE (CNATION = 'UNITED STATES') AND (SNATION = 'UNITED STATES') AND (year >= 1992) AND (year <= 1997)
GROUP BY
  CCITY,
  SCITY,
  year
ORDER BY
  year ASC,
  revenue DESC
```
- Q3.3
```sql
SELECT
  CCITY,
  SCITY,
  toYear(LOORDERDATE) AS year,
  sum(LOREVENUE) AS revenue
FROM lineorder_flat_all
WHERE ((CCITY = 'UNITED KI1') OR (CCITY = 'UNITED KI5')) AND ((SCITY = 'UNITED KI1') OR (SCITY = 'UNITED KI5')) AND (year >= 1992) AND (year <= 1997)
GROUP BY
  CCITY,
  SCITY,
  year
ORDER BY
  year ASC,
  revenue DESC
```
- Q3.4
```sql
SELECT
  CCITY,
  SCITY,
  toYear(LOORDERDATE) AS year,
  sum(LOREVENUE) AS revenue
FROM lineorder_flat_all
WHERE ((CCITY = 'UNITED KI1') OR (CCITY = 'UNITED KI5')) AND ((SCITY = 'UNITED KI1') OR (SCITY = 'UNITED KI5')) AND (toYYYYMM(LOORDERDATE) = 199712)
GROUP BY
  CCITY,
  SCITY,
  year
ORDER BY
  year ASC,
  revenue DESC
```
- Q4.1
```sql
SELECT
  toYear(LOORDERDATE) AS year,
  CNATION,
  sum(LOREVENUE - LOSUPPLYCOST) AS profit
FROM lineorder_flat_all
WHERE (CREGION = 'AMERICA') AND (SREGION = 'AMERICA') AND ((PMFGR = 'MFGR#1') OR (PMFGR = 'MFGR#2'))
GROUP BY
  year,
  CNATION
ORDER BY
  year ASC,
  CNATION ASC
```
- Q4.2
```sql
SELECT
  toYear(LOORDERDATE) AS year,
  SNATION,
  PCATEGORY,
  sum(LOREVENUE - LOSUPPLYCOST) AS profit
FROM lineorder_flat_all
WHERE (CREGION = 'AMERICA') AND (SREGION = 'AMERICA') AND ((year = 1997) OR (year = 1998)) AND ((PMFGR = 'MFGR#1') OR (PMFGR = 'MFGR#2'))
GROUP BY
  year,
  SNATION,
  PCATEGORY
ORDER BY
  year ASC,
  SNATION ASC,
  PCATEGORY ASC
```
- Q4.3
```sql
SELECT
  toYear(LOORDERDATE) AS year,
  SCITY,
  PBRAND,
  sum(LOREVENUE - LOSUPPLYCOST) AS profit
FROM lineorder_flat_all
WHERE (SNATION = 'UNITED STATES') AND ((year = 1997) OR (year = 1998)) AND (PCATEGORY = 'MFGR#14')
GROUP BY
  year,
  SCITY,
  PBRAND
ORDER BY
  year ASC,
  SCITY ASC,
  PBRAND ASC
```
# 并发测试
```sql
SELECT
  toYear(LOORDERDATE) AS year,
  CNATION,
  sum(LOREVENUE - LOSUPPLYCOST) AS profit
FROM lineorder_flat_all
WHERE (CREGION = 'AMERICA') AND (SREGION = 'AMERICA') AND ((PMFGR = 'MFGR#1') OR (PMFGR = 'MFGR#2'))
GROUP BY
  year,
  CNATION
ORDER BY
  year ASC,
  CNATION ASC;
```
## benchmark并发测试
```sh
./clickhouse benchmark --host=host --port=9000 --user=xxx --password=xxx --database=ssb --concurrency=1 --iterations=1 < test-ck.sql
./clickhouse benchmark --host=host --port=9000 --user=xxxx --password=xxxx --database=ssb --concurrency=10 --iterations=10 < test-ck.sql
./clickhouse benchmark --host=host --port=9000 --user=xxx --password=xxx--database=ssb --concurrency=100 --iterations=100 < test-ck.sql
```
- Q3.3
```sql
```