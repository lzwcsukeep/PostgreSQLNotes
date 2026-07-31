本文解释 postgresql 执行计划里 pipeline 以及 pipeline breaker的概念。

这是理解 PostgreSQL 执行器（Executor）和各种 Join 算法最重要的概念之一。

实际上，**Pipeline（流水线）** 并不是 PostgreSQL 独有，而是所有 Volcano Model（火山模型）数据库（PostgreSQL、Oracle、MySQL、SQL Server 等）共同的概念。

------

# 一、什么叫 Pipeline？

一句话来说：

> **Pipeline 就是数据可以一条一条地向上传递，而不需要等子节点全部执行完成。**

假设有 SQL：

```sql
SELECT *
FROM users
WHERE age > 18;
```

执行计划

```
Seq Scan
   │
Filter(age>18)
```

实际上执行器是这样的：

```
ExecProcNode(Filter)
      │
      ▼
ExecProcNode(SeqScan)
```

调用过程：

```
Filter
   │
   ├── 我要一条数据
   ▼
SeqScan
   │
   ├── 从磁盘读第一页
   ├── 返回第一条tuple
   ▲
Filter
   │
   ├── 判断 age>18 ?
   ├── 是
   ▼
返回给客户端
```

注意：

**这里只有一条 tuple 在流动。**

并没有：

```
Seq Scan
    ↓
全部扫描完
    ↓
Filter
```

而是

```
tuple1 →
tuple2 →
tuple3 →
tuple4 →
...
```

这就是 Pipeline。

------

# 二、为什么叫 Pipeline（流水线）

因为整个执行器就像工厂流水线。

例如：

```
原料
 │
 ▼
切割
 │
 ▼
打磨
 │
 ▼
喷漆
 │
 ▼
包装
```

不用等所有零件切完再打磨。

而是：

```
零件1
切割
↓
打磨
↓
喷漆
↓
包装

零件2
切割
↓
打磨
↓
喷漆
↓
包装
```

数据库完全一样。

例如：

```
Seq Scan
    │
Filter
    │
Projection
    │
Aggregate
```

每得到一条 tuple：

```
Scan
 ↓
Filter
 ↓
Project
 ↓
Output
```

马上就可以输出。

所以：

**Pipeline = Tuple-at-a-time Processing**

------

# 三、Volcano Model 为什么天然支持 Pipeline？

看看 PG 最核心的函数：

```c
TupleTableSlot *
ExecProcNode(PlanState *node)
{
    return node->ExecProcNode(node);
}
```

所有算子都是：

```
我要一条
```

而不是：

```
我要全部
```

例如：

```
Sort
```

调用：

```
Sort
   │
   ▼
Seq Scan
```

Seq Scan 返回：

```
tuple1
```

Sort 收到。

再次调用：

```
tuple2
```

再次：

```
tuple3
```

...

每次都是：

```
next tuple
```

所以 Pipeline 天然存在。

------

# 四、Pipeline 的优点

例如：

```
100GB 数据
```

如果不是 Pipeline：

```
Scan全部
↓

Filter全部
↓

Join全部
↓

Output
```

需要：

```
100GB
+
100GB
+
100GB
```

大量中间结果。

Pipeline：

```
tuple1
↓

Filter

↓

Join

↓

Output
```

几乎没有中间结果。

因此：

- 内存占用低
- 延迟低
- 可以边扫描边输出
- CPU Cache 更友好

------

# 五、什么时候 Pipeline 会断？

这就涉及：

> **Pipeline Breaker**

Pipeline Breaker：

**必须等所有输入数据收集完成之后，才能继续往上输出。**

例如：

```
Seq Scan
    │
 Sort
    │
 Output
```

Sort 不可能：

```
收到 tuple1

↓

输出 tuple1
```

为什么？

因为：

```
tuple100
```

可能更小。

必须：

```
全部读完
```

才能：

```
排序
```

然后：

```
输出
```

所以：

```
Seq Scan

↓↓↓↓↓↓↓↓↓

Sort缓存全部

↓↓↓↓↓↓↓↓↓

Output
```

Pipeline 被打断。

所以：

```
Sort
```

就是 Pipeline Breaker。

------

# 六、Pipeline Breaker 的定义

定义非常简单：

> **必须消费（consume）所有输入，才能产生第一条输出。**

也就是：

```
Input

↓

全部读完

↓

开始输出
```

这种节点就是：

```
Blocking Operator
```

也叫：

```
Pipeline Breaker
```

------

# 七、哪些算子是 Pipeline Breaker？

## ① Sort（最典型）

```
Seq Scan

↓

Sort

↓

Output
```

必须：

```
全部扫描
↓

排序
↓

输出
```

------

## ② Hash Join（Build Side）

