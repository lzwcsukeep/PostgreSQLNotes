# #算子介绍

结合你最近在看的 PostgreSQL 源码（`PortalRun`、`ExecProcNode`、`TupleTableSlot`、Volcano Model），**执行器算子（Executor Operator）** 是理解 PostgreSQL 执行器最重要的概念。

------

# 一、什么是执行器算子（Operator）

一句话概括：

> **执行器算子就是执行计划（Plan Tree）中的一个节点，每个节点负责完成一种数据库操作。**

例如下面 SQL：

```sql
SELECT name
FROM student
WHERE age > 18;
```

优化器可能生成这样的执行计划：

```
Seq Scan
  Filter: age > 18
```

这里只有一个算子：

```
SeqScan
```

如果 SQL 复杂一点：

```sql
SELECT *
FROM student s
JOIN score sc
ON s.id = sc.sid;
```

计划可能变成

```
Hash Join
├── Seq Scan(student)
└── Hash
      └── Seq Scan(score)
```

这里就有四个算子：

```
Hash Join
Hash
Seq Scan(student)
Seq Scan(score)
```

所以：

> **Plan Tree 上的每一个 PlanNode，在执行阶段都会变成一个执行器算子（PlanState）。**

------

# 二、Plan 和 执行器算子的关系

优化器生成的是：

```
Plan Tree
```

例如：

```
HashJoin
    |
+---------+
|         |
Hash    SeqScan
 |
SeqScan
```

里面都是

```
Plan
HashJoin
Hash
SeqScan
```

执行器开始执行时：

```
ExecutorStart()
```

会把它们变成

```
HashJoinState
HashState
SeqScanState
```

即：

```
Plan
        ↓ ExecutorStart
PlanState
```

所以：

```
SeqScan Plan
        ↓
SeqScanState（真正工作的对象）
```

------

# 三、为什么叫算子（Operator）

因为数据库把整个查询拆成很多小操作。

例如

```
SELECT ...
FROM A
JOIN B
GROUP BY
ORDER BY
LIMIT
```

实际上对应很多算子：

```
Limit
 Sort
  Aggregate
    Hash Join
      SeqScan
      IndexScan
```

每个算子只干一件事情。

例如：

SeqScan：

> 我负责扫描表。

Hash：

> 我负责建立哈希表。

Hash Join：

> 我负责根据哈希表做 Join。

Sort：

> 我负责排序。

Limit：

> 我负责限制返回条数。

这就是 Operator（算子）的含义。

------

# 四、执行器算子的共同特点

PostgreSQL 所有算子都遵循 **Volcano Model**。

接口几乎统一：

```
Init
↓

ExecProcNode()

↓

End
```

对应源码：

```
ExecInitNode()
ExecProcNode()
ExecEndNode()
```

例如：

```
ExecInitSeqScan()

ExecSeqScan()

ExecEndSeqScan()
```

Hash Join：

```
ExecInitHashJoin()

ExecHashJoin()

ExecEndHashJoin()
```

所以所有算子都有一样的生命周期。

------

# 五、所有算子都实现 ExecProcNode()

这是整个 Executor 最核心的思想。

例如：

```
TupleTableSlot *ExecSeqScan(...)
TupleTableSlot *ExecHashJoin(...)
TupleTableSlot *ExecSort(...)
```

全部返回：

```
TupleTableSlot *
```

为什么？

因为 PostgreSQL 规定：

> **所有算子都只负责产生"下一条Tuple"。**

而不是一次返回全部结果。

------

# 六、算子之间如何协作

例如：

```
SELECT *
FROM A
JOIN B;
```

计划：

```
Hash Join
├── Seq Scan(A)
└── Hash
      └── Seq Scan(B)
```

执行时：

```
ExecHashJoin()
```

发现：

需要左表一条记录。

于是调用：

```
ExecProcNode(lefttree)
```

即：

```
ExecSeqScan(A)
```

得到：

```
TupleSlot(A1)
```

再需要右边：

```
ExecProcNode(righttree)
```

进入：

```
ExecHash()
```

Hash 又继续调用：

