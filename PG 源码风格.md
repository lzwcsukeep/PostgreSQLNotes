问题：

```c
typedef struct TupleDescData
{
	int			natts;			/* number of attributes in the tuple */
	Oid			tdtypeid;		/* composite type ID for tuple type */
	int32		tdtypmod;		/* typmod for tuple type */
	int			tdrefcount;		/* reference count, or -1 if not counting */
	TupleConstr *constr;		/* constraints, or NULL if none */
	/* attrs[N] is the description of Attribute Number N+1 */
	FormData_pg_attribute attrs[FLEXIBLE_ARRAY_MEMBER];
}			TupleDescData;
typedef struct TupleDescData *TupleDesc;
```

为什么 PG 喜欢`typedef struct xxxData * xxx` 这种写法？

---

这是 PostgreSQL 源码中一种**非常典型的设计风格**。你会发现大量这样的定义：

```c
typedef struct TupleDescData *TupleDesc;
typedef struct RelationData *Relation;
typedef struct HeapTupleData *HeapTuple;
typedef struct EState *EState;
typedef struct ExprContext *ExprContext;
typedef struct MemoryContextData *MemoryContext;
typedef struct SnapshotData *Snapshot;
```

也就是说：

```c
struct XxxData {
    ...
};

typedef struct XxxData *Xxx;
```

而不是

```c
typedef struct XxxData Xxx;
```

这种设计并不是 C 语言要求，而是 PostgreSQL 有意采用的一种 API 风格。

------

# 1. 先理解这种写法

例如：

```c
typedef struct TupleDescData
{
    ...
} TupleDescData;

typedef TupleDescData *TupleDesc;
```

实际上等价于

```c
struct TupleDescData
{
    ...
};

typedef struct TupleDescData *TupleDesc;
```

于是：

```c
TupleDesc desc;
```

实际上就是

```c
struct TupleDescData *desc;
```

注意：

**TupleDesc 永远是指针。**

------

# 2. 为什么这么做？

主要有几个原因。

------

## 原因一：避免到处写 *

如果不用这种写法：

```c
typedef struct TupleDescData TupleDesc;
```

那么变量必须写：

```c
TupleDesc *desc;
```

函数返回值：

```c
TupleDesc *CreateTupleDesc(...);
```

参数：

```c
void ExecStoreTuple(TupleDesc *desc);
```

整个 PostgreSQL 到处都是：

```
*****
*******
*******
```

非常难读。

采用 pointer typedef 后：

```c
TupleDesc desc;

TupleDesc CreateTupleDesc(...);

void ExecStoreTuple(TupleDesc desc);
```

API 看起来就非常干净。

------

例如：

MemoryContext

如果不是 pointer typedef：

```c
MemoryContextData *TopMemoryContext;

MemoryContextData *CurrentMemoryContext;

MemoryContextData *AllocSetContextCreate(...);
```

PG 实际写成：

```c
MemoryContext TopMemoryContext;

MemoryContext CurrentMemoryContext;

MemoryContext AllocSetContextCreate(...);
```

是不是舒服很多。

------

# 原因二：强调这是一个"句柄(Handle)"

很多 PG 对象，本质上不是一个值，而是一个对象。

例如：

```
Relation
TupleDesc
Snapshot
MemoryContext
ExprContext
Portal
```

这些对象都应该：

- 创建
- 引用
- 修改
- 删除

而不是：

```
直接复制
```

所以作者希望：

```
Relation
```

看起来就像：

```
FILE *
DIR *
MYSQL *
sqlite3 *
```

一样，是一个 Handle。

例如：

```c
Relation rel;
```

直观上就是：

> 一个 Relation 对象

而不是：

> 一个 RelationData 结构体

------

# 原因三：实现"面向对象"风格

PostgreSQL 的源码其实很 OO。

例如：

MemoryContext：

```c
struct MemoryContextData
{
    NodeTag type;

    MemoryContextMethods *methods;

    MemoryContext parent;
    MemoryContext firstchild;
    ...
};
```

外面全部写：

```c
MemoryContext ctx;
```

