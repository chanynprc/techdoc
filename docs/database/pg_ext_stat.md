## PostgreSQL's Extended Statistics

### 背景

当查询过滤条件中的多个列存在相关性时，会导致过滤后行数估算不准的情况，从而生成较差执行计划，导致查询语句执行缓慢。

优化器一般会认为多个条件之间是相互独立的

### 引用

[1] https://www.postgresql.org/docs/current/static/planner-stats.html#PLANNER-STATS-EXTENDED