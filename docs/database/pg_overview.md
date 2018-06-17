## PostgreSQL概览

### 简介

PostgreSQL是一个被广泛应用的开源数据库系统，由它也构筑了MySQL之外的又一开源数据库生态。PostgreSQL本身是一个单机数据库，有很多公司和组织基于PostgreSQL开发了分布式的数据库系统，比如Greenplum和Postgre-XL等。
 
### 逻辑结构和物理结构


### PostgreSQL的扩展
 
#### Postgres-XL

Postgres-XL基于PostgreSQL开发而来，是一个横向可扩展的开源数据库集群，它可以灵活地应用于多种负载。如OLTP写入密集型负载、满足商业智能需求的MPP并行处理、Key-Value存储、多租户等。

它由三个主要部分组成，分别是：

- Global Transaction Monitor (GTM)
- Coordinator：管理用户会话，并与GTM和Data Nodes进行交互，对查询语句进行解析，制定执行计划，并将序列化的全局计划发送给所涉及的组件。
- Data Node：Data Node是实际数据存储的位置，数据的分布方式由DBA配置。

其基础架构为：

![](/techdoc/docs/database/images/xl_cluster_architecture.jpg)

### 引用

[0] http://www.interdb.jp/pg/

[1] http://www.postgres-xl.org/overview/
