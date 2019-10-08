## ClickHouse Overview

### 存储结构

- 数据文件存储于`/var/lib/clickhouse/data`目录
- 数据文件目录下按照database级别划分目录，每个database一个目录。如有两个database：default和system，则data目录下有两个文件夹default和system
- database目录下按照table级别划分目录，每个table一个目录。目录名为table名
- table目录下按照partition级别划分目录，每个partition一个目录。目录名为一个四元组组合成的字符串
- partition目录下按照column级别划分文件，每个column两个文件，分别为`columnname.bin`文件和`columnname.mrk2`文件，此外还包含`checksums.txt`、 `columns.txt`、 `count.txt`、 `minmax_b.idx`、 `partition.dat`、 `primary.idx`文件

### 写入逻辑

- 数据写入时，会新建一个partition目录，将新数据写入到该目录的column文件中
- 后台会定期做`optimize`操作，将需要合并的partition合并到一起，`optimize`操作也可以手动触发。合并后，原被合并分区会被标记非active
- 后台会定期清理非active的partition的目录

### 基本操作

查看集群情况

```sql
SELECT * FROM system.clusters;
```

查看partition情况

```sql
SELECT table, name, partition, active
FROM system.parts
WHERE database='default';
```

建表

```sql
CREATE TABLE t3 (a int, b int, c int, d int)
ENGINE = MergeTree()
PARTITION BY b
ORDER BY b;

CREATE TABLE t3_d AS t3
ENGINE = Distributed(test_cluster_two_shards_localhost, default, t3, a);
```