例如：

```
users
JOIN dept
```

假设：

```
Hash Join

      /      \
 users     dept
```

PG 会先：

```
dept

↓

全部扫描

↓

建立Hash Table
```

之后：

```
users

↓

一边扫描

↓

Probe Hash
```

因此：

```
dept
```

这一侧：

Pipeline 被打断。

但是：

```
users
```

这一侧：

还是 Pipeline。

所以 Hash Join 是：

> **半个 Pipeline Breaker（只阻塞 Build Side）。**

例如：

```
dept

↓

tuple1

↓

Hash

↓

tuple2

↓

Hash

↓

tuple3

↓

Hash

↓

全部完成

↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓

开始 Probe
```

------

## ③ Hash Aggregate

例如：

```sql
SELECT deptid,count(*)
FROM users
GROUP BY deptid;
```

HashAgg：

```
Seq Scan

↓

HashAgg
```

需要：

```
全部扫描

↓

Hash Table统计

↓

输出结果
```

必须等所有数据处理完成才能输出最终聚合结果，因此通常也是 Pipeline Breaker。（某些特殊聚合在其他数据库可能支持流式输出，但 PostgreSQL 的 HashAgg 一般需要先完成聚合。）

------

## ④ Materialize

```
Seq Scan

↓

Materialize
```

Materialize：

```
全部缓存

↓

以后重复读取
```

也是 Breaker（虽然在 PostgreSQL 中 Materialize 有时会边读边缓存，并不总是严格阻塞，但它的作用就是切断上游重复扫描，因此通常被视为 Pipeline 的边界）。

------

## ⑤ CTE Materialize

例如：

```sql
WITH t AS (...)
```

如果需要 Materialize：

```
CTE

↓

全部执行

↓

缓存

↓

后续引用
```

也是 Breaker。

------

# 八、哪些算子不是 Pipeline Breaker？

例如：

## Seq Scan

```
读一条

↓

返回一条
```

------

## Index Scan

```
找到一条

↓

返回一条
```

------

## Filter

```
收到一条

↓

判断

↓

返回
```

------

## Projection

```
收到

↓

计算表达式

↓

返回
```

------

## Nested Loop Join

```
Outer tuple

↓

Inner Scan

↓

Match

↓

立即返回
```

虽然每个外表元组都会触发一次内表扫描，但它不需要等整个外表或内表处理完才开始产生结果，因此仍然属于流式执行。

------

## Merge Join

很多人容易误解。

Merge Join：

```
Index Scan

↓

Merge Join

↓

Output
```

如果：

```
左右输入已经有序
```

Merge Join：

```
左一条

右一条

↓

比较

↓

输出
```

可以一直输出。

**Merge Join 自己不是 Pipeline Breaker。**

真正 Break 的往往是：

```
Seq Scan

↓

Sort

↓

Merge Join
```

因为：

```
Sort
```

才是真正 Blocking。

------

# 九、一张图理解 Pipeline

没有 Breaker：

```
 SeqScan
     │
 Filter
     │
 Project
     │
 Output
```

数据流：

```
tuple1 ─────────►

tuple2 ─────────►

tuple3 ─────────►
```

一直流动。

------

有 Breaker：

```
 SeqScan
     │
    Sort
     │
 Filter
     │
 Output
```

执行过程：

```
tuple1
tuple2
tuple3
tuple4
tuple5
...
        ↓

     Sort缓存

        ↓

排序完成

        ↓

tuple1
tuple2
tuple3
...
```

可以看到：

**Sort 前后形成了两个独立的 Pipeline。**

------

# 十、在 PostgreSQL 中如何理解 Pipeline

如果把你最近研究的 `ExecProcNode()` 放进来，可以把整个执行器看成由多个 Pipeline 组成。

```
ExecProcNode(Output)
          │
          ▼
ExecProcNode(Filter)
          │
          ▼
ExecProcNode(Hash Join)
       ┌─────────────┐
       │             │
   Probe Pipeline   Build Phase（先完成）
       │             │
       ▼             ▼
   SeqScan(users) SeqScan(dept)
```

这里可以清楚地看到：

- **左侧（Probe）**：数据一条一条流动，属于一个 Pipeline。
- **右侧（Build）**：必须先把 `dept` 全部扫描并建立哈希表，形成一个 Pipeline Break。
- **Hash Join 节点**：在 Build 完成后，重新开始新的 Pipeline，继续向上传递结果。

因此，从 PostgreSQL 执行器的角度，可以把 **Pipeline 理解为一条连续的 `ExecProcNode()` 调用链**；而 **Pipeline Breaker** 就是那些必须先消费完全部（或某一侧全部）输入，才能开始产生输出的算子，它们把一条调用链切分成多个独立的执行阶段。