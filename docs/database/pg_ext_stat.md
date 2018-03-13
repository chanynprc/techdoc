## PostgreSQL's Extended Statistics

### 背景

当查询过滤条件中的多个属性列存在相关性时，会导致过滤后行数估算不准的情况，从而生成较差执行计划，导致查询语句执行缓慢。

优化器一般会认为多个条件之间是相互独立的，不能处理多个属性存在相关性的情况。传统的统计信息是面向单个属性的，我们无法从中获取属性之间相关性的信息。

PostgreSQL在10.0版本中，提供了Extended Statistics特性，考虑了多个属性之间了相关性以及多列distinct值。

### 支持的统计信息

#### Functional Dependencies

在关系数据理论中，关系应该满足一定的要求，从而有1NF~5NF的范式等级。函数依赖应该在模式的分解中被消除，而只存在于主键（primary keys）和超键（superkeys）之间。但是，在实际情况下，很多数据库中都存在部分的函数依赖。

函数依赖会使得行数估算收到影响。如果一个查询的过滤条件包含存在函数依赖的多个列，那么依赖于其他列的列上的条件就不会很明显地进行结果集过滤，如果我们并不知道函数依赖的信息，则可能对结果集行数进行低估。

#### Multivariate N-Distinct Counts

比较容易理解，就是多个列的distinct值。

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

#### 收集统计信息：Functional Dependencies

假设有关系R(a,b)，PostgreSQL 10.0的实现方式是统计“a=>b”和“b=>a”的函数依赖比例。所谓函数依赖比例，以“a=>b”为例，是指a直接决定了唯一的b的行数的比例。

```
关系R(a,b,...,z)，计算a,b,...,z列的函数依赖关系，共n个列

for k = 2..n
    生成(k-1个列 => 1个列)的函数依赖列表，共k个
	foreach (k-1个列 => 1个列) in 函数依赖列表
		将采样的数据保存到内存
		对采样的数据进行排序
		对前k-1列进行聚集，最后1列进行distinct值计数
		若某一聚集的最后1列的distinct值计数为1，则记下此聚集在做聚集前的行数，加到计数行数
		函数依赖比例 = 计数行数 / 总行数
``` 

#### 收集统计信息：Multivariate N-Distinct Counts

```
关系R(a,b,...,z)，计算a,b,...,z列的多列distinct值，共n个列


```

### 统计信息的使用

#### Functional Dependencies

#### Multivariate N-Distinct Counts

### 引用

[1] https://www.postgresql.org/docs/current/static/planner-stats.html#PLANNER-STATS-EXTENDED