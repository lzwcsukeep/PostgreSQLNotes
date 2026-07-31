# 1. t_cid：Command ID

`t_cid` 是：

```c
CommandId
```

本质：

```c
typedef uint32 CommandId;
```

占 4 字节。

表示：

> 在同一个事务内部，第几个 SQL 命令创建或删除了这条 tuple。

------

例如：

```sql
BEGIN;

INSERT INTO t VALUES (1);      -- cid=0

INSERT INTO t VALUES (2);      -- cid=1

DELETE FROM t WHERE id=1;      -- cid=2

COMMIT;
```

事务号：

```text
xmin = 938
```

不变。

但 CommandId：

```text
0
1
2
```

不断增加。

------

# 为什么需要 t_cid？

因为：

一个事务内部可能执行很多 SQL。

例如：

```sql
BEGIN;

INSERT INTO t VALUES (1);

SELECT * FROM t;

COMMIT;
```

同一个事务：

```text
xmin = 当前事务
```

如果没有 `cid`，

系统无法知道：

```text
SELECT 能不能看到刚插入的记录？
```

------

PostgreSQL 通过：

```text
xmin + cmin
```

判断：

```text
当前 command 是否早于 tuple 创建 command
```

从而实现：

### 同一事务内的可见性。

------

# 实际上 header 中没有真正的 t_cid

源码：

```c
union
{
    HeapTupleFields t_heap;
    DatumTupleFields t_datum;
} t_choice;
```

其中：

```c
typedef struct HeapTupleFields
{
    TransactionId t_xmin;
    TransactionId t_xmax;

    union
    {
        CommandId t_cid;
        TransactionId t_xvac;
    } t_field3;
} HeapTupleFields;
```

因此：

### t_cid 和 t_xvac 共用空间。

------

# cmin 和 cmax

实际上：

创建 command：

```text
cmin
```

删除 command：

```text
cmax
```

都不一定存放在 tuple 中。

因为只有 4 字节空间。

所以：

当需要同时保存：

```text
cmin
cmax
```

时：

会用

```text
combocid
```

再查：

```text
ComboCidMap
```

得到：

```text
cmin
cmax
```

------

# 你的输出：

```text
t_cid = 0
```

表示：

### t_cid = 0

这条记录是在事务 938 的：

```text
第 0 条 SQL 命令
```

产生的。

例如：

```sql
BEGIN;

INSERT INTO actor ...;
```

这是事务中的第一条命令。