## 开放表格式

### Iceberg概述

- 官网：https://iceberg.apache.org/
- SDK：Java、Go、Python、Rust
- Catalog支持：Hive MetaStore、Glue、JDBC
- OCC

元数据和数据的组织分为3层：

- Iceberg Catalog：存储元数据的位置，指向当前的metadata file。存储Iceberg Catalog的存储必须支持原子操作
- Metadata Layer：包含元数据文件（metadata file）、清单列表（manifest list）、清单文件（manifest file）
  - 元数据文件（metadata file）：包含表的表模式、分区、快照以及当前快照的信息，每次数据操作都会生成一个新的文件。此文件是json格式
  - 清单列表（manifest list）：包含一系列清单文件及其信息（分区列的上下界信息等），这些文件构成了一个快照。此文件是Avro格式
  - 清单文件（manifest file）：除了跟踪数据文件，还记录了每个文件的统计信息和其他详细信息，也包含数据文件的格式信息。此文件是Avro格式
- Data Layer：每个清单文件跟踪一组数据文件

![](/techdoc/docs/database/images/iceberg_arch.png)

### Catalog概述

在大数据生态中有Catalog的概念，在访问外部数据的时候无需建额外创建外表，直接通过指定外部表名即可对外部数据进行访问。这个功能实际上是在同一套系统内将外部数据的管理访问与内部数据的管理访问方式拉齐到同一水平。

在使用Catalog能力时，需要访问外部的2套服务/系统：（1）元数据服务，（2）存储系统。一般元数据服务有Hive Metastore和AWS Glue。

使用Catalog之前，一般要创建Catalog，主要是需要指定元数据服务的访问参数和存储系统的访问参数。在创建Catalog并使用Catalog后，可以使用类似访问内表的方式，访问外表的方式。

### 数据库/数据仓库对Iceberg的支持

**ClickHouse**

- 支持读取AWS S3上的Iceberg表，且Iceberg表需要已存在于AWS S3上，不支持在AWS S3上创建新的Iceberg表
- 只支持只读，不支持写
- 只支持Iceberg v1，暂时不支持Iceberg v2

可以通过Iceberg的表引擎去定义Iceberg外表，语句示例如下：

```sql
CREATE TABLE iceberg_table
ENGINE=Iceberg('http://test.s3.amazonaws.com/clickhouse-bucket/test_table', 'test', 'test');
```

**Starrocks （显式创建外表方式）**

- 在访问文件系统或对象存储系统外，还需保证可访问Iceberg依赖的元数据服务，Starrocks对Iceberg的访问需要元数据服务的支持
- 外表方式只支持只读，不支持写
- 支持Iceberg v1，3.0版开始支持Iceberg v2+ORC，3.1版开始支持Iceberg v2+Parquet
- 2.3版开始支持同步Iceberg表结构，此前版本若有表结构变化需删除外表并重建

第1步，创建Iceberg资源：

```sql
CREATE EXTERNAL RESOURCE "iceberg0"
PROPERTIES (
   "type" = "iceberg",
   "iceberg.catalog.type" = "HIVE", -- 取值可以是HIVE或CUSTOM（SR 2.3开始）
   "iceberg.catalog.hive.metastore.uris" = "thrift://xx.xx.xx.xx:9083" -- Hive Metastore 的 URI
);

CREATE EXTERNAL RESOURCE "iceberg1"
PROPERTIES (
   "type" = "iceberg",
   "iceberg.catalog.type" = "CUSTOM",
   "iceberg.catalog-impl" = "com.starrocks.IcebergCustomCatalog" -- 开发的custom catalog的全限定类名
);
```

第2步，创建Iceberg外表：

```sql
CREATE EXTERNAL TABLE `iceberg_tbl`
(
    `id` bigint NULL,
    `data` varchar(200) NULL
)
ENGINE=ICEBERG
PROPERTIES
(
    "resource" = "iceberg0", -- Iceberg资源
    "database" = "iceberg", -- Iceberg表的数据库
    "table" = "iceberg_table" -- Iceberg表的名称
);
```

**Starrocks（Catalog方式）**

- Iceberg Catalog方式支持写
- 只支持Parquet文件格式

第1步，创建Catalog：

