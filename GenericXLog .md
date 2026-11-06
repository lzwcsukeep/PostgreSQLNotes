`GenericXLog` 是 PostgreSQL WAL 子系统里一个很重要、但容易被普通 SQL 层用户忽略的机制。它主要服务于**自定义存储结构 / Index AM / Table AM 等需要自己修改页面（page）的代码**。

你可以先把它理解成：

> **GenericXLog = PostgreSQL 提供的一套“帮你把页面修改转换成 WAL，并保证崩溃恢复正确”的通用接口。**

它解决的问题是：如果一个存储引擎直接修改 `Page`，那么数据库崩溃后，WAL 必须能够重新执行这些修改，否则磁盘页面可能处于不一致状态。

------

# 1. 为什么需要 GenericXLog？

假设你实现了一个自己的存储结构：

```text
MyIndexPage
+----------------------+
| header               |
| item 1               |
| item 2               |
| item 3               |
+----------------------+
```

现在要插入一个 item：

```c
page = ReadBuffer(...);

PageAddItem(page, item, ...);
```

看起来非常简单。

但是 PostgreSQL 是 WAL 架构。

正常情况下：

```text
修改 Page
   ↓
产生 WAL
   ↓
WAL flush
   ↓
Page 写入磁盘
```

如果数据库在这里：

```text
Page 已修改
WAL 还没持久化
↓
服务器 crash
```

就可能出现：

```text
磁盘上的 Page
      ↓
没有对应 WAL
      ↓
无法正确恢复
```

所以 PostgreSQL 要求：

> **任何持久化数据页的修改，都必须遵守 WAL 规则。**

对于 PostgreSQL 自己的 heap、btree 等模块，它们有自己的 WAL record 格式。

例如 B-tree 有自己的：

```text
XLOG_BTREE_INSERT_LEAF
XLOG_BTREE_SPLIT
XLOG_BTREE_DELETE
...
```

但如果你写一个新的 Access Method：

```text
MyIndex
MyTableAM
MyStorage
```

难道你还要自己设计：

```text
XLOG_MYAM_INSERT
XLOG_MYAM_SPLIT
XLOG_MYAM_DELETE
...
```

以及：

```text
WAL record
   ↓
redo function
   ↓
解析 page modification
   ↓
重新修改 page
```

这工作量很大。

因此 PostgreSQL 提供了：

```text
Generic WAL
```

也就是：

```text
GenericXLog
```

------

# 2. GenericXLog 的核心思想

GenericXLog 不关心你的 page 是什么结构。

它只关心：

> **这个 page 从修改前变成了修改后。**

例如：

```text
修改前

Page A
+----------------+
| aaa            |
| bbb            |
| ccc            |
+----------------+

        ↓

修改

        ↓

修改后

Page B
+----------------+
| aaa            |
| bbb            |
| xxx            |
| ccc            |
+----------------+
```

GenericXLog 可以记录：

```text
Page A → Page B
```

因此 WAL recovery 时：

```text
old page
   ↓
Generic WAL record
   ↓
apply recorded changes
   ↓
new page
```

这就是它最核心的思想。

------

# 3. GenericXLog 主要位于哪里？

源码主要看：

```text
src/backend/access/transam/generic_xlog.c
src/include/access/generic_xlog.h
```

你可以重点看：

```c
GenericXLogState
```

以及几个核心 API：

```c
GenericXLogStart()
GenericXLogRegisterBuffer()
GenericXLogUnregisterBuffer()
GenericXLogFinish()
GenericXLogAbort()
```

整体关系可以先记成：

```text
GenericXLogStart()
        │
        ▼
GenericXLogState
        │
        ├── RegisterBuffer()
        │
        ├── 修改 page
        │
        ├── RegisterBuffer()
        │
        └── Finish()
                 │
                 ▼
              WAL
```

------

# 4. 一个最简单的使用流程

典型代码大概是：

```c
GenericXLogState *state;
Buffer buffer;
Page page;

state = GenericXLogStart();

buffer = ReadBuffer(rel, blkno);

page = GenericXLogRegisterBuffer(state, buffer, 0);

/*
 * 修改 page
 */
...

GenericXLogFinish(state);
```

这个流程非常重要。

你可以把它理解成一个“小型事务”：

