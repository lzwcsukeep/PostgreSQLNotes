```c
/*
 * Enable for extra debugging. This is rather expensive, so it's not enabled by
 * default even when USE_ASSERT_CHECKING.
 */
/* #define ILIST_DEBUG */

/*
 * Node of a doubly linked list.
 *
 * Embed this in structs that need to be part of a doubly linked list.
 */
typedef struct dlist_node dlist_node;
struct dlist_node
{
	dlist_node *prev;
	dlist_node *next;
};

/*
 * Head of a doubly linked list.
 *
 * Non-empty lists are internally circularly linked.  Circular lists have the
 * advantage of not needing any branches in the most common list manipulations.
 * An empty list can also be represented as a pair of NULL pointers, making
 * initialization easier.
 */
typedef struct dlist_head
{
	/*
	 * head.next either points to the first element of the list; to &head if
	 * it's a circular empty list; or to NULL if empty and not circular.
	 *
	 * head.prev either points to the last element of the list; to &head if
	 * it's a circular empty list; or to NULL if empty and not circular.
	 */
	dlist_node	head;
} dlist_head;

```

**postgres 里设计的 doubly linked list 结构体里面为啥就两个指针，其余什么成员也没有？**

这是 PostgreSQL 中一个很典型的设计：`dlist_node` 只负责维护链表的连接关系，而不负责存储链表元素的数据。 元素本身的数据保存在其他结构体中，`dlist_node` 只是嵌入其中的一个成员。

这种设计叫作 侵入式双向链表（Intrusive Doubly Linked List）。

我们结合你贴出的 PostgreSQL 源码来分析。

## 1. 为什么只有 `prev` 和 `next` 两个指针？

先看结构体：

```c
typedef struct dlist_node dlist_node;

struct dlist_node
{
    dlist_node *prev;
    dlist_node *next;
};
```

这两个指针已经足以维护双向链表的连接关系：

双向链表示意图

![img](data:image/svg+xml;charset=utf-8,%3Csvg%20font-family%3D%22-apple-system-body%2C%20ui-sans-serif%2C%20-apple-system%2C%20system-ui%2C%20%26quot%3BSegoe%20UI%26quot%3B%2C%20Helvetica%2C%20%26quot%3BApple%20Color%20Emoji%26quot%3B%2C%20Arial%2C%20sans-serif%2C%20%26quot%3BSegoe%20UI%20Emoji%26quot%3B%2C%20%26quot%3BSegoe%20UI%20Symbol%26quot%3B%22%20font-weight%3D%22400%22%20data-d-component%3D%22svg%22%20fill%3D%22currentColor%22%20style%3D%22color%3Argb(13%2C%2013%2C%2013)%22%20viewBox%3D%220%200%20320%20115%22%20width%3D%22100%25%22%20xmlns%3D%22http%3A%2F%2Fwww.w3.org%2F2000%2Fsvg%22%3E%3Crect%20x%3D%228%22%20y%3D%2227%22%20width%3D%2288%22%20height%3D%2256%22%20rx%3D%227%22%20fill%3D%22%2364748b%22%20fill-opacity%3D%220.14%22%20stroke%3D%22%2364748b%22%2F%3E%3Crect%20x%3D%22116%22%20y%3D%2227%22%20width%3D%2288%22%20height%3D%2256%22%20rx%3D%227%22%20fill%3D%22%230d9488%22%20fill-opacity%3D%220.14%22%20stroke%3D%22%230d9488%22%2F%3E%3Crect%20x%3D%22224%22%20y%3D%2227%22%20width%3D%2288%22%20height%3D%2256%22%20rx%3D%227%22%20fill%3D%22%2364748b%22%20fill-opacity%3D%220.14%22%20stroke%3D%22%2364748b%22%2F%3E%3Cg%20font-size%3D%2211%22%20fill%3D%22currentColor%22%20text-anchor%3D%22middle%22%3E%3Ctext%20x%3D%2252%22%20y%3D%2248%22%3ENode%20A%3C%2Ftext%3E%3Ctext%20x%3D%2252%22%20y%3D%2264%22%3Eprev%20%2F%20next%3C%2Ftext%3E%3Ctext%20x%3D%22160%22%20y%3D%2248%22%3ENode%20B%3C%2Ftext%3E%3Ctext%20x%3D%22160%22%20y%3D%2264%22%3Eprev%20%2F%20next%3C%2Ftext%3E%3Ctext%20x%3D%22268%22%20y%3D%2248%22%3ENode%20C%3C%2Ftext%3E%3Ctext%20x%3D%22268%22%20y%3D%2264%22%3Eprev%20%2F%20next%3C%2Ftext%3E%3C%2Fg%3E%3Cg%20stroke%3D%22%230d9488%22%20stroke-width%3D%221.5%22%20fill%3D%22none%22%3E%3Cpath%20d%3D%22M96%2042%20H116%20M111%2038%20L116%2042%20L111%2046%22%2F%3E%3Cpath%20d%3D%22M116%2068%20H96%20M101%2064%20L96%2068%20L101%2072%22%2F%3E%3Cpath%20d%3D%22M204%2042%20H224%20M219%2038%20L224%2042%20L219%2046%22%2F%3E%3Cpath%20d%3D%22M224%2068%20H204%20M209%2064%20L204%2068%20L209%2072%22%2F%3E%3C%2Fg%3E%3C%2Fsvg%3E)

