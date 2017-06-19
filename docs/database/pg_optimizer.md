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

```cpp
static Plan *inheritance_planner(PlannerInfo *root)
{
    grouping_planner
}
```

```cpp
static Plan *grouping_planner(PlannerInfo *root,
                              double tuple_fraction)
{
    query_planner
    create_plan
}
```

进行grouping、aggregation等的处理。并且，调用query_planner生成cheapest_path和sorted_path，然后选择best_path，并调用create_plan生成计划。

```cpp
RelOptInfo *query_planner(PlannerInfo *root,
                          List *tlist,
                          query_pathkeys_callback qp_callback,
                          void *qp_extra)
{
    //为simple_rel_array、simple_rte_array申请空间
    //初始化simple_rte_array
    setup_simple_rel_arrays

    //根据join tree，构造基表的RelOptInfo，并存于simple_rel_array
    add_base_rels_to_query

    make_one_rel
}
```

生成path。并不生成最优路径（best path），而是生成考虑了join的cheapest_path和sorted_path。

如果查询无`FROM`子句，直接生成路径并返回。否则，进行一系列预处理，并调用make_one_rel生成最终RelOptInfo，包含cheapest_path和sorted_path等信息。

```cpp
Plan *create_plan(PlannerInfo *root,
                  Path *best_path)
{
}
```

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