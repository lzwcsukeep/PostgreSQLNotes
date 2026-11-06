# PostgreSQL Visibility Map（VM，可见性映射）详解

在 PostgreSQL 中，Visibility Map（简称 VM，可见性映射）是一个用于记录 Heap Page 中元组可见性和冻结状态的辅助数据结构。

前面我们介绍了 FSM（Free Space Map），它负责记录 Heap Page 的空闲空间，而 VM 负责记录 Heap Page 的可见性状态。

两者都是 PostgreSQL Relation 的独立 Fork，但用途完全不同。

## 一、为什么 PostgreSQL 需要 Visibility Map？

首先回顾 PostgreSQL 的 MVCC 机制。

假设有一张表：

```
CREATE TABLE test (
    id INT,
    name TEXT
);

INSERT INTO test VALUES (1, 'Alice');
INSERT INTO test VALUES (2, 'Bob');
```

PostgreSQL 的 Heap Page 中保存着两个元组：

```
Heap Page 0
+--------------------------------+
| PageHeaderData                 |
+--------------------------------+
| ItemId[0] -> Tuple A           |
| ItemId[1] -> Tuple B           |
+--------------------------------+
| Free Space                     |
+--------------------------------+
| Tuple B                        |
|   xmin = 101                   |
|   xmax = 0                     |
+--------------------------------+
| Tuple A                        |
|   xmin = 100                   |
|   xmax = 0                     |
+--------------------------------+
```

当 PostgreSQL 执行查询时：

```
SELECT * FROM test;
```

需要根据 MVCC 规则判断哪些元组对当前事务可见。

例如，判断元组的：

- `xmin`：创建元组的事务 ID。
- `xmax`：删除或更新元组的事务 ID。
- 事务提交状态。
- 当前事务的 Snapshot。

假设一张表包含 100 万个 Heap Page，其中绝大多数 Page 中的元组都已经由很久以前的事务提交，并且没有任何事务会再修改它们。

如果 PostgreSQL 每次都需要逐个检查元组的 MVCC 信息，就可能产生额外开销。

因此，PostgreSQL 引入了 Visibility Map。

它可以记录：

某个 Heap Page 中的所有元组，是否已经对所有事务可见。

例如：

```
Heap Relation

Page 0    -> All Visible
Page 1    -> All Visible
Page 2    -> Not All Visible
Page 3    -> All Visible
Page 4    -> Not All Visible
...
```

这样，某些数据库操作就可以直接利用 VM 中的信息，而不需要重新检查整个 Heap Page 中每个元组的可见性。

不过要注意：普通顺序扫描通常仍然需要读取 Heap Page 来获取元组数据，VM 并不意味着普通 `SELECT *` 可以跳过读取 Heap Page。

## 二、Visibility Map Fork 在磁盘上如何存储？

与 FSM 类似，VM 也是 Relation 的一个独立 Fork。

例如：

```
$PGDATA/base/16384/

24576         <-- Main Fork
24576_fsm     <-- Free Space Map
24576_vm      <-- Visibility Map
```

这三个文件分别负责：

| Fork      | 作用                              |
| --------- | --------------------------------- |
| Main Fork | 保存 Heap Page 和元组数据         |
| FSM Fork  | 保存 Heap Page 的空闲空间信息     |
| VM Fork   | 保存 Heap Page 的可见性与冻结状态 |

VM 不保存任何 HeapTuple，也不保存 `xmin`、`xmax` 的具体数值。

它只使用少量 bit 来描述 Heap Page 的整体状态。

### VM 的两个状态位

从 PostgreSQL 9.6 开始，每个 Heap Page 对应 VM 中的两个 bit。

分别是：

1. All-Visible bit：该 Page 中所有元组都对所有事务可见。
2. All-Frozen bit：该 Page 中所有元组都已经冻结，或者不再需要未来的事务 ID 冻结处理。

可以理解为：

```
Heap Page 0 -> [All-Visible = 1, All-Frozen = 1]
Heap Page 1 -> [All-Visible = 1, All-Frozen = 0]
Heap Page 2 -> [All-Visible = 0, All-Frozen = 0]
```

正常情况下，All-Frozen 意味着 All-Visible。

因此：

