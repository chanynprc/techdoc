## PostgreSQL概览

### 简介

PostgreSQL是一个被广泛应用的开源数据库系统，由它也构筑了MySQL之外的又一开源数据库生态。PostgreSQL本身是一个单机数据库，有很多公司和组织基于PostgreSQL开发了分布式的数据库系统，比如Greenplum和Postgre-XL等。

### 逻辑结构和物理结构

#### 逻辑结构

在逻辑结构上，我们称PostgreSQL数据库系统为一个Database Cluster，这里的Cluster并不表示有很多物理服务器，而是表示一个PostgreSQL实例可以有多个Database。每个Database包含各种属于这个Database的对象，这些对象可以是Table、Index、View、Function、Sequence等。一个对象只属于某一个Database，用户访问时，每次也只能访问一个Database。多个Database共同组成了PostgreSQL数据库系统。

#### 物理结构

物理结构是指数据库中Database和对象的存储方式。在初始化数据库后，在```PGDATA```目录下有一个目录```base```文件夹被用来存储数据文件。在这个```base```文件夹中有多个表示Database的文件夹，每个Database一个文件夹，文件夹的名字是Database的OID。每个Database中的对象的文件都存放于其属于的Database的文件夹中。

与Database不同，Database中的对象并不直接使用OID作为它们的文件名，而是使用一个```relfilenode```来作为对象的文件名。一般情况下，```relfilenode```与对象的OID相同，但是在一些特殊的操作后（如TRUNCATE、REINDEX、CLUSTER），```relfilenode```可能会发生改变，从而不等于对象的OID。同时，每个对象文件的大小也有规定，可以通过参数设定，对象文件大小超过一定阈值后，一个新的文件将产生，名字为```relfilenode.1```，如果再超过阈值，第3个名为```relfilenode.2```新文件将产生。除此之外，还有表示Free Space Map的```relfilenode_fsm```文件和表示Visibility Map的```relfilenode_vm```文件。所以，一个对象可能有多个文件与之对应。

#### TableSpace
TableSpace被用于在```PGDATA```目录外存放数据文件。建立一个TableSpace后，PostgreSQL将在```PGDATA/pg_tblspc```目录下建立一个链接，指向TableSpace的目录。在TableSpace的目录下，会先建立一个与PostgreSQL版本号相关的文件夹，此版本号文件夹类似于内部文件目录```base```，在版本号文件夹下是表示Database的文件夹。如果对象被建立在一个已有的Database中，那么版本号文件夹下会出现一个同名（Database的OID）的文件夹，如果对象被建立在一个新的Database中，则会使用新的Database的OID作为文件夹名字。

#### 对象文件

对象文件内部由Page/Block组成，它们的大小是固定的（默认8K）。对象文件和Page的组织形式如图所示：

![](/techdoc/docs/database/images/pg_heap_table_file.png)

Tuple Identifier（TID）被用于标记每一个Tuple，由两个数字组成，Block Number（Page在数据文件中的偏移）和Offset Number（Tuple在Page中的偏移）。在索引的叶节点中，存放着目标数值（Key）和它的TID，通过这个TID可以直接定位到目标数据的Tuple。

当一个Tuple的大小超过2K时，PostgreSQL采用TOAST（The Oversized-Attribute Storage Technique）技术进行存储。

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

### Foreign Data Wrappers（FDW）

PostgreSQL从9.1版本开始支持SQL Management of External Data (SQL/MED，从SQL 2003成为标准，用于访问远程数据)。在SQL/MED中，远程的数据表成为外表（foreign table），在PostgreSQL中，可以像访问本地表一样访问外表。



### 并发控制（Concurrency Control）

并发控制是为处理一致性（consistency）和隔离性（isolation）所设计的机制。有3种主流的并发控制技术：

- MVCC (Multi-version Concurrency Control)：写操作创建一个新版本数据，并保留旧版本数据，读操作选择一个保持事务隔离性的版本进行数据读取，MVCC的特点是读不阻塞写，写不阻塞读
- S2PL (Strict Two-Phase Locking)：S2PL在写数据时需要加排他锁，所以写会阻塞读
- OCC (Optimistic Concurrency Control)

PostgreSQL和一些RDBMS（如Oracle）使用的是MVCC的一个变种，快照隔离（Snapshot Isolation, SI）。Oracle和PostgreSQL之间不同的是，Oracle使用回滚段（rollback segments），当需要写新的数据时，老数据被写入回滚段，然后直接在原数据区域写入数据。PostgreSQL在原数据页直接插入新数据，读数据时根据可见性规则选择相应的数据版本。

在PostgreSQL 9.1版本中，加入了Serializable Snapshot Isolation (SSI)以检测serialization anomalies并处理其带来的冲突。所以从9.1版本以后，PostgreSQL支持了真正的可串行化隔离级别。其他数据库中，SQL Server使用的是SSI，Oracle使用的是SI。

PostgreSQL中的隔离级别如下：

隔离级别 | Dirty<br/>Reads | Non-repeatable<br/>Read | Phantom<br/>Read | Serialization<br/>Anomaly
--- | --- | --- | --- | ---
 READ COMMITTED | × | ○ | ○ | ○ 
 REPEATABLE READ | × | × | ×(PG)<br/>○(ANSI SQL) | ○
 SERIALIZABLE | × | × | × | ×

×: Not Possible<br/>○: Possible

