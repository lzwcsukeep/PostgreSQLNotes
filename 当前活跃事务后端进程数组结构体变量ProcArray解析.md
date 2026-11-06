文件 `src/backend/storage/ipc/procarray.c` 中全局变量 `static ProcArrayStruct *procArray;`  解析

# PostgreSQL `procArray` 全局变量

## 1. 定义

PostgreSQL 中有一个全局变量：

```c
static ProcArrayStruct *procArray;
```

对应的结构体定义：

```c
typedef struct ProcArrayStruct
{
    int         numProcs;
    int         maxProcs;

    int         maxKnownAssignedXids;
    int         numKnownAssignedXids;
    int         tailKnownAssignedXids;
    int         headKnownAssignedXids;
    slock_t     known_assigned_xids_lck;

    TransactionId lastOverflowedXid;

    TransactionId replication_slot_xmin;
    TransactionId replication_slot_catalog_xmin;

    int         pgprocnos[FLEXIBLE_ARRAY_MEMBER];
} ProcArrayStruct;
```

`procArray` 是 PostgreSQL **共享内存中的 ProcArray 数据结构**，用于维护当前加入 **ProcArray** 的 `PGPROC`，同时保存与事务快照、Hot Standby、复制槽等相关的全局信息。

可以简单记成：

> **`procArray` 是 PostgreSQL 用来管理“当前参与 ProcArray 的进程/事务”的共享数据结构。**

------

# 2. `procArray` 和 `PGPROC` 的关系

首先要区分：

```text
PGPROC
```

和：

```text
procArray
```

### `PGPROC`

`PGPROC` 描述**一个 PostgreSQL backend/process 的状态**：

```text
PGPROC
├── pid
├── xid
├── xmin
├── lxid
├── pgprocno
├── pgxactoff
├── procLatch
├── locks
├── subxids
└── ...
```

可以理解为：

> **一个进程对应一个 PGPROC。**

------

### `procArray`

`procArray` 则负责维护：

> **当前属于 ProcArray 的这些 PGPROC。**

它本身并不直接保存完整的 `PGPROC`，而是通过：

```c
int pgprocnos[FLEXIBLE_ARRAY_MEMBER];
```

保存 `PGPROC` 的编号。

例如：

```text
ProcGlobal->allProcs

pgprocno
   0       1       2       3       4
   │       │       │       │       │
   ↓       ↓       ↓       ↓       ↓
 PGPROC  PGPROC  PGPROC  PGPROC  PGPROC
   A       B       C       D       E
```

假设当前 ProcArray 中只有：

```text
A、C、E
```

那么：

```text
procArray->pgprocnos

[0] = 0
[1] = 2
[2] = 4
```

因此可以理解成：

```text
procArray
    │
    └── pgprocnos[]
            │
            ├── 0 ──→ allProcs[0] → PGPROC A
            ├── 2 ──→ allProcs[2] → PGPROC C
            └── 4 ──→ allProcs[4] → PGPROC E
```

------

# 3. `procArray` 并不是 `allProcs`

这是理解 PostgreSQL ProcArray 机制最重要的一点。

`ProcGlobal` 中：

```c
PGPROC *allProcs;
```

表示：

> **所有可分配的 PGPROC。**

而：

```c
procArray->pgprocnos[]
```

表示：

> **当前已经加入 ProcArray 的 PGPROC。**

所以：

```text
allProcs
    ↓
所有 PGPROC
    │
    ├── 当前使用
    ├── 当前使用
    ├── 空闲
    ├── 当前使用
    └── 空闲
```

而：

```text
procArray
    ↓
当前 ProcArray 成员
    │
    ├── PGPROC A
    ├── PGPROC C
    └── PGPROC D
```

因此：

> **`allProcs` 是 PGPROC 的存储池；`procArray` 是当前需要参与 ProcArray 管理的 PGPROC 的索引集合。**

------

# 4. `pgprocnos[]` 是 `procArray` 最核心的成员

定义：

```c
int pgprocnos[FLEXIBLE_ARRAY_MEMBER];
```

注释：

```c
/* indexes into allProcs[], has PROCARRAY_MAXPROCS entries */
```

意思是：

> `pgprocnos[]` 中存放的是 `allProcs[]` 的下标。

例如：

```text
allProcs

[0] → PGPROC A
[1] → PGPROC B
[2] → PGPROC C
[3] → PGPROC D
[4] → PGPROC E
```

如果：

```text
procArray->pgprocnos = [0, 2, 4]
```

那么 ProcArray 中的成员就是：

```text
allProcs[0] → A
allProcs[2] → C
allProcs[4] → E
```

------

# 5. `numProcs`

```c
int numProcs;
```

表示：

> **当前 `pgprocnos[]` 中有多少个有效的 PGPROC。**

例如：

```text
pgprocnos：

[0] = 2
[1] = 5
[2] = 8
[3] = ...
```

如果：

```c
numProcs = 3;
```

那么只有：

```text
pgprocnos[0]
pgprocnos[1]
pgprocnos[2]
```

有效。

------

