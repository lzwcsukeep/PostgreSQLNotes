`src/include/storage/proc.h` 文件中结构体 `PGPROC` 解析

这个 `PGPROC` 是 **PostgreSQL 后端进程（backend process）在共享内存中的核心状态结构**。可以把它理解成：

> **每一个 PostgreSQL backend 进程，在共享内存里都有一个 `PGPROC`，用来记录“这个进程是谁、正在执行什么事务、持有哪些锁、正在等待什么、是否在同步复制中等待”等运行状态。**

它位于 PostgreSQL 源码的 `src/include/storage/proc.h`。

需要特别注意：**`PGPROC` 不是操作系统进程本身**。
它是 PostgreSQL 为 OS 进程建立的一份**共享内存控制块/状态描述对象**。

------

# 1. 先从整体关系理解

可以先把它想象成：

```text
                    PostgreSQL Shared Memory
┌─────────────────────────────────────────────────────────┐
│                                                         │
│   ProcGlobal                                             │
│      │                                                  │
│      ├── allProcs[]                                     │
│      │      │                                           │
│      │      ├── PGPROC  ← backend process #1            │
│      │      ├── PGPROC  ← backend process #2            │
│      │      ├── PGPROC  ← backend process #3            │
│      │      └── ...                                     │
│      │                                                  │
│      ├── xids[]                                         │
│      ├── statusFlags[]                                  │
│      ├── subxidStates[]                                 │
│      └── ...                                            │
│                                                         │
└─────────────────────────────────────────────────────────┘
          ↑
          │
      OS Backend
      PID = 12345
```

例如：

```text
psql
  │
  │ TCP / Unix socket
  ↓
postgres backend
PID = 12345
  │
  │ 对应共享内存中的
  ↓
PGPROC
```

所以：

```c
PGPROC *MyProc;
```

通常可以理解成：

> **当前 backend 在 PostgreSQL 共享内存中的“身份证 + 运行状态 + 锁/事务状态”。**

------

# 2. 为什么需要 PGPROC？

PostgreSQL 是多进程架构。

假设有：

```text
Client A → Backend A
Client B → Backend B
Client C → Backend C
```

这些 backend 是不同的 OS 进程。

但是它们需要互相知道：

- A 正在执行哪个事务？
- A 的 `xmin` 是多少？
- A 是否持有某个锁？
- B 是否正在等待锁？
- C 是否正在等待 WAL flush？
- 哪些事务正在运行？
- 哪些事务可以阻止 VACUUM 回收 tuple？
- 哪个进程持有某个 LWLock？
- 哪个进程正在等待某个 LWLock？

所以 PostgreSQL 在共享内存中为每个 backend 保存一个：

```c
PGPROC
```

这样其他 backend 就能够查看它。

------

# 3. `links`：PGPROC 可以挂到各种队列

```c
dlist_node links;
dlist_head *procgloballist;
```

首先看：

```c
dlist_node links;
```

注释特别强调：

```c
/* proc->links MUST BE FIRST IN STRUCT */
```

也就是说：

```text
PGPROC
┌──────────────────────────────┐
│ links                        │ ← 必须第一个
├──────────────────────────────┤
│ procgloballist               │
│ sem                          │
│ waitStatus                   │
│ procLatch                    │
│ ...                          │
└──────────────────────────────┘
```

为什么必须第一个？

因为 PostgreSQL 的某些链表/等待机制会直接通过 `PGPROC` 的地址和 `links` 做转换。

`links` 可以让一个 `PGPROC` 被挂到各种 list 中。

例如：

```text
Lock A
  │
  └── waitProcs
         │
         ├── PGPROC backend1
         ├── PGPROC backend2
         └── PGPROC backend3
```

如果 backend 正在等待锁：

```c
proc->links
```

就可能被用于连接到这个 lock 的等待队列。

而空闲的 PGPROC：

```text
ProcGlobal->freeProcs
        │
        ├── PGPROC
        ├── PGPROC
        └── PGPROC
```

也会通过这个链表节点连接。

------

# 4. `sem`：backend 睡眠/唤醒机制

```c
PGSemaphore sem;
ProcWaitStatus waitStatus;
```

这是 PostgreSQL 进程等待机制的重要部分。

例如：

```sql
BEGIN;

-- Backend A
LOCK TABLE t;
```

如果锁拿不到：

```text
Backend A
    │
    ├── 请求 Lock
    │
    ├── Lock 冲突
    │
    └── sleep
          ↓
      semaphore
```

