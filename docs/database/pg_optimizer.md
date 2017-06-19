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
{
    planner
}
```

无实质处理，调用planner函数进行处理。

```cpp
PlannedStmt *planner(Query *parse,
                     int cursorOptions,
                     ParamListInfo boundParams)
{
    planner_hook
    pgxc_planner
    standard_planner
}
```

无实质处理，根据条件调用不同的计划生成接口。

```cpp
PlannedStmt *pgxc_planner(Query *query,
                          int cursorOptions,
                          ParamListInfo boundParams)
{
    pgxc_FQS_planner
    standard_planner
}
```

无实质处理，调用pgxc_FQS_planner生成FQS的计划，如果未生成FQS计划，则调用standard_planner进行处理。

```cpp
PlannedStmt *standard_planner(Query *parse,
                              int cursorOptions,
                              ParamListInfo boundParams)
{
    subquery_planner
}
```

构造PlannerGlobal，调用subquery_planner生成计划树，构造PlannedStmt并返回。

```cpp
Plan *subquery_planner(PlannerGlobal *glob,
                       Query *parse,
                       PlannerInfo *parent_root,
                       bool hasRecursion,
                       double tuple_fraction,
                       PlannerInfo **subroot)
{
    inheritance_planner
    grouping_planner
}
```

subquery_planner构造生成计划的主入口，它被每一个`SELECT`递归调用。

首先，构造PlannerInfo结构，其次进行一系列查询重写（如CTE处理、子链接提升、子查询提升）和表达式处理，然后调用inheritance_planner或grouping_planner生成计划，最后进行一些收尾处理后返回Plan。

### 主要数据结构

```cpp
Query
PlannedStmt
PlannerGlobal
PlannerInfo
```

### 常见缩写

```cpp
SS
bms
```