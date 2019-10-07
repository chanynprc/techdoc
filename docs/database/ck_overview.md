## ClickHouse Overview

### 基本操作

查看集群情况

```sql
select * from system.clusters;
```

建表

```sql
CREATE TABLE t3 (a int, b int, c int, d int)
ENGINE = MergeTree()
PARTITION BY b
ORDER BY b;
```
