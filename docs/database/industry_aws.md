## 竞品分析：AWS

### 产品简述

流计算产品：

- Amazon Kinesis Data Streams：一种流服务，对标Apache Kafka、Amazon MSK，用于收集和存储数据流。数据源是应用程序、日志、网络点击数据、IoT数据等。
- Amazon Kinesis Data Firehose：将数据移动、加载到数据存储中。下游有S3、Redshift、Open-Search、Amazon Kinesis Data Analytics。
- Amazon Kinesis Data Analytics：一种流处理技术，对标Apache Flink、Apache Kafka Streams（嵌入Kafka的Java库），使用Data Analytics Studio或Apache Flink分析数据流。数据源是Amazon Kinesis Data Streams、Amazon MSK、Amazon MQ等。
- Amazon Managed Streaming for Apache Kafka (MSK)：AWS上托管的Kafka服务。

数据库、数据仓库产品：

- Amazon Aurora：关系型数据库
- Amazon Redshift：数据仓库
- Amazon DynamoDB：键值数据库
- Amazon EMR
- Amazon OpenSearch Service：日志和搜索分析
- Amazon Athena：交互式查询，用于联邦查询和数据湖分析，使用SQL分析存储于S3中的数据

生态工具产品：

- Amazon Glue：一个数据集成系统。
- Amazon AppFlow：在应用程序、S3、Redshift间安全地传输数据，打破数据孤岛
- Amazon Lambda：有轮询事件、自动通知的功能。
- 
- Amazon SageMaker Data Wrangler：大规模构建、训练、部署机器学习模型

### Amazon Redshift

部分新特性及值得关注的特性列表：

- 【订阅Kafka】：Redshift支持订阅Kafka上的数据，可以通过`CREATE EXTERNAL SCHEMA evdata FROM KINESIS`语句映射Kinesis上的数据，并通过`CREATE MATERIALIZED VIEW ev_station_data AUTO REFRESH YES AS SELECT xxx FROM evdata.ev_station_data`语句创建物化视图去引用数据，物化视图带上`AUTO REFRESH YES`则会自动刷新，如果不带则需要使用`REFRESH MATERIALIZED VIEW ev_station_data`去刷新，刷新时会从Kafka中读取数据并将数据加载到化视图中。

- 【资源管理】：Redshift支持手动WLM和自动WLM。手动WLM可以对内存和并发度进行管理，无法对CPU进行管理，可以给每个组分配一定的内存比例，组内各个并发槽再平分内存。自动WLM只能设置组的查询优先级，共5个普通级别（HIGHEST、HIGH、NORMAL、LOW、LOWEST，此外还有1个用于超级用户的级别CRITICAL），会根据负载情况动态调整各个组内的执行并发度。此外，还有一些其他特性：（1）Redshift资源控制粒度包含用户组和查询组两类，（2）Redshift支持短查询加速SQA，避免短查询排在队列中的长时查询后面等待，只在手动WLM时可用，（3）Redshift支持基于规则进行负载管理，比如查询扫描的数据量达到一定量后移动到别的组内执行。

- 物化视图支持Apache Iceberg和标准AWS Glue表上实体化视图的增量刷新（

  []: https://aws.amazon.com/cn/about-aws/whats-new/2023/11/amazon-redshift-incremental-refresh-materialized-views-data-lake-tables-preview/	"链接"

  ）