其他进程释放锁以后：

```text
Backend B
   │
   └── Unlock
        │
        ↓
    唤醒 Backend A
```

所以 `sem` 是 PGPROC 与进程等待/唤醒相关的重要设施。

------

# 5. `procLatch`：Latch

```c
Latch procLatch;
```

PostgreSQL 的 `Latch` 是一种非常重要的**进程唤醒机制**。

例如一个 backend 正在：

```c
WaitLatch(...)
```

另一个进程可以通过：

```c
SetLatch(...)
```

通知它。

它和 semaphore 都与等待有关，但用途和实现机制不同。

你之前看过 PostgreSQL 的：

```text
WaitEventSetWait()
    ↓
epoll_wait()
```

这一套等待机制时，就会经常看到 `Latch`。

可以粗略理解为：

```text
PGPROC
 ├── sem        → 某些进程同步/锁等待
 │
 └── procLatch  → PostgreSQL 通用进程唤醒
```

------

# 6. `xid`：当前事务 ID

这是 PGPROC 中非常重要的字段：

```c
TransactionId xid;
```

注释：

```text
id of top-level transaction currently being
executed by this proc
```

例如：

```sql
BEGIN;

INSERT INTO t VALUES (...);
```

如果这个 backend 的事务已经分配 XID：

```text
PGPROC
   │
   └── xid = 500
```

其他 PostgreSQL 机制可以通过 ProcArray 找到：

```text
XID 500
   ↓
哪个 backend 正在运行？
   ↓
对应 PGPROC
```

它直接参与 PostgreSQL 的 MVCC 判断。

------

# 7. `xmin`：非常重要的 MVCC 字段

```c
TransactionId xmin;
```

这个字段和 VACUUM/MVCC 密切相关。

它表示：

> 当前事务开始时，系统认为仍然可能影响可见性的最小运行事务 ID。

注释里：

```text
vacuum must not remove tuples deleted by
xid >= xmin
```

例如：

```text
Transaction A
xmin = 100

Transaction B
    DELETE tuple
    xid = 150
```

如果 A 仍然可能需要看到旧版本：

```text
VACUUM
   │
   └── 不能随便清理 xid >= 100 相关的旧 tuple
```

所以：

```text
PGPROC->xmin
```

是 PostgreSQL MVCC / VACUUM 判断的重要依据之一。

------

# 8. `lxid`

```c
LocalTransactionId lxid;
```

这个和：

```c
TransactionId xid;
```

不要混淆。

简单区分：

```text
xid
    ↓
全局事务 ID
    ↓
MVCC / clog / transaction visibility

lxid
    ↓
本地事务 ID
    ↓
backend 本地使用
```

------

# 9. `pid`

```c
int pid;
```

这个非常直观：

```text
PGPROC
   │
   └── pid = 12345
```

对应 OS 层：

```bash
ps
```

看到的 PostgreSQL backend PID。

特殊情况：

```c
pid == 0
```

表示它可能是：

> **prepared transaction 对应的 dummy PGPROC**

也就是注释里说的：

```text
prepared-xact PGPROC
```

它不是一个真正正在运行的 backend。

但是 PostgreSQL 需要让 prepared transaction：

- 出现在 ProcArray
- 看起来像仍然持有事务
- 正确显示锁

所以创建了一个“假的 PGPROC”。

------

# 10. `pgxactoff` 和 `pgprocno`

这两个非常容易混淆。

```c
int pgxactoff;
int pgprocno;
```

## `pgprocno`

```c
ProcGlobal->allProcs[n].pgprocno == n
```

所以它就是：

> 这个 PGPROC 在 `allProcs[]` 中的编号。

例如：

```text
allProcs[]

[0] PGPROC
[1] PGPROC
[2] PGPROC ← pgprocno = 2
[3] PGPROC
```

------

## `pgxactoff`

这个更加特殊。

PostgreSQL 把 PGPROC 中一些经常访问的字段，另外复制到了更加紧凑的数组中。

例如：

```text
PGPROC
 ├── xid
 ├── statusFlags
 ├── subxidStatus
 └── ...

              mirrored
                  ↓

ProcGlobal
 ├── xids[]
 ├── statusFlags[]
 ├── subxidStates[]
 └── ...
```

`pgxactoff` 就是：

> PGPROC 在这些 ProcGlobal 紧凑数组中的索引。

例如：

```text
PGPROC
    pgxactoff = 10

ProcGlobal
    xids[10]
    statusFlags[10]
    subxidStates[10]
```