每个节点只需要知道前一个节点和后一个节点的地址，就能向前遍历、向后遍历，也能在常数时间内完成已知节点的插入和删除。

例如，假设有三个节点：

```
A.next = &B;
B.prev = &A;

B.next = &C;
C.prev = &B;
```

从 B 删除节点时，只需要调整相邻节点的指针：

```
B.prev->next = B.next;
B.next->prev = B.prev;
```

这里不需要知道 B 保存了什么业务数据，也不需要在链表节点里额外保存数据长度、节点类型或元素值。

因此，`prev` 和 `next` 已经完成了链表节点最核心的职责。

## 2. 那么，链表元素的数据存在哪里？

这才是 PostgreSQL 这个设计最值得理解的地方。

PostgreSQL 的 `dlist_node` 采用的是侵入式链表设计：将链表节点直接嵌入业务结构体，而不是单独分配一个包含数据指针的链表节点。

例如，我们自己定义一个任务结构体：

```
typedef struct Task
{
    int task_id;
    char task_name[32];

    dlist_node node;
} Task;
```

这里的 `Task` 才是真正保存业务数据的对象：

- `task_id`：任务编号。
- `task_name`：任务名称。
- `node`：用于将任务连接到双向链表。

假设创建三个任务：

```
Task task1 = {.task_id = 1, .task_name = "task1"};
Task task2 = {.task_id = 2, .task_name = "task2"};
Task task3 = {.task_id = 3, .task_name = "task3"};
```

每个任务对象内部都包含一个 `dlist_node`。

Task 对象在内存中的逻辑布局

Task 1

task_id = 1

task_name = "task1"

node：prev + next

Task 2

task_id = 2

task_name = "task2"

node：prev + next

绿色区域是链表管理所需的节点，其他成员由业务逻辑使用。

链表只需要操作 `node` 成员，但通过节点地址，我们仍然可以找到它所属的 `Task` 对象，进而访问任务数据。

这就意味着，PostgreSQL 不需要为每个链表元素额外分配一个独立的链表包装对象，也不需要额外存储一个指向业务数据的 `void *data` 指针。

## 3. 既然 `dlist_node` 不保存数据，怎么找到所属的结构体？

这是理解 PostgreSQL 链表实现的关键。

假设现在有一个：

```
dlist_node *node;
```

我们想得到包含它的 `Task *`。

因为 `node` 是 `Task` 的一个成员，所以可以通过成员偏移量反推出 `Task` 对象的起始地址。

PostgreSQL 的代码中经常使用 `container_of` 思路实现这种操作；具体辅助宏和函数名称需要以相应版本的源码为准。

简单演示如下：

```c
#include <stddef.h>

typedef struct Task
{
    int task_id;
    dlist_node node;
} Task;

Task *task = /* 某个 Task 对象 */;
dlist_node *node = &task->node;

/* 从 node 的地址反推出 Task 的地址 */
Task *owner = (Task *)(
    (char *)node - offsetof(Task, node)
);
```

这里：

- `offsetof(Task, node)`：计算 `node` 成员相对于 `Task` 起始地址的偏移量。
- `(char *)node - offsetof(...)`：从成员地址反向计算结构体起始地址。

这样，链表节点本身不需要知道自己属于什么类型的对象，也不需要保存一个额外的 `owner` 指针。