```text
GenericXLogStart
        │
        │ 开始记录页面修改
        ▼
RegisterBuffer
        │
        │ 获取可修改的 page
        ▼
修改 page
        │
        ▼
GenericXLogFinish
        │
        │ 生成 WAL
        ▼
结束
```

------

# 5. GenericXLogRegisterBuffer 是关键

最重要的 API 之一：

```c
Page
GenericXLogRegisterBuffer(GenericXLogState *state,
                          Buffer buffer,
                          int flags);
```

例如：

```c
page = GenericXLogRegisterBuffer(state, buffer, 0);
```

它并不是简单地：

```c
PageGetContents(buffer)
```

而是在 GenericXLog 内部建立一个：

```text
原始 Page
     +
工作副本
```

概念上可以理解成：

```text
Buffer
  │
  ▼
原始 page
  │
  │ copy
  ▼
GenericXLog 工作 page
```

然后你修改的是：

```text
工作 page
```

而不是直接把 shared buffer 里的 page 随便改掉。

这是 GenericXLog 能够计算 WAL 的基础。

------

# 6. 为什么要保存“修改前”和“修改后”？

假设：

```text
Before:

+------------------+
| A                |
| B                |
| C                |
+------------------+
```

修改：

```c
page = GenericXLogRegisterBuffer(...);

PageAddItem(page, ...);
```

得到：

```text
After:

+------------------+
| A                |
| B                |
| X                |
| C                |
+------------------+
```

GenericXLog 可以比较：

```text
Before
   │
   │ diff
   ▼
After
```

然后生成 WAL。

因此 Generic WAL 不需要知道：

```text
X 是什么结构
```

它只需要知道：

```text
哪些 page bytes 发生了变化
```

这也是它叫 **Generic** 的原因。

------

# 7. GenericXLogFinish 做什么？

最后：

```c
GenericXLogFinish(state);
```

是整个过程的核心。

它需要完成类似：

```text
工作 Page
   │
   ├── 与原 Page 比较
   │
   ├── 生成 WAL record
   │
   ├── 写入 WAL
   │
   ├── 将修改应用到 shared buffer
   │
   └── 完成页面状态更新
```

概念上：

```text
                 GenericXLogFinish
                        │
             ┌──────────┴──────────┐
             │                     │
             ▼                     ▼
        WAL record             Page update
             │                     │
             ▼                     ▼
        WAL subsystem          shared buffer
```

这样才能满足 WAL 的基本要求：

```text
WAL
 ↓
先持久化
 ↓
Page
```

即经典的：

> **Write-Ahead Logging**

------

# 8. Generic WAL 和普通 WAL 有什么区别？

这是理解 GenericXLog 最重要的一点。

普通 PostgreSQL WAL：

```text
BTREE INSERT
```

可以记录成高度结构化的信息：

```text
relation
block
offset
item
...
```

然后 redo：

```c
_bt_restore_page(...)
```

根据 WAL record 的语义重新构造 page。

而 Generic WAL 更接近：

```text
Page before
      ↓
byte-level changes
      ↓
Page after
```

所以：

```text
普通 WAL

语义化
   ↓
INSERT
SPLIT
DELETE
UPDATE
```

而：

```text
Generic WAL

页面变化
   ↓
page bytes
```

可以粗略理解为：

```text
             WAL
              │
       ┌──────┴──────┐
       │             │
       ▼             ▼
普通 WAL          Generic WAL
       │             │
       ▼             ▼
结构化操作       页面变化
       │             │
       ▼             ▼
BTREE redo       Generic redo
```

------

# 9. 为什么不所有 PostgreSQL 都使用 GenericXLog？

因为 Generic WAL 有明显的代价。

假设一个页面：

```text
8 KB
```

只修改了：

```text
10 bytes
```

Generic WAL 需要记录：

```text
这些 bytes 的变化
```

而一个高度优化的专用 WAL 可以只记录：

```text
offset = 123
length = 10
data = ...
```

甚至记录更高层的语义：

```text
insert tuple #5
```

专用 WAL 通常可以更加紧凑、高效。

因此 PostgreSQL 内置模块一般自己定义 WAL。

例如：

```text
heapam
  ↓
heap WAL

nbtree
  ↓
btree WAL

gin
  ↓
GIN WAL

gist
  ↓
GiST WAL
```