------

# 11. 为什么要 mirror？

这是 PostgreSQL 性能设计中比较重要的一点。

如果：

```c
PGPROC
```

非常大，而 ProcArray 又需要频繁扫描很多 backend：

```text
扫描 1000 个 PGPROC
```

那么 CPU cache locality 不好。

因此 PostgreSQL 把经常一起扫描的数据单独压紧：

```text
PGPROC
┌──────────────────────┐
│ 很多字段              │
│ 很多锁相关数据        │
│ 很多等待相关数据      │
│ ...                   │
└──────────────────────┘


ProcGlobal arrays

xids:
┌────┬────┬────┬────┬────┐
│ X1 │ X2 │ X3 │ X4 │ ...│
└────┴────┴────┴────┴────┘

statusFlags:
┌────┬────┬────┬────┬────┐
│ S1 │ S2 │ S3 │ S4 │ ...│
└────┴────┴────┴────┴────┘
```

所以 `PGPROC` 里面有：

```c
xid
statusFlags
subxidStatus
```

而这些又分别 mirror 到：

```c
ProcGlobal->xids[]
ProcGlobal->statusFlags[]
ProcGlobal->subxidStates[]
```

------

# 12. backend 基本身份信息

```c
BackendId backendId;
Oid databaseId;
Oid roleId;
Oid tempNamespaceId;
bool isBackgroundWorker;
```

这些可以看成：

```text
PGPROC
│
├── backendId
│
├── databaseId
│
├── roleId
│
├── tempNamespaceId
│
└── isBackgroundWorker
```

分别表示：

### `backendId`

PostgreSQL 内部 backend ID。

### `databaseId`

当前连接的数据库：

```text
databaseId
    ↓
pg_database.oid
```

### `roleId`

当前使用的 role：

```text
roleId
    ↓
pg_authid.oid
```

### `tempNamespaceId`

当前 backend 的临时 schema：

```sql
CREATE TEMP TABLE ...
```

背后会有自己的 temp namespace。

### `isBackgroundWorker`

判断：

```text
这个 PGPROC
    ↓
普通 backend？
还是
background worker？
```

------

# 13. `recoveryConflictPending`

```c
bool recoveryConflictPending;
```

主要和：

> Hot Standby / recovery

有关。

例如 standby 正在 replay WAL：

```text
Primary
   │
   │ WAL
   ↓
Standby
   │
   └── replay
```

如果 WAL replay 与某个正在运行的查询发生冲突：

```text
WAL replay
    │
    └── conflict
          │
          ↓
     backend query
```

这个字段用于记录相关 conflict signal 状态。

------

# 14. LWLock 等待信息

这一组：

```c
uint8 lwWaiting;
uint8 lwWaitMode;
proclist_node lwWaitLink;
```

非常重要。

如果 backend 正在等待一个 LWLock：

```text
Backend A
   │
   └── 请求 LWLock
           │
           └── 没拿到
                 ↓
              waiting
```

PGPROC 中会记录：

```text
lwWaiting
lwWaitMode
lwWaitLink
```

其中：

```c
lwWaitLink
```

用于把当前 PGPROC 放到 LWLock 的等待队列里。

------

# 15. Condition Variable

```c
proclist_node cvWaitLink;
```

用于 PostgreSQL condition variable。

也就是：

```text
Condition Variable
       │
       └── waiting processes
               │
               ├── PGPROC A
               ├── PGPROC B
               └── PGPROC C
```

------

# 16. Lock Manager 等待信息

这是 `PGPROC` 最核心的一组内容之一：

```c
LOCK	   *waitLock;
PROCLOCK   *waitProcLock;
LOCKMODE	waitLockMode;
LOCKMASK	heldLocks;
pg_atomic_uint64 waitStart;
```

如果：

```sql
LOCK TABLE foo;
```

但是被其他事务阻塞：

```text
Backend A
    │
    │ LOCK foo
    ↓
  LOCK object
    │
    ├── Backend B holds lock
    │
    └── Backend A waits
```

那么：

```c
PGPROC A
```

里面：

```c
waitLock
```

指向正在等待的 `LOCK` 对象。

------

## `waitProcLock`

这个和 PostgreSQL Lock Manager 的内部结构有关。

大致关系：

```text
LOCK
 │
 ├── PROCLOCK ← Backend A
 │
 ├── PROCLOCK ← Backend B
 │
 └── ...
```

`PROCLOCK` 表示：

> 某个 PGPROC 与某个 LOCK 之间的关系。