PostgreSQL使用了MVCC和2PL混合的方式。对于DML语句（SELECT, UPDATE, INSERT, DELETE）使用SSI，对于DDL语句（CREATE TABLE等）使用2PL。

#### 事务ID

一个事务开始后，将被分配一个ID。在PostgreSQL中事务ID称为txid（transaction id），是一个32位无符号整形。通过事务ID，可以比较事务的先后顺序以及判别数据修改的可见性。由于32位无符号整形是有限的，PostgreSQL将事务ID看成一个环，环上某一点的前面一半事务ID看成是Past事务ID，后面一半事务ID看成是Future事务ID。

PostgreSQL中，有3个特殊的事务ID：

- 0：Invalid txid
- 1：Bootstrap txid，只在初始化数据库时使用
- 2：Frozen txid

#### Tuple结构

前面介绍过Table的Page结构，在Page中，每一个元组是以Tuple形式存储的，Tuple的结构如下：

```
+--------+--------+-------+--------+-------------+------------+--------+--------+-----------+
| t_xmin | t_xmax | t_cid | t_ctid | t_infomask2 | t_infomask | t_hoff | t_bits | User Data |
+--------+--------+-------+--------+-------------+------------+--------+--------+-----------+
```

各字段的含义如下：
- t_xmin：insert tuple的事务的txid
- t_xmax：当insert tuple时设置为0，当update和delete tuple时设置为事务的txid
- t_cid：操作tuple的语句在事务中的index，从0开始
- t_ctid：保存自己的ctid，当tuple被update时，保存新tuple的ctid

#### Tuple的insert, update和delete

首先，insert一条数据：

```sql
-- txid = 100
BEGIN;
INSERT INTO t VALUES ('A');
COMMIT;
```

则一条新的记录被添加到page中：

```
+--------+--------+-------+--------+-----------+
| t_xmin | t_xmax | t_cid | t_ctid | User Data |
+--------+--------+-------+--------+-----------+
| 100    | 0      | 0     | (0,1)  | 'A'       |
+--------+--------+-------+--------+-----------+
```

然后，update这条数据：

```sql
-- txid = 101
BEGIN;
UPDATE t SET data='B';
UPDATE t SET data='C';
COMMIT;
```

两条语句会在page中插入两个新的tuple，并对之前的tuple作更新，标记（逻辑上）删除。

第1条语句执行后，原tuple的t_xmax、t_ctid会被更新，并插入一条新的数据：

```
+--------+--------+-------+--------+-----------+
| t_xmin | t_xmax | t_cid | t_ctid | User Data |
+--------+--------+-------+--------+-----------+
| 100    | 101    | 0     | (0,2)  | 'A'       |
+--------+--------+-------+--------+-----------+
| 101    | 0      | 0     | (0,2)  | 'B'       |
+--------+--------+-------+--------+-----------+
```

第2条语句执行后，需要更新的字段与第1条语句类似，由于是事务的第2条语句，新插入的tuple的t_cid为1：

```
+--------+--------+-------+--------+-----------+
| t_xmin | t_xmax | t_cid | t_ctid | User Data |
+--------+--------+-------+--------+-----------+
| 100    | 101    | 0     | (0,2)  | 'A'       |
+--------+--------+-------+--------+-----------+
| 101    | 101    | 0     | (0,3)  | 'B'       |
+--------+--------+-------+--------+-----------+
| 101    | 0      | 1     | (0,3)  | 'C'       |
+--------+--------+-------+--------+-----------+
```

最后，delete这条数据：

```sql
-- txid = 102
BEGIN;
DELETE FROM t;
COMMIT;
```

系统不会立即删除这条记录，而是将记录标记为删除，即设置其t_xmax，系统运行VACUUM命令后，将对存储空间进行回收：

```
+--------+--------+-------+--------+-----------+
| t_xmin | t_xmax | t_cid | t_ctid | User Data |
+--------+--------+-------+--------+-----------+
| 100    | 101    | 0     | (0,2)  | 'A'       |
+--------+--------+-------+--------+-----------+
| 101    | 101    | 0     | (0,3)  | 'B'       |
+--------+--------+-------+--------+-----------+
| 101    | 102    | 1     | (0,3)  | 'C'       |
+--------+--------+-------+--------+-----------+
```

#### Commit Log (clog)

Commit log被用于保存事务的状态，存放于shared memory中。PostgreSQL的事务状态有4种：

- IN_PROGRESS
- COMMITTED
- ABORTED
- SUB_COMMITTED（用于子事务）

clog是一个数组结构，数组下标是事务ID（txid），数组内容是对应txid的事务状态。clog的数组是按照page（8KB）分配的，如果txid超过了目前的数组长度，则会扩展新的8KB给clog。

当PostgreSQL停机或触发checkpoint时，clog将被从内存中写入磁盘的pg_log（pg_xact）文件夹进行持久化，这些文件被命名为`0000`、`0001`等，每个文件的大小上限是256KB，也就是说每个文件最多容纳32个clog的page。当PostgreSQL启动时，pg_log（pg_xact）文件夹下的文件将被加载成clog。当然，并非clog中的所有数据都是有用的，Vacuum会对clog的page以及文件进行清理。

#### 事务快照（Transaction Snapshot）

事务快照用于表示对于某个特性的事务、某个特定的时间点而言，哪些事务是active的。

