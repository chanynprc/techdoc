## Greenplum分布式事务

### 事务ID

在PostgreSQL的事务ID基础之上，Greenplum构建了一套在分布式集群上保持事务一致性的全局事务ID。在Greenplum 7中，全局事务ID被改成了64位，此前是32位。

```c
typedef uint32 TransactionId;

typedef uint64 DistributedTransactionId;
```

#### 本地事务ID和全局事务ID之间的映射

在Greenplum中，使用DLog（Distributed Log）来保存本地事务ID和全局事务ID之间的映射。本质上，DLog是一个DistributedTransactionId的数组，数组的下标和本地事务ID相关，内容是全局事务ID。DLog保存在共享内存中并持久化存储在pg_distributedlog目录下。

```c
typedef struct DistributedLogEntry
{
	DistributedTransactionId distribXid;
} DistributedLogEntry;
```

本地事务ID到全局事务ID的转换逻辑，就是在DLog数组中通过本地事务ID计算的下标定位全局事务ID的过程。

```c
#define ENTRIES_PER_PAGE (BLCKSZ / sizeof(DistributedLogEntry))

#define TransactionIdToPage(localXid) ((localXid) / (TransactionId) ENTRIES_PER_PAGE)
#define TransactionIdToEntry(localXid) ((localXid) % (TransactionId) ENTRIES_PER_PAGE)

void
DistributedLog_GetDistributedXid(
	TransactionId 						localXid,
	DistributedTransactionId 			*distribXid)
{
	int			page = TransactionIdToPage(localXid);
	int			entryno = TransactionIdToEntry(localXid);
	int			slotno;
	DistributedLogEntry *ptr;

	Assert(!IS_QUERY_DISPATCHER());

	slotno = SimpleLruReadPage_ReadOnly(DistributedLogCtl, page, localXid);
	ptr = (DistributedLogEntry *) DistributedLogCtl->shared->page_buffer[slotno];
	ptr += entryno;
	*distribXid = ptr->distribXid;
	LWLockRelease(DistributedLogControlLock);
}
```

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

分布式Snapshot中的信息，和PostgreSQL的Snapshot类似，记录了xmin、xmax、xip_list等信息。关于PostgreSQL的相关信息，可以参考[PostgreSQL概览](/techdoc/docs/database/pg_overview)。

#### 分布式事务可见性判断

在执行查询时，QD会将分布式事务信息（包含当前事务的分布式Snapshot）序列化后通过libpq协议发送给QE，QE反序列化后获得QD的分布式事务信息。所有的QE都是用QD发送的同一份分布式事务信息来判断数据的可见性，进而保证了整个集群数据的一致性。

在QE上判断一个元组对某个Snapshot的可见性算法如下：

1. 如果写入元组的事务（即元组头中的xmin字段）还没有提交，则该元组不可见
2. 否则，需要判断写入元组的事务xid对Snapshot是否可见
  1. 首先，会根据分布式Snapshot信息判断，将元组的xmin从分布式事务提交日志（DLog）中找到其对应的分布式事务gxid，然后判断此事务的gxid对分布式Snapshot是否可见
    1. 如果元组事务gxid < 分布式Snapshot->xmin，则元组可见
    2. 如果元组事务gxid > 分布式Snapshot->xmax，则元组不可见
    3. 如果分布式Snapshot->inProgressXidArray包含元组事务gxid，则元组不可见
    4. 否则元组可见
  2. 如果不能根据分布式Snapshot判断可见性，或者不需要根据分布式Snapshot判断可见性，则使用本地Snapshot信息判断，这个逻辑和PostgreSQL的可见性判断逻辑一样

> [尚未确认] 为了提高判断本地 xid 可见性的效率，避免每次访问全局事务提交日志，Greenplum 引入了本地事务-分布式事务提交缓存，如下图所示。每个 QE 都维护了这样一个缓存，通过该缓存，可以快速查到本地 xid 对应的全局事务distribXid 信息，进而根据全局快照判断可见性，避免频繁访问共享内存或磁盘。

#### 共享本地Snapshot

Greenplum中一个SQL的查询计划可能含有多个Slices，在每个segment上每个Slice对应一个QE进程。任一Segment上，同一会话（处理同一个用户SQL）的不同QE必须有相同的可见性。然而每个QE进程都是独立的PostgreSQL backend进程，它们之间互相不知道对方的存在，因而其事务和快照信息都是不一样的。

Greenplum中每个Segment上执行同一个SQL的QE进程放在一起被称为SegMate进程组，同时又把SegMate进程组中的QE分为QE writer和QE reader：

- QE writer：有且只有一个，可以修改数据库状态
- QE reader：可以没有或者多个，不能修改数据库的状态，且需要使用和QE writer一样的快照信息以保持与QE writer一致的可见性

为了保证同一个SegMate中跨Slice可见性的一致性，Greenplum引入了“共享本地快照（Shared Local Snapshot）”的概念，每个Segment上执行同一个SQL的不同QEs（QE Writer和QE Readers）通过共享内存数据结构SharedSnapshotSlot来共享会话和事务信息。

```c
typedef struct SharedSnapshotSlot
{
	int32			slotindex;  /* where in the array this one is. */
	int32	 		slotid;
	PGPROC			*writer_proc;
	PGXACT			*writer_xact;

	/* only used by cursor dump identification, dose not always set */
	volatile DistributedTransactionId distributedXid;

	volatile bool			ready;
	volatile uint32			segmateSync;
	SnapshotData	snapshot;
	LWLock		   *slotLock;

	volatile int    cur_dump_id;
	volatile SnapshotDump    dump[SNAPSHOTDUMPARRAYSZ];
	/* for debugging only */
	FullTransactionId	fullXid;
	TimestampTz		startTimestamp;
} SharedSnapshotSlot;
```

既然叫共享“本地”Snapshot，那就意味着这个Snapshot是Segment的本地Snapshot。Segment的共享内存中有一个区域存储共享本地Snapshot，该区域被分成很多槽（slots），一个SegMate进程组对应一个槽，通过的会话ID来标志，SharedSnapshotSlot.slotid就是gp_session_id。一个Segment可能有多个SegMate进程组，每个进程组对应一个用户的会话。

```c
typedef struct SharedSnapshotStruct
{
	int 		numSlots;		/* number of valid Snapshot entries */
	int			maxSlots;		/* allocated size of sharedSnapshotArray */
	int 		nextSlot;		/* points to the next avail slot. */

	/*
	 * We now allow direct indexing into this array.
	 *
	 * We allocate the XIPS below.
	 *
	 * Be very careful when accessing fields inside here.
	 */
	SharedSnapshotSlot	   *slots;

	TransactionId	   *xips;		/* VARIABLE LENGTH ARRAY */
} SharedSnapshotStruct;

static volatile SharedSnapshotStruct *sharedSnapshotArray;
```

QE Writer创建本地事务后，在共享内存中获得一个共享本地Snapshot槽，并它自己的本地事务和Snapshot信息拷贝到共享内存槽中，SegMate进程组中的其他QE Readers从该共享内存中获得事务和Snapshot信息。QE Readers会等待QE Writer，直到QE Writer设置好共享本地Snapshot信息。

只有QE Writer参与全局事务，也只有该QE需要处理commit及abort等事务命令。

### GTM（Global Transaction Manager）

在Greenplum中，master节点上的GTM会维护全局的分布式事务状态。

### 引用