```
ExecSeqScan(B)
```

所以调用链就是：

```
HashJoin
    ↓
 Hash
    ↓
SeqScan
```

每个人都向自己的子节点：

> 给我下一条 Tuple。

------

# 七、真正执行时的数据流

假设：

```
student

1
2
3
score

1
2
```

执行过程：

```
HashJoin
```

↓

```
ExecProcNode(left)
```

↓

```
SeqScan(student)
```

返回：

```
student(1)
```

然后：

```
ExecProcNode(right)
```

↓

```
Hash
```

↓

```
SeqScan(score)
```

返回：

```
score(1)
```

Join 成功：

```
(student1, score1)
```

返回给：

```
PortalRun()
```

再返回客户端。

下一次：

```
ExecHashJoin()
```

再去要：

```
student(2)
```

继续。

整个过程一直重复。

------

# 八、所有算子组成一棵流水线

例如：

```
Limit
   |
 Sort
   |
Hash Join
 |      |
Scan   Scan
```

真正执行顺序并不是：

```
Scan
↓

Hash

↓

Join

↓

Sort

↓

Limit
```

而是：

```
Limit
  │
  ▼
Sort
  │
  ▼
HashJoin
  │
 ┌┴───────┐
 ▼        ▼
Scan    Scan
```

当 `Limit` 需要一条记录时，它向下调用 `Sort`；`Sort` 再向下调用 `HashJoin`；`HashJoin` 再向两个子节点要数据。最终最底层 `Scan` 读取一条记录，然后结果沿调用栈逐层返回。

也就是说：

```
Scan
↑
Hash
↑
Join
↑
Sort
↑
Limit
↑
Client
```

**调用方向是自顶向下，数据返回方向是自底向上。**

------

# 九、源码中的体现

你最近已经看到过：

```c
TupleTableSlot *
ExecProcNode(PlanState *node)
{
    return node->ExecProcNode(node);
}
```

其中：

```
PlanState
```

里面保存了：

```c
struct PlanState
{
    Plan       *plan;

    PlanState  *lefttree;

    PlanState  *righttree;

    TupleTableSlot *(*ExecProcNode)(PlanState *);
};
```

因此：

```
HashJoinState
```

执行时：

```c
ExecProcNode(node->lefttree);
```

实际上会调用：

```
ExecSeqScan()
```

如果：

```
lefttree
```

是：

```
SeqScanState
```

如果：

```
lefttree
```

是：

```
SortState
```

那么调用的就是：

```
ExecSort()
```

这种通过函数指针实现的多态，使执行器无需关心子节点的具体类型，只需要不断调用 `ExecProcNode()` 即可。

------

# 十、总结：执行器算子的本质

可以把 PostgreSQL 的执行器理解成一组遵循统一接口的"积木"：

```
            SQL
             │
             ▼
      Optimizer
             │
             ▼
         Plan Tree
             │
      ExecutorStart
             ▼
      PlanState Tree
             │
        ExecProcNode()
             ▼
      一条 Tuple 一条 Tuple 地返回
             │
             ▼
          PortalRun
             ▼
           Client
```

每个**执行器算子（PlanState）**都只负责一种操作（扫描、连接、排序、聚合等），并实现统一的 `ExecProcNode()` 接口。它需要结果时向子算子"拉取"一条 `TupleTableSlot`，加工后再向父算子返回一条 `TupleTableSlot`。这种**统一接口 + 拉模型（Pull Model）**就是 PostgreSQL 执行器采用的 **Volcano Model** 核心思想。

# #火山模型实现

> 拉取和返回体现在哪里？ 从代码的角度证明

这是 PostgreSQL Executor 最核心的一点。**"拉取（Pull）"和"返回（Return）"并不是一句理论，而是直接体现在 `ExecProcNode()` 的调用方式上。** 我们从源码一步一步证明。

------

# 一、首先看统一入口：ExecProcNode()

你之前已经看过这个函数（`src/include/executor/executor.h`）：

