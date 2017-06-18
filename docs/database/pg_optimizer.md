## Postgres-XL优化器概述

### 代码结构

#### pg_plan_queries

输入：

- List *querytrees：
- int cursorOptions：
- ParamListInfo boundParams：

输出：

- List *：

调用：

- pg_plan_query

说明：

对输入的每一个query tree调用pg_plan_query进行处理（Utility命令除外）。



### 引用