PostgreSQL 的链表实现就可以被不同类型的业务结构体复用，而不用把链表代码绑定到某一种具体的数据结构。

## 4. 为什么 `dlist_head` 也只有一个 `head`？

再看你贴出的另一个结构体：

```c
typedef struct dlist_head
{
    dlist_node head;
} dlist_head;
```

乍看之下，`dlist_head` 似乎也没有保存链表长度、首元素指针、尾元素指针等成员。

实际上，它把 `head` 本身作为哨兵节点（sentinel node）。

当链表非空时，首尾节点可以围绕哨兵节点形成一个环：

```
          ┌────────────────────────┐
          ↓                        │
      +--------+    +--------+    +--------+
      |  head  | ←→ | Node A | ←→ | Node B |
      +--------+    +--------+    +--------+
          ↑                        │
          └────────────────────────┘
```

这里的箭头表示双向连接；逻辑上，`head.next` 指向第一个元素，`head.prev` 指向最后一个元素。

这样做的好处是：

- 插入首元素时，不需要单独处理空链表的特殊情况。
- 删除首元素或尾元素时，不需要为边界情况编写另一套主要逻辑。
- 插入和删除通常只需要修改相邻节点的几个指针。

而空链表既可以表示为 `head.next == NULL && head.prev == NULL`，也可以表示为循环链表形式，即两个指针都指向 `&head`。你贴出的注释正是在说明这两种表示方式。

至于链表长度，`dlist_head` 不保存它。如果需要遍历统计，就遍历节点；如果业务需要高频获取长度，则可以在业务结构体或其他管理结构中单独维护。

## 5. 与普通非侵入式链表对比

| 对比项       | PostgreSQL 的侵入式链表                    | 常见的非侵入式链表                         |
| ------------ | ------------------------------------------ | ------------------------------------------ |
| 节点结构     | `prev`、`next`                             | `prev`、`next`、`data`                     |
| 数据存放     | 嵌入节点的业务结构体中                     | 通常由 `data` 指针指向                     |
| 类型耦合     | 链表实现与业务类型解耦，但节点嵌入业务对象 | 链表容器可持有不同类型的元素指针           |
| 额外内存开销 | 不需要单独的链表包装对象                   | 通常需要额外的节点对象                     |
| 适用场景     | 内核、数据库、内存管理、进程管理等底层代码 | 通用容器、业务代码、需要动态管理元素的场景 |

总结： PostgreSQL 的 `dlist_node` 不是一个完整的业务对象，而是一个可嵌入其他结构体的链表连接组件。两个指针足以维护双向连接，业务数据留在外层结构体中，而 `dlist_head` 利用哨兵节点简化链表操作。这种设计特别适合 PostgreSQL 这种大量使用 C 语言、强调性能和内存布局控制的系统。

---

# 结构体成员结合  offsetof() 反推结构体起始地址

> 因为 `node` 是 `Task` 的一个成员，所以可以通过成员偏移量反推出 `Task` 对象的起始地址。 解释一下这句话

你问的这句话，实际上涉及 C 语言中一个非常重要的知识点：已知结构体内部某个成员的地址，如何反推出整个结构体的起始地址？

这也是理解 PostgreSQL 侵入式链表的关键。我们从内存布局、`offsetof()` 的原理、指针运算，再到 PostgreSQL 源码中的实际应用，一步一步讲解。

## 一、先理解：结构体和成员在内存中是什么关系？

假设我们有以下 C 代码：

```
#include <stdio.h>

typedef struct Task
{
    int task_id;
    char task_name[32];
    dlist_node node;
} Task;
```

这里的 `Task` 是一个结构体类型，它包含三个成员：

- `task_id`：任务编号。
- `task_name`：任务名称。
- `node`：双向链表节点。

现在创建一个对象：

```
Task task;
```

编译器会为 `task` 分配一块足够大的连续内存，用于存储这个结构体的全部成员。

假设为了便于理解，这块内存的起始地址是 `1000`，内存布局如下。这里的地址是假设值，不是真实运行结果。

Task 对象的假设内存布局

task_id

### 地址 1000

占用 4 字节（假设）

task_name

### 地址 1004

占用 32 字节

node.prev

### 地址 1036

假设 node 从此处开始

node.next

### 地址 1044

假设指针占 8 字节

