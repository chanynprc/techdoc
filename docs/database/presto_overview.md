## Presto概览

Presto是一个开源的分布式SQL引擎。它主要面向大数据量的数据仓库分析服务。

Presto并不是一个传统意义上的数据库，Presto被设计作为HDFS上的查询分析工具，可以替代Hive和Pig。但是，Presto并不仅仅只用于HDFS的访问，它的数据还可以来自关系型数据库和类似Cassandra等的数据源，通过其丰富的连接器与这些数据源进行连接。Presto被主要用于OLAP场景。

Presto的架构图如下：

![](/techdoc/docs/database/images/presto-overview.png)

### 节点类型

Presto有两种节点，Coordinator和Worker。

- **Coordinator**：```Coordinator```节点的任务是对查询语句进行解析、计划生成，以及对```Worker```节点进行管理。它是Presto的控制单元，也同时负责接收来自客户端的查询请求，此外，还负责将```Worker```的执行结果返回给客户端

- **Worker**：```Worker```节点的任务是执行```Task```并处理数据。它从```Connnector```拿到数据，执行相应逻辑，与其他```Worker```交换数据

客户端与```Coordinator```之间、```Coordinator```与```Worker```之间，以及```Worker```与```Worker```之间的通信使用的是REST API。

### 数据源

Presto的数据源有这么几个概念：```Connector```、```Catalog```、```Schema```、```Table```。

- **Connector**：```Connector```是指Presto的数据源连接器，每一种数据源有一个```Connector```与之对应
- **Catalog**：```Catalog```可以理解为```Connector```的实例，它的一个重要属性是其```Connector```的名字。在Presto中，一个查询可以跨```Catalog```
- **Schama**：```Schema```类似于数据库中scheme的概念，是一系列表的集合
- **Table**：数据表。在Presto中，数据表的完整表示形式是catalog_name.schema_name.table_name

### 查询执行模型

- **Statement**：指的是符合ANSI的SQL语句
- **Query**：指Statement经过Presto处理后的内部数据结构（包括Query Plan在内），包含执行该Statement所需的更多的信息
- **Stage**：指可在单个Worker中执行的一个分布式Query Plan片段。一个Query Plan可能由多个Stage一层一层叠加在一起组成
- **Task**：是Stage的实体，被分配到多个Worker上执行。Stage在分布式系统中被展开成Task就是分布式系统的MPP并行的概念
- **Split**：即分片，是对数据的划分，不同的Task工作在不同的数据Split上，达到并行计算的目的
- **Driver**：Driver是Task内部并行计算的概念（SMP），一个Task可以包含多个Driver并行执行
- **Operator**：即算子。其中，Exchange算子负责在stage间传递数据（Shuffle）

### 引用

[0] https://prestodb.io/overview.html

[1] https://prestodb.io/docs/current/index.html