而 GenericXLog 更适合：

```text
自定义 Access Method
自定义 index
自定义 page-oriented storage
```

尤其是：

> **你希望快速实现一个 WAL-safe 的 page-based 存储结构。**

------

# 10. GenericXLog 可以操作多个 Buffer

它并不一定只能处理一个 page。

例如一个操作同时修改：

```text
Buffer A
Buffer B
Buffer C
```

可以：

```c
state = GenericXLogStart();

page_a = GenericXLogRegisterBuffer(state, buffer_a, 0);
page_b = GenericXLogRegisterBuffer(state, buffer_b, 0);
page_c = GenericXLogRegisterBuffer(state, buffer_c, 0);

...

GenericXLogFinish(state);
```

最终：

```text
             GenericXLogState
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
     Buffer A    Buffer B    Buffer C
        │           │           │
        ▼           ▼           ▼
     Page A       Page B       Page C
        │           │           │
        └───────────┼───────────┘
                    ▼
               Generic WAL
```

这对于一些需要同时修改多个页面的存储操作很重要。

------

# 11. GenericXLogState 是什么？

可以把：

```c
GenericXLogState
```

理解成：

> **一次 Generic WAL 操作的上下文。**

类似：

```text
GenericXLogState
        │
        ├── Buffer 1
        │      ├── original page
        │      └── modified page
        │
        ├── Buffer 2
        │      ├── original page
        │      └── modified page
        │
        └── WAL information
```

所以它不是数据库事务本身。

注意这一点：

```text
GenericXLogState
       ≠
Transaction
```

它只是：

```text
一次 Generic WAL page modification operation
```

------

# 12. GenericXLogAbort

如果中间决定放弃：

```c
GenericXLogAbort(state);
```

那么：

```text
GenericXLogStart
       ↓
RegisterBuffer
       ↓
修改工作副本
       ↓
发现错误
       ↓
GenericXLogAbort
```

修改不会作为一次成功的 Generic WAL 操作提交。

因此基本模式就是：

```c
state = GenericXLogStart();

...

if (error)
{
    GenericXLogAbort(state);
    return;
}

GenericXLogFinish(state);
```

------

# 13. 它和 Buffer Lock 是什么关系？

这也是你后面研究 PostgreSQL Storage Engine 时非常值得注意的一点。

GenericXLog 并没有取代：

```c
LockBuffer()
```

通常仍然需要正确管理 buffer lock。

大致是：

```text
Buffer
  │
  ▼
LockBuffer
  │
  ▼
GenericXLogRegisterBuffer
  │
  ▼
修改 Page
  │
  ▼
GenericXLogFinish
```

具体锁模式和调用顺序取决于使用场景。

核心原则是：

> **GenericXLog 负责 WAL/page-change tracking，不负责替代 PostgreSQL 的 buffer locking 体系。**

------

# 14. 和 Full Page Write 的关系

这里还有一个非常重要的概念：

```text
full-page image
```

PostgreSQL 在某些情况下会把整个页面写进 WAL。

原因是：

```text
Page 修改
   ↓
数据库 crash
   ↓
磁盘 Page 可能只写了一半
```

这就是：

> **torn page**

例如：

```text
磁盘 Page

+----------------+
| 新数据         |
| 新数据         |
| 新数据         |
| OLD             ← crash
| OLD             |
+----------------+
```

如果 WAL 中有：

```text
full page image
```

恢复时就可以先：

```text
FPW
 ↓
恢复完整 Page
 ↓
继续应用 WAL
```

GenericXLog 也需要和 PostgreSQL 的这一套 WAL / full-page-write 机制配合。

所以它不是：

```text
自己解决 crash recovery
```

而是：

```text
GenericXLog
      ↓
PostgreSQL WAL
      ↓
WAL persistence
      ↓
checkpoint
      ↓
recovery
```

------

# 15. Generic WAL 的恢复

你可以把 recovery 想象成：

```text
服务器 crash
     │
     ▼
读取 WAL
     │
     ▼
发现 Generic WAL record
     │
     ▼
Generic WAL redo
     │
     ▼
根据 WAL 中保存的页面变化
恢复 page
```

Generic WAL 的 redo 代码同样在：

```text
src/backend/access/transam/generic_xlog.c
```

相关 WAL resource manager 通常是：

