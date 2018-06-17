## PostgreSQL概览

### 简介

PostgreSQL是一个被广泛应用的开源数据库系统，由它也构筑了MySQL之外的又一开源数据库生态。PostgreSQL本身是一个单机数据库，有很多公司和组织基于PostgreSQL开发了分布式的数据库系统，比如Greenplum和Postgre-XL等。
 
### 逻辑结构和物理结构
 
在逻辑结构上，我们称PostgreSQL数据库系统为一个Database Cluster，这里的Cluster并不表示有很多物理服务器，而是表示一个PostgreSQL实例可以有多个Database。每个Database包含各种属于这个Database的对象，这些对象可以是Table、Index、View、Function、Sequence等。一个对象只属于某一个Database，用户访问时，每次也只能访问一个Database。多个Database共同组成了PostgreSQL数据库系统。
 
物理结构是指数据库中Database和对象的存储方式。在初始化数据库后，一般有一个目录```base```被用来存储数据文件。在这个```base```目录中有多个表示Database的文件夹，每个Database一个文件夹，文件夹的名字是Database的OID。每个Database中的对象的文件都存放于其属于的Database的文件夹中。
 
与Database不同，Database中的对象并不直接使用OID作为它们的文件名，而是使用一个```relfilenode```来作为对象的文件名。一般情况下，```relfilenode```与对象的OID相同，但是在一些特殊的操作后（如TRUNCATE、REINDEX、CLUSTER），```relfilenode```可能会发生改变，从而不等于对象的OID。同时，每个对象文件的大小也有规定，可以通过参数设定，对象文件大小超过一定阈值后，一个新的文件将产生，名字为```relfilenode.1```，如果再超过阈值，第3个名为```relfilenode.2```新文件将产生。除此之外，还有表示Free Space Map的```relfilenode_fsm```文件和表示Visibility Map的```relfilenode_vm```文件。

### 进程结构
 
PostgreSQL是一个多进程的架构，它的进程主要有：
 
- Server Process：所有其他进程的父进程，负责对整个系统的管理
- Background Processes：针对于每个独立功能的进程，如Vacuum进程、Checkpoint进程
- Replication Associated Processes
- Background Worker Process
- Backend Processes：处理来自客户端的连接，处理查询请求，针对每个连接，都有一个独立的Backend Process
 
在老版本的PostgreSQL中，Server Process被称为Postmaster。当使用pg_ctl命令启动数据库时，Server Process就被启动了，然后它会在内存中申请一块Shared Memory，并启动Background Processes、Replication Associatied Processes和Background Worker Process。当收到来自客户端的连接请求时，启动Backend Process。
 
Backend Process也被称为Postgres进程，它被启动用于处理来自一个客户端的所有查询请求，在客户端关闭连接时被销毁。一个Backend Process只被允许访问一个Database实例，所以在客户端连接PostgreSQL的服务端时，需要指定连接哪一个数据库。PostgreSQL允许多个客户端同时连接服务端，但有一个参数控制最大连接数。因为PostgreSQL没有实现连接池的机制，当服务端频繁连接和断连服务端时，性能会因建连接和启动Backend Process而降低，但有一些第三方中间件可用于处理连接池问题。

Background Processes用于一些独立而特殊的功能：
 
- Background Writer：将Shared Buffer Pool中的脏页持久化
- Checkpointer：处理Check Point相关的操作
- AutoVacuum Launcher：定期进行Vacuum
- WAL Writer：定期将WAL Buffer中的WAL数据持久化
- Statistics Collector：进行统计信息收集
- Logging Collector：将日志信息写入Log文件
- Archiver：进行日志归档

> 为了防止因Server Failure而丢失数据，PostgreSQL支持了WAL机制，WAL数据也被称为XLOG，是PostgreSQL的事务日志

### 内存结构
 
PostgreSQL的内存主要分为两大类，每个Backend Process的私有内存，以及所有进程共用的共享内存。
 
私有内存有：
 
- work_mem：算子执行时所用的内存
- maintenance_work_mem：用于维护的语句/算子所用的内存，如Vacuum和Reindex等
- temp_buffers：用于存放临时表
 
共享内存有：
 
- Shared Buffer Pool：从持久化存储读取表和索引的页面后，存放于此区域
- WAL Buffer：WAL数据被持久化前，存放于此
- Commit Log：存放CLOG

> CLOG（Commit Log）用于记录并发控制（Concurrency Control，CC）机制的事务状态（in_progress、committed、aborted等）
 
此外，还有一些其他的共享内存用于访问控制（semaphores、lightweight locks、shared and exclusive locks等）、Background Processes、事务处理（save point、two phase commit等）。

### 查询处理
 
一条查询语句被发送到PostgreSQL的服务端，需要经过词法解析、语法解析、语义分析、重写、优化和执行的阶段。
 
PostgreSQL借助词法分析工具Lex和语法分析工具Yacc来进行词法解析和语法解析，生成了分析树。语义分析阶段会检查查询语句中是否有不符合语义规定的成分，检查语句是否能够被正常执行，并将分析树中的内容转换为数据库内部数据（如将表名字符串转换为表的OID），最终生成查询树。重写阶段会根据定义好的重写规则对查询树进行重写和转换。因SQL语言是一个描述性语言，对其执行方式的优化更为重要，在优化阶段，将根据规则和代价模型，生成并挑选最优的执行计划。执行阶段将根据执行计划执行各个算子，并生成最终的结果。
 
查询处理是整个数据库中最为深奥复杂的一部分，尤其是作为数据库大脑的优化器部分，此处不作展开，笔者将在其他文章中作重点阐述。

### 并发控制

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

[1] 彭智勇，彭煜玮，PostgreSQL数据库内核分析，2012，机械工业出版社

[2] http://www.postgres-xl.org/overview/
