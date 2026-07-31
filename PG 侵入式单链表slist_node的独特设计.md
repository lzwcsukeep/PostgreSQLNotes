# 为什么 `slist_node` 只存放 `next` 指针，而不存放数据？

这是 PostgreSQL `ilist.h` 中最容易让人疑惑的地方，也是它与教科书中链表实现最大的区别。

**答案是：因为 PostgreSQL 使用的是"侵入式链表（Intrusive Linked List）"。\**\*\*`slist_node`\*\**\* 本身不是一个完整的链表节点，它只是一个"链接器（Link）"。真正的数据存放在包含它的业务结构体中。**

------

## 1. 教科书中的普通链表

大多数数据结构教材都会这样定义链表节点：

```c
typedef struct Node
{
    int data;
    struct Node *next;
} Node;
```

其内存布局如下：

```
+----------------------+
| data = 100           |
| next --------------+ |
+---------------------|-+
                      |
                      ▼
               +----------------------+
               | data = 200           |
               | next = NULL          |
               +----------------------+
```

这里，`Node` 同时承担两个职责：

- 保存数据（`data`）
- 保存链表关系（`next`）

因此，每插入一个元素，都需要额外 `malloc` 一个 `Node`。

------

## 2. PostgreSQL 的设计

PostgreSQL 定义的是：

```c
struct slist_node
{
    slist_node *next;
};
```

这里没有任何业务数据，只有一个 `next` 指针。

这是因为 PostgreSQL 认为：

> **链表只负责维护对象之间的连接关系，而不负责保存对象本身的数据。**

------

## 3. 真实例子：Background Worker

例如：

```c
typedef struct RegisteredBgWorker
{
    pid_t pid;

    BackgroundWorker worker;

    TimestampTz crashed_at;

    slist_node rw_lnode;

} RegisteredBgWorker;
```

它的内存布局可以理解为：

```
RegisteredBgWorker

+-------------------------------------------+
| pid                                       |
|                                           |
| BackgroundWorker                          |
|                                           |
| crashed_at                                |
|                                           |
| rw_lnode                                  |
|     next ------------------------------+  |
+----------------------------------------|--+
                                         |
                                         ▼
```

真正的数据包括：

- pid
- BackgroundWorker
- crashed_at

而：

```
rw_lnode
```

只是告诉链表：

> **下一个对象是谁。**

------

## 4. 链表中真正连接的是什么？

假设有三个对象：

```c
RegisteredBgWorker w1;
RegisteredBgWorker w2;
RegisteredBgWorker w3;
```

链表实际上连接的是：

```
head
 │
 ▼
w1.rw_lnode ----> w2.rw_lnode ----> w3.rw_lnode ----> NULL
```

注意：

链表连接的是 `rw_lnode`，而不是整个 `RegisteredBgWorker`。

但是，每个 `rw_lnode` 都属于某一个 `RegisteredBgWorker`。

------

## 5. 如何通过 `slist_node` 找回真正的数据？

遍历链表时，得到的是：

```c
slist_node *node;
```

但真正需要的是：

```c
RegisteredBgWorker *rw;
```

PostgreSQL 使用：

```c
slist_container()
```

完成转换：

```c
RegisteredBgWorker *rw;

rw = slist_container(
        RegisteredBgWorker,
        rw_lnode,
        node);
```

其原理与 Linux Kernel 的 `container_of()` 完全相同。

本质上就是：

```
rw_lnode 指针
        │
        ▼
减去 rw_lnode 在结构体中的偏移量
        │
        ▼
得到 RegisteredBgWorker *
```

因此：

**链表保存的是 \**\*\*`slist_node`\*\**\*，真正操作的是包含它的业务对象。**

------

# 6. 为什么 PostgreSQL 采用这种设计？

主要有四个原因。

## （1）无需额外分配内存

普通链表：

```
业务对象
      │
      ▼
malloc(Node)
      │
      ▼
Node 保存 data*
```

需要两块内存。

而 PostgreSQL：

```
RegisteredBgWorker
        │
        ▼
内部直接包含 slist_node
```

只有一块内存。

例如：

```c
RegisteredBgWorker rw;
```

创建对象时，链表节点已经存在，无需再次 `malloc()`。

------

## （2）缓存局部性（Cache Locality）更好

普通链表：

```
Node
 │
 ▼
data*
 │
 ▼
真正对象
```

访问对象需要多一次指针跳转。

而 PostgreSQL：

```
RegisteredBgWorker
        │
        ├── pid
        ├── worker
        ├── crashed_at
        └── next
```

所有数据都在同一块内存中，更容易命中 CPU Cache，性能更高。

------

## （3）一个对象可以属于多个链表

例如：

```c
typedef struct
{
    int pid;

    slist_node wait_node;

    slist_node run_node;

} Worker;
```

这样：

同一个 `Worker` 可以同时存在于：

- 等待队列
- 运行队列

因为它拥有两个不同的链表节点。

如果使用传统链表，就无法做到这一点。

------

## （4）链表完全不知道业务数据类型

`slist` 根本不知道自己连接的是：

- RegisteredBgWorker
- PGPROC
- Relation
- BufferDesc

它唯一知道的是：

```c
next
```

因此：

任何结构体，只要包含：

```c
slist_node node;
```

就可以加入 `slist`。

这使 `slist` 成为 PostgreSQL 内部一种完全通用的基础数据结构。

------

# 7. 如何理解 `slist_node`？

可以把它理解成：

它不是一个"箱子（Node）"，而是一个"挂钩（Hook）"。

例如：

```
Worker A
+---------------------------+
| pid                       |
| worker                    |
| hook -------------------+
+---------------------------+
                            |
                            ▼
Worker B
+---------------------------+
| pid                       |
| worker                    |
| hook -------------------+
+---------------------------+
```

真正有价值的是整个 `Worker` 对象。

而 `hook`（即 `slist_node`）唯一的职责，就是把这些对象连接起来。

------

# 总结

传统链表与 PostgreSQL 侵入式链表的区别如下：

### 普通链表（非侵入式）

```
Node
├── data
└── next
```

Node 本身就是链表节点，同时保存数据。

------

### PostgreSQL / Linux Kernel（侵入式链表）

```
业务对象
├── 各种业务字段
└── slist_node（仅包含 next）
```

`slist_node` 只是业务对象中的一个成员，用于维护链表关系。

因此，**真正的数据始终保存在业务结构体（如 \**\*\*`RegisteredBgWorker`\*\**\*）中，而 \**\*\*`slist_node`\*\**\* 的唯一职责就是维护链表结构。**

正因为采用了这种设计，PostgreSQL 和 Linux Kernel 才能够做到：

- 无需额外分配链表节点内存；
- 更好的缓存局部性；
- 同一个对象可以同时加入多个链表；
- 链表实现完全独立于业务数据类型，具有极高的通用性。