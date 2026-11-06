快照数据结构定义

```c
typedef struct SnapshotData
{
	SnapshotType snapshot_type; /* type of snapshot */

	TransactionId xmin;			/* all XID < xmin are visible to me */
	TransactionId xmax;			/* all XID >= xmax are invisible to me */

	TransactionId *xip;
	uint32		xcnt;			/* # of xact ids in xip[] */

} SnapshotData;
```

获取快照的函数

```c
/*
 * GetSnapshotData -- returns information about running transactions.
 *
 * The returned snapshot includes xmin (lowest still-running xact ID),
 * xmax (highest completed xact ID + 1), and a list of running xact IDs
 * in the range xmin <= xid < xmax.  It is used as follows:
 *		All xact IDs < xmin are considered finished.
 *		All xact IDs >= xmax are considered still running.
 *		For an xact ID xmin <= xid < xmax, consult list to see whether
 *		it is considered running or not.
 * This ensures that the set of transactions seen as "running" by the
 * current xact will not change after it takes the snapshot.
 *
 * All running top-level XIDs are included in the snapshot, except for lazy
 * VACUUM processes.  We also try to include running subtransaction XIDs,
 * but since PGPROC has only a limited cache area for subxact XIDs, full
 * information may not be available.  If we find any overflowed subxid arrays,
 * we have to mark the snapshot's subxid data as overflowed, and extra work
 * *may* need to be done to determine what's running (see XidInMVCCSnapshot()).
 *
 */
Snapshot
GetSnapshotData(Snapshot snapshot){}
```

## 首先理解 MVCC 快照的三个核心概念

源码开头的注释已经说明了快照的核心结构：

```
/*
 * The returned snapshot includes xmin
 * (lowest still-running xact ID),
 * xmax (highest completed xact ID + 1),
 * and a list of running xact IDs
 * in the range xmin <= xid < xmax.
 */
```

主要是：

- `xmin`
- `xmax`
- `xip[]`

### 1. xmin：快照中的最小事务 ID 边界

`xmin` 通常代表快照中最早仍然活跃的事务 ID。

例如：

```
当前正在运行的事务：

事务 100
事务 103
事务 107
```

那么：

```
snapshot->xmin = 100;
```

对于比 `xmin` 更早的普通事务 ID，快照认为它们已经结束。

注意，已经结束不代表一定提交成功，也可能是回滚。具体提交状态还需要结合事务提交状态信息判断。

### 2. xmax：快照中的最大事务 ID 边界

源码：

```
latest_completed = ShmemVariableCache->latestCompletedXid;

xmax = XidFromFullTransactionId(latest_completed);

TransactionIdAdvance(xmax);
```

也就是说：

```
xmax = latestCompletedXid + 1
```

这里的 `latestCompletedXid` 是 PostgreSQL 维护的最新完成事务 ID 高水位信息，不能简单理解成当前最大的已提交事务 ID。

例如：

```
latestCompletedXid = 110

那么：

snapshot->xmax = 111
```

对于：

```
xid >= 111
```

的事务，当前快照一律认为它们仍在运行，即它们的修改对这个快照不可见。

这样即使后续有新的事务开始并提交，也不会改变这个快照的可见性边界。

### 3. xip[]：快照中正在运行的事务列表

例如：

```
snapshot->xmin = 100;
snapshot->xmax = 111;

snapshot->xip = {100, 103, 107};
snapshot->xcnt = 3;
```

表示在获取快照时：

```
100：正在运行
103：正在运行
107：正在运行
```

而：

```
101
102
104
105
106
108
109
110
```

这些不在 `xip[]` 中的事务，对于该快照而言不再属于正在运行的事务集合。

注意，这里先不考虑子事务、事务回绕以及当前事务自身的特殊处理。

## 快照 xmin,xmax，xip[] 数据来源

xmax 来自 PG 维护的最新的已完成的事务ID /* newest full XID that has committed or aborted */

```c
	latest_completed = ShmemVariableCache->latestCompletedXid;
	xmax = XidFromFullTransactionId(latest_completed);
	TransactionIdAdvance(xmax);   // 加一
	Assert(TransactionIdIsNormal(xmax));
```

xmin 和 xip[] , xcnt 数据来源于：

**ProcGlobal->xids , ProcGlobal->subxidStates , ProcGlobal->statusFlags 和 procArray 。**

xmin 会遍历 xids ，然后排除无效的xid,最终 xmin 等于 xids 里面有效的里面最小的。

遍历的过程中，会把有效的，正在运行的 xid 记录到 xip[] 中去，更新 xcnt。

```c
		TransactionId *other_xids = ProcGlobal->xids;
		xmin = xmax;
		for (int pgxactoff = 0; pgxactoff < numProcs; pgxactoff++)
		{
            TransactionId xid = UINT32_ACCESS_ONCE(other_xids[pgxactoff]);
            
            /*
			 * If the transaction has no XID assigned, we can skip it; it
			 * won't have sub-XIDs either.
			 */
            if (likely(xid == InvalidTransactionId))
				continue;

			Assert(TransactionIdIsNormal(xid));

			/*
			 * If the XID is >= xmax, we can skip it; such transactions will
			 * be treated as running anyway (and any sub-XIDs will also be >=
			 * xmax).
			 */
			if (!NormalTransactionIdPrecedes(xid, xmax))
				continue;

			/*
			 * Skip over backends doing logical decoding which manages xmin
			 * separately (check below) and ones running LAZY VACUUM.
			 */
			statusFlags = allStatusFlags[pgxactoff];
			if (statusFlags & (PROC_IN_LOGICAL_DECODING | PROC_IN_VACUUM))
				continue;

			if (NormalTransactionIdPrecedes(xid, xmin))
				xmin = xid;

			/* Add XID to snapshot. */
			xip[count++] = xid;
        }
		
snapshot->xmin = xmin;
snapshot->xmax = xmax;
snapshot->xcnt = count;

```



重要总结： xmin, xip[] 的数据来源就是 **ProcGlobal->xids 和 procArray** ， xids 里面的xid 并不完全是有效的,需要排除。排除之后，xmin 是里面之中最小的。

其余有效的记录在 xip[] 里面，个数使用 xcnt 记录。

xmax 是PG 单独维护的最近已完成的最新XID+1 。



重要结论： 快照数据来源就是来源于 公共共享内存里面的数据结构！！！