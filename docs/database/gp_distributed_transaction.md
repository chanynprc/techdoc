## Greenplum分布式事务

### Two-Phase Commit

### 分布式Snapshot

在Greenplum中，当一个事务开始时，Coordinator节点会生成一个分布式Snapshot，并将其与查询一起发送到各个Segment节点。每个Segment节点在收到分布式Snapshot后，会创建一个本地Snapshot，并将本地事务ID（xid）映射到分布式事务ID（gxid）。这样，整个集群中的所有Segment节点都会使用相同的快照，从而确保数据的一致性。

分布式Snapshot的数据结构如下(Greenplum 7)：

```c
typedef struct DistributedSnapshot
{
	/*
	 * The lowest distributed transaction being used for distributed snapshots.
	 */
	DistributedTransactionId xminAllDistributedSnapshots;

	/*
	 * Unique number identifying this particular distributed snapshot.
	 */
	DistributedSnapshotId distribSnapshotId;

	DistributedTransactionId xmin;	/* XID < xmin are visible to me */
	DistributedTransactionId xmax;	/* XID >= xmax are invisible to me */
	int32		count;		/*  # of distributed xids in inProgressXidArray */

	/* Array of distributed transactions in progress. */
	DistributedTransactionId        *inProgressXidArray;
} DistributedSnapshot;
```

分布式Snapshot中的信息，和单机PostgreSQL的Snapshot类似，记录了xmin、xmax、xip等信息。

在执行查询时，QD会将分布式事务

执行查询时，QD 将分布式事务和快照等信息序列化，通过libpq协议发送给 QE。QE 反序列化后，获得 QD 的分布式事务和快照信息。这些信息被用于确定元组的可见性（HeapTupleSatisfiesMVCC）。所有参与查询的 QEs 都使用QD 发送的同一份分布式事务和快照信息判断元组的可见性，因而保证了整个集群数据的一致性，避免前面例子中出现的不一致现象。

在 QE 上判断一个元组对某个快照的可见性流程如下：

如果创建元组的事务：xid （即元组头中的xmin字段）还没有提交，则不需要使用分布式事务和快照信息；

否则判断创建元组的事务 xid 对快照是否可见

首先根据分布式快照信息判断。根据创建元组的 xid 从分布式事务提交日志中找到其对应的分布式事务：distribXid，然后判断 distribXid 对分布式快照是否可见：

如果 distribXid < distribSnapshot->xmin，则元组可见

如果 distribXid > distribSnapshot->xmax，则元组不可见

如果 distribSnapshot->inProgressXidArray 包含 distribXid，则元组不可见

否则元组可见

如果不能根据分布式快照判断可见性，或者不需要根据分布式快照判断可见性，则使用本地快照信息判断，这个逻辑和 PostgreSQL 的判断可见性逻辑一样。

和 PostgreSQL 的提交日志 clog 类似，Greenplum 需要保存全局事务的提交日志，以判断某个事务是否已经提交。这些信息保存在共享内存中并持久化存储在 distributedlog 目录下。

为了提高判断本地 xid 可见性的效率，避免每次访问全局事务提交日志，Greenplum 引入了本地事务-分布式事务提交缓存，如下图所示。每个 QE 都维护了这样一个缓存，通过该缓存，可以快速查到本地 xid 对应的全局事务distribXid 信息，进而根据全局快照判断可见性，避免频繁访问共享内存或磁盘。

CENTER_PostgreSQL_Community

共享本地快照（Shared Local Snapshot）

Greenplum 中一个 SQL 查询计划可能含有多个 slices，每个 Slice 对应一个 QE 进程。任一 segment 上，同一会话（处理同一个用户SQL）的不同 QE 必须有相同的可见性。然而每个 QE 进程都是独立的 PostgreSQL backend进程，它们之间互相不知道对方的存在，因而其事务和快照信息都是不一样的。如下图所示。

CENTER_PostgreSQL_Community

为了保证跨slice可见性的一致性，Greenplum引入了 “共享本地快照(Shared Local Snapshot)” 的概念。每个 segment 上的执行同一个SQL的不同 QEs 通过共享内存数据结构 SharedSnapshotSlot 共享会话和事务信息。这些进程称为 SegMate 进程组。

Greenplum 把 SegMate 进程组中的 QE 分为 QE writer 和 QE reader。QE writer 有且只有一个，QE reader 可以没有或者多个。QE writer 可以修改数据库状态；QE reader 不能修改数据库的状态，且需要使用和 QE writer 一样的快照信息以保持与 QE writer 一致的可见性。如下图所示。

CENTER_PostgreSQL_Community

“共享”意味着该快照在 QE writer 和 readers 间共享，“本地” 意味着这个快照是 segment 的本地快照，同一用户会话在不同的 segment 上可以有不同的快照。segment 的共享内存中有一个区域存储共享快照，该区域被分成很多槽（slots）。一个 SegMate 进程组对应一个槽，通过的会话id标志。一个 segment 可能有多个 SegMate 进程组，每个进程组对应一个用户的会话，如下图所示。

CENTER_PostgreSQL_Community

QE Writer 创建本地事务后，在共享内存中获得一个 SharedLocalSnapshot 槽，并它自己的本地事务和快照信息拷贝到共享内存槽中，SegMate 进程组中的其他 QE Reader 从该共享内存中获得事务和快照信息。Reader QEs 会等待 Writer QE 直到 Writer 设置好共享本地快照信息。

只有 QE writer 参与全局事务，也只有该 QE 需要处理 commit/abort 等事务命令。

### GTM（Global Transaction Manager）

在Greenplum中，master节点上的GTM会维护全局的分布式事务状态。

### 引用


