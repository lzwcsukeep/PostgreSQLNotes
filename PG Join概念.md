本文解释Join 的概念。

------

# 一、什么是 Join？

简单来说：

> **Join 的作用就是把两张表按照某种条件关联起来，生成一张新的结果表。**

例如：

```sql
SELECT u.name, d.dept_name
FROM users u
JOIN dept d
ON u.deptid = d.id;
```

假设两张表：

## users

| id   | name  | deptid |
| ---- | ----- | ------ |
| 1    | Tom   | 10     |
| 2    | Jerry | 20     |
| 3    | Lucy  | 10     |

## dept

| id   | dept_name |
| ---- | --------- |
| 10   | Sales     |
| 20   | IT        |

Join 后得到

| name  | dept_name |
| ----- | --------- |
| Tom   | Sales     |
| Jerry | IT        |
| Lucy  | Sales     |

可以理解为：

```
users            dept

Tom 10      <---->   10 Sales

Jerry20     <---->   20 IT

Lucy10      <---->   10 Sales
```

Join 的目的就是

> **根据 Join 条件，把左右两张表的记录匹配起来。**

------

# 二、为什么数据库需要 Join？

因为数据库设计遵循**关系模型**。

例如

用户

```
users
------
id
name
deptid
```

部门

```
dept
-----
id
dept_name
```

不能写成

```
users

id
name
deptid
dept_name
```

否则

```
Sales
Sales
Sales
Sales
...
```

重复很多次。

因此数据库进行了规范化（Normalization）。

查询的时候，再把它们拼回来。

这个"拼回来"就是 Join。

------

# 三、Join 在 PostgreSQL 中是什么？

很多初学者认为 Join 是 SQL 语法。

其实不是。

```sql
JOIN
```

只是告诉优化器：

> 我需要把两张关系连接起来。

真正执行 Join 的，是执行器中的 Join 算子。

例如

```
Nested Loop
Hash Join
Merge Join
```

这些都是 **Plan Node（执行器算子）**。

例如

```
EXPLAIN
SELECT *
FROM users u
JOIN dept d
ON u.deptid=d.id;
```

可能看到

```
Hash Join
   Hash Cond:
      u.deptid = d.id

   -> Seq Scan users

   -> Hash
        -> Seq Scan dept
```

Plan Tree

```
           Hash Join
           /       \
    Seq Scan     Hash
                    \
                 Seq Scan
```

这里

```
Seq Scan
```

负责产生 tuple。

```
Hash
```

负责建立 Hash Table。

```
Hash Join
```

负责匹配。

------

# 四、Join 可以理解成什么？

可以理解成

> Join = 两个输入 + 一个匹配规则 + 一个输出

即

```
          Join

     左输入      右输入

        |
      Join Condition

        |

      输出Tuple
```

例如

```
users.deptid

      =

dept.id
```

就是 Join Condition。

------

# 五、Join 在执行器里的位置

PostgreSQL 的 Plan Tree 是一棵树。

例如

```
SELECT *
FROM users
JOIN dept
ON ...
WHERE age>20;
```

Plan

```
              Hash Join

          /                 \

Seq Scan(users)         Hash

                             \

                          Seq Scan(dept)
```

整个执行器就是

```
ExecProcNode()

↓

HashJoin

↓

左边要tuple？
↓

SeqScan(users)

↓

右边要tuple？
↓

Hash

↓

SeqScan(dept)
```

所以

Join 本身也是一个 Plan Node。

也有

```
ExecHashJoin()
```

函数。

------

# 六、Join 的核心问题

Join 本质只有一个问题：

> 左边的一条记录，如何快速找到右边匹配的记录？

不同算法回答这个问题的方法不同。

PostgreSQL 提供三种主要 Join 算法。

------

## ① Nested Loop Join（嵌套循环）

最容易理解。

思想：

```
左边每一行

    去扫描右边全部数据
```

例如

```
users

Tom
Jerry
Lucy
```

右边

```
Sales

IT
```

执行过程

```
Tom

    Sales ?

    IT ?

------------

Jerry

    Sales ?

    IT ?

------------

Lucy

    Sales ?

    IT ?
```

