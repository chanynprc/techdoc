## Greenplum优化器代码走读

### 关键数据结构

- Query：优化器的输入，关系代数表达式树
- Plan：优化器的输出，执行计划

### PostgreSQL Planner处理流程及关键函数

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

### GPDB调用GPORCA的过程

从GP侧的优化器入口开始，逐层进行调用：

- planner
- standard_planner
- optimize_query (orca.c)
- GPOPTOptimizedPlan (CGPOptimizer.cpp)
- CGPOptimizer::GPOPTOptimizedPlan (CGPOptimizer.cpp)
- COptTasks::GPOPTOptimizedPlan (COptTasks.cpp)
- COptTasks::Execute (COptTasks.cpp)

以上处理流程只是做一些基础的准备工作基于try-catch，下面将进入主逻辑调用函数：

- COptTasks::OptimizeTask (COptTasks.cpp)

在这个函数中做主要的处理流程。其内部逻辑如下：

1. 处理metadata的cache，是否需要被初始化、重置或调整大小
2. 加载search strategy（LoadSearchStrategy）
3. 加载metadata accessor
4. 设置Cost Model
5. 进行参数设置
6. 将Query结构转换为DXL结构
7. 调用COptimizer::PdxlnOptimize生成执行计划（DXL结构）
8. 根据需要，将执行计划序列化为XML格式以及PlannedStmt结构体
9. 输出缺失的统计信息

在这个函数中，将调用ORCA中的COptimizer::PdxlnOptimize进行计划生成。后续流程将进入ORCA代码。

### GPORCA处理流程及关键函数

- COptimizer::PdxlnOptimize：总入口，在GP中被调用

在此函数中，处理流程如下：

1. 根据mini dump参数，初始化mini dump
2. 初始化优化器上下文，包含metadata accessor、expr evaluator、config参数
3. 将DXL转换为CExpression
4. 处理CTE
5. 调用COptimizer::PexprOptimize，将logical expression tree转换为physical expression tree（主要的计划生成逻辑）
6. 将CExpression转换为DXL
7. 处理mini dump相关逻辑
8. 清理空间并返回

- COptimizer::PexprOptimize：将logical expression tree转换为physical expression tree，函数较简单，在其内部初始化一个CEngine，并进行优化调度
- CEngine::Optimize：初始化Job、Scheduler等结构，并且根据每一层的search stage生成一个root group，并且进行任务（Job）的调度（Scheduler）并选择该层的最优计划
- CScheduler::Run：简单的调用逻辑，直接调用CScheduler::ExecuteJobs
- CScheduler::ExecuteJobs：一个循环，一个个执行队列中的Job，这些Job可能是CJobGroup及其子类、CJobGroupExpression及其子类以及CJobTransformation，并对任务执行结果进行判断（完成、挂起等状态）
- CScheduler::FExecute：调度队列中的任务。在此函数中，会调用相应具体Job的FExecute
- CJobXXX::FExecute：具体Job的FExecute，在此函数中会调用状态机中的FRun方法
- CJobStateMachine::FRun：此方法中获取处理当前状态需要的函数，并执行。这类处理函数在各个CJobXXX类之中以EevtXXX形式呈现

在CJobTransformation中，状态机会调用如下的函数：

- CJobTransformation::EevtTransform
- CGroupExpression::Transform：在此函数中会调用被分配的Xform的Transform方法

### GPORCA关键类

CTask

COptimizer

CEngine

CScheduler

CJob
 -> CJobGroup
     -> CJobGroupExploration
     -> CJobGroupImplementation
     -> CJobGroupOptimization
 -> CJobGroupExpression
     -> CJobGroupExpressionExploration
     -> CJobGroupExpressionImplementation
     -> CJobGroupExpressionOptimization
 -> CJobTransformation

CXform
 -> CXformExploration
 -> CXformImplementation

CGroup

CGroupExpression

COperator
 -> CLogical
 -> CPhysical
 -> CScalar

CCostModelGPDB



