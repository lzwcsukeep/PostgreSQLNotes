# Sort 算子行为

之前的文章提到 执行计划里有 pipeline 和 pipeline breaker 的概念，Sort 算子就是一个 pipeline breaker。

本文先解释 Sort 算子行为，然后从源码角度分析验证这种行为。

首先回答：sort 算子的父节点向 sort 请求下一个tuple 会发生什么？

答案是：

> **Sort 的父节点第一次调用 `ExecProcNode(Sort)` 时，Sort 并不会立即返回 tuple，而是会先完成整个排序过程。排序完成后，才返回第一条 tuple。**

这个问题和答案也是理解 **Volcano Model 如何处理 Pipeline Breaker** 的关键。

我们来看 PostgreSQL 的实际执行流程。

------

## 假设执行计划

```text
          Result
             │
           Sort
             │
         Seq Scan
```

父节点（Result）需要数据：

```c
ExecProcNode(Result)
```

Result 会调用：

```c
ExecProcNode(Sort)
```

------

## 第一次进入 Sort

实际上会进入：

```c
ExecSort()
```

（位于 `nodeSort.c`）

它的大致逻辑可以简化为：

```c
if (!sort_Done)
{
    // 第一次进入

    while ((slot = ExecProcNode(outerPlan)) != NULL)
    {
        tuplesort_puttupleslot(...);
    }

    tuplesort_performsort(...);

    sort_Done = true;
}

return tuplesort_gettupleslot(...);
```

注意第一件事：

**并不是立即返回 tuple。**

而是：

```text
父节点
    │
    ▼
ExecSort()

    │
    ├── 我要所有tuple
    │
    ▼
Seq Scan
```

于是：

------

Seq Scan 开始不断返回

```text
tuple1

↓

Sort缓存
```

继续

```text
tuple2

↓

Sort缓存
```

继续

```text
tuple3

↓

Sort缓存
```

……

直到

```text
NULL
```

表示：

```text
Seq Scan结束
```

------

## 然后开始排序

Sort 调用

```c
tuplesort_performsort()
```

这一步：

可能是

```
内存排序
```

或者

```
外部归并排序
```

完全由 tuplesort 模块负责。

------

## 排序结束以后

现在 Sort 内部已经有

```text
1

2

3

4

5
```

于是：

第一次调用

```c
ExecSort()
```

终于可以

```c
tuplesort_gettupleslot()
```

返回

```text
tuple1
```

注意：

**这是父节点第一次收到 tuple。**

虽然它已经等了很久。

------

## 第二次父节点调用

父节点再次：

```c
ExecProcNode(Sort)
```

这时候：

```text
sort_Done == true
```

所以：

不会再扫描 Seq Scan。

直接：

```c
tuplesort_gettupleslot()
```

返回

```text
tuple2
```

第三次：

```text
tuple3
```

……

直到

```text
NULL
```

------

## 整个调用栈是什么样？

第一次调用：

```text
Result

↓

Sort

↓

Seq Scan

↓

tuple1

↓

Sort缓存

↓

Seq Scan

↓

tuple2

↓

Sort缓存

↓

...

↓

NULL

↓

Sort开始排序

↓

返回tuple1

↓

Result收到tuple1
```

所以：

**虽然父节点只调用了一次 `ExecProcNode(Sort)`，但这一次调用在内部递归调用了成千上万次 `ExecProcNode(SeqScan)`。**

------

## 为什么父节点感觉不到？

这是 Volcano Model 最巧妙的地方。

父节点代码可能只是：

```c
while ((slot = ExecProcNode(child)) != NULL)
{
    ...
}
```

它根本不知道：

第一次

```c
ExecProcNode(child)
```

内部到底干了什么。

可能：

Seq Scan：

```text
1ms
```

返回。

也可能：

Sort：

```text
扫描100GB

↓

排序

↓

10分钟后

↓

返回tuple1
```

父节点完全不知道。

它只知道：

```text
给我下一条tuple
```

所以：

**Pipeline Breaker 对父节点来说是透明的。**

------

## 用一个生活中的比喻

假设你去餐厅点菜。

普通算子（Filter、Projection、Seq Scan）像快餐窗口：

```text
你：

来一份。

↓

马上给你。
```

而 Sort 更像定制蛋糕：

```text
你：

来一份。

↓

厨师：

等等。

↓

准备全部材料

↓

烘焙

↓

装饰

↓

完成

↓

给你第一块。
```

之后你说：

```text
再来一块。
```

厨师直接切：

```text
第二块
```

不用重新做蛋糕。

------

## 结合 PostgreSQL 源码再看

如果你接下来阅读 `src/backend/executor/nodeSort.c`，建议重点关注这几个函数：

```c
ExecInitSort()      // 初始化 SortState
ExecSort()          // 第一次调用时完成"读完输入 + 排序"，之后逐条返回
ExecEndSort()       // 释放 tuplesort 等资源
ExecReScanSort()    // 重新扫描时决定是否重新排序或复用结果
```

其中，`ExecSort()` 正是 **Pipeline Breaker** 行为的具体实现：**第一次调用时不急着向父节点返回数据，而是先把下层所有 tuple 消费完并完成排序，然后才开始逐条返回结果。**

