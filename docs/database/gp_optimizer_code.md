## Greenplum优化器代码走读

### 关键数据结构

- Query：优化器的输入，关系代数表达式树
- Plan：优化器的输出，执行计划

### 关键函数

- planner：优化器的入口函数。在此函数中选择使用用户自定义优化器或自带优化器
- standard_planner：自带优化器的入口函数。在此函数中判断是否使用ORCA，若不使用，继续走PostgreSQL Planner流程
- subquery_planner：查询块处理入口函数。负责查询重写、表达式预处理等，调用Upper Path的处理流程
- grouping_planner：Upper Path的处理流程入口。在处理过程中会调用Join Path的处理流程
- query_planner：Join Path的处理流程入口。做各种初始化的工作，然后调用make_one_rel进行路径生成
- make_one_rel：生成路径的入口。在其内部生成基表路径，以及Join的路径
- set_base_rel_pathlists：生成基表路径的入口
- make_rel_from_joinlist：生成Join的路径的入口。生成第一层的路径，并调用后续Join逻辑（可自己实现）
- standard_join_search：动态规划路径生成框架入口
- join_search_one_level：每一层动态规划路径生成入口
- make_join_rel：生成一个Join的路径。在这里分Join顺序进行代价计算
- add_paths_to_joinrel：生成某种Join顺序下各种Join实现路径
- sort_inner_and_outer、match_unsorted_outer、hash_inner_and_outer：根据Join的不同形式，对分支进行处理，然后尝试各种Join实现
- try_nestloop_path、try_mergejoin_path、try_hashjoin_path：生成各种Join的路径
- create_plan：根据生成的路径转换成计划