```sql

CREATE EXTERNAL CATALOG <catalog_name>
[COMMENT <comment>]
PROPERTIES
(
    "type" = "iceberg",
    MetastoreParams,
    StorageCredentialParams
)

CREATE EXTERNAL CATALOG iceberg_catalog_hms
PROPERTIES
(
    "type" = "iceberg",
    "iceberg.catalog.type" = "hive",
    "hive.metastore.uris" = "thrift://xx.xx.xx.xx:9083",
    "aliyun.oss.access_key" = "<user_access_key>",
    "aliyun.oss.secret_key" = "<user_secret_key>",
    "aliyun.oss.endpoint" = "<oss_endpoint>"
);
```

第2步，创建Iceberg数据库：

```sql
CREATE DATABASE <database_name>
[PROPERTIES ("location" = "<prefix>://<path_to_database>/<database_name.db>/")]
```

第3步，创建Iceberg表：

```sql
CREATE TABLE [IF NOT EXISTS] [database.]table_name
(column_definition1[, column_definition2, ...
partition_column_definition1,partition_column_definition2...])
[partition_desc]
[PROPERTIES ("location" = "value", "file_format" = 'value', "compression_codec" = "", ...)]
[AS SELECT query]
```

### 代码调研

#### Iceberg代码调研

- 

#### Starrocks访问Iceberg代码调研

设计文档：https://docs.google.com/document/d/1LtIX7v_i5PdazthzC6tpeGe_TR3Zza8LW2SPNUCJdmA/edit?usp=sharing

基础提交：https://github.com/StarRocks/starrocks/pull/2225

Starrocks的FE代码中涉及Iceberg的代码有：

```
// apache iceberg类及封装类
fe/fe-core/src/main/java/org/apache/iceberg/*
fe/fe-core/src/main/java/org/apache/iceberg/StarRocksIcebergTableScan.java

// catalog
fe/fe-core/src/main/java/com/starrocks/catalog/IcebergTable.java
fe/fe-core/src/main/java/com/starrocks/catalog/IcebergResource.java
fe/fe-core/src/main/java/com/starrocks/catalog/IcebergPartitionKey.java

// connector
fe/fe-core/src/main/java/com/starrocks/connector/iceberg/*
fe/fe-core/src/main/java/com/starrocks/connector/metadata/iceberg/*

// 算子
fe/fe-core/src/main/java/com/starrocks/planner/IcebergMetadataScanNode.java
fe/fe-core/src/main/java/com/starrocks/planner/IcebergScanNode.java
fe/fe-core/src/main/java/com/starrocks/planner/IcebergTableSink.java

// 优化器
fe/fe-core/src/main/java/com/starrocks/sql/optimizer/*

// factory
fe/fe-core/src/main/java/com/starrocks/server/IcebergTableFactory.java
```

Starrocks的BE代码中涉及Iceberg的代码有：

```
// iceberg
be/src/exec/iceberg/iceberg_delete_file_iterator.cpp
be/src/exec/iceberg/iceberg_delete_builder.cpp

// pipeline
be/src/exec/pipeline/sink/iceberg_table_sink_operator.cpp

// connector
be/src/connector/iceberg_chunk_sink.cpp
be/src/connector/iceberg_connector.cpp

// runtime
be/src/runtime/iceberg_table_sink.cpp
```

Starrocks中其他涉及Iceberg的代码有：

```
//
java-extensions/iceberg-metadata-reader/src/main/java/com/starrocks/connector/iceberg
java-extensions/iceberg-metadata-reader/src/main/java/com/starrocks/connector/iceberg/IcebergMetadataScanner.java
java-extensions/iceberg-metadata-reader/src/main/java/com/starrocks/connector/iceberg/IcebergMetadataScannerFactory.java
java-extensions/iceberg-metadata-reader/src/main/java/com/starrocks/connector/iceberg/IcebergMetadataColumnValue.java

//
java-extensions/hadoop-ext/src/main/java/com/starrocks/connector/share/iceberg
java-extensions/hadoop-ext/src/main/java/com/starrocks/connector/share/iceberg/IcebergMetricsBean.java
```



### 引用

- [Dremio Document: Apache Iceberg](https://docs.dremio.com/current/sonar/query-manage/data-formats/apache-iceberg/)
- [Apache Iceberg: An Architectural Look Under the Covers](https://www.dremio.com/resources/guides/apache-iceberg-an-architectural-look-under-the-covers/)
- [Apache Iceberg 101 – Your Guide to Learning Apache Iceberg Concepts and Practices](https://www.dremio.com/blog/apache-iceberg-101-your-guide-to-learning-apache-iceberg-concepts-and-practices/)



