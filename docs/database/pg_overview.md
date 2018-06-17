## PostgreSQL概览

### 简介

PostgreSQL是一个被广泛应用的开源数据库系统，由它也构筑了MySQL之外的又一开源数据库生态。PostgreSQL本身是一个单机数据库，有很多公司和组织基于PostgreSQL开发了分布式的数据库系统，比如Greenplum和Postgre-XL等。
 
### 逻辑结构和物理结构
 
### 进程和内存结构
 
PostgreSQL有一个多进程的架构，它的进程主要有：
 
- Server Process：所有其他进程的父进程，负责对整个系统的管理
- Background Processes：针对于每个独立功能的进程，如Vacuum进程、Checkpoint进程
- Replication Associated Processes
- Background Worker Process
- Backend Processes：处理来自客户端的连接，处理查询请求，针对每个连接，都有一个独立的Backend Processes
 
在老版本的PostgreSQL中，Server Process被称为Postmaster。当使用pg_ctl命令启动数据库时，Server Process就被启动了，然后它会在内存中申请一块Shared Memory，并启动Background Processes、Replication Associatied Processes和Background Worker Process。当收到来自客户端的连接请求时，启动Backend Process。
 
Backend Process也被称为Postgres进程，它被启动用于处理来自一个客户端的所有查询请求，在客户端关闭连接时被销毁。一个Backend Process只被允许访问一个Database实例，所以在客户端连接PostgreSQL的服务端时，需要指定连接哪一个数据库。PostgreSQL允许多个客户端同时连接服务端，但有一个参数控制最大连接数。因为PostgreSQL没有实现连接池的机制，当服务端频繁连接和断连服务端时，性能会因建连接和启动Backend Process而降低，但有一些第三方中间件可用于处理连接池问题。
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