```c
static inline TupleTableSlot *
ExecProcNode(PlanState *node)
{
    if (node->chgParam != NULL)
        ExecReScan(node);

    return node->ExecProcNode(node);
}
```

这里实际上做了两件事：

1. 调用当前节点自己的执行函数

```c
node->ExecProcNode(node);
```

例如：

```
SeqScanState
    ↓
ExecSeqScan()

HashJoinState
    ↓
ExecHashJoin()

SortState
    ↓
ExecSort()
```

所以：

> **每一次 ExecProcNode() 都是在问当前算子："给我下一条 Tuple。"**

注意：

这里没有循环。

没有数组。

没有一次返回全部数据。

只有：

```c
return ...
```

说明：

> **一次调用，只返回一条 Tuple。**

------

# 二、拉取(Pull)体现在哪里？

假设执行计划：

```
Hash Join
├── Seq Scan(A)
└── Seq Scan(B)
```

看看 `ExecHashJoin()`。

源码（`nodeHashjoin.c`）里面大量出现：

```c
outerTupleSlot = ExecProcNode(outerNode);
```

这里：

```c
outerNode = outerPlanState(node);
```

就是：

```
HashJoinState
      |
      +------lefttree
```

于是：

```c
outerTupleSlot =
    ExecProcNode(lefttree);
```

是不是 HashJoin 主动调用了左孩子？

也就是说：

```
HashJoin
      │
      ▼
ExecProcNode(lefttree)
```

HashJoin 在说：

> **"左孩子，给我一条Tuple。"**

这就是：

> **Pull（拉取）**

不是：

```
SeqScan
主动发送
```

而是：

```
HashJoin
主动要
```

------

## 再看看右孩子

同样：

```c
hashtable = ExecHashTableCreate(hashNode);

slot = ExecProcNode(innerNode);
```

又出现：

```c
ExecProcNode(innerNode)
```

说明：

```
HashJoin
      │
      ▼
Hash
      │
      ▼
SeqScan
```

仍然是：

父节点：

> 给我数据。

不是：

子节点：

> 我给你数据。

所以：

**谁调用 ExecProcNode()，谁就是在 Pull。**

------

# 三、返回(Return)体现在哪里？

继续看 SeqScan。

例如：

```c
TupleTableSlot *
ExecSeqScan(...)
{
    ...
    return ExecScan(...);
}
```

继续：

```c
ExecScan(...)
```

最后：

```c
return slot;
```

是不是：

```
SeqScan
    │
    ▼
return slot
```

回到哪里？

回到：

```c
outerTupleSlot =
    ExecProcNode(lefttree);
```

于是：

```
SeqScan
   │
return slot
   │
   ▼
HashJoin
```

注意：

不是：

```
HashJoin
去内存里面找slot
```

而是：

```
HashJoin调用

↓

ExecProcNode()

↓

SeqScan return
```

这就是：

**Return。**

------

# 四、为什么说它是"拉模型"？

看整个调用栈。

例如：

```
Limit
  |
Sort
  |
HashJoin
  |
SeqScan
```

客户端需要一条数据。

PortalRun()

调用：

```
ExecProcNode(limit)
```

进入：

```
ExecLimit()
```

发现：

没有Tuple。

于是：

```c
slot = ExecProcNode(outerPlanState(node));
```

进入：

```
ExecSort()
```

Sort发现：

没有排序结果。

于是：

```c
slot = ExecProcNode(outerPlanState(node));
```

进入：

```
ExecHashJoin()
```

HashJoin发现：

需要左表。

于是：

```c
slot =
ExecProcNode(lefttree);
```

进入：

```
ExecSeqScan()
```

SeqScan终于：

```c
return slot;
```

于是调用栈开始返回：

```
SeqScan

↓

HashJoin

↓

Sort

↓

Limit

↓

PortalRun

↓

Client
```

所以：

调用方向：

```
Limit

↓

Sort

↓

Join

↓

Scan
```

返回方向：

```
Scan

↑

Join

↑

Sort

↑

Limit
```

完全就是函数调用栈。

------

# 五、真正的代码证明

假设 HashJoin：