PostgreSQL使用一个格式化的字符串表示事务快照。它的格式是`xmin:xmax:xip_list`，各字段的含义为：

- xmin：最早的active的txid，早于这个txid的事务要么已提交，要么已回滚
- xmax：最早的未被分配的txid
- xip_list：此快照中active的txid，取值范围为[xmin, xmax)

```
+-----------------+----+----+-----+-----+-----+-----+-----+-----+
| Snapshot        | 98 | 99 | 100 | 101 | 102 | 103 | 104 | 105 |
+-----------------+----+----+-----+-----+-----+-----+-----+-----+
| 100:100:        | o  | o  | +   | +   | +   | +   | +   | +   |
+-----------------+----+----+-----+-----+-----+-----+-----+-----+
| 100:104:100,102 | o  | o  | +   | o   | +   | o   | +   | +   |
+-----------------+----+----+-----+-----+-----+-----+-----+-----+

+：active   , invisible           , in progress or not yet started
o：inactive , visible if commited , commited or aborted
```

可以总结规律为：（1）xmax之前在xip_list的txid为active，（2）xmax及xmax以后的txid均active，（3）active的txid不可见。

在不同的隔离级别下，事务获取事务快照的时间点是有差别的。在READ COMMITTED隔离级别下，每条SQL语句都会获取一个事务快照。在REPEATABLE READ和SERIALIZABLE隔离级别下，事务只在第1条语句的时候获取事务快照。

#### 可见性规则

可见性规则是利用tuple的xmin、xmax、clog和snapshot来判断tuple的可见性。此处只讨论简化了的可见性规则。

**Clog(t_xmin) = ABORTED**

如果tuple的xmin指示的事务为ABORTED状态，说明插入此tuple的事务没有成功，此tuple应为不可见。

> **Rule 1**: If Clog(t_xmin) = ABORTED ⇒ Invisible

**Clog(t_xmin) = IN_PROGRESS**

如果tuple的xmin指示的事务为IN_PROGRESS状态，则此tuple的可见性需要分情况讨论。

情况1：如果tuple的xmin指示的事务是当前事务，说明此tuple在当前事务中被插入。

情况1.1：如果此tuple的xmax为INVALID，说明此tuple在当前事务中insert后未作修改，则此tuple可见。

> **Rule 2**: If Clog(t_xmin) = IN_PROGRESS ∧ t_xmin = current_txid ∧ t_xmax = INVAILD ⇒ Visible

情况1.2：如果tuple的xmax不为INVALID，说明此tuple被当前事务update或delete了，此tuple不可见。

> **Rule 3**: If Clog(t_xmin) = IN_PROGRESS ∧ t_xmin = current_txid ∧ t_xmax ≠ INVAILD ⇒ Invisible

情况2：如果tuple的xmin指示的事务不是当前事务，说明此tuple在其他IN_PROGRESS状态的事务中被插入，此tuple应为不可见。

> **Rule 4**: If Clog(t_xmin) = IN_PROGRESS ∧ t_xmin ≠ current_txid ⇒ Invisible

**Clog(t_xmin) = COMMITTED**

如果tuple的xmin指示的事务为COMMITTED状态，则此tuple的可见性需要分情况讨论。

情况1：如果snapshot中显示xmin指示的事务为active状态，则该xmin指示的事务应被视为IN_PROGRESS，而当前事务一定不是COMMITTED，所以xmin指示的事务不是当前事务，这就可以使用`Rule 4`确定此tuple不可见。

> **Rule 5**: If Clog(t_xmin) = COMMITTED ∧ Snapshot(t_xmin) = active ⇒ Invisible

情况2：如果snapshot中显示xmin指示的事务为inactive状态，则该xmin指示的事务可视为COMMITTED。

情况2.1：如果tuple的xmax为INVALID，说明此tuple未被修改，或者如果tuple的xmax指示的事务为ABORTED状态，说明此tuple的后续更改失败了，则此tuple可见。

> **Rule 6**: If Clog(t_xmin) = COMMITTED ∧ Snapshot(t_xmin) = inactive ∧ (t_xmax = INVALID ∨ Clog(t_xmax) = ABORTED) ⇒ Visible

情况2.2：如果tuple的xmax指示的事务为IN_PROGRESS状态，则需根据xmax指示的事务是不是当前事务来决定。

情况2.2.1：如果xmax指示的事务是当前事务，说明此tuple被当前事务update或delete了，此tuple不可见。

> **Rule 7**: If Clog(t_xmin) = COMMITTED ∧ Snapshot(t_xmin) = inactive ∧ Clog(t_xmax) = IN_PROGRESS ∧ t_xmax = current_txid ⇒ Invisible

情况2.2.2：如果xmax指示的事务不是当前事务，说明此tuple被某个IN_PROGRESS状态的其他事务update或delete了，但此事务未提交，所以其update或delete被认为未生效，则此tuple可见。

> **Rule 8**: If Clog(t_xmin) = COMMITTED ∧ Snapshot(t_xmin) = inactive ∧ Clog(t_xmax) = IN_PROGRESS ∧ t_xmax ≠ current_txid ⇒ Visible

情况2.3：如果tuple的xmax指示的事务为COMMITTED状态，则需要根据snapshot中该事务的状态判断。

情况2.3.1：如果snapshot中显示xmax指示的事务为active状态，则该事务可认为未提交，其update或delete未生效，此tuple可见。此情况可归约为`Rule 8`。

