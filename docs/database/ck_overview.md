## ClickHouse Overview

### 存储结构

- 数据文件存储于`/var/lib/clickhouse/data`目录
- 数据文件目录下按照database级别划分目录，每个database一个目录。如有两个database：default和system，则data目录下有两个文件夹default和system
- database目录下按照table级别划分目录，每个table一个目录。目录名为table名
- table目录下按照partition级别划分目录，每个partition一个目录。目录名为一个有规律的组合字符串
- partition目录下按照column级别划分文件，每个column两个文件，分别为`columnname.bin`文件和`columnname.mrk2`文件，此外还包含`checksums.txt`、`columns.txt`、`count.txt`、`minmax_b.idx`、`partition.dat`、`primary.idx`文件

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