```c
TupleTableSlot *
ExecHashJoin(...)
{
    TupleTableSlot *outerSlot;

    outerSlot =
        ExecProcNode(outerPlanState(node));

    ...
}
```

这里：

```
ExecProcNode(...)
```

就是：

> 我需要数据。

然后：

SeqScan：

```c
TupleTableSlot *
ExecSeqScan(...)
{
    ...

    return slot;
}
```

HashJoin继续：

```c
outerSlot
```

已经拿到了。

说明：

```
HashJoin

↓

ExecProcNode()

↓

SeqScan

↓

return slot

↓

HashJoin
```

是不是和普通C函数完全一样？

例如：

```c
int add()
{
    return 3;
}

int main()
{
    int x = add();
}
```

对应Executor：

```c
TupleTableSlot *
ExecSeqScan(...)
{
    return slot;
}

TupleTableSlot *
ExecHashJoin(...)
{
    slot =
        ExecProcNode(lefttree);
}
```

所以：

> **Executor 本质就是利用 C 语言函数调用机制实现了 Volcano Model。**

------

# 六、为什么不是 Push（推模型）？

如果是 Push 模型（例如很多流式计算框架），代码通常会是这样：

```c
ExecSeqScan(...)
{
    while (...)
    {
        slot = ...

        ExecHashJoin(slot);
    }
}
```

即：

```
SeqScan

↓

HashJoin(slot)
```

数据是**子节点主动推给父节点**。

而 PostgreSQL 从来不是这样。

它永远是：

```c
HashJoin
{
    slot =
        ExecProcNode(lefttree);
}
```

因此：

```
HashJoin

↓

SeqScan
```

是：

> **父节点主动向子节点要数据（Pull）**。

------

# 七、从 `PortalRun` 到 `SeqScan` 的完整调用链

把你最近分析的调用流程串起来，可以更直观地看到整个"拉取—返回"过程：

```text
PortalRun()
    │
    ▼
ExecProcNode(LimitState)
    │
    ▼
ExecLimit()
    │
    ├─ slot = ExecProcNode(SortState)      ← 拉取
    │
    ▼
ExecSort()
    │
    ├─ slot = ExecProcNode(HashJoinState)  ← 拉取
    │
    ▼
ExecHashJoin()
    │
    ├─ slot = ExecProcNode(SeqScanState)   ← 拉取
    │
    ▼
ExecSeqScan()
    │
    ├─ 从 Heap 读取一条记录
    └─ return slot                         ← 返回
          ▲
          │
HashJoin 得到 slot，完成 Join，return 新 slot
          ▲
          │
Sort 得到 slot，排序后 return slot
          ▲
          │
Limit 得到 slot，检查 LIMIT 后 return slot
          ▲
          │
PortalRun 得到 slot，发送给客户端
```

整个过程中，**每一层都是通过 `ExecProcNode(child)` 向下拉取数据，再通过 C 函数 `return` 把 `TupleTableSlot *` 一层层返回给调用者**。这就是 PostgreSQL Volcano（Pull）执行模型最直接、最有力的源码证明。

# #ExecProcNode 语义

> ```c
> static inline TupleTableSlot *
> ExecProcNode(PlanState *node)
> {
>     if (node->chgParam != NULL)
>         ExecReScan(node);
> 
>     return node->ExecProcNode(node);
> }
> ```
>
> 上面这个ExecProcNode函数就是执行算子 node, 返回一条 tuple ? 然后谁调用ExecProcNode 就返回给谁？因此调用动作就是拉取，ExecProcNode 执行完毕返回就是子节点推给父节点？

你的理解**已经非常接近了**，但最后一句 "**返回就是子节点推给父节点**" 容易把 Pull 和 Push 混淆，我建议稍微修正一下概念。

# 先看这个函数

```c
static inline TupleTableSlot *
ExecProcNode(PlanState *node)
{
    if (node->chgParam != NULL)
        ExecReScan(node);

    return node->ExecProcNode(node);
}
```

它实际上可以理解成一句话：

> **执行当前算子，获取一条 Tuple。**

例如：