> **Rule 9**: If Clog(t_xmin) = COMMITTED ∧ Snapshot(t_xmin) = inactive ∧ Clog(t_xmax) = COMMITTED ∧ Snapshot(t_xmax) = active ⇒ Visible

情况2.3.2：如果snapshot中显示xmax指示的事务为inactive状态，即为真正的COMMITTED状态，此tuple被该事务update或delete，此tuple不可见。

> **Rule 10**: If Clog(t_xmin) = COMMITTED ∧ Snapshot(t_xmin) = inactive ∧ Clog(t_xmax) = COMMITTED ∧ Snapshot(t_xmax) ≠ active ⇒ Invisible

#### 并发异常的处理

**脏读（Dirty Reads, wr-conflicts）**

脏读是指一个事务读取了另一个未提交事务的脏数据。

```
+----+--------------------------+--------------------------+
|    | txid = 101               | txid = 102               |
+----+--------------------------+--------------------------+
| T1 | BEGIN;                   | BEGIN;                   |
| T2 | UPDATE t SET data = 'B'; |                          |
| T3 | SELECT * FROM t;         | SELECT * FROM t;         |
| T4 | COMMIT;                  |                          |
| T5 |                          | SELECT * FROM t;         |
| T6 |                          | COMMIT;                  |
+----+--------------------------+--------------------------+

T1:
+---------+--------+--------+-------+--------+-----------+
|         | t_xmin | t_xmax | t_cid | t_ctid | User Data |
+---------+--------+--------+-------+--------+-----------+
| Tuple_1 | 100    | 0      | 0     | (0,1)  | 'A'       |
+---------+--------+--------+-------+--------+-----------+

After T2:
+---------+--------+--------+-------+--------+-----------+
|         | t_xmin | t_xmax | t_cid | t_ctid | User Data |
+---------+--------+--------+-------+--------+-----------+
| Tuple_1 | 100    | 101    | 0     | (0,2)  | 'A'       |
+---------+--------+--------+-------+--------+-----------+
| Tuple_2 | 101    | 0      | 0     | (0,2)  | 'B'       |
+---------+--------+--------+-------+--------+-----------+
```

在T3时刻，不论事务隔离级别被如何设置，两个事务拿到的snapshot是“100:100:”，clog的情况是Clog(100) = COMMITTED，Clog(101) = IN_PROGRESS，Clog(102) = IN_PROGRESS。那么，根据上文所述的可见性规则，txid为101的事务可见的是Tuple_2，读取到的内容是‘B’，txid为102的事务可见的是Tuple_1，读取到的内容是‘A’。无论txid为101的事务后续是否失败，在txid为102的事务中不会出现读取到txid为101事务未提交的修改的情况。

可以看出，在PostgreSQL中，无论何种事务隔离级别，不会出现一个事务读取另一个未提交事务的脏数据的情况，这样就避免了脏读异常。

**重复读（Repeatable Reads）**

重复读是指一个事务中两次读取到的数据内容出现了差异。

继续使用上述示例，在T5时刻，如果txid为102事务的隔离级别设置的为READ COMMITTED，其snapshot为“101:101:”，如果txid为102事务的隔离级别设置为REPEATABLE READ，其snapshot为“100:100:”，clog的情况是Clog(100) = COMMITTED，Clog(101) = COMMITTED，Clog(102) = IN_PROGRESS。那么，根据可见性规则，如果txid为102事务的隔离级别设置的为READ COMMITTED，Tuple_1不可见，Tuple_2可见，如果txid为102事务的隔离级别设置为REPEATABLE READ，Tuple_1可见，Tuple_2不可见。

可以看出，在PostgreSQL中，READ COMMITTED隔离级别可能会导致一个事务中两次读取到的数据内容出现差异，但是在REPEATABLE READ隔离级别中，不会出现这一情况，这样就避免了重复读异常。

**幻读（Phantom Reads）**

幻读是指一个事务中两次读取到的数据条数出现了差异。

```
+----+----------------------------+----------------------------+
|    | txid = 100                 | txid = 101                 |
+----+----------------------------+----------------------------+
| T1 | BEGIN;                     | BEGIN;                     |
| T2 |                            | SELECT * FROM t;           |
| T3 | INSERT INTO t VALUES('A'); |                            |
| T4 | COMMIT;                    |                            |
| T5 |                            | SELECT * FROM t;           |
| T6 |                            | COMMIT;                    |
+----+----------------------------+----------------------------+

T1:
+---------+--------+--------+-------+--------+-----------+
|         | t_xmin | t_xmax | t_cid | t_ctid | User Data |
+---------+--------+--------+-------+--------+-----------+
+---------+--------+--------+-------+--------+-----------+

After T3:
+---------+--------+--------+-------+--------+-----------+
|         | t_xmin | t_xmax | t_cid | t_ctid | User Data |
+---------+--------+--------+-------+--------+-----------+
| Tuple_1 | 100    | 0      | 0     | (0,1)  | 'A'       |
+---------+--------+--------+-------+--------+-----------+
```

在T2时刻，读取的t表为空。

在T5时刻，如果txid为101事务的隔离级别为READ COMMITTED，其snapshot为“100:100:”，如果txid为101事务的隔离级别设置为REPEATABLE READ，其snapshot为“99:99:”，clog的情况是Clog(99) = COMMITTED，Clog(100) = COMMITTED，Clog(101) = IN_PROGRESS。那么，根据可见性规则，如果txid为101事务的隔离级别为READ COMMITTED，Tuple_1可见，如果txid为101事务的隔离级别设置为REPEATABLE READ，Tuple_1不可见。