所以：

```c
waitProcLock
```

表示当前 backend 与等待锁之间对应的 `PROCLOCK`。

------

## `waitLockMode`

例如：

```text
AccessShareLock
RowExclusiveLock
AccessExclusiveLock
...
```

表示：

> 当前 backend 正在等待什么类型的锁。

------

## `heldLocks`

```c
LOCKMASK heldLocks;
```

记录：

> 当前 backend 在这个 lock object 上已经持有哪些 lock mode。

------

## `waitStart`

```c
pg_atomic_uint64 waitStart;
```

记录：

> 开始等待锁的时间。

这对于：

```text
lock wait monitoring
```

非常重要。

例如：

```sql
SELECT *
FROM pg_stat_activity;
```

里面看到：

```text
wait_event_type = Lock
wait_event = ...
```

背后的等待状态，就和这一整套机制有关。

------

# 17. `delayChkptFlags`

```c
int delayChkptFlags;
```

用于控制：

> checkpoint 是否需要因为当前 backend 的某些操作而延迟。

也就是 PostgreSQL checkpoint 与 backend 协作时的一些状态。

------

# 18. `statusFlags`

```c
uint8 statusFlags;
```

记录 backend 的一些状态标志。

注释：

```text
mirrored in
ProcGlobal->statusFlags[pgxactoff]
```

所以：

```text
PGPROC->statusFlags
       ↕
ProcGlobal->statusFlags[pgxactoff]
```

两边需要保持一致。

------

# 19. 同步复制相关字段

这一组：

```c
XLogRecPtr	waitLSN;
int			syncRepState;
dlist_node	syncRepLinks;
```

用于：

> Synchronous Replication

例如：

```text
Primary
  │
  ├── WAL
  ↓
WAL Sender
  │
  ↓
Standby
```

如果事务要求同步复制：

```text
COMMIT
  │
  ↓
等待 standby 确认
  │
  └── waitLSN
```

`waitLSN`：

> 当前 backend 等待 standby 至少 replay/flush 到哪个 LSN。

`syncRepState`：

> 当前同步复制等待状态。

`syncRepLinks`：

> 把 backend 加入 synchronous replication wait queue。

------

# 20. `myProcLocks[]`：这个 backend 持有的所有锁

这是 Lock Manager 非常重要的一部分：

```c
dlist_head myProcLocks[NUM_LOCK_PARTITIONS];
```

它的意思是：

> 当前 PGPROC 持有/等待的所有 `PROCLOCK`，按照 lock partition 分类存放。

可以理解成：

```text
PGPROC A
   │
   └── myProcLocks[]
          │
          ├── partition 0
          │     ├── PROCLOCK
          │     └── PROCLOCK
          │
          ├── partition 1
          │     └── PROCLOCK
          │
          └── partition N
                └── PROCLOCK
```

因此可以从：

```text
PGPROC
   ↓
myProcLocks
   ↓
PROCLOCK
   ↓
LOCK
```

找到：

> 这个 backend 与哪些锁对象有关。

------

# 21. 子事务 XID

```c
XidCacheStatus subxidStatus;
struct XidCache subxids;
```

PostgreSQL 支持：

```sql
BEGIN;

SAVEPOINT a;
...
SAVEPOINT b;
...
```

这些 savepoint 对应的 subtransaction 会有自己的：

```text
SubTransactionId
```

所以一个顶层事务可能有：

```text
top-level XID
       │
       ├── subxid 1
       ├── subxid 2
       ├── subxid 3
       └── ...
```

`subxids` 就是 backend 对这些 subtransaction XID 的缓存。

而：

```c
subxidStatus
```

表示当前 subxid cache 的状态。

它也被 mirror 到：

```c
ProcGlobal->subxidStates[]
```

------

# 22. ProcArray Group：批量清除 XID

这一组：

```c
bool procArrayGroupMember;
pg_atomic_uint32 procArrayGroupNext;
TransactionId procArrayGroupMemberXid;
```

是 PostgreSQL 的一种**组协作优化**。

为什么需要？

假设多个 backend 同时需要更新/清除某些事务状态：

```text
Backend A ─┐
Backend B ─┤
Backend C ─┼──→ group
Backend D ─┘
```

没必要让每个 backend 都独立完成全部工作。

可以：

```text
leader
  │
  ├── A
  ├── B
  ├── C
  └── D
```

一起处理。

这里的：

```c
procArrayGroupMember
procArrayGroupNext
procArrayGroupMemberXid
```

就是为这种 group operation 服务的。

