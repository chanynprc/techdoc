## Greenplum ORCA Overview

本文基于Greenplum 6.0 beta和ORCA v3.52.0。

### 组件

GPORCA有4个组件：

- libnaucrates：包含DXL相关的类、统计信息相关的类
- libgpopt：包含优化器引擎、元数据访问、逻辑/物理算子、变换规则、DXL转换器的代码
- libgpdbcost：包含代价模型代码
- libgpos：包含内存申请、调度、错误处理、测试框架等

### 入口

在Greenplum中，从优化器的standard_planner函数中，对optimizer参数等进行了判断。如果满足使用ORCA的条件，则调用optimize_query函数，这是ORCA在Greenplum中的主入口。在Greenplum侧，调用关系比较简单，调用顺序和关系如下：

- optimize_query：在optimize_query函数中，
	1. 对Query树进行处理（preprocess_query_optimizer），目前只是进行常量折叠
	1. 开始优化流程（GPOPTOptimizedPlan）
	1. 记录log信息（log_optimizer）
	1. 还有一系列的后续处理
- GPOPTOptimizedPlan：此函数是C函数，调用CGPOptimizer::GPOPTOptimizedPlan这个C++方法
- CGPOptimizer::GPOPTOptimizedPlan：此方法主要调用COptTasks::GPOPTOptimizedPlan方法，其次做一些异常的catch工作
- COptTasks::GPOPTOptimizedPlan：此方法中使用COptTasks::Execute调用COptTasks::OptimizeTask
- COptTasks::OptimizeTask：此方法做一系列与ORCA交互前的处理，调用ORCA生成执行计划，并且做了后续处理
	1. 初始化元数据缓存（根据情况进行初始化、重置或变更大小）
	1. 加载搜索策略
	1. 初始化代价模型、优化器配置等参数
	1. 将Query树转换为DXL
	1. 调用COptimizer::PdxlnOptimize方法，基于Query的DXL，调用ORCA，生成计划的DXL
	1. 将计划的DXL转换成需要的PlannedStmt结构或XML格式

其中COptimizer::PdxlnOptimize是ORCA的代码，为Greenplum调用ORCA的入口。