可以看出，在PostgreSQL中，READ COMMITTED隔离级别可能会导致一个事务中两次读取到的数据条数出现差异，但是在REPEATABLE READ隔离级别中，不会出现这一情况，这样就避免了幻读异常。

**丢失更新（Lost Updates, ww-conflict）**

丢失更新是指两个事务对同一条数据进行更新操作会导致更新被覆盖的问题。

分3种情况来讨论：

1. 目标数据正在被更新：即目标数据被一个另外的事务更新，但该事务仍然IN_PROGRESS。这种情况下，当前事务会等待更新目标数据的事务结束，然后进行情况2或情况3的操作。
2. 目标数据已被更新：如果更新目标数据的事务被正确提交，那么目标数据已被更改。在READ COMMITTED隔离级别下，会进行update操作，此时就出现了丢失更新。在REPEATABLE READ或SERIALIZABLE隔离级别下，当前事务会被abort，从而避免丢失更新。
3. 目标数据未被更新：数据可被正常update。

在PostgreSQL中，SI使用first-updater-win，而SSI使用first-committer-win。从而在丢失更新的处理中，情况1的当前事务会进行等待。

#### Serializable Snapshot Isolation (SSI)

从PostgreSQL 9.1版本后，引入了Serializable Snapshot Isolation (SSI)以实现了真正的可串行化（SERIALIZABLE）隔离级别。这里只以Write-Skew（串行化异常的一种）为例作简要讨论。

这里引入依赖图（precedence graph, dependency graph或serialization graph）对串行化异常问题进行描述。如果依赖图中出现了环，则表示出现了串行化异常。

SSI的总体思想是：

1. 将所有对tuple、page、relation的访问记SIREAD锁
2. 当数据被写入时，根据SIREAD锁检测rw-conflicts
3. 当检测到串行化异常时，中止事务

SIREAD锁，也叫predicate lock，形式为一个对象和一个虚拟txid，用来表示这个对象被谁访问。在SERIALIZABLE隔离级别下，DML语句会触发创建SIREAD锁。例如，如果txid为100的事务访问了Tuple_1，那么{Tuple_1, {100}}将被建立，如果txid为101的事务也访问了Tuple_1，那么这个SIREAD锁将被更新为{Tuple_1, {100, 101}}。SIREAD锁有3种级别，tupel、page和relation，会根据情况优先建立低级别的SIREAD锁，当需要锁的数据变多后，会对SIREAD锁进行合并，并升级为更高级别的SIREAD锁。在Index Scan中，会直接对index page加SIREAD锁，在sequential scan中，会直接对数据relation加SIREAD锁。

rw_conflicts（TBA）

#### 维护流程

PostgreSQL需要以下的维护流程，来保证并发控制的有效性。

- 移除dead tuple，并相应更新索引
- 移除CLog中的冗余内容
- Freeze过老的txid
- 更新FSM、VM和统计信息

以上大部分流程在Vacuum逻辑中处理。

对于Freeze过老的txid，由于txid是有限的，会存在翻转的问题，比如一个tuple数据很久未被更新，其txid面临失效的风险，会导致该航数据不可见。对于这种长时间未更新的tuple，需要对器txid进行处理，保证在可见性判断算法中不会出错。在9.4版本之前，txid年龄大于vacuum_freeze_min_age的tuple的t_xmin会在Vacuum逻辑中置为2。从9.4版本开始，t_infomask列XMIN_FROZEN列会被置1。

### Vacuum

Vacuum的主要任务是移除dead tuple和Freeze过老的txid。Vacuum分为两种：

- Vacuum（Concurrent Vacuum）：为每个数据页移除dead tuple，此表对其他查询可读
- Vacuum full：移除dead tuple并且重新整理数据文件，此表对其他查询不可读

#### (Concurrent) Vacuum

Vacuum的总体流程是：

1. 对目标表加ShareUpdateExclusiveLock锁
1. 扫描所有的数据Page，将dead tuple存到一个列表中（内存大小受maintenance_work_mem控制），并且freeze dead tuple
1. 更新指向dead tuple的索引
1. 对每个数据Page，移除dead tuple并整理数据Page的heap tuples部分（line pointers部分的dead tuple指针并不移除），更新FSM和VM
1. 如果最后一个数据Page没有元组了，将其truncate
1. 更新和目标表相关的统计信息和系统表信息
1. 释放目标表的ShareUpdateExclusiveLock锁
1. 更新和Vacuum处理过程相关的统计信息和系统表信息
1. 清理无用的CLog

#### Visibility Map

Visibility Map被设计用来加快Vacuum的过程，它标记了哪些数据Page有dead tuple（VM中标记为0），哪些数据Page没有dead tuple（VM中标记为1），当某数据Page对应的VM值为1时（表示无dead tuple），Vacuum会直接跳过该数据Page。

#### Freeze处理逻辑

Freeze处理有两种模式，lazy mode和eager mode。在lazy mode下，Vacuum只扫描被VM标记含有dead tuple的数据Page。在eager mode下，Vacuum扫描所有的数据Page。

1、Lazy mode

