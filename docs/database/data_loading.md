## 数据导入



### Starrocks的数据导入

| 导入方式        | 同步、异步 | 串行、并行                       | 主要场景                   | 技术原理                                                     |
| --------------- | ---------- | -------------------------------- | -------------------------- | ------------------------------------------------------------ |
| INSERT+FILES()  | 同步       | 串行，从FE导入                   | 从云存储或HDFS中导入文件   |                                                              |
| Broker Load     | 异步       | 并行，从BE导入                   | 文件较多、较大场景         | 各个 BE 从数据源拉取数据并把数据导入到 StarRocks 中          |
| Pipe            | 持续异步   | 串行，从FE导入                   | 大规模导入、不间断导入场景 | 持续的INSERT+FILES() 大量文件分批导入，在多个事务中持续导入，并可以监听文件变化，导入新文件，中间结果用户可见 |
| Stream Load     | 同步       | 串行，单BE作为Coordinator BE导入 | 文件较少、较小场景         | 基于HTTP PUT，非SQL 作业请求提交到FE，轮询选定BE执行作业     |
| Kafka connector |            |                                  | 消费Kafka                  |                                                              |
| Routine Load    |            |                                  |                            |                                                              |
| Spark connector |            |                                  |                            |                                                              |
| Spark Load      |            |                                  |                            |                                                              |

### Snowflake的数据导入

Snowflake的数据导入，都是基于数据文件的，数据文件可以存放于2种不同的位置：

- 外部存储（external stage）：可以从AWS S3、Google Cloud Storage、Microsoft Azure中加载数据，但不能加载存储在归档存储中的数据（Amazon S3 Glacier Flexible Retrieval、Glacier Deep Archive storage class、Microsoft Azure Archive Storage），此存储位置可以使用CREATE STAGE去创建。
- 内部存储（internal stage）：在Snowflake的账户中维护了几个存储数据的位置，包括用户（分别分配给了每个用户，用于单个用户的文件暂存和管理，不支持更新和删除）、表（分别分配给了每个表，可多个用户共用，不支持更新和删除）、命名（Named，可以被多个用户暂存和管理加到到多个表的文件，此方式可以使用CREATE STAGE去创建）。

Snowflake提供批量导入和连续加载的导入方式：

- 批量导入：可以使用COPY命令进行批量导入。COPY的数据来源可以来自外部存储和内部存储中的文件。在COPY的过程中，可以对数据进行基本的转换，比如列的重新排序、列省略、类型转换、截断超出目标列长度的文本字符串，数据文件无需与目标表有相同数量和顺序的列。
- 连续加载（Snowpipe）：使用Snowpipe进行连续加载，旨在加载微批数据。在数据被添加到stage并提交到ingestion后的几分钟内，Snowpipe会加载数据。在内部的实现也是用COPY，在数据转换能力方面和批量导入一致。
- 连续加载（Snowpipe Streaming）：Snowpipe Streaming可以让数据直接将数据写入到Snowflake的数据表中，无需落到stage中，可以支持更低的延迟，使Snowflake具备处理近乎实时的数据的能力。此外，Snowpipe Streaming可以和Snowflake Connector for Kafka结合。

Snowflake还支持联邦查询去直接查询外部数据，无需将数据导入到Snowflake中：

- 外表：数据可以存储在外部云存储中，可以在这些数据的子集上创建物化视图，以提高查询性能。