------

# 23. `wait_event_info`

```c
uint32 wait_event_info;
```

这个字段和你之前看到的：

```sql
pg_stat_activity
```

里面的：

```text
wait_event_type
wait_event
```

非常相关。

例如：

```text
Backend
   │
   └── waiting
          │
          ├── Lock
          ├── LWLock
          ├── IO
          └── Client
```

PostgreSQL 会记录 wait event information。

这个字段就是 backend 当前等待事件信息的重要载体。

------

# 24. CLOG group：批量更新事务状态

下面这一组：

```c
bool clogGroupMember;
pg_atomic_uint32 clogGroupNext;
TransactionId clogGroupMemberXid;
XidStatus clogGroupMemberXidStatus;
int clogGroupMemberPage;
XLogRecPtr clogGroupMemberLsn;
```

是另一个非常重要的性能优化：

> **Group CLOG update**

假设多个 backend 同时要更新事务状态：

```text
A → XID 100 → COMMITTED
B → XID 101 → COMMITTED
C → XID 102 → COMMITTED
```

它们可能进行 group：

```text
        group
       /  |  \
      A   B   C
       \  |  /
        CLOG update
```

这里记录：

- 哪个 backend 是 group member
- 下一个 member
- 对应 XID
- XID 状态
- CLOG page
- WAL LSN

------

# 25. Fast-Path Locks

这一组非常值得单独理解：

```c
LWLock		fpInfoLock;
uint64		fpLockBits;
Oid			fpRelId[FP_LOCK_SLOTS_PER_BACKEND];
bool		fpVXIDLock;
LocalTransactionId fpLocalTransactionId;
```

PostgreSQL 对一些常见的 relation lock 做了优化。

普通 Lock Manager：

```text
Backend
   ↓
Lock Manager
   ↓
Lock hash table
```

如果每一个很简单的 relation lock 都走完整的 Lock Manager，成本比较高。

所以 PostgreSQL 有：

> **Fast-Path Lock**

可以直接在 PGPROC 里面保存。

例如：

```text
PGPROC
 │
 ├── fpRelId[0] = relation A
 ├── fpRelId[1] = relation B
 ├── fpRelId[2] = relation C
 └── ...
```

`fpLockBits` 则记录对应 slot 上的锁模式。

------

# 26. `fpInfoLock`

```c
LWLock fpInfoLock;
```

保护：

```text
fpLockBits
fpRelId[]
fpVXIDLock
fpLocalTransactionId
```

这些 fast-path 状态。

------

# 27. `fpVXIDLock`

```c
bool fpVXIDLock;
LocalTransactionId fpLocalTransactionId;
```

和：

> Virtual Transaction ID（VXID）

有关。

PostgreSQL 中有：

```text
TransactionId
```

也有：

```text
VirtualTransactionId
```

VXID 特别用于一些锁等待场景。

例如某个事务尚未分配真正的 XID，但其他事务需要等待它结束。

这时候可以使用：

```text
VirtualXID lock
```

来等待。

------

# 28. Lock Group

最后：

```c
PGPROC *lockGroupLeader;
dlist_head lockGroupMembers;
dlist_node lockGroupLink;
```

用于：

> Lock Groups

可以把多个 backend 组成一个 lock group。

例如：

```text
             Leader
               │
        ┌──────┼──────┐
        ↓      ↓      ↓
      Proc A Proc B Proc C
```

组内成员可以共享某些锁管理语义。

------

# 29. 把整个 PGPROC 按功能重新分类

你看源码时，不建议按照结构体从上到下死记。

更好的方式是把它分成这几个模块：

```text
PGPROC
│
├── ① 链表 / 等待基础设施
│      ├── links
│      ├── sem
│      ├── procLatch
│      └── waitStatus
│
├── ② 事务 / MVCC
│      ├── xid
│      ├── xmin
│      ├── lxid
│      ├── subxids
│      └── subxidStatus
│
├── ③ Backend 身份
│      ├── pid
│      ├── backendId
│      ├── databaseId
│      ├── roleId
│      └── isBackgroundWorker
│
├── ④ LWLock / Condition Variable 等待
│      ├── lwWaiting
│      ├── lwWaitMode
│      ├── lwWaitLink
│      └── cvWaitLink
│
├── ⑤ Heavyweight Lock
│      ├── waitLock
│      ├── waitProcLock
│      ├── waitLockMode
│      ├── heldLocks
│      └── myProcLocks[]
│
├── ⑥ 同步复制
│      ├── waitLSN
│      ├── syncRepState
│      └── syncRepLinks
│
├── ⑦ Fast-Path Lock
│      ├── fpInfoLock
│      ├── fpLockBits
│      ├── fpRelId[]
│      └── fpVXIDLock
│
├── ⑧ Group 操作优化
│      ├── ProcArray group
│      └── CLOG group
│
├── ⑨ Wait Event
│      └── wait_event_info
│
└── ⑩ Lock Group
       ├── lockGroupLeader
       ├── lockGroupMembers
       └── lockGroupLink
```

