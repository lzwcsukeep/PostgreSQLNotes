PostgreSQL 源码里面设计的双向链表和教科书上的不太一样，很有特色，叫做侵入式双向链表。下面讲解这种结构。

双向链表的源码定义如下：

**1. 表头与节点**

核心定义很简单，见 [ilist.h (line 137)](/home/lzw/workspace/ext_projs/postgresql-16.11/src/include/lib/ilist.h:137)：

```
typedef struct dlist_node dlist_node;

struct dlist_node
{
    dlist_node *prev;
    dlist_node *next;
};

typedef struct dlist_head
{
    dlist_node head;
} dlist_head;
```

- `dlist_node`：节点，只保存前后指针。
- `dlist_head`：表头，其中的 `head` 是一个**哨兵节点**，不代表业务数据。

例如，`PGPROC` 中嵌入了一个节点：

```
struct PGPROC
{
    dlist_node links;
    /* 其他进程状态字段 */
};
```

链表中的指针指向 `PGPROC.links`，通过这个成员把不同的 `PGPROC` 串起来，不需要额外分配链表节点。

`dlist_head`就是 `PROC_HDR.freeProcs` 使用的链表。

**2. 链表是循环的**

设 `H` 表示表头里的哨兵 `head`，有两个业务节点 A、B：

```
沿 next：
H → A.links → B.links → H

沿 prev：
H → B.links → A.links → H
```

因此：

```
H.next             // 第一个节点
H.prev             // 最后一个节点
H.next->prev == &H
H.prev->next == &H
```

调用 `dlist_init()` 后，空链表是：

```
// 空链表判定条件
H.next = &H;
H.prev = &H;
```

哨兵让首尾节点的操作可以复用普通节点的指针修改逻辑。PG 也支持将表头清零来表示空链表；正常插入后会使用循环形式。

**3. 插入：修改四个指针**

假设要把节点 N 插入到相邻的 A、B 之间：

```
原来：A ⇄ B
之后：A ⇄ N ⇄ B
```

核心操作是：

```
N->prev = A;
N->next = B;
A->next = N;
B->prev = N;
```

头插 `dlist_push_head()` 就是把新节点插到哨兵和原首节点之间：

```
dlist_push_head(&ProcGlobal->freeProcs, &proc->links);
```

这表示：把 `proc` 的槽位放进空闲链表，供以后使用。插入只修改指针，不复制 `PGPROC`。

**4. 删除：让前后节点绕过当前节点**

```
原来：A ⇄ N ⇄ B
之后：A ⇄ B
```

核心操作：

```
N->prev->next = N->next;
N->next->prev = N->prev;
```

调用：

```
dlist_delete(&proc->links);
```

**删除节点只表示脱链，不会释放 `proc` 的内存。** `dlist` 不负责内存分配或释放。

因为节点自带 `prev`，已知节点地址时，插入和删除都是 O(1)，不需要从头查找。

**5. 怎么从节点找到 `PGPROC`？**

链表操作拿到的是 `dlist_node *`，业务代码通常需要的是 `PGPROC *`：

```
dlist_node *node = dlist_pop_head_node(&ProcGlobal->freeProcs);

PGPROC *proc = dlist_container(PGPROC, links, node);
```

`dlist_container()` 根据成员 `links` 在 `PGPROC` 中的偏移，反推出整个结构体的地址，概念上等价于：

```
PGPROC *proc =
    (PGPROC *) ((char *) node - offsetof(PGPROC, links));
```

因此，节点成员不必放在结构体开头。

你在 `proc.c` 中会看到直接强转：

```
MyProc = (PGPROC *) dlist_pop_head_node(procgloballist);
```

这是因为 `PGPROC.links` 恰好是第一个成员，二者起始地址相同；一般情况下应使用 `dlist_container()`。

**6. 遍历链表**

```
dlist_iter iter;

dlist_foreach(iter, &ProcGlobal->freeProcs)
{
    PGPROC *proc = dlist_container(PGPROC, links, iter.cur);

    /* 使用 proc */
}
```

其逻辑是从 `head.next` 开始，沿 `next` 前进，遇到哨兵时结束。

如果需要边遍历边删除当前节点，应使用专门的可修改迭代器：

```
dlist_mutable_iter iter;

dlist_foreach_modify(iter, &list)
{
    dlist_delete(iter.cur);
}
```

它会提前保存下一个节点。这个接口允许删除当前节点，但不能任意修改相邻节点。

**7. 放回 `freeProcs` 的场景理解**

`PGPROC` 的空闲链表大致这样工作：

```
初始化：预分配 PGPROC，将空闲槽位串入 freeProcs
    ↓
进程启动：从链表头摘下一个槽位，作为 MyProc
    ↓
进程运行：槽位仍在 allProcs 数组中，但已不在空闲链表中
    ↓
进程退出：清理槽位，再挂回空闲链表
```

数组和链表是同一批对象的两种组织方式：`allProcs` 提供固定编号访问，`freeProcs` 提供空闲槽位的快速领取与归还。

最后要注意，**`dlist` 本身没有并发保护**。例如这些进程空闲链表在并发领取、归还时，由调用方使用 `ProcStructLock` 保护。