# Postgresql 共享内存

Postgresql 共享内存是 整个 Postgresql 实例下所有进程都共同访问的 一块内存区域，比如 postmaster、checkpointer 、background writer、backend等所有当前实例下的进程，PostgreSQL 的共享内存最终依赖的是**操作系统提供的共享内存机制**。比如 System V Shared Memory 的 `shmget`，`shmat`。

Postgresql 创建共享内存的入口函数是 `CreateSharedMemoryAndSemaphores` ，该函数内部调用 `PGSharedMemoryCreate`来创建共享内存，创建的是 PostgreSQL 整个实例（instance）唯一的一块主共享内存（Main Shared Memory Segment），几乎所有 backend 都通过它共享数据。

# PG 共享内存布局

Postgresql 共享内存会划分成不同的组成部分：

```tex
                 PG Shared Memory
+-------------------------------------------------------------+
| PGShmemHeader                                                |
+-------------------------------------------------------------+
| ShmemIndex                                                  |
+-------------------------------------------------------------+
| LWLock Array                                                |
+-------------------------------------------------------------+
| ProcGlobal (PGPROC数组)                                     |
+-------------------------------------------------------------+
| XACT / CLOG / SUBTRANS 等共享状态                           |
+-------------------------------------------------------------+
| WAL 共享控制结构                                             |
+-------------------------------------------------------------+
| BufferDesc[]                                                |
+-------------------------------------------------------------+
| Buffer Pool(shared_buffers真正的数据页)                     |
+-------------------------------------------------------------+
| Lock Manager                                                |
+-------------------------------------------------------------+
| Predicate Lock                                               |
+-------------------------------------------------------------+
| Replication                                                  |
+-------------------------------------------------------------+
| Dynamic Shared Memory Control                               |
+-------------------------------------------------------------+
| ...                                                         |
+-------------------------------------------------------------+
```

其中：

- PGShmemHeader   ： 共享内存头部信息
- ShmemIndex   ： 共享内存目录索引， 是一个哈希表，保存共享内存模块名字到地址的映射。
- BufferShmemSize ： buffer pool区域
- XLOGShmemSize ： wal buffers 区域

上述内存布局产生的源代码：

```c
// src/backend/storage/ipc/ipci.c
void
CreateSharedMemoryAndSemaphores(void)
{

		/* Compute the size of the shared-memory block */
		size = CalculateShmemSize(&numSemas);

		/*
		 * Create the shmem segment
		 */
		seghdr = PGSharedMemoryCreate(size, &shim);

		InitShmemAccess(seghdr);

	/*
	 * Set up shmem.c index hashtable
	 */
	InitShmemIndex();

	dsm_shmem_init();

	/*
	 * Set up xlog, clog, and buffers
	 */
	XLOGShmemInit();
	XLogPrefetchShmemInit();
	XLogRecoveryShmemInit();
	CLOGShmemInit();
	CommitTsShmemInit();
	SUBTRANSShmemInit();
	MultiXactShmemInit();
	InitBufferPool();

	/*
	 * Set up lock manager
	 */
	InitLocks();


	/*
	 * Set up interprocess signaling mechanisms
	 */
	PMSignalShmemInit();
	ProcSignalShmemInit();
	CheckpointerShmemInit();
	AutoVacuumShmemInit();
	ReplicationSlotsShmemInit();
	ReplicationOriginShmemInit();
	WalSndShmemInit();
	WalRcvShmemInit();
	PgArchShmemInit();
	ApplyLauncherShmemInit();

	/*
	 * Set up other modules that need some shared memory space
	 */
	SnapMgrInit();
	BTreeShmemInit();
	SyncScanShmemInit();
	AsyncShmemInit();
	StatsShmemInit();
}
```

真正使用系统调用创建共享内存之前会调用函数 `CalculateShmemSize` 计算共享内存的大小，共享内存的大小包括：