在lazy mode下，Vacuum只扫描被VM标记含有dead tuple的数据Page（VM=0）。

当某数据Page的VM=0时，表示该数据Page含有dead page，该Page上的所有元组会被扫描，当目标元组的xmin < freezeLimit_txid时，该元组会被标记为frozen。其中：

```
freezeLimit_txid = OldestXmin - vacuum_freeze_min_age
```

OldestXmin为当前所有运行的事务的最小txid，如果除了Vacuum外没有其他事务，OldestXmin就是Vacuum事务的txid。vacuum_freeze_min_age是一个系统参数，默认值是50,000,000。

当某数据Page的VM=1时，表示该数据Page没有dead tuple，则该Page不会被扫描，若这个数据Page上的元组的txid < freezeLimit_txid时，也不会被标记frozen。

所以，在Lazy mode下，因为会根据VM跳过一些没有dead tuple的Page，会导致这些Page中的元组不会被Freeze。

2、Eager mode（version <= 9.5）

Eager mode时Lazy mode的补充，它扫描所有Page的所有Tuple。Vacuum时，它在满足以下条件时触发：

```
pg_database.datfrozenxid < (OldestXmin − vacuum_freeze_table_age)
```

pg_database.datfrozenxid是pg_database表的datfrozenxid列，表示当前database最老的frozen txid。vacuum_freeze_table_age是一个系统参数，默认值为150,000,000。

当Eager mode被触发后，具体哪些Tuple需要被Freeze，和Lazy mode一致，只是所有的Page都会被扫描。

在某表执行完Eager mode的Freeze后，需要额外对两个系统表进行更新：pg_class（relfrozenxid列）和pg_database（datfrozenxid列）。pg_class.relfrozenxid表示当前表最近一次Eager mode Freeze执行的freezeLimit_txid，目标表中所有x_min小于pg_class.relfrozenxid的Tuple都已经被Freeze。每次Eager mode Freeze后，需要把当次的freezeLimit_txid写入pg_class.relfrozenxid。pg_database.datfrozenxid表示当前Database中所有表的pg_class.relfrozenxid的最小值，当一个表的pg_class.relfrozenxid更新后，pg_database.datfrozenxid并不一定会更新。

此外，Vacuum还有个Freeze的选项，如果该选项被打开，则所有x_min < OldestXmin的Tuple都会被Freeze。

3、Eager mode（version >= 9.6）

在9.5版本前，Eager mode扫描了所有的Page，其实，有些Page是不需要扫描的。

在9.6版以后，VM中不光标记了Page中是否含有dead tuple，而且还标记了Page中的Tuple是否全部被Freeze了。若所有Tuple被Freeze了，VM(freeze)=1，若有Tuple未被Freeze，VM(freeze)=0。

在Eager mode中，若所有Tuple已经被Freeze了（VM(freeze)=1），则Freeze的过程会跳过该Page。

4、Lazy mode和Eager mode的图示

![](/techdoc/docs/database/images/lazy_eager_mode.jpg)

#### 清理CLog

前面有介绍CLog，是一个数组结构，下标为txid，值为事务状态。在磁盘上，按照8KB一个Page，256KB一个文件存储。

在Vacuum时，所有Database的最小pg_database.datfrozenxid所在CLog文件之前的CLog文件都可以被删除，因为在Database Cluster级别（包括所有Database），这之前的txid都已经被Freeze。

#### AutoVacuum

AutoVacuum守护进程会启动自动的Vacuum。默认情况下，会开启3个autovacuum_worker后台进程，启动多少进程由参数autovacuum_max_workers确定，这些进程会每隔1min睡眠一次，由参数autovacuum_naptime确定。

AutoVacuum的详细介绍可以参考：https://www.percona.com/blog/2018/08/10/tuning-autovacuum-in-postgresql-and-autovacuum-internals/

#### Full Vacuum

Concurrent VACUUM有一些不足，比如它不能缩小表大小。因为只有当最后一个Page中全部为dead tuple时，最后一个Page才会被移除，但是，如果Tuple都零星地分布在中间的Page上时，这些Page不会被移除。这样在查询时会带来性能问题。

Full Vacuum执行逻辑如下：

1. 给表加AccessExclusiveLock锁
2. 创建一个新的数据文件
3. 将live tuple拷贝到新的数据文件，如果有必要，Freeze相应Tuple
4. 移除旧的数据文件，重建索引，更新FSM、VM、统计信息和相关系统表
5. 释放AccessExclusiveLock锁
6. 更新CLog

需要注意，在Full Vacuum的时候，表不可读不可写。在执行过程中，可能会占用两倍于表的大小，需要提前检查磁盘空间。

可以使用pg_freespacemap分析何时需要进行Full Vacuum。

### Index Scan相关特性

#### Heap Only Tuple (HOT)

设计HOT的目的是为了减少Vacuum。当更新一个Tuple时，如果更新的Tuple和原Tuple在同一个Page中时，可以使用HOT。

1、没有HOT的行为

当没有HOT时，如果要更新一个带索引的表时，需要同时向索引的相应Page和数据的相应Page都插入一个Tuple。这样就导致索引Page和数据Page都存在一个dead tuple，会增加更新和Vacuum的代价。

2、有HOT的行为

有HOT时，如果未更新索引列，并且新Tuple和老Tuple在同一个数据Page上时，就可以避免更改索引Page。HOT通过t_informask2字段的HEAP_HOT_UPDATED位和HEAP_ONLY_TUPLE位标记老Tuple和新Tuple。步骤如下：