------

# 30. `PGPROC` 和 `PGXACT` / `ProcGlobal` 的关系尤其值得注意

你现在看的是 PostgreSQL 新版本源码，所以这里很容易产生一个误解：

> “既然 `PGPROC` 里面已经有 `xid`、`statusFlags`，为什么 `ProcGlobal` 里面还要保存？”

答案就是源码开头说的：

```text
Mirrored fields
```

也就是：

```text
                 ┌──────────────┐
                 │    PGPROC    │
                 │              │
                 │ xid          │
                 │ statusFlags  │
                 │ subxidStatus │
                 └──────┬───────┘
                        │
                     mirror
                        │
                        ↓
              ┌───────────────────┐
              │    ProcGlobal     │
              │                   │
              │ xids[]            │
              │ statusFlags[]     │
              │ subxidStates[]    │
              └───────────────────┘
```

对应关系：

```c
PGPROC->xid
    ↕
ProcGlobal->xids[pgxactoff]

PGPROC->statusFlags
    ↕
ProcGlobal->statusFlags[pgxactoff]

PGPROC->subxidStatus
    ↕
ProcGlobal->subxidStates[pgxactoff]
```

而：

```c
pgxactoff
```

就是连接两边的索引。

------

# 31. 一个具体例子：两个事务互相看到什么

假设：

```text
Backend A
PID = 1000
XID = 100

Backend B
PID = 2000
XID = 101
```

共享内存大概是：

```text
ProcGlobal
│
├── allProcs[]
│      │
│      ├── PGPROC A
│      │      ├── pid = 1000
│      │      ├── xid = 100
│      │      ├── xmin = ...
│      │      └── ...
│      │
│      └── PGPROC B
│             ├── pid = 2000
│             ├── xid = 101
│             ├── xmin = ...
│             └── ...
│
├── xids[]
│      ├── ...
│      ├── 100
│      └── 101
│
└── statusFlags[]
```

因此当 Backend B 做 MVCC 判断时，系统可以通过 ProcArray/ProcGlobal 判断：

```text
XID 100 是否仍然运行？
       │
       ↓
ProcArray
       │
       ↓
PGPROC A
       │
       └── xid = 100
```

这就是为什么 `PGPROC` 是 PostgreSQL **事务系统 + 锁系统 + 进程管理系统的交汇点**。

------

# 32. 最后给你一个最重要的理解

如果你接下来继续阅读 PostgreSQL 源码，可以把 `PGPROC` 记成一句话：

> **PGPROC = “一个 PostgreSQL backend 在共享内存中的运行时控制块”。**

它主要回答 5 类问题：

```text
① 我是谁？
   pid / backendId / databaseId / roleId

② 我现在运行什么事务？
   xid / xmin / lxid / subxids

③ 我现在持有什么锁、等待什么锁？
   myProcLocks / waitLock / waitProcLock / waitLockMode

④ 我现在在等待什么？
   lwWaiting / waitStatus / wait_event_info / procLatch

⑤ 我和其他 backend 怎么协作？
   ProcArray group / CLOG group / SyncRep / LockGroup
```

而你贴出的这个结构体最值得深入跟踪的其实是三条线：

```text
PGPROC
  │
  ├────────→ ProcArray
  │            └── MVCC / xmin / running transactions
  │
  ├────────→ Lock Manager
  │            └── LOCK / PROCLOCK / wait queue
  │
  └────────→ ProcGlobal
               └── xids[] / statusFlags[] / subxidStates[]
```

如果你现在是在**跟 PostgreSQL 源码里的某个 SQL 执行过程**，尤其是想弄清楚“一个 backend 如何通过 `PGPROC` 被其他 backend 看到”，那么下一步最值得看的是 **`PGPROC → PGXACT/ProcGlobal → ProcArray` 这条关系**；这会把 `xid/xmin`、事务可见性、`pg_stat_activity` 以及 VACUUM 为什么能判断“哪些事务还活着”串起来。