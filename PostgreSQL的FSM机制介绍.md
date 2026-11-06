# PostgreSQL FSM（Free Space Map）详解

FSM（Free Space Map，空闲空间映射）是 PostgreSQL 用来记录表中各个 Page 剩余可用空间，并帮助 INSERT 等操作快速找到合适 Page 的一种辅助存储结构。

结合前面讨论的 VACUUM，可以这样理解：

- VACUUM：负责清理死亡元组，回收 Heap Page 内部的空间。
- FSM：负责记录哪些 Heap Page 有空闲空间，以及大概有多少。
- INSERT：通过 FSM 寻找能够容纳新元组的 Page，尽量复用已有空间，避免表文件不断增长。

需要注意：FSM 本身不存储元组，也不负责回收死亡元组。它只管理空闲空间信息。

## 一、为什么 PostgreSQL 需要 FSM？

假设一张表包含 100 万个 Heap Page，每个 Page 为 8KB。

执行大量 UPDATE、DELETE，再经过 VACUUM 后，部分 Page 中出现了可复用空间：

```
Heap Relation

Page 0       Free Space = 100 bytes
Page 1       Free Space = 500 bytes
Page 2       Free Space = 2000 bytes
Page 3       Free Space = 50 bytes
Page 4       Free Space = 4000 bytes
...
Page 999999  Free Space = 1000 bytes
```

现在执行：

```
INSERT INTO test VALUES (100, 'Alice');
```

假设新元组需要 300 字节。PostgreSQL 需要找到一个至少有 300 字节可用空间的 Page。

如果没有 FSM，就可能需要逐个读取 Heap Page，检查 `pd_upper - pd_lower`，直到找到合适的位置。

对于一张拥有 100 万个 Page 的表，这种查找方式显然效率很低。因此，PostgreSQL 引入 FSM。

它相当于为整个表维护了一份空闲空间目录：

```
FSM

Page 0       -> 100 bytes
Page 1       -> 500 bytes
Page 2       -> 2000 bytes
Page 3       -> 50 bytes
Page 4       -> 4000 bytes
...
Page 999999  -> 1000 bytes
```

当需要插入 300 字节的数据时，PostgreSQL 可以通过 FSM 快速找到合适的 Page，而不需要扫描整个表。

不过，上面只是概念示意。PostgreSQL 实际并不是直接为每个 Page 存储一个精确的空闲字节数。

## 二、FSM 在磁盘上是如何存储的？

PostgreSQL 的每个普通 Heap 表通常有多个 Fork：

```
base/
  16384/
    24576          <-- Main Fork
    24576_fsm      <-- FSM Fork
    24576_vm       <-- Visibility Map Fork
```

其中：

| 文件        | 作用                              |
| ----------- | --------------------------------- |
| `24576`     | 保存 Heap Page 和元组数据         |
| `24576_fsm` | 保存各个 Heap Page 的空闲空间信息 |
| `24576_vm`  | 保存 Page 的可见性和冻结状态信息  |

这些文件属于同一个 Relation 的不同 Fork。

可以通过下面的 SQL 查看：

```
SELECT pg_relation_filepath('test');
```

还可以分别查看它们的大小：

```
SELECT
    pg_size_pretty(pg_relation_size('test', 'main')) AS main_size,
    pg_size_pretty(pg_relation_size('test', 'fsm')) AS fsm_size,
    pg_size_pretty(pg_relation_size('test', 'vm')) AS vm_size;
```

例如，可能得到：

```
 main_size | fsm_size | vm_size
-----------+----------+---------
 100 MB    | 32 kB    | 8192 bytes
```

这些数值只是示例。

FSM 是独立于 Heap 数据文件的辅助结构，它不会直接修改 Heap Page 的 `pd_lower` 或 `pd_upper`。

## 三、FSM 的内部数据结构

FSM 并不是简单的数组，而是采用了树形结构。

这里需要区分两个层次：

1. FSM 文件中的多层 Page 组织。
2. 每个 FSM Page 内部的二叉树。

### 1. FSM Page 内部的二叉树

PostgreSQL 将 Heap Page 的可用空间大小量化为一个字节。

一个 FSM 条目只需要 1 byte，就可以表示某个 Heap Page 的空闲空间类别。

例如，默认 8KB Page 下，空闲空间大致按照 32 字节一个等级进行量化。

```
Heap Page 0 -> 100 bytes
Heap Page 1 -> 500 bytes
Heap Page 2 -> 2000 bytes
Heap Page 3 -> 4000 bytes
```

FSM 中保存的是量化后的空间类别，而不是精确字节数。

在一个 FSM Page 内部，PostgreSQL 使用类似最大值二叉树的结构：

```
                   125
                 /     \
               62       125
              /  \      /  \
             3   62    125  1
             |    |     |   |
           Page0 Page1 Page2 Page3
```

这里的数字表示空间类别，仅用于说明结构。

树的叶子节点对应 Heap Page 的空闲空间类别。

每个内部节点保存其子树中的最大空闲空间类别。

例如：

