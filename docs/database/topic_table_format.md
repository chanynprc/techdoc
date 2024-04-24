## 开放表格式

### Iceberg

- 官网：https://iceberg.apache.org/


#### 数据库/数据仓库对Iceberg的支持

**ClickHouse**

- 支持读取AWS S3上的Iceberg表，且Iceberg表需要已存在于AWS S3上，不支持在AWS S3上创建新的Iceberg表
- 只支持只读，不支持写
- 只支持Iceberg v1，暂时不支持Iceberg v2

可以通过Iceberg的表引擎去定义Iceberg外表，语句示例如下：

```sql
CREATE TABLE iceberg_table
ENGINE=Iceberg('http://test.s3.amazonaws.com/clickhouse-bucket/test_table', 'test', 'test');
```

**Starrocks**

- 在访问文件系统或对象存储系统外，还需保证可访问Iceberg依赖的元数据服务，Starrocks对Iceberg的访问需要元数据服务的支持
- 外表方式只支持只读，不支持写，Iceberg Catalog方式支持写
- 支持Iceberg v1，3.0版开始支持Iceberg v2+ORC，3.1版开始支持Iceberg v2+Parquet
- 2.3版开始支持同步Iceberg表结构，此前版本若有表结构变化需删除外表并重建

定义Iceberg外表之前，需要创建Iceberg资源：

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

然后重建Iceberg外表：

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

### Catalog

在大数据生态中，有一个Catalog的概念，在访问外部数据的时候，无需建表，直接通过指定表名，即可对外部数据进行访问。这个功能实际上是在同一套系统内实现了内部和外部数据的管理。

在使用Catalog能力时，外部有2套服务/系统：元数据服务、存储系统。一般元数据服务有Hive Metastore和AWS Glue。

使用Catalog之前，一般要创建Catalog，主要是需要指定元数据服务的访问参数和存储系统的访问参数。在创建Catalog并使用Catalog后，可以使用类似访问内表的方式，访问外表的方式