```c
ExecProcNode(seqScanState);
```

实际调用的是

```c
ExecSeqScan(seqScanState);
```

返回

```c
TupleTableSlot *
```

如果

```c
ExecProcNode(hashJoinState);
```

实际调用的是

```c
ExecHashJoin(hashJoinState);
```

返回也是

```c
TupleTableSlot *
```

所以：

> **ExecProcNode() 的语义就是："给我一条 Tuple"。**

------

# 谁调用 ExecProcNode，就返回给谁？

完全正确。

例如：

```c
TupleTableSlot *slot;

slot = ExecProcNode(lefttree);
```

执行过程就是

```
HashJoin
    │
    │ 调用
    ▼
ExecProcNode(lefttree)
    │
    ▼
ExecSeqScan()
    │
    ▼
return slot
    │
    ▲
HashJoin 得到 slot
```

所以：

> **C 语言函数 return 天然保证了：返回给调用者。**

没有什么消息队列。

没有什么通知机制。

就是普通函数调用。

------

# 为什么说这是 Pull？

重点就在于：

是谁主动发起调用？

例如

```c
slot = ExecProcNode(lefttree);
```

不是

```
SeqScan：
"我给你一条数据吧。"
```

而是

```
HashJoin：
"左孩子，给我一条数据。"
```

主动的是：

```
HashJoin
```

所以叫

> **Pull（拉）**

真正的动作是：

```
HashJoin
    │
    │ 我要
    ▼
SeqScan
```

------

# return 算 Push 吗？

这里很多人第一次学 Executor 都容易误会。

严格来说：

**不是。**

因为 Push Model 在数据库里有专门的定义。

Push Model 指的是：

```
SeqScan
{
    while (...)
    {
        slot = ...

        HashJoin(slot);   // 主动送过去
    }
}
```

注意这里：

父节点没有请求。

子节点自己决定：

> "我产生了一条数据，我现在推给你。"

这才叫 Push。

------

而 PostgreSQL 完全不是。

PostgreSQL 是：

```
HashJoin
{
    slot =
        ExecProcNode(lefttree);
}
```

意思是：

```
HashJoin：
"给我一条。"
```

然后

```
SeqScan
return slot;
```

return 只是：

> **函数调用结束，把结果交还给调用者。**

**它不是 Push Model 意义上的"推送数据"。**

------

# 所以更准确的描述应该是：

你可以把整个过程理解成：

```
HashJoin
    │
    │ Pull
    ▼
ExecProcNode(lefttree)
    │
    ▼
ExecSeqScan
    │
    │ 读取一条Tuple
    ▼
return slot
    │
    ▲
HashJoin继续执行
```

这里只有一个 Pull。

return 只是：

> **Pull 请求的响应（response）**

不是新的 Push。

------

# 用生活中的例子

假设：

老板（HashJoin）

员工（SeqScan）

老板说：

> "给我一份报表。"

员工去写：

```
return report;
```

这里：

老板主动发起请求。

员工只是回应请求。

没人会说：

> 员工主动推送了一份报表。

因为：

如果老板没要，

员工根本不会干。

这就是 Pull。

------

# 从源码角度可以总结成一句经典的话

可以把 `ExecProcNode()` 看成 Executor 的统一接口：

```c
TupleTableSlot *
ExecProcNode(PlanState *node)
```

它的语义就是：

> **"向这个执行器节点请求（Pull）下一条 Tuple，如果有，则通过函数返回值（return）把这条 Tuple 返回给调用者；如果没有，则返回一个空的 `TupleTableSlot`（通常通过 `TupIsNull(slot)` 判断）。"**

因此：

- **Pull**：体现在**调用** `ExecProcNode(child)`，父节点主动向子节点请求数据。
- **Return**：体现在 C 函数 `return slot`，只是对这次请求的正常响应。

这也是 PostgreSQL 的 Volcano Executor 被称为 **Pull-based Iterator Model（基于拉取的迭代器模型）** 的根本原因。



# 链接

https://chatgpt.com/c/6a55f7d2-3818-83ea-98fd-b41a679bdda2