## Presto概览

Presto是一个开源的分布式SQL引擎。它主要面向大数据量的数据仓库分析服务。

Presto并不是一个传统意义上的数据库，Presto被设计作为HDFS上的查询分析工具，可以替代Hive和Pig。但是，Presto并不仅仅只用于HDFS的访问，它的数据还可以来自关系型数据库和类似Cassandra等的数据源。Presto被主要用于OLAP场景。

### 节点类型

Presto有两种节点，coordinator和worker。

- **Coordinator**：```Coordinator```节点的任务是对查询语句进行解析、计划生成，以及对```Worker```节点进行管理。它是Presto的控制单元，也同时负责接收来自客户端的查询请求，此外，还负责将```Worker```的执行结果返回给客户端。

- **Worker**：```Worker```节点的任务是执行```Task```并处理数据。它从```Connnector```拿到数据，执行相应逻辑，与其他```Worker```交换数据。

客户端与```Coordinator```之间、```Coordinator```与```Worker```之间，以及```Worker```与```Worker```之间的通信使用的是REST API。

### 数据源

### 查询执行模型

### 引用

[1] https://prestodb.io/docs/current/index.html