就是

```
for each left tuple

    for each right tuple

        if 满足条件

            输出
```

复杂度

```
O(N×M)
```

------

## ② Hash Join（PG 最常见）

思想：

不要一直扫描右边。

先建立 Hash Table。

例如

右边

```
dept

10 Sales

20 IT
```

建立

```
Hash Table

10 -> Sales

20 -> IT
```

然后扫描 users

```
Tom

deptid=10

Hash查找

↓

Sales
```

整个流程

```
Seq Scan dept

↓

Build Hash Table

↓

Seq Scan users

↓

Hash Lookup

↓

Output
```

复杂度接近

```
O(N+M)
```

因此 PostgreSQL 最喜欢 Hash Join。

------

## ③ Merge Join

适用于

```
左右都已经排序
```

例如

左边

```
10
20
30
40
```

右边

```
20
30
50
```

像归并排序一样。

```
10

20 ←→20

30 ←→30

40

50
```

只扫描一遍。

复杂度

```
O(N+M)
```

------

# 七、为什么 PostgreSQL 要有三种 Join？

因为没有一种算法适合所有情况。

例如

| 场景            | 最佳 Join   |
| --------------- | ----------- |
| 右表很小        | Nested Loop |
| 大部分普通 Join | Hash Join   |
| 已经排序        | Merge Join  |

优化器会估算

```
Cost
```

例如

```
Nested Loop

Cost = 10000

Hash Join

Cost = 100

Merge Join

Cost = 120
```

最后选择

```
Hash Join
```

因此

**Join 类型是优化器决定的，而不是 SQL 决定的。**

------

# 八、Join 在源码中的结构

Join 节点有统一的继承关系（通过结构体嵌套实现）：

```
Plan
  │
  ▼
Join
  │
  ├── NestLoop
  ├── HashJoin
  └── MergeJoin
```

执行器状态节点也对应：

```
PlanState
   │
   ▼
JoinState
   │
   ├── NestLoopState
   ├── HashJoinState
   └── MergeJoinState
```

因此执行时会调用不同的函数，例如：

```
ExecNestLoop()

ExecHashJoin()

ExecMergeJoin()
```

它们都遵循相同的 Volcano Model 接口：每调用一次，就返回一条满足 Join 条件的结果元组。

------

# 九、结合 Volcano Model 理解 Join（这是理解执行器的关键）

你之前已经学习过：

```c
ExecProcNode(node)
```

对于 Join 节点来说，它的执行逻辑可以概括为：

```
ExecHashJoin()
        │
        ├── 向左孩子要一条 tuple
        │       ↓
        │   ExecProcNode(left)
        │
        ├── 向右孩子要数据（可能先构建 Hash 表）
        │       ↓
        │   ExecProcNode(right)
        │
        ├── 判断是否满足 Join 条件
        │
        └── 如果匹配，返回一条新的 Join 后的 tuple
```

因此，**Join 自己并不保存数据，它只是不断从左右子节点"拉取（pull）"数据，按照 Join 算法进行匹配，再向父节点返回结果**。这正是 PostgreSQL Volcano Model 的体现：每个算子都通过 `ExecProcNode()` 向下游请求数据、向上游返回一条结果，整个执行器像流水线一样协同工作。

------

## 建议的学习顺序

结合你目前正在阅读 PostgreSQL 16.0 源码，建议按下面的顺序深入：

1. 理解 `Join`、`NestLoop`、`HashJoin`、`MergeJoin` 三个 Plan 结构体（定义在 `plannodes.h`）。
2. 理解对应的 `JoinState`、`HashJoinState` 等执行状态结构体（`execnodes.h`）。
3. 阅读 `ExecInitNestLoop()`、`ExecInitHashJoin()` 等初始化流程。
4. 重点阅读 `ExecHashJoin()`，观察它如何调用左右子节点的 `ExecProcNode()` 获取数据，这会把 Join 算法和 Volcano Model 自然联系起来。
5. 最后结合 `EXPLAIN (ANALYZE, VERBOSE)` 对照执行计划和源码，会更容易建立执行器整体框架。