- ```c
  CalculateShmemSize 
  	size = add_size(size, PGSemaphoreShmemSize(numSemas));
  	size = add_size(size, SpinlockSemaSize());
  	size = add_size(size, hash_estimate_size(SHMEM_INDEX_SIZE,
  											 sizeof(ShmemIndexEnt)));
  	size = add_size(size, dsm_estimate_size());
  	size = add_size(size, BufferShmemSize());       // buffer pool 
  	size = add_size(size, LockShmemSize());
  	size = add_size(size, PredicateLockShmemSize());
  	size = add_size(size, ProcGlobalShmemSize());
  	size = add_size(size, XLogPrefetchShmemSize());
  	size = add_size(size, XLOGShmemSize());        // wal buffers
  	size = add_size(size, XLogRecoveryShmemSize());
  	size = add_size(size, CLOGShmemSize());
  	size = add_size(size, CommitTsShmemSize());
  	size = add_size(size, SUBTRANSShmemSize());
  	size = add_size(size, TwoPhaseShmemSize());
  	size = add_size(size, BackgroundWorkerShmemSize());
  	size = add_size(size, MultiXactShmemSize());
  	size = add_size(size, LWLockShmemSize());
  	size = add_size(size, ProcArrayShmemSize());
  	size = add_size(size, BackendStatusShmemSize());
  	size = add_size(size, SInvalShmemSize());
  	size = add_size(size, PMSignalShmemSize());
  	size = add_size(size, ProcSignalShmemSize());
  	size = add_size(size, CheckpointerShmemSize());
  	size = add_size(size, AutoVacuumShmemSize());
  	size = add_size(size, ReplicationSlotsShmemSize());
  	size = add_size(size, ReplicationOriginShmemSize());
  	size = add_size(size, WalSndShmemSize());
  	size = add_size(size, WalRcvShmemSize());
  	size = add_size(size, PgArchShmemSize());
  	size = add_size(size, ApplyLauncherShmemSize());
  	size = add_size(size, SnapMgrShmemSize());
  	size = add_size(size, BTreeShmemSize());
  	size = add_size(size, SyncScanShmemSize());
  	size = add_size(size, AsyncShmemSize());
  	size = add_size(size, StatsShmemSize());
  // ...
  ```

初始化头部信息，哈希索引，以及各个模块的函数为

```c
// 设置共享内存基础指针
InitShmemAccess
// 设置共享内存哈希表目录
InitShmemIndex
    
// 切分 WAL buffers
XLOGShmemInit
// 切分出 Buffer Pool
InitBufferPool();
...
```

# 哪些进程会使用共享内存

这是理解 PostgreSQL 进程模型最关键的问题之一。

答案是：

> **几乎所有属于同一个 PostgreSQL 实例（同一个 DataDir）的进程都会映射并使用这块共享内存。**

可以先记住一句话：

> **这块共享内存就是整个 PostgreSQL 实例所有进程共同的数据中心。**

------

## 哪些进程会使用？

假设启动 PostgreSQL：

```bash
pg_ctl start
```

进程树大概是：

```text
                 postmaster
                      │
    ┌─────────────────┼──────────────────┐
    │                 │                  │
checkpointer      background writer   wal writer
    │                 │                  │
    ├──────────────┬──┴──────────────┐
    │              │                 │
 autovacuum     wal sender       logical launcher
    │
    ├─────────────────────────────────────────────
    │
 client1
    │
 backend1

 client2
    │
 backend2

 client3
    │
 backend3
```

**上面这些进程，全部都会 attach 同一块共享内存。**

也就是说：

```text
                 Shared Memory
+------------------------------------------------------+
| Buffer Pool                                          |
| BufferDesc                                           |
| PGPROC                                               |
| ProcArray                                            |
| LWLock                                               |
| Lock Manager                                         |
| WAL Control                                          |
| ...                                                  |
+------------------------------------------------------+
        ▲          ▲          ▲          ▲
        │          │          │          │
   postmaster   backend1   backend2   checkpointer
```

大家看到的是**同一块物理内存**。

------

## 为什么每个进程都需要？

因为每个进程负责不同工作，但需要共享状态。

------

### ① Postmaster

虽然 Postmaster 基本不执行 SQL，

但它也 attach 共享内存。

因为它要：

例如：

```text
创建共享内存

初始化 Header

启动后台进程

检查共享内存状态
```

所以：

```text
postmaster
```

也是共享内存的拥有者。

------

### ② Backend（最重要）

例如：

客户端：

```sql
SELECT * FROM t;
```

backend：

首先：

看：

```text
Buffer Pool
```

有没有：

```text
table page
```

如果：

有：

直接：

返回。

如果：

没有：

读磁盘。

放进：

```text
Shared Buffer
```