示例假设没有填充字节，且指针占 8 字节。实际布局由编译器的对齐规则决定。

注意一个关键事实：

`Task` 对象的起始地址是 1000，但它内部 `node` 成员的起始地址是 1036。

因此：

C

```
Task *task_ptr = &task;
dlist_node *node_ptr = &task.node;
```

假设地址符合上面的示例：

```
task_ptr = 1000
node_ptr = 1036
```

那么，既然我们知道 `node_ptr` 的地址，也知道 `node` 在 `Task` 中的偏移量，就可以反推出 `Task` 的起始地址。

公式就是：

结构体起始地址=成员地址−成员偏移量

在这个例子里：

1000=1036−36

接下来要解决两个问题：

1. 如何计算 `node` 的偏移量？
2. 如何根据偏移量和成员地址得到结构体指针？

## 二、`offsetof(Task, node)` 到底是什么？

C 标准库提供了一个宏：`offsetof`，定义在 `<stddef.h>` 中。

用法：

```c
offsetof(结构体类型, 成员名称)
```

例如：

```c
#include <stddef.h>

size_t offset = offsetof(Task, node);
```

这里得到的是：

`node` 成员相对于 `Task` 对象起始位置的字节偏移量。

假设：

```
Task 起始地址：1000
node 起始地址：1036
```

那么：

```c
offsetof(Task, node) == 36
```

这里的 `36` 不是内存地址，而是一个相对偏移量。

这两个概念必须区分：

| 表达式                 | 含义                         | 示例值 |
| ---------------------- | ---------------------------- | ------ |
| `&task`                | 整个对象的起始地址           | 1000   |
| `&task.node`           | `node` 成员的地址            | 1036   |
| `offsetof(Task, node)` | 成员相对于对象起始位置的偏移 | 36     |

### 1. 为什么不能直接假设偏移量？

因为 C 编译器需要满足结构体成员的对齐要求。

例如：

```c
struct Example
{
    char a;
    int b;
};
```

虽然 `a` 只占 1 字节，但 `b` 通常需要按照 4 字节边界对齐。

在常见的平台上，`b` 的偏移量可能是 4，而不是 1。

因此，正确的做法是使用：

```c
offsetof(struct Example, b)
```

而不是自己计算前面成员的大小并简单相加。

同样，`Task` 中 `node` 的真实偏移量应该由编译器布局规则决定，而不是硬编码成 36。

## 三、如何通过成员地址反推出结构体地址？

有了 `offsetof()`，反推地址就很直接了。

先看完整示例：

```c
#include <stdio.h>
#include <stddef.h>

typedef struct dlist_node
{
    struct dlist_node *prev;
    struct dlist_node *next;
} dlist_node;

typedef struct Task
{
    int task_id;
    char task_name[32];
    dlist_node node;
} Task;

int main(void)
{
    Task task = {
        .task_id = 42,
        .task_name = "demo"
    };

    Task *task_ptr = &task;
    dlist_node *node_ptr = &task.node;

    size_t offset = offsetof(Task, node);

    Task *owner = (Task *)(
        (char *)node_ptr - offset
    );

    printf("task       = %p\n", (void *)&task);
    printf("task_ptr   = %p\n", (void *)task_ptr);
    printf("node_ptr   = %p\n", (void *)node_ptr);
    printf("offset     = %zu\n", offset);
    printf("owner      = %p\n", (void *)owner);

    printf("task_id    = %d\n", owner->task_id);
    printf("task_name  = %s\n", owner->task_name);

    return 0;
}
```

可以在 Ubuntu 上保存为 `container_of_demo.c`，然后编译运行：

```bash
gcc -Wall -Wextra -std=c11 \
    container_of_demo.c -o container_of_demo

./container_of_demo
```

输出中的地址每次运行可能不同，但你会看到：

```
task       = 0x...
task_ptr   = 0x...
node_ptr   = 0x...
offset     = ...
owner      = 0x...
task_id    = 42
task_name  = demo
```

其中最重要的是：

```
task == task_ptr == owner
```

这三个表达式代表同一个 `Task` 对象的起始位置。

### 2. 逐步拆解关键代码

```c
Task *owner = (Task *)(
    (char *)node_ptr - offsetof(Task, node)
);
```

我们把它拆成三个步骤。

第一步：将 `node_ptr` 转换为 `char *`。

```
(char *)node_ptr
```