理解了这一点，你再去看 `nodeHash.c` 的 `ExecHash()`，会发现它和 `ExecSort()` 非常相似：第一次调用时都会先完成一个耗时的准备阶段（Build Hash Table），然后后续调用才进入正常的逐条返回阶段。这也是为什么我们把它们都称为 **Blocking Operator（Pipeline Breaker）**。

# Sort 算子源码分析

### 先重复Sort算子行为

PostgreSQL 的 `Sort` 必须先把下层所有 tuple 全部读完。然后完成排序（内存排序或外部归并排序）。

之后才能向父节点返回第一条 tuple。所以它不是流式（streaming）的。而是一个真正的 Pipeline Breaker。

### 源码里是怎么实现的？

核心文件：

```
src/backend/executor/nodeSort.c
```

核心函数：

```
static TupleTableSlot *
ExecSort(PlanState *pstate)
```

它的逻辑（简化后）大概是：

```c
static TupleTableSlot *
ExecSort(PlanState *pstate)
{
	SortState  *node = castNode(SortState, pstate);
	/*
	 * If first time through, read all tuples from outer plan and pass them to
	 * tuplesort.c. Subsequent calls just fetch tuples from tuplesort.
	 */
	if (!node->sort_Done)
	{

		if (node->datumSort)
		{
			for (;;)
			{
				slot = ExecProcNode(outerNode);

				if (TupIsNull(slot))
					break;
				slot_getsomeattrs(slot, 1);
				tuplesort_putdatum(tuplesortstate,
								   slot->tts_values[0],
								   slot->tts_isnull[0]);
			}
		}
		else
		{
			for (;;)
			{
				slot = ExecProcNode(outerNode);

				if (TupIsNull(slot))
					break;
				tuplesort_puttupleslot(tuplesortstate, slot);
			}
		}

		/*
		 * Complete the sort.
		 */
		tuplesort_performsort(tuplesortstate);
		node->sort_Done = true;
	}

	if (node->datumSort)
	{
        ...
	}
	else
		(void) tuplesort_gettupleslot(tuplesortstate,
									  ScanDirectionIsForward(dir),
									  false, slot, NULL);

	return slot;
}
```

### 注意这个代码里的 for (;;)

```
			for (;;)
			{
				slot = ExecProcNode(outerNode);

				if (TupIsNull(slot))
					break;
				tuplesort_puttupleslot(tuplesortstate, slot);
			}
```

这意味着：

一次父节点调用

```
ExecProcNode(Sort)
```

内部会发生

```
ExecProcNode(SeqScan)
ExecProcNode(SeqScan)
ExecProcNode(SeqScan)
...
ExecProcNode(SeqScan)
```

直到：

```
NULL
```

也就是：

把下层全部耗尽（consume all input）

再排序

再返回第一条

### 为什么 Sort 天生没法做 Pipeline？

假设输入是：

```
5
1
3
2
```

如果你刚读到：

```
5
```

你敢输出吗？

不敢。

因为后面可能还有：

```
1
```

所以：

必须看到全部输入

才能确定最小值是谁

才能输出第一条

这是排序问题本身决定的，不是 PostgreSQL 特有的。

### 执行时会发生什么？

假设：

```
SELECT * FROM users ORDER BY age;
```

计划：

```
Sort
  - Seq Scan on users
```

父节点第一次：

```
ExecProcNode(Sort)
```

Sort 内部：

```
SeqScan -> tuple1
SeqScan -> tuple2
SeqScan -> tuple3
...
SeqScan -> tupleN
SeqScan -> NULL
```

然后：

```
tuplesort_performsort()
```

可能：

- 快速排序（内存够）
- 外部归并排序（超过 work_mem）

排序完成后：

```
return smallest_tuple;
```

于是父节点终于得到：

```
第一条结果
```

### 所以 Pipeline 在哪里断了？

原本：

```
SeqScan -> Filter -> Output
```

是连续流：

```
tuple1 ---------->
tuple2 ---------->
```

加入 Sort：

```
SeqScan -> Sort -> Output
```

变成：

```
tuple1
tuple2
tuple3
...
tupleN

   ↓ 全部缓存

Sort

   ↓

sorted1 ---->
sorted2 ---->
sorted3 ---->
```

这里就是 Pipeline Break。

### 一个更准确的说法

你可以把 PostgreSQL 的执行器看成：

```
[Pipeline 1]
SeqScan
   ↓
Filter

   ↓  (Pipeline Break)

Sort

   ↓

[Pipeline 2]
Output
```

也就是说：

Sort 前面是一条 Pipeline

Sort 后面是另一条 Pipeline

Sort 把两条 Pipeline 切开了

### 和 Merge Join 的关系（非常重要）

很多人以为：

```
Merge Join
```

是 Breaker。

其实通常不是。

真正 Break 的是：

```
SeqScan
   ↓
Sort
   ↓
Merge Join
```

因为 Merge Join 需要有序输入，而 Sort 为了产生有序输入，必须先阻塞。

### 一句话记忆

PostgreSQL 的 Sort 算子是一个真正的 Pipeline Breaker。

它的执行模式是：

```
consume all input
        ↓
sort
        ↓
produce output
```

而不是：

```
consume one
        ↓
produce one
```

这也是你在阅读 `ExecSort()` 时看到那个`for(;;)`循环的根本原因。