1. 索引不变，依然指向老Tuple
2. 在数据Page的line pointers区域增加一个单元，并且在heap tuples区域写入新Tuple，line pointers的这个新单元指向新Tuple
3. 让老Tuple的ctid指向新Tuple
4. 标记老Tuple的t_informask2字段的HEAP_HOT_UPDATED位，标记新Tuple的t_informask2字段HEAP_ONLY_TUPLE位

```
假设目标Tuple在第5个Page的第1个Tuple位置，更新的txid是100

更新前：
+--------+--------+--------+------------------+-----------+
| t_xmin | t_xmax | t_ctid |     t_informask2 | User Data |
+--------+--------+--------+------------------+-----------+
|     99 |      0 |  (5,1) |                  |     1,'A' |
+--------+--------+--------+------------------+-----------+

更新后：
+--------+--------+--------+------------------+-----------+
| t_xmin | t_xmax | t_ctid |     t_informask2 | User Data |
+--------+--------+--------+------------------+-----------+
|     99 |    100 |  (5,2) | HEAP_HOT_UPDATED |     1,'A' |
+--------+--------+--------+------------------+-----------+
|    100 |      0 |  (5,2) |  HEAP_ONLY_TUPLE |     1,'B' |
+--------+--------+--------+------------------+-----------+
```

在当前的操作后，如果需要读取这个新Tuple，会首先找到索引，然后通过索引找到老Tuple，通过并发控制（concurrency control）判断可见性，找到新Tuple。但是，如果老Tuple被当成dead tuple移除后，新Tuple将无法被访问到，所以，PostgreSQL设计了Pruning来解决这个问题：

5. Pruning：在合适的时间（决策较为复杂）将老Tuple的line pointer指向新Tuple的line pointer
6. defragmentation：在Pruning后，老Tuple就可以被移除了（因为无需操作索引，所以defragmentation过程比Vacuum轻量）

使用HOT，可以减少索引Page和数据Page的大小，减少Vacuum的次数，同时，在更新和Vacuum时，也可以减少对索引操作，提升了性能。

#### Index-Only Scans

如果Target list的所有列都包含在索引列中，就可以使用Index-Only Scan。这时候，一个查询只需要扫描索引的Page就可以了。

但是，因为索引不包含事务信息（没有heap tuple的t_xmin、t_xmax信息），所以通过只扫描索引，并无法获知数据的可见性。

这时候，可以依赖Visibility Map来进行辅助。Visibility Map标记了各数据Page是否有dead tuple（没有标1，有标0）。所以：

- 对于没有dead tuple的Page，直接使用索引数据即可，无需访问数据Page
- 对于有dead tuple的Page，在扫描索引后，需要访问相应数据Page，进行可见性判断

### 缓存管理（Buffer Manager）

#### 概述

Buffer Manager主要对shared memory和持久化存储间的数据传输进行管理。Buffer Manager由buffer table、buffer descriptors、buffer pool组成。

![](/techdoc/docs/database/images/pg_buffer_pool.png)

- **Buffer pool**：用于保存数据文件的Page（包括数据、索引、FM、VM）。Buffer pool是一个数组，每个元素的内容是一个Page，其数组下标被称为buffer_ids
- **Buffer descriptors**：保存Buffer pool中Page的元数据信息，是一个与Buffer pool一一对应的数组，所以其数组下标也是buffer_ids
- **Buffer table**：保存了buffer_tags（数据页面标记信息，下面介绍）和buffer_ids的对应关系，主体是一个hash表，能够由buffer_tag得到buffer_id

#### buffer_tag

在PostgreSQL中，每个数据Page都有一个标记，称为buffer_tag。buffer_tag的组成如下：

```
# buffer_tag

| RelFileNode                                  | fork number | block number |
| tablespace oid | database oid | relation oid | fork number | block number |
```

其中，

- RelFileNode包含3部分，tablespace oid、database oid和relation oid
- fork number用于标记该Page是数据Page（0）或FM Page（1）还是VM Page（2）
- block number是指数据文件的第多少个block（page）

#### 页面的请求

当一个Backend进程请求一个页面的时候，逻辑如下：

1. 将拼装好的buffer_tag发送给Buffer Manager
2. 如果Buffer Manager中有相应数据页面，则返回buffer_ID，如果Buffer Manager中没有相应的数据页面，则会去持久化存储中读取相应页面，并返回buffer_ID
3. Backend进程访问Buffer pool中buffer_ID下标的数据页

当Backend进程请求的页面不在Buffer pool中，且Buffer pool中的页面已满，则需要用页面替换算法进行页面的替换。8.1版本前，PG使用LRU算法，从8.1版本起，使用clock sweep算法。

#### 数据结构

1、Buffer table

Buffer table：负责将buffer_tag映射成buffer_id。内部是一个Hash表结构，由hash function、bucket slots、data entries组成。

2、Buffer descriptor

Buffer descriptor的主要结构有：

- tag：即数据页面的buffer_tag
- buffer_id：即数据页面在Buffer pool的下标buffer_id
- refcount：也叫pin count，记录数据页正在被多少个进程访问，有新进程访问该数据页时需+1，访问结束后-1
- usage_count：数据页被换入buffer pool后被引用了的次数，用于数据页换入换出算法
- context_lock、io_in_progress_lock：用于做访问控制
- flag：标记数据页的状态。标记数据页是否为脏页、是否是合法的，以及是否正在进行IO读写，由相应的标记位标记
- freeNext：freelist的下一个元素