在 C 语言中，`char *` 指针运算以字节为单位。

例如：

```
char *p = (char *)node_ptr;
p - 1
```

表示向前移动 1 个字节。

如果直接对 `dlist_node *` 做减法：

```
node_ptr - 1
```

则表示向前移动 `sizeof(dlist_node)` 个字节，而不是 1 个字节。

因此，这里先转换成 `char *`，是为了方便按字节计算地址偏移。

第二步：减去成员偏移量。

```
(char *)node_ptr - offsetof(Task, node)
```

假设 `node_ptr` 指向地址 1036，而偏移量为 36：

```
1036 - 36 = 1000
```

这样就回到了 `Task` 对象的起始位置。

第三步：转换成 `Task *`。

```
(Task *)(...)
```

现在得到的是对象起始位置的地址。由于我们知道这个地址对应一个 `Task` 对象，因此可以将其转换成 `Task *`，再通过：

```
owner->task_id
owner->task_name
```

访问原来的成员。

这里有一个前提：`node_ptr` 必须确实指向某个有效 `Task` 对象的 `node` 成员。不能拿任意地址执行这个操作，否则可能产生未定义行为。

## 四、PostgreSQL 为什么需要这种技巧？

回到你最初看的代码：

```
typedef struct dlist_node dlist_node;

struct dlist_node
{
    dlist_node *prev;
    dlist_node *next;
};
```

PostgreSQL 希望 `dlist_node` 能够嵌入不同类型的结构体。

例如，某个结构体保存事务相关数据：

C

```
typedef struct TransactionInfo
{
    int transaction_id;
    dlist_node node;
} TransactionInfo;
```

另一个结构体保存缓存信息：

C

```
typedef struct CacheEntry
{
    int cache_id;
    char *value;
    dlist_node node;
} CacheEntry;
```

两个结构体都可以放进同一套双向链表操作逻辑中，因为链表操作只需要访问 `prev` 和 `next`。

但它们的数据布局不同：

```
TransactionInfo
├── transaction_id
└── node
    ├── prev
    └── next

CacheEntry
├── cache_id
├── value
└── node
    ├── prev
    └── next
```

如果链表节点额外保存：

C

```
void *data;
```

那么每个节点还需要维护一个指向业务对象的指针。

侵入式链表省去了这个额外指针，也不需要单独分配链表包装节点。

不过需要注意：侵入式链表并不是完全没有代价。业务结构体必须包含链表节点成员；而且，如果一个对象需要同时加入多个独立链表，通常就要在结构体中嵌入多个 `dlist_node` 成员。

## 五、这与 `container_of` 有什么关系？

你在 Linux 内核源码、PostgreSQL 或其他底层 C 代码中，都可能看到类似的写法。

Linux 内核广泛使用 `container_of` 这个宏。概念上，它做的就是我们刚才演示的事情：

C

```
container_of(ptr, type, member)
```

含义是：

- `ptr`：某个结构体成员的地址。
- `type`：这个成员所属的外层结构体类型。
- `member`：该成员在外层结构体中的名称。

例如：

C

```
Task *task = container_of(node_ptr, Task, node);
```

相当于：

C

```
Task *task = (Task *)(
    (char *)node_ptr - offsetof(Task, node)
);
```

实际项目中的宏可能会额外进行类型检查，具体实现也可能使用编译器扩展。

需要区分： `container_of` 是一种常见的实现模式，不代表 PostgreSQL 的每个版本都直接使用这个宏。你正在阅读的 PostgreSQL 源码，应该继续查看对应版本的链表头文件（通常是 `src/include/lib/ilist.h`），确认它实际使用的辅助宏或函数。

## 六、最后用一句话把原理串起来

当 PostgreSQL 遍历双向链表时，拿到的是 `dlist_node *`。这个指针指向某个业务结构体内部的成员，而不是独立的业务对象。

因为编译时已经知道成员在外层结构体中的偏移量，所以可以：

owner=node 地址−offsetof⁡(外层类型,node)\text{owner} = \text{node 地址} - \operatorname{offsetof}(\text{外层类型},\text{node})owner=node 地址−offsetof(外层类型,node)

由此恢复外层结构体的地址，再访问完整的业务数据。

这就是为什么 `dlist_node` 只需要两个指针：它只负责连接节点，而不需要知道节点属于哪一种业务对象。