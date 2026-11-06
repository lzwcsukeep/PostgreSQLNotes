## procArray 变量介绍

在 PostgreSQL 中，`procArray` 是一个 `ProcArrayStruct` 的全局变量，存放在主共享内存之中。

```c
static ProcArrayStruct *procArray;
```

`ProcArrayStruct` 结构体用于管理参与事务状态跟踪的进程。

可以简化理解为：

```
typedef struct ProcArrayStruct
{
    int numProcs;
    int maxProcs;

    int pgprocnos[FLEXIBLE_ARRAY_MEMBER];

} ProcArrayStruct;
```

其中：

| 字段          | 含义                            |
| ------------- | ------------------------------- |
| `numProcs`    | 当前 procArray 中注册的进程数量 |
| `maxProcs`    | procArray 可容纳的最大进程数量  |
| `pgprocnos[]` | 存储已注册进程的 `PGPROC` 编号  |

例如，假设：

```
procArray->numProcs = 3;

procArray->pgprocnos = [1, 4, 7];
```

表示当前注册了三个 `PGPROC`，编号分别为 1、4、7。

可以通过全局数组 `allProcs` 找到对应的结构体：

```
PGPROC *proc = &allProcs[4];
```

这里的 `allProcs` 是指向全局 `PGPROC` 数组的指针。

`procArray` 是所有活跃 Backend 的紧凑集合, 按照槽位编号大小排列。

## procArray 作用

**作用：记录当前注册的进程。**

每一个新连接上来，都会在固定的PGPROC 数组中选择一个空闲的 PGPROC,作为自己的进程结构记录，后续使用可以是这个 PGPROC 的槽位。

**这个新连接的Backend 对应的 PGPROC 就会注册到 变量 procArray 之中。** **注册使用的是 PGPROC 的槽位。**

procArray 结构体成员 `numProcs` 和 `pgprocnos` 共同确定有效的 PGPROC 数组范围。

```c
procArray->numProcs = 3;
procArray->pgprocnos = [1, 4, 7];
```

## procArray 与 另外三个紧凑数组的关系

在另一个进程管理的变量`ProcGlobal` 中，有三个成员和这个 procArray 关系密切。

```c
PROC_HDR   *ProcGlobal = NULL;

typedef struct PROC_HDR
{
	/* Array of PGPROC structures (not including dummies for prepared txns) */
	// 整个实例运行期间不变的 PGPROC 数组的起始位置。
	// [PGPROC1][PGPROC2][PGPROC3][PGPROC4][PGPROC5]
	PGPROC	   *allProcs;

	/* Array mirroring PGPROC.xid for each PGPROC currently in the procarray */
	TransactionId *xids;
	/*
	 * Array mirroring PGPROC.subxidStatus for each PGPROC currently in the
	 * procarray.
	 */
	XidCacheStatus *subxidStates;
	/*
	 * Array mirroring PGPROC.statusFlags for each PGPROC currently in the
	 * procarray.
	 */
	uint8	   *statusFlags;
} PROC_HDR;
```

这三个成员是 

```c
TransactionId *xids;
XidCacheStatus *subxidStates;
XidCacheStatus *subxidStates;
```

我称之为三个紧凑数组。 这三个紧凑数组的长度跟 `procArray` 保持一致。这三个紧凑数组的数据来源就是 `procArray` 注册的进程。

一句话总结：

PG主共享内存里面预留了固定大小的 PGPROC 数组。这个固定的大小只限制了能开启的最大后端连接数。但是实际有哪些后端进程在运行在

`procArray` 这个变量里面注册。procArray 里的保存当前注册进程的数组成员`pgprocnos` 是按照槽位号有序排列的。

另一个进程管理的全局变量ProcGlobal 的三个成员 xids，subxidStates， statusFlags 大小和 procArray 大小一样。并且这三个成员数据取自注册到 procArray 的进程。

