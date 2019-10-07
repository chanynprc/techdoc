## ClickHouse Overview

### 基本操作

建表

```sql
CREATE TABLE t3 (`a` Int32, `b` Int32, `c` Int32, `d` Int32)
ENGINE = MergeTree()
PARTITION BY b
ORDER BY b;
```
