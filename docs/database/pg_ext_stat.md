## PostgreSQL's Extended Statistics

### 背景

当查询过滤条件中的多个属性列存在相关性时，会导致过滤后行数估算不准的情况，从而生成较差执行计划，导致查询语句执行缓慢。

优化器一般会认为多个条件之间是相互独立的，不能处理多个属性存在相关性的情况。传统的统计信息是面向单个属性的，我们无法从中获取属性之间相关性的信息。

PostgreSQL在10.0版本中，提供了Extended Statistics特性，考虑了多个属性之间了相关性以及多列distinct值。

### 统计信息的收集

由于列之间的组合数非常巨大，不能让统计信息收集模块自动收集多属性列的统计信息。PostgreSQL的Extended Statistics通过先指定，再收集的方式进行。

#### 指定收集统计信息

PostgreSQL添加了一个新的语句，用于指定需要收集的统计信息：

```sql
CREATE STATISTICS [ IF NOT EXISTS ] statistics_name
[ ( statistics_kind [, ... ] ) ]
ON column_name, column_name [, ...]
FROM table_name
```

当CREATE STATISTICS后，会在pg_statistic_ext系统表中添加一行，用于标记需要收集的统计信息，但是真正的统计信息并未被收集，还需要用户运行ANALYZE后，才会收集统计信息并存入系统表。


#### 收集统计信息

### 统计信息的使用

### 引用

[1] https://www.postgresql.org/docs/current/static/planner-stats.html#PLANNER-STATS-EXTENDED