# 6. `maxProcs`

```c
int maxProcs;
```

表示：

> `pgprocnos[]` 分配的最大容量。

例如：

```text
maxProcs = 100
numProcs = 20
```

表示：

```text
已经使用：20
最大容量：100
```

所以：

```text
numProcs <= maxProcs
```

------

# 7. `ProcArrayAdd()` 和 `procArray`

你刚才分析的：

```c
void
ProcArrayAdd(PGPROC *proc)
```

就是修改这个全局对象。

一开始：

```text
procArray

numProcs = 3

pgprocnos:
[0] = 2
[1] = 5
[2] = 8
```

调用：

```c
ProcArrayAdd(proc);
```

假设：

```c
proc->pgprocno = 6;
```

因为 ProcArray 按 `pgprocno` 排序：

```text
原来：

[2, 5, 8]

加入 6：

[2, 5, 6, 8]
```

于是：

```text
numProcs = 4
```

------

# 8. 为什么 `pgprocnos[]` 要排序？

源码：

```c
/*
 * Keep the procs array sorted by (PGPROC *) so that we can utilize
 * locality of references much better.
 */
```

从代码实际比较的是：

```c
arrayP->pgprocnos[index] > proc->pgprocno
```

所以它实际上保持：

```text
pgprocno
```

的升序。

例如：

```text
[1, 3, 5, 8, 12]
```

而不是：

```text
[8, 1, 12, 3, 5]
```

这样遍历 ProcArray 时，访问：

```c
allProcs[pgprocnos[i]]
```

具有更好的 cache locality。

------

# 9. `procArray` 与 `ProcGlobal` 的关系

这两个结构非常容易混淆。

PostgreSQL 有：

```c
PROC_HDR *ProcGlobal;
```

以及：

```c
static ProcArrayStruct *procArray;
```

可以理解成：

```text
共享内存
│
├── ProcGlobal
│
│   ├── allProcs[]
│   ├── xids[]
│   ├── subxidStates[]
│   ├── statusFlags[]
│   └── ...
│
└── procArray
    │
    ├── numProcs
    ├── maxProcs
    ├── known assigned XIDs
    ├── replication slot xmin
    └── pgprocnos[]
```

其中：

```text
ProcGlobal
    ↓
管理 PGPROC 及其 dense arrays

procArray
    ↓
管理当前 ProcArray 成员及相关事务状态
```

------

# 10. `procArray` 和 `ProcGlobal->xids[]` 的配合

这是你前面代码中非常重要的一组关系。

假设：

```text
procArray->pgprocnos

[0] = 2
[1] = 5
[2] = 8
```

那么：

```text
ProcGlobal->xids

[0] = allProcs[2].xid
[1] = allProcs[5].xid
[2] = allProcs[8].xid
```

所以：

```text
ProcArray index
       │
       ├───────────────┐
       ↓               ↓
pgprocnos[i]        xids[i]
       │               │
       ↓               ↓
allProcs[...]       PGPROC.xid
```

例如：

```text
i = 1

procArray->pgprocnos[1] = 5

ProcGlobal->xids[1] = allProcs[5].xid
```

而：

```c
allProcs[5].pgxactoff = 1;
```

所以：

```text
                    i = 1
                      │
          ┌───────────┴───────────┐
          ↓                       ↓
procArray->pgprocnos[1]     ProcGlobal->xids[1]
          │                       │
          ↓                       ↓
      pgprocno = 5             xid = ...
          │
          ↓
     allProcs[5]
          │
          └── pgxactoff = 1
```

这就是 `pgxactoff` 存在的意义。

------

# 11. `known_assigned_xids`

`procArray` 里面还有一组和 **Hot Standby / WAL replay** 有关的字段：

```c
int maxKnownAssignedXids;
int numKnownAssignedXids;
int tailKnownAssignedXids;
int headKnownAssignedXids;
slock_t known_assigned_xids_lck;
```

它们维护：

> **Known Assigned XIDs**

也就是在 standby/recovery 场景下，用于记录已经知道被分配的事务 ID。

可以先把它理解成一个环形数组：

```text
known_assigned_xids
        ↓

      tail
       ↓
+---+---+---+---+---+---+
| X | X | X |   |   | X |
+---+---+---+---+---+---+
                   ↑
                  head
```

相关字段：

```text
maxKnownAssignedXids
    ↓
数组容量

numKnownAssignedXids
    ↓
当前有效 XID 数量

tailKnownAssignedXids
    ↓
最老元素位置

headKnownAssignedXids
    ↓
下一个写入位置

known_assigned_xids_lck
    ↓
保护 head/tail
```

这部分和普通 backend ProcArray 成员管理是不同的一套功能。

------

# 12. `lastOverflowedXid`

```c
TransactionId lastOverflowedXid;
```

表示：

> KnownAssignedXids 数组发生溢出时，记录相关的最高 XID。

这个机制主要也是为了处理 Hot Standby 下事务 ID 信息过多、无法完整缓存的问题。

你现在如果主要是在理解 `ProcArrayAdd()`，可以暂时把这一字段放到第二阶段学习。