```
Page 0 -> 3
Page 1 -> 15
Page 2 -> 62
Page 3 -> 125
```

则：

```
                 125
               /     \
             15       125
            /  \      /  \
           3   15    62  125
```

假设现在需要寻找一个空闲空间类别至少为 50 的 Page。

查找过程为：

```
Root = 125
    |
    v
Left = 15 < 50
    |
    v
Right = 125
    |
    v
找到满足要求的叶子节点
```

这样可以快速跳过不满足条件的整个子树。

这就是 FSM 使用树形结构的重要原因。

### 2. FSM 文件本身也具有树形层次

如果一个 FSM Page 无法记录整张表的空闲空间信息，就需要多个 FSM Page。

PostgreSQL 使用多层 FSM Page 来组织这些信息。

概念上可以理解为：

```
                  FSM Root Page
                  /           \
                 /             \
          FSM Page A         FSM Page B
          /       \          /       \
         /         \        /         \
    FSM Leaf 0  FSM Leaf 1  ...    FSM Leaf N
       |             |
       v             v
  Heap Pages     Heap Pages
  0 ... K        K+1 ... M
```

实际 PostgreSQL 的 FSM 是固定层数的树形组织，并不完全等同于普通动态 B-Tree。

在 PostgreSQL 16 中，FSM 使用三级 FSM Page 结构（叶层、中间层、根层），每个 FSM Page 内部再使用二叉树管理空闲空间类别。

因此，FSM 可以在很大的表中快速定位具有足够空闲空间的 Heap Page。

## 四、VACUUM 如何更新 FSM？

结合前面讨论的 Page 空间回收过程。

假设 Heap Page 100 中有三个元组：

```
Page 100

+--------------------------+
| PageHeaderData           |
+--------------------------+
| ItemId[0] -> Tuple A     |
| ItemId[1] -> Tuple B     |
| ItemId[2] -> Tuple C     |
+--------------------------+
| Free Space = 500 bytes   |
+--------------------------+
| Tuple C (Live)           |
| Tuple B (Dead)           |
| Tuple A (Live)           |
+--------------------------+
```

此时 FSM 中记录：

```
FSM:

Page 100 -> 500 bytes（近似）
```

执行：

```
DELETE FROM test WHERE id = 2;
VACUUM test;
```

当 VACUUM 确认 Tuple B 可以回收，并完成必要的索引清理及页面整理后：

```
Page 100

+--------------------------+
| PageHeaderData           |
+--------------------------+
| ItemId[0] -> Tuple A     |
| ItemId[1] -> LP_UNUSED   |
| ItemId[2] -> Tuple C     |
+--------------------------+
|                          |
| Free Space = 600 bytes   |
|                          |
+--------------------------+
| Tuple C (Live)           |
| Tuple A (Live)           |
+--------------------------+
```

VACUUM 可以将新的空闲空间信息记录到 FSM。

从逻辑上看：

```
VACUUM 前：

FSM[100] = 500 bytes

VACUUM 后：

FSM[100] = 600 bytes
```

注意，FSM 实际存储的是量化后的类别。

### 相关源码

PostgreSQL 16 中，可以关注：

```
RecordPageWithFreeSpace(...)
```

源码位置：

```
src/backend/storage/freespace/freespace.c
```

该函数用于记录某个 Relation Page 的可用空间信息。

另一个重要函数是：

```
GetRecordedFreeSpace(...)
```

它用于获取 FSM 中记录的空闲空间大小。

## 五、INSERT 如何利用 FSM 找到合适的 Page？

假设执行：

```
INSERT INTO test VALUES (10, 'New User');
```

PostgreSQL 需要寻找一个能够容纳新元组的 Heap Page。

相关核心函数是：

```
RelationGetBufferForTuple(...)
```

源码位置：

```
src/backend/access/heap/hio.c
```

简化后的执行流程：

```
INSERT
   |
   v
heap_insert()
   |
   v
RelationGetBufferForTuple()
   |
   v
计算新元组所需空间
   |
   v
查找 FSM
   |
   +-- 找到满足条件的 Heap Page
   |       |
   |       v
   |   读取并锁定 Heap Page
   |       |
   |       v
   |   检查实际空闲空间
   |       |
   |       +-- 足够：插入元组
   |       |
   |       +-- 不足：更新 FSM 并继续查找
   |
   +-- 没找到合适 Page
           |
           v
       扩展 Relation
           |
           v
       分配新的 Heap Page
```

这里还有一个值得注意的细节：

FSM 中记录的空闲空间不一定与 Heap Page 当前的实际空闲空间完全一致。

例如：

```
FSM 记录：

Page 100 -> 2000 bytes
```

但由于其他事务已经向 Page 100 插入数据：

```
实际 Heap Page：

Free Space = 500 bytes
```

因此 PostgreSQL 不能完全相信 FSM。

它必须在获取并锁定目标 Heap Page 后，再检查 Page 的真实可用空间。

如果空间不足，就更新 FSM，并尝试寻找其他 Page。

这也是为什么 FSM 是一种空间查找辅助结构，而不是严格实时、完全精确的空间分配表。