Buffer descriptors有3种状态：

- Empty：refcount和usage_count均为0
- Pinned：refcount和usage_count大于等于1
- Unpinned：usage_count大于等于1，但refcount为0

在初始状态，Buffer descriptors中的每个元素都是Empty状态，他们被一个freelist串起来。当有一个页面请求到来时，将进行如下步骤：

1. 在Buffer descriptors的freelist中申请一个Empty的descriptor，并pin它
1. 在Buffer table中插入一个<buffer_tag, buffer_id>对
1. 从存储中加载一个Page到Buffer pool
1. 将这个Page的元信息存入申请的Buffer descriptor

Buffer descriptors中的descriptor不会轻易地被返回到freelist中，在下面情况会被返回freelist：

- 表、索引、数据库被DROP
- 表、索引被VACUUM FULL

3、Buffer pool

Buffer pool是一个Page数组，元素大小为8KB，和数据Page的大小一致。

#### 锁

1、Buffer table的锁

BufMappingLock是Buffer table上的一个锁，保证了整个Buffer table的数据一致性。在查找一个页面中的内容时，backend进程持有shared BufMappingLock，在插入/删除一个页面中的内容时，backend进程持有exclusive BufMappingLock。

BufMappingLock被分成多个区域（默认128个区域），来减少在Buffer table上的锁冲突。每个BufMappingLock管理Buffer table中Hash table的一部分bucket。

2、Buffer descriptor的锁

每个Buffer descriptor使用content_lock和io_in_progress_lock来控制对应Buffer pool slot的访问。当Buffer descriptor中的值被读取或改变时，还需要使用spinlock。

- content_lock用于访问控制，有shared和exclusive两种模式。在读取一个Page时，会在相应的Buffer descriptor上加shared content_lock。在插入/删除/更新/Vacuum/HOT/Freezing的时候，会加exclusive content_lock
- io_in_progress_lock用于等待buffer上的IO操作。在从存储中加载或写入Page时，会在相应的Buffer descriptor上加exclusive io_in_progress_lock
- spinlock用于Buffer descriptor自身被读取或修改的时候，一般步骤是加spinlock，修改Buffer descriptor（包括refcount、usage_count等），然后解锁。在PG 9.6版本中，spinlock被改为原子操作

#### Buffer Manager的工作流程

当一个backend进程想要访问一个Page时，需要调用ReadBufferExtended方法。此方法基于下面的几个应用场景。

1. 访问一个在Buffer pool中的Page
1. 访问一个在Buffer pool中的没有的Page，并将该Page从存储中加载到空slot中
1. 将一个Page从存储中加载到需替换的slot中

1、访问一个在Buffer pool中的Page

其步骤如下：

1. 查找buffer_tag
    1.1 为目标Page创建一个buffer_tag，并通过Buffer table的hash函数计算出相应的hash bucket slot
    1.1 在Buffer table的对应区域加shared BufMappingLock
    1.1 在Buffer table中得到与buffer_tag对应的buffer_id
1. Pin相应的Buffer descriptor，将Buffer descriptor的refcount和usage_count都加一
1. 释放Buffer table的对应区域的shared BufMappingLock
1. 访问Buffer pool中buffer_id对应的slot，如果是读取Page，需在Buffer descriptor上加shared content_lock，如果写/修改Page，需在Buffer descriptor上加exclusive content_lock，并修改Buffer descriptor的脏页标记
1. 访问结束后，修改Buffer descriptor的refcount，做减一操作

2、访问一个在Buffer pool中的没有的Page，并将该Page从存储中加载到空slot中

1. 为目标Page创建一个buffer_tag，并通过Buffer table的hash函数计算出相应的hash bucket slot
1. 在Buffer table的对应区域加shared BufMappingLock
1. 在Buffer table中查找与buffer_tag对应的buffer_id，但没找到
1. 释放Buffer table的对应区域的shared BufMappingLock
1. 从freelist中获取一个空的Buffer descriptor，并Pin它
1. 在Buffer table的对应区域加exclusive BufMappingLock 
1. 创建一个<buffer_tag, buffer_id>对，并插入Buffer table
1. 在相应Buffer descriptor上加exclusive io_in_progress_lock锁
1. 设置Buffer descriptor的io_in_progress标记为1
1. 将存储中的Page加载到Buffer pool中对应的slot
1. 修改Buffer descriptor，将io_in_progress标记置0，valid标记置1（是不是还要修改refcount和usage_count？）
1. 释放Buffer table的对应区域的exclusive BufMappingLock
1. 访问数据

3、将一个Page从存储中加载到需替换的slot中

1. 为目标Page创建一个buffer_tag，并通过Buffer table的hash函数计算出相应的hash bucket slot
1. 在Buffer table的对应区域加shared BufMappingLock
1. 在Buffer table中查找与buffer_tag对应的buffer_id，但没找到
1. 释放Buffer table的对应区域的shared BufMappingLock
1. 利用Clock Sweep算法选择一个待替换的slot，并Pin它的Buffer descriptor
1. 如果这个Page是脏页，flush这个Page的数据


#### Clock Sweep算法

#### Ring Buffer

#### 刷脏页

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