调用：

```c
ctx->methods->alloc(ctx, size);
```

非常像：

```cpp
ctx.alloc(size);
```

Relation 也是：

```c
Relation rel;

rel->rd_rel;

rel->rd_index;

rel->rd_att;
```

几乎就是对象访问。

------

# 原因四：隐藏实现（Information Hiding）

这是 C 库最经典的技巧。

例如：

头文件：

```c
typedef struct RelationData *Relation;
```

用户：

```c
Relation rel;
```

但是：

```c
struct RelationData
```

可以根本不放头文件。

于是：

外部代码：

只能：

```c
Relation rel;
```

不能：

```c
rel->xxxx
```

因为不知道结构体内容。

PG 有很多地方就是这么做的。

例如：

```c
struct MemoryContextData;
```

只有 memorycontext.c 知道真正定义。

其它模块只能拿 Handle。

这就是 C 里面实现：

> opaque pointer（不透明指针）

的方法。

------

# 原因五：减少复制

例如：

TupleDesc

里面有：

```c
FormData_pg_attribute attrs[];
```

几十个字段。

如果：

```c
TupleDesc desc;
```

表示整个结构体，

那么：

```c
foo(desc);
```

就是整个结构体复制。

PG 根本不希望这样。

他们希望：

```
TupleDesc
```

永远就是：

```
指针
```

没有任何复制。

------

# 3. 为什么结构体名字叫 XXXData？

因为：

```
TupleDesc
```

已经被 typedef 成：

```
TupleDescData *
```

所以真正的数据结构只能另外命名。

于是：

```
TupleDescData
RelationData
HeapTupleData
SnapshotData
PortalData
MemoryContextData
```

都是：

> 真正的数据

而：

```
TupleDesc
Relation
HeapTuple
Snapshot
Portal
MemoryContext
```

都是：

> Handle（对象引用）

例如：

```
TupleDesc
        │
        ▼
+----------------------+
| TupleDescData        |
|----------------------|
| natts                |
| tdtypeid             |
| tdtypmod             |
| attrs[]              |
+----------------------+
```

------

# 4. PostgreSQL 几乎所有核心对象都是这种风格

例如：

```c
typedef struct RelationData *Relation;
typedef struct HeapTupleData *HeapTuple;
typedef struct SnapshotData *Snapshot;
typedef struct MemoryContextData *MemoryContext;
typedef struct ExprContext *ExprContext;
typedef struct TupleTableSlot *TupleTableSlot;
typedef struct EState *EState;
typedef struct PortalData *Portal;
```

所以源码里：

```c
Relation rel;
HeapTuple tuple;
TupleDesc desc;
Snapshot snapshot;
MemoryContext context;
Portal portal;
```

虽然没有 `*`，但实际上全部都是指针。

------

## 总结

PostgreSQL 使用 `typedef struct XxxData *Xxx;` 的设计主要是为了以下几个目的：

| 原因             | 好处                                                  |
| ---------------- | ----------------------------------------------------- |
| 简化 API         | 不必到处写 `*`，函数签名更简洁                        |
| 强调对象语义     | `Relation`、`TupleDesc` 等表示对象句柄，而非值类型    |
| 支持面向对象风格 | 通过 `obj->field`、函数操作对象，接近 OO 编程习惯     |
| 支持不透明类型   | 可以只暴露 `Xxx`，隐藏 `XxxData` 的内部实现，实现封装 |
| 避免误复制       | 大多数对象体积较大且具有共享状态，只应传递引用而非值  |

不过，这种风格也有一个代价：**容易让人忘记 `TupleDesc`、`Relation` 等其实都是指针**。例如：

```c
TupleDesc a, b;
```

很多初学者会以为定义了两个结构体对象，实际上它等价于：

```c
struct TupleDescData *a;
struct TupleDescData *b;
```

因此，这种写法虽然让接口更整洁，但也要求开发者熟悉 PostgreSQL 的命名约定：**凡是 `XxxData` 是结构体本体，而 `Xxx` 通常就是指向它的指针（对象句柄）**。