## 六、FSM 与 PageHeaderData 的关系

这是理解 FSM 时非常重要的一点。

FSM 不直接管理元组，也不直接保存 `pd_lower`、`pd_upper`。

真正的空闲空间位于 Heap Page 内部：

```
Heap Page 100

pd_lower = 1000
pd_upper = 4000

Free Space = 3000 bytes
```

而 FSM 保存的是：

```
FSM:

Page 100 -> 约 3000 bytes
```

两者关系如下：

```
                  Heap Page
                      |
                      v
             PageHeaderData
                      |
             +--------+--------+
             |                 |
          pd_lower          pd_upper
             |                 |
             +--------+--------+
                      |
                      v
              计算可用空间
                      |
                      v
            RecordPageWithFreeSpace()
                      |
                      v
                     FSM
                      |
                      v
             后续 INSERT 查找
```

实际计算可用空间时，PostgreSQL 通常使用：

```
PageGetHeapFreeSpace(page)
```

而不是简单地直接使用：

```
pd_upper - pd_lower
```

原因是 `PageGetHeapFreeSpace()` 还会考虑是否需要为新元组分配新的行指针。

例如：

```
Page 100

pd_upper - pd_lower = 1000 bytes
```

如果不存在可复用的 `LP_UNUSED` 行指针，插入一个新元组还需要分配一个新的 `ItemIdData`（4 字节）。

因此实际可用于元组数据的空间可能是：

```
1000 - 4 = 996 bytes
```

如果存在可以复用的 `LP_UNUSED` 行指针，就可能不需要额外的 4 字节。

因此，FSM 记录的可用空间，更接近于 Heap Page 能够容纳的新元组大小，而不只是 `pd_upper - pd_lower` 的差值。

## 七、如何通过 SQL 查看 FSM？

PostgreSQL 提供了一个非常实用的扩展：

```
CREATE EXTENSION pg_freespacemap;
```

创建测试表：

```
CREATE TABLE fsm_test (
    id INT,
    data TEXT
);

INSERT INTO fsm_test
SELECT i, repeat('a', 100)
FROM generate_series(1, 10000) i;
```

执行：

```
VACUUM fsm_test;
```

然后查看某个 Heap Page 在 FSM 中记录的空闲空间：

```
SELECT pg_freespace('fsm_test', 0);
```

其中：

- `fsm_test`：表名。
- `0`：Heap Block Number。

也可以查看整张表的 FSM 信息：

```
SELECT * FROM pg_freespace('fsm_test');
```

返回结果类似：

```
 blkno | avail
-------+-------
     0 |    32
     1 |   128
     2 |   512
     3 |  1024
     4 |  2048
```

这些数据表示 FSM 记录的各个 Heap Page 的近似可用空间。

还可以结合 `pageinspect` 扩展：

```
CREATE EXTENSION pageinspect;
```

查看 PageHeaderData：

```
SELECT
    lower,
    upper,
    special,
    upper - lower AS contiguous_free_space
FROM page_header(get_raw_page('fsm_test', 0));
```

再与 FSM 信息对比：

```
SELECT pg_freespace('fsm_test', 0);
```

两者可能不完全一致，因为 FSM 的数据经过量化，并且可能不是最新状态。

## 八、总结

可以将 PostgreSQL 的空间管理分成三个层次：

| 层次        | 核心结构                    | 作用                                       |
| ----------- | --------------------------- | ------------------------------------------ |
| 元组层      | HeapTuple / ItemIdData      | 保存元组及其物理位置                       |
| 页面层      | PageHeaderData / Free Space | 管理单个 Page 内部的空闲空间               |
| Relation 层 | FSM                         | 管理整个 Relation 中各 Page 的空闲空间信息 |

三者之间的关系：

```
                  VACUUM
                     |
                     v
              清理死亡元组
                     |
                     v
           整理 Heap Page 空间
                     |
                     v
             pd_upper 发生变化
                     |
                     v
           计算 Page 可用空间
                     |
                     v
               更新 FSM
                     |
                     v
               后续 INSERT
                     |
                     v
            通过 FSM 寻找 Page
                     |
                     v
              复用空闲空间
```

一句话总结：VACUUM 负责产生可复用空间，FSM 负责帮助后续插入操作找到这些空间。

如果继续研究 PostgreSQL 16 内核源码，可以按照下面的调用关系阅读：

```
src/backend/storage/freespace/freespace.c
    |
    +-- RecordPageWithFreeSpace()
    |
    +-- GetPageWithFreeSpace()
    |
    +-- GetRecordedFreeSpace()

src/backend/access/heap/hio.c
    |
    +-- RelationGetBufferForTuple()

src/backend/storage/page/bufpage.c
    |
    +-- PageGetHeapFreeSpace()
```

其中最值得重点研究的是 `GetPageWithFreeSpace()` 如何利用 FSM 的树形结构定位 Heap Page，以及 `RelationGetBufferForTuple()` 如何处理 FSM 信息与实际 Page 空闲空间不一致的情况。