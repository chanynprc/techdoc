## Postgres-XL优化器概述

### 主要函数

```cpp
List *pg_plan_queries(List *querytrees,
                      int cursorOptions,
                      ParamListInfo boundParams)
{
    pg_plan_query
}
```

无实质处理，对输入的每一个query tree调用pg_plan_query进行处理（Utility命令除外）。

```cpp
PlannedStmt *pg_plan_query(Query *querytree,
                           int cursorOptions,
                           ParamListInfo boundParams)
```

输入：
- Query *querytree
- int cursorOptions
- ParamListInfo boundParams

输出：
- PlannedStmt *

调用：
- planner

说明：

无实质处理，调用planner函数进行处理

```cpp
PlannedStmt *planner(Query *parse,
                     int cursorOptions,
                     ParamListInfo boundParams)
```

输入：
- Query *parse
- int cursorOptions
- ParamListInfo boundParams

输出：
- PlannedStmt *

调用：
- _planner_hook_
- pgxc_planner
- standard_planner

说明：

无实质处理，根据条件调用不同的计划生成接口

```cpp
PlannedStmt *pgxc_planner(Query *query,
                          int cursorOptions,
                          ParamListInfo boundParams)
```

输入：
- Query *query
- int cursorOptions
- ParamListInfo boundParams

输出：
- PlannedStmt *

调用：
- pgxc_FQS_planner
- standard_planner

说明：

无实质处理，调用pgxc_FQS_planner生成FQS的计划，如果未生成FQS计划，则调用standard_planner进行处理



### 引用

