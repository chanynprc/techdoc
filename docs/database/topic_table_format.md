## 开放表格式

### Iceberg概述

- 官网：https://iceberg.apache.org/
- SDK：Java、Go、Python、Rust
- Catalog支持：Hive MetaStore、Glue、JDBC

#### 数据组织

![](/techdoc/docs/database/images/iceberg_arch.png)

如上图所示，Iceberg中元数据和数据的组织分为3层：

- Iceberg Catalog：存储基础元数据，每个表保存表名到“current metadata pointer”的映射指向该表当前的Metadata File
- Metadata Layer：包含元数据文件（metadata file）、清单列表（manifest list）、清单文件（manifest file）
  - Metadata File：记录表的元数据，包含表的表模式、分区等基本元数据，以及快照以及当前快照的信息
  - Manifest List：记录一系列Manifest File，这些文件构成了一个快照，还包含各Manifest File的分区列的上下界信息、文件数信息、行数信息
  - Manifest File：记录数据文件、每个文件的统计信息（如行数、值范围、空值、数据大小、分区成员）和其他详细信息（如数据文件的格式）
- Data Layer：每个清单文件跟踪一组数据文件

**更多关于Iceberg Catalog**：使用不同的Iceberg Catalog存储，Iceberg Catalog的存储组织形式也会不同。（1）如果使用HDFS存储，在表的元数据文件夹中会有一个名为version-hint.text的文件去存储current metadata pointer。（2）如果使用Hive Metastore存储，在表的属性表中会有一个字段存储current metadata pointer。

**更多关于Metadata File**：（1）此文件是json格式。（2）每次数据写操作都会生成一个新的文件，包含此前的snapshot并添加新的snapshot。（3）在文件中，每个snapshot会包含这个snapshot的timestamp信息，用于Time Travel。如果查询不带Time Travel时间点，则会使用current snapshot。

**更多关于Manifest List**：（1）此文件是Avro格式。（2）每次数据写操作都会生成一个新的文件，只包含当前snapshot的写入及变更。（3）因为包含分区列上下界信息，所以一些在分区列上的过滤裁剪可以在读取到Manifest List文件时进行。

**更多关于Manifest FIle**：（1）此文件是Avro格式。

#### 读写操作



#### 事务

Iceberg在每次写入或更新时，会更新Iceberg Catalog中每个表的current metadata pointer，存储Iceberg Catalog的存储必须支持原子操作。

在更新Iceberg Catalog中每个表的current metadata pointer之前，所有的事务读取到的数据都基于此前的current metadata pointer。当current metadata pointer被更新之后，新的事务的都会按照当前current metadata pointer去寻找最新的snapshot数据，所以current metadata pointer一旦被更新，最新的数据降对其他事务立即可见。

【OCC】

#### 小结

综上，Iceberg的关键点如下：

- 存储Iceberg Catalog的存储必须支持原子操作，如HDFS、Hive Metastore、Nessie

### Catalog概述

在大数据生态中有Catalog的概念，在访问外部数据的时候无需建额外创建外表，直接通过指定外部表名即可对外部数据进行访问。这个功能实际上是在同一套系统内将外部数据的管理访问与内部数据的管理访问方式拉齐到同一水平。

在使用Catalog能力时，需要访问外部的2套服务/系统：（1）元数据服务，（2）存储系统。一般元数据服务有Hive Metastore和AWS Glue。

使用Catalog之前，一般要创建Catalog，主要是需要指定元数据服务的访问参数和存储系统的访问参数。在创建Catalog并使用Catalog后，可以使用类似访问内表的方式，访问外表的方式。

### 数据库/数据仓库对Iceberg的支持

#### ClickHouse

- 支持读取AWS S3上的Iceberg表，且Iceberg表需要已存在于AWS S3上，不支持在AWS S3上创建新的Iceberg表
- 只支持只读，不支持写
- 只支持Iceberg v1，暂时不支持Iceberg v2

可以通过Iceberg的表引擎去定义Iceberg外表，语句示例如下：

```sql
CREATE TABLE iceberg_table
ENGINE=Iceberg('http://test.s3.amazonaws.com/clickhouse-bucket/test_table', 'test', 'test');
```

#### Starrocks （显式创建外表方式）

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

#### Starrocks（Catalog方式）

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

#### DuckDB

- DuckDB通过extension形式提供Iceberg表格式的支持，extension的代码不默认包含在DuckDB代码中
- 只支持读取Iceberg，不支持写
- 不支持Iceberg Catalog，读取Iceberg文件通过类似iceberg_scan的函数实现，数据的地址通过函数的参数传入
- Iceberg Catalog组织不支持HMS，只支持和数据文件在一起的类似HDFS的存储方式，通过version-hint.text文件保存current metadata pointer
- 支持数据在S3上，在S3上的数据读取，貌似是需要手动指定具体的current metadata pointer，而不支持读取version-hint.text文件（实际也不会有场景在S3上维护Iceberg Catalog）。需要httpfs extension加入去支持S3上文件的读取（这也是DuckDB的一个extension）
- 从代码中看，只支持Parquet格式

```sql
INSTALL iceberg;
LOAD iceberg;

SELECT count(*)
FROM iceberg_scan('data/iceberg/lineitem_iceberg', allow_moved_paths = true);

SELECT count(*)
FROM iceberg_scan('data/iceberg/lineitem_iceberg/metadata/02701-1e474dc7-4723-4f8d-a8b3-b5f0454eb7ce.metadata.json');

SELECT count(*)
FROM iceberg_scan('s3://bucketname/lineitem_iceberg/metadata/02701-1e474dc7-4723-4f8d-a8b3-b5f0454eb7ce.metadata.json', allow_moved_paths = true);

SELECT *
FROM iceberg_metadata('data/iceberg/lineitem_iceberg', allow_moved_paths = true);

SELECT *
FROM iceberg_snapshots('data/iceberg/lineitem_iceberg');
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

#### DuckDB访问Iceberg代码调研

- DuckDB访问Iceberg的代码不在DuckDB代码库中，在单独的代码库中：https://github.com/duckdb/duckdb_iceberg

duckdb_iceberg extension中，没有调用Iceberg任何SDK，而是自己实现了一套针对FileSystem（本地文件系统或S3）的文件读取逻辑，并实现各级别元数据文件及数据文件的解析逻辑，从读取version-hint.text到metadata file、manifest list、manifest file，再到读取数据文件。

在duckdb_iceberg extension中，对外放出3个函数，这3个函数用于直接在select语句中调用去访问Iceberg数据或元数据：

- iceberg_scan
- iceberg_metadata
- iceberg_snapshots

具体到元数据及数据文件的解析上，元数据的Json文件是调用了duckdb_iceberg extension自带的Json解析器，元数据的Avro文件是调用了系统的Avro库，数据的Parquet文件是调用DuckDB内部的parquet_scan函数，这个函数在DuckDB的parquet extension中。

### 引用

- [Dremio Document: Apache Iceberg](https://docs.dremio.com/current/sonar/query-manage/data-formats/apache-iceberg/)
- [Apache Iceberg: An Architectural Look Under the Covers](https://www.dremio.com/resources/guides/apache-iceberg-an-architectural-look-under-the-covers/)
- [Apache Iceberg 101 – Your Guide to Learning Apache Iceberg Concepts and Practices](https://www.dremio.com/blog/apache-iceberg-101-your-guide-to-learning-apache-iceberg-concepts-and-practices/)