```c
RM_GENERIC_ID
```

你可以进一步追：

```text
rmgr.c
rmgrlist.h
generic_xlog.c
```

------

# 16. 从 PostgreSQL 架构看 GenericXLog

结合你最近在研究的 **Table AM / Storage Engine**，我建议你把它放在这个位置理解：

```text
                       SQL
                        │
                        ▼
                    Executor
                        │
                        ▼
                    Table AM
                        │
                        ▼
                  Storage Engine
                        │
                ┌───────┴────────┐
                │                │
                ▼                ▼
             Buffer           Page
                │                │
                │        GenericXLog
                │                │
                └────────┬───────┘
                         ▼
                        WAL
                         │
                         ▼
                    WAL Receiver
                    / Archive
                    / Recovery
```

不过要注意：

**不是 Table AM 必须使用 GenericXLog。**

例如你自己的 Table AM 完全可以设计自己的 WAL record。

GenericXLog 更像是：

```text
PostgreSQL 给 Access Method 开发者提供的
一个通用 WAL 工具箱。
```

------

# 17. 一个非常简化的例子

假设我们实现：

```text
MyAM
```

每个 page：

```c
typedef struct MyPageHeader
{
    uint32 magic;
    uint32 nitems;
} MyPageHeader;
```

插入一个 item：

```c
GenericXLogState *state;
Page page;

state = GenericXLogStart();

page = GenericXLogRegisterBuffer(state, buffer, 0);

MyPageHeader *hdr = (MyPageHeader *) PageGetContents(page);

hdr->nitems++;

GenericXLogFinish(state);
```

这里：

```c
hdr->nitems++;
```

看起来只是：

```text
100 → 101
```

但实际上 GenericXLog 帮你把：

```text
Page before
```

和：

```text
Page after
```

之间的修改纳入：

```text
WAL
```

最终形成：

```text
MyAM page modification
        │
        ▼
GenericXLog
        │
        ▼
Generic WAL record
        │
        ▼
WAL
```

------

# 18. 你可以重点记住这几个 API

| API                             | 作用                                          |
| ------------------------------- | --------------------------------------------- |
| `GenericXLogStart()`            | 开始一次 Generic WAL 操作                     |
| `GenericXLogRegisterBuffer()`   | 把 Buffer 注册进这次操作，并获得可修改的 Page |
| `GenericXLogUnregisterBuffer()` | 从操作中移除 Buffer                           |
| `GenericXLogFinish()`           | 完成操作、生成并写入 WAL                      |
| `GenericXLogAbort()`            | 放弃这次操作                                  |

最核心的使用模型：

```c
state = GenericXLogStart();

page = GenericXLogRegisterBuffer(state, buffer, 0);

/* modify page */

GenericXLogFinish(state);
```

------

# 19. 如果你准备研究源码，建议这样追

你现在正在从：

```text
fmgr
  ↓
Table AM
  ↓
Buffer
  ↓
Page
```

往 PostgreSQL 存储层深入，那么 GenericXLog 很适合作为下一步。

建议直接从：

```text
src/include/access/generic_xlog.h
```

开始。

然后：

```text
generic_xlog.h
       │
       ▼
GenericXLogStart()
       │
       ▼
GenericXLogRegisterBuffer()
       │
       ▼
GenericXLogFinish()
       │
       ├── XLogBeginInsert()
       ├── XLogRegisterBuffer()
       ├── XLogInsert()
       └── ...
       │
       ▼
WAL
```

最终把它和 PostgreSQL 的通用 WAL 接口串起来：

```text
GenericXLog
      │
      ▼
XLogBeginInsert()
      │
      ▼
XLogRegisterBuffer()
      │
      ▼
XLogRegisterBufData()
      │
      ▼
XLogInsert()
      │
      ▼
WAL
```

**这一条链非常值得你结合源码和 GDB 跑一遍。**

尤其是你之前研究 `TableAM` 时看到的：

```c
table_tuple_insert()
```

如果继续往下研究自定义存储引擎，你最终一定会碰到这个问题：

> **“我已经拿到了 Buffer，也修改了 Page，那么 PostgreSQL 到底是在哪里把这个 Page 修改变成 WAL 的？”**

GenericXLog 正好就是回答这个问题的一条非常典型的源码路径。