所以：

backend：

天天访问：

```text
Shared Memory
```

------

例如：

```sql
UPDATE t
```

backend：

需要：

访问：

```text
Lock Table

PGPROC

ProcArray

BufferDesc

WAL
```

全部：

共享内存。

------

### ③ Checkpointer

Checkpoint：

需要：

扫描：

```text
BufferDesc[]
```

看看：

哪些：

```text
dirty page
```

需要：

刷盘。

所以：

不停访问：

```text
Buffer Pool
```

------

### ④ Background Writer

作用：

提前：

刷：

```text
dirty buffer
```

所以：

也一直：

访问：

```text
BufferDesc

Buffer Pool
```

------

### ⑤ WAL Writer

它：

看：

```text
XLOG Control
```

例如：

```text
InsertPos

WritePos

FlushPos
```

然后：

写：

WAL。

------

### ⑥ Autovacuum

VACUUM：

需要：

访问：

```text
ProcArray

Buffer

Lock
```

等等。

------

### ⑦ WAL Sender

流复制：

需要：

知道：

```text
WAL Position
```

所以：

看：

```text
XLogCtl
```

------

### ⑧ Logical Replication

例如：

```text
Replication Slot

Origin
```

也：

共享。

------

## 有没有进程不用？

基本没有。

**凡是 PostgreSQL 自己启动的后台进程，都 attach 这块共享内存。**

例如：

```text
checkpointer

walwriter

autovacuum

stats

backend

parallel worker

logical worker
```

全部：

attach。

------

## 为什么 fork 后不用重新 attach？

Linux是：

```text
postmaster

↓

fork()

↓

backend
```

fork会继承：

```text
虚拟地址空间
```

因此：

```text
postmaster

0x7f3000000000
```

backend：

也是：

```text
0x7f3000000000
```

共享内存已经映射好了。

所以 Linux backend通常不用再次：

```text
shmat()
```

------

## 哪些东西在共享内存中最常访问？

几乎所有 backend 每执行一条 SQL，都会访问下面这些：

| 模块              | 谁使用                               | 作用                                |
| ----------------- | ------------------------------------ | ----------------------------------- |
| **Shared Buffer** | 所有 backend、checkpointer、bgwriter | 数据页缓存                          |
| **BufferDesc**    | 所有 backend                         | Buffer 元数据（是否脏页、pin 数等） |
| **LWLock**        | 所有进程                             | 同步访问共享数据                    |
| **PGPROC**        | 所有 backend                         | 每个进程的运行状态                  |
| **ProcArray**     | 所有 backend                         | MVCC 快照、活跃事务                 |
| **Lock Manager**  | 所有 backend                         | 表锁、行锁、事务锁                  |
| **XLogCtl**       | backend、walwriter、walsender        | WAL 插入、写入、刷盘状态            |
| **SInval**        | 所有 backend                         | 系统缓存失效通知                    |

------

## 一个 SQL 执行时会发生什么？

例如：

```sql
UPDATE t SET name='Tom' WHERE id=1;
```

backend 大概会访问：

```text
Shared Memory

│
├── ProcArray
│      获取 Snapshot
│
├── Lock Manager
│      获取 Row Lock
│
├── BufferDesc
│      找 Buffer
│
├── Shared Buffer
│      修改 Page
│
├── XLogCtl
│      写 WAL
│
└── PGPROC
       更新事务状态
```

可以看到，一条普通 SQL 就会涉及多个共享内存模块的协同工作。

------

## 总结

可以把 PostgreSQL 的共享内存理解成一个**所有数据库进程共同维护的"中央工作区"**：

```text
                     PostgreSQL Shared Memory
+--------------------------------------------------------------+
| Buffer Pool | Lock Manager | PGPROC | ProcArray | WAL | ...  |
+--------------------------------------------------------------+
      ▲             ▲             ▲          ▲          ▲
      │             │             │          │          │
 postmaster     backend1      backend2   bgwriter   checkpointer
      │             │             │          │          │
      └───────────── 所有 PostgreSQL 进程共享访问 ─────────────┘
```

**除了 PostgreSQL 自己启动的这些进程之外，其他普通用户进程（例如 `ls`、`vim`、你的应用程序）不会使用这块共享内存，除非它们主动知道共享内存的 key 并调用 `shmat()` 去映射它。**