------

# 13. `replication_slot_xmin`

```c
TransactionId replication_slot_xmin;
```

表示：

> 所有 replication slot 中最老的 `xmin`。

它和 PostgreSQL 的复制槽、vacuum、数据保留有关。

例如：

```text
Replication Slot A
    xmin = 100

Replication Slot B
    xmin = 120

Replication Slot C
    xmin = 105
```

那么：

```text
replication_slot_xmin = 100
```

因为：

```text
100 是最老的 xmin
```

它可以防止 vacuum 过早清理复制槽仍然需要的数据。

------

# 14. `replication_slot_catalog_xmin`

```c
TransactionId replication_slot_catalog_xmin;
```

和上面的类似，但针对：

> replication slot 对 catalog tuple 的 xmin 保留需求。

可以简单记：

```text
replication_slot_xmin
    ↓
普通数据的 xmin 保留

replication_slot_catalog_xmin
    ↓
系统 catalog 的 xmin 保留
```

------

# 15. `FLEXIBLE_ARRAY_MEMBER` 是什么？

最后这个：

```c
int pgprocnos[FLEXIBLE_ARRAY_MEMBER];
```

不是一个真正固定长度的数组。

它是 C 的：

> **Flexible Array Member（柔性数组成员）**

也就是说结构体本身的尾部预留了一段空间：

```text
ProcArrayStruct
┌──────────────────────────────┐
│ numProcs                     │
│ maxProcs                     │
│ ...                          │
│ replication_slot_catalog_xmin│
├──────────────────────────────┤
│ pgprocnos[0]                 │
│ pgprocnos[1]                 │
│ pgprocnos[2]                 │
│ ...                          │
│ pgprocnos[maxProcs - 1]      │
└──────────────────────────────┘
```

因此实际分配共享内存时，会分配：

```text
sizeof(ProcArrayStruct)
+
maxProcs * sizeof(int)
```

左右的空间。

所以虽然源码写的是：

```c
pgprocnos[FLEXIBLE_ARRAY_MEMBER]
```

实际上运行时它后面可以有很多个元素。

------

# 16. `procArray` 最核心的理解

如果你的 Markdown 笔记只保留几个关键点，我建议写成：

~~~markdown
## procArray

```c
static ProcArrayStruct *procArray;
~~~

`procArray` 指向 PostgreSQL 共享内存中的 `ProcArrayStruct`。

它用于维护当前加入 ProcArray 的 PGPROC，并保存与事务可见性、
Hot Standby、复制槽等相关的信息。

核心成员：

- `numProcs`：当前 ProcArray 中有效 PGPROC 数量
- `maxProcs`：`pgprocnos[]` 的容量
- `pgprocnos[]`：当前 ProcArray 成员对应的 `pgprocno`
- `known_assigned_xids` 相关字段：Hot Standby 下的 Known Assigned XIDs
- `replication_slot_xmin`：所有 replication slot 中最老的 xmin
- `replication_slot_catalog_xmin`：所有 replication slot 中最老的 catalog xmin

### 与 allProcs 的关系

`ProcGlobal->allProcs[]` 保存所有 PGPROC。

`procArray->pgprocnos[]` 保存当前 ProcArray 成员在
`allProcs[]` 中的索引。

例如：

```
allProcs:
[0] PGPROC A
[1] PGPROC B
[2] PGPROC C
[3] PGPROC D

procArray->pgprocnos:
[0] = 0
[1] = 2
[2] = 3
```

则当前 ProcArray 成员为：

```
allProcs[0] → A
allProcs[2] → C
allProcs[3] → D
```

### 与 ProcGlobal->xids[] 的关系

ProcArray 中的每个成员在 `ProcGlobal->xids[]` 中都有一个
对应的 xid 镜像。

相同的 index 对应同一个 PGPROC：

```
procArray->pgprocnos[i]
          ↓
      allProcs[pgprocno]
          ↓
      PGPROC

ProcGlobal->xids[i]
          ↓
    PGPROC.xid
```

PGPROC->pgxactoff 保存该 PGPROC 当前在这些 dense arrays
中的 index。

### 主要操作

`ProcArrayAdd(PGPROC *proc)`
→ 将 PGPROC 加入 ProcArray

`ProcArrayRemove(PGPROC *proc)`
→ 将 PGPROC 从 ProcArray 移除

`GetSnapshotData()`
→ 遍历 ProcArray，获取当前运行事务的信息并构造 Snapshot

因此可以把 ProcArray 理解为：

```
“当前事务系统需要关注的 PGPROC 集合”
---

### 最核心的一句话

> **`procArray` 不是“所有 PGPROC 的数组”，而是一个共享的数据结构，它通过 `pgprocnos[]` 索引 `ProcGlobal->allProcs[]`，维护当前加入 ProcArray 的 PGPROC 集合；同时为事务快照、事务可见性以及 Hot Standby 等功能提供全局事务状态信息。**

这句话基本可以作为你这部分 PostgreSQL 源码学习的核心定义。
```