| All-Visible | All-Frozen | 含义                               |
| ----------- | ---------- | ---------------------------------- |
| 0           | 0          | 不能保证整个 Page 对所有事务可见   |
| 1           | 0          | 所有元组可见，但还可能需要冻结处理 |
| 1           | 1          | 所有元组可见，且已满足冻结条件     |
| 0           | 1          | 非正常组合                         |

需要特别强调：

All-Visible 并不等于 All-Frozen。

一个 Page 中所有元组都可见，并不意味着这些元组的事务 ID 已经被冻结。

## 三、VM 与 Heap Page 的对应关系

VM 采用 Bitmap（位图）结构。

每个 Heap Page 使用两个 bit，因此一个字节可以记录四个 Heap Page 的状态。

例如：

```
VM Byte 0

bit 7 6 | bit 5 4 | bit 3 2 | bit 1 0
--------+---------+---------+--------
Page 3  | Page 2  | Page 1  | Page 0
```

假设：

```
Page 0: All-Visible=1, All-Frozen=1
Page 1: All-Visible=1, All-Frozen=0
Page 2: All-Visible=0, All-Frozen=0
Page 3: All-Visible=1, All-Frozen=1
```

对应的位图为：

```
Page 3   Page 2   Page 1   Page 0
  11       00       01       11

二进制：11000111
```

注意，这里的每组 bit 低位是 All-Visible，高位是 All-Frozen。

### 一个 VM Page 可以管理多少个 Heap Page？

PostgreSQL 默认 Page 大小为 8192 字节。

VM Page 需要保留 PageHeaderData：

```
Page Size = 8192 bytes
PageHeaderData = 24 bytes
```

因此，可用于 Bitmap 的空间为：

```
8192 - 24 = 8168 bytes
```

每个字节管理四个 Heap Page：

\[ 8168 \times 4 = 32672 \]

所以：

一个 8KB 的 VM Page 可以记录 32672 个 Heap Page 的状态。

而这些 Heap Page 对应的主数据文件大小约为：

\[ 32672 \times 8192 = 255.25\ \text{MiB} \]

因此，VM 的空间开销非常小。

## 四、VACUUM 如何更新 Visibility Map？

这部分与前面讨论的普通 VACUUM 直接相关。

假设 Heap Page 100 中存在：

```
Page 100

Tuple A：Live
Tuple B：Dead
Tuple C：Live
```

此时：

```
VM[100]:

All-Visible = 0
All-Frozen  = 0
```

因为存在死亡元组，该 Page 不能被标记为 All-Visible。

执行：

```
VACUUM test;
```

VACUUM 会扫描 Page，判断元组状态，并清理可以安全回收的死亡元组。

清理完成后：

```
Page 100

Tuple A：Live
Tuple C：Live
```

如果 VACUUM 确认该 Page 中剩余的元组对所有事务可见，那么可以设置：

```
VM[100]:

All-Visible = 1
All-Frozen  = 0
```

如果该 Page 中的元组同时满足冻结条件，则还可以设置：

```
VM[100]:

All-Visible = 1
All-Frozen  = 1
```

这里要注意，VACUUM 清理完死亡元组，不代表一定能够立即设置 All-Visible。

例如，Page 中仍然存在尚未提交事务插入的元组，那么该 Page 就不能被标记为 All-Visible。

### VACUUM 后为什么可能跳过某些 Page？

普通 VACUUM 在扫描 Heap 时，可以利用 VM 判断哪些 Page 已经是 All-Visible。

对于已经 All-Visible 的 Page，如果没有冻结处理需求，就可能跳过扫描。

这可以大幅减少 VACUUM 的 I/O 开销。

例如：

```
Heap Relation：100000 Pages

Page 0       All-Visible
Page 1       All-Visible
Page 2       All-Visible
...
Page 99998   Not All-Visible
Page 99999   All-Visible
```

如果大多数 Page 都已经 All-Visible，后续普通 VACUUM 就不需要每次完整扫描所有 Heap Page。

不过，冻结维护、激进 VACUUM 等情况可能仍然需要访问部分已经 All-Visible 的 Page。

## 五、UPDATE、DELETE 如何影响 VM？

假设 Page 100 当前状态为：

```
VM[100]:

All-Visible = 1
All-Frozen  = 1
```

现在执行：

```
UPDATE test
SET name = 'David'
WHERE id = 1;
```

如果 UPDATE 修改了 Page 100 中的元组，那么原有的 All-Visible 状态就可能失效。

PostgreSQL 必须清除相应 VM 状态位。

例如：

```
UPDATE 前：

All-Visible = 1
All-Frozen  = 1


UPDATE 后：

All-Visible = 0
All-Frozen  = 0
```

因为 UPDATE 产生了新的元组版本，新版本不一定对所有事务可见。

同样，DELETE 也会使受影响 Page 的 All-Visible 状态失效。

对于 INSERT，是否清除 All-Visible 位还涉及插入时的具体可见性规则；普通插入通常会使目标 Page 不再满足 All-Visible 条件。

VM 的状态维护与 Heap Page 的修改需要协调，避免出现 VM 错误地声称某个 Page 已经 All-Visible 的情况。

## 六、VM 最重要的用途：Index-Only Scan

VM 的一个非常重要的用途，是支持 PostgreSQL 的 Index-Only Scan（仅索引扫描）。

先看普通 Index Scan。

假设：

```
CREATE TABLE users (
    id INT,
    name TEXT
);

CREATE INDEX idx_users_id ON users(id);
```

执行：

```
SELECT id FROM users WHERE id = 100;
```

普通 Index Scan：

```
B-Tree Index
    |
    v
找到 id = 100
    |
    v
获取 TID
    |
    v
访问 Heap Page
    |
    v
检查元组 MVCC 可见性
    |
    v
返回结果
```

即使索引中已经包含 `id`，PostgreSQL 仍可能需要访问 Heap Page 来确认元组对当前事务是否可见。

原因是 B-Tree 索引条目本身不包含完整的 MVCC 可见性信息。

### Index-Only Scan 如何利用 VM？

当查询所需列都能从索引中获取时，PostgreSQL 可以选择 Index-Only Scan。

执行过程：

```
B-Tree Index
    |
    v
找到 id = 100
    |
    v
获取 TID
    |
    v
检查对应 Heap Page 的 VM
    |
    +-- All-Visible = 1
    |       |
    |       v
    |   无需访问 Heap
    |       |
    |       v
    |   直接返回索引中的数据
    |
    +-- All-Visible = 0
            |
            v
        访问 Heap
            |
            v
        检查 MVCC 可见性
            |
            v
        返回可见元组
```

因此：

VM 的 All-Visible bit 是 Index-Only Scan 能够避免访问 Heap Page 的关键。

注意，Index-Only Scan 并不意味着完全不访问 Heap。

如果目标 Page 没有被标记为 All-Visible，仍然需要进行 Heap Fetch。

### 实际验证

```
CREATE TABLE vm_test (
    id INT,
    name TEXT
);

INSERT INTO vm_test
SELECT i, 'user_' || i
FROM generate_series(1, 100000) i;

CREATE INDEX idx_vm_test_id ON vm_test(id);

VACUUM vm_test;
```

执行：

```
EXPLAIN (ANALYZE, BUFFERS)
SELECT id
FROM vm_test
WHERE id BETWEEN 100 AND 200;
```

可能得到：

```
Index Only Scan using idx_vm_test_id on vm_test
  Index Cond: ((id >= 100) AND (id <= 200))
  Heap Fetches: 0
```

这里：

```
Heap Fetches: 0
```

表示本次 Index-Only Scan 不需要回表检查元组可见性。

当然，具体执行计划由优化器决定，不保证一定选择 Index-Only Scan。

## 七、VM 的另一个重要用途：减少事务冻结扫描

PostgreSQL 的事务 ID（XID）是 32 位的，会发生回绕（Wraparound）。

为了避免旧元组的事务 ID 在回绕后产生可见性问题，PostgreSQL 需要进行事务冻结（Freeze）。

例如：

```
Tuple A:
    xmin = 100

Tuple B:
    xmin = 200
```

当事务 ID 足够旧时，VACUUM 可能需要对元组执行冻结处理。

如果一个 Page 已经被标记为 All-Frozen：

```
VM[100]:

All-Visible = 1
All-Frozen  = 1
```

那么 PostgreSQL 就知道该 Page 中的元组不再需要进一步的 XID 冻结处理。

因此，在需要进行冻结维护的 VACUUM 中，可以跳过这些 All-Frozen Page。

这可以显著减少大表的冻结维护开销。

需要注意：All-Frozen 表示不再需要相应的冻结处理，并不一定意味着所有元组头部的 `xmin` 字段都被物理改写成同一个特殊值。

## 八、PostgreSQL 16 中 VM 的相关源码

可以重点阅读：

```
src/backend/access/heap/visibilitymap.c
src/include/access/visibilitymap.h
```

核心函数包括：

```
visibilitymap_get_status(...)
```

用于读取某个 Heap Page 对应的 VM 状态。

```
visibilitymap_set(...)
```

用于设置 VM 中的 All-Visible 或 All-Frozen 位。

```
visibilitymap_clear(...)
```

用于清除 VM 状态位。

在 VACUUM 实现中，还可以结合：

```
src/backend/access/heap/vacuumlazy.c
```

理解 VACUUM 如何根据 Page 中的元组状态设置 VM，以及如何利用 VM 跳过不需要扫描的 Page。

## 九、如何通过 SQL 查看 Visibility Map？

PostgreSQL 提供了扩展：

```
CREATE EXTENSION pg_visibility;
```

查看指定 Heap Page 的 VM 状态：

```
SELECT *
FROM pg_visibility_map('vm_test', 0);
```

返回结果类似：

```
 blkno | all_visible | all_frozen
-------+-------------+-----------
     0 | t           | f
```

其中：

- `blkno`：Heap Block Number。
- `all_visible`：VM 中的 All-Visible 状态。
- `all_frozen`：VM 中的 All-Frozen 状态。

查看整张表的 VM 状态：

```
SELECT *
FROM pg_visibility_map('vm_test');
```

也可以统计整个表的状态：

```
SELECT *
FROM pg_visibility_map_summary('vm_test');
```

例如：

```
 all_visible | all_frozen
-------------+------------
         500 |        100
```

表示有 500 个 Heap Page 被标记为 All-Visible，其中 100 个被标记为 All-Frozen。

## 十、FSM 和 VM 的区别

最后，将前面介绍的 FSM 与 VM 放在一起比较。

| 对比项                | FSM                    | VM                      |
| --------------------- | ---------------------- | ----------------------- |
| 全称                  | Free Space Map         | Visibility Map          |
| 文件后缀              | `_fsm`                 | `_vm`                   |
| 记录内容              | Page 的可用空间类别    | Page 的可见性、冻结状态 |
| 数据结构              | 多层树 + Page 内二叉树 | Bitmap                  |
| 每个 Heap Page 的信息 | 1 byte 空间类别        | 2 bit                   |
| 主要服务对象          | INSERT、VACUUM         | VACUUM、Index-Only Scan |
| 是否保存元组数据      | 否                     | 否                      |
| VACUUM 的作用         | 更新空闲空间信息       | 设置可见性、冻结状态    |
| UPDATE/DELETE 的影响  | 可能改变可用空间记录   | 清除相关可见性位        |

从整个 PostgreSQL Heap 存储系统来看：

```
                   PostgreSQL Relation
                            |
          +-----------------+-----------------+
          |                 |                 |
          v                 v                 v
      Main Fork          FSM Fork          VM Fork
          |                 |                 |
          v                 v                 v
      Heap Pages       Free Space Map    Visibility Map
          |                 |                 |
          v                 v                 v
     保存元组数据       记录空闲空间       记录可见性状态
          |                 |                 |
          +-----------------+-----------------+
                            |
                            v
                  PostgreSQL 存储管理
```

可以这样记忆：

- Main Fork：数据存在哪里？
- FSM Fork：哪里还有空间可以存新数据？
- VM Fork：哪些 Page 的数据已经对所有事务可见，哪些 Page 不再需要冻结？

这三个 Fork 共同支持 PostgreSQL 的 MVCC、空间复用、VACUUM 和高效查询。

如果接下来继续阅读 PostgreSQL 16 内核，建议重点研究 `visibilitymap_set()` 和 `visibilitymap_clear()`，尤其是 Heap Page 中的 `PD_ALL_VISIBLE` 标志与 VM Fork 中的 All-Visible bit 为什么要同时存在，以及 PostgreSQL 如何保证两者的一致性。这涉及 VACUUM、WAL 和并发控制之间非常重要的设计。