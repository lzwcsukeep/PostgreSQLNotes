在阅读内存上下文相关代码时遇到了 结构体数组的一些没见过的写法，如下：

代码写法：

```c
/*****************************************************************************
 *	  GLOBAL MEMORY															 *
 *****************************************************************************/

static const MemoryContextMethods mcxt_methods[] = {
	/* aset.c */
	[MCTX_ASET_ID].alloc = AllocSetAlloc,
	[MCTX_ASET_ID].free_p = AllocSetFree,
	[MCTX_ASET_ID].realloc = AllocSetRealloc,
	[MCTX_ASET_ID].reset = AllocSetReset,
	[MCTX_ASET_ID].delete_context = AllocSetDelete,
	[MCTX_ASET_ID].get_chunk_context = AllocSetGetChunkContext,
	[MCTX_ASET_ID].get_chunk_space = AllocSetGetChunkSpace,
	[MCTX_ASET_ID].is_empty = AllocSetIsEmpty,
	[MCTX_ASET_ID].stats = AllocSetStats,
#ifdef MEMORY_CONTEXT_CHECKING
	[MCTX_ASET_ID].check = AllocSetCheck,
#endif

	/* generation.c */
	[MCTX_GENERATION_ID].alloc = GenerationAlloc,
	[MCTX_GENERATION_ID].free_p = GenerationFree,
	[MCTX_GENERATION_ID].realloc = GenerationRealloc,
	[MCTX_GENERATION_ID].reset = GenerationReset,
	[MCTX_GENERATION_ID].delete_context = GenerationDelete,
	[MCTX_GENERATION_ID].get_chunk_context = GenerationGetChunkContext,
	[MCTX_GENERATION_ID].get_chunk_space = GenerationGetChunkSpace,
	[MCTX_GENERATION_ID].is_empty = GenerationIsEmpty,
	[MCTX_GENERATION_ID].stats = GenerationStats,
#ifdef MEMORY_CONTEXT_CHECKING
	[MCTX_GENERATION_ID].check = GenerationCheck,
#endif

	/* slab.c */
	[MCTX_SLAB_ID].alloc = SlabAlloc,
	[MCTX_SLAB_ID].free_p = SlabFree,
	[MCTX_SLAB_ID].realloc = SlabRealloc,
	[MCTX_SLAB_ID].reset = SlabReset,
	[MCTX_SLAB_ID].delete_context = SlabDelete,
	[MCTX_SLAB_ID].get_chunk_context = SlabGetChunkContext,
	[MCTX_SLAB_ID].get_chunk_space = SlabGetChunkSpace,
	[MCTX_SLAB_ID].is_empty = SlabIsEmpty,
	[MCTX_SLAB_ID].stats = SlabStats,
#ifdef MEMORY_CONTEXT_CHECKING
	[MCTX_SLAB_ID].check = SlabCheck,
#endif

	/* alignedalloc.c */
	[MCTX_ALIGNED_REDIRECT_ID].alloc = NULL,	/* not required */
	[MCTX_ALIGNED_REDIRECT_ID].free_p = AlignedAllocFree,
	[MCTX_ALIGNED_REDIRECT_ID].realloc = AlignedAllocRealloc,
	[MCTX_ALIGNED_REDIRECT_ID].reset = NULL,	/* not required */
	[MCTX_ALIGNED_REDIRECT_ID].delete_context = NULL,	/* not required */
	[MCTX_ALIGNED_REDIRECT_ID].get_chunk_context = AlignedAllocGetChunkContext,
	[MCTX_ALIGNED_REDIRECT_ID].get_chunk_space = AlignedAllocGetChunkSpace,
	[MCTX_ALIGNED_REDIRECT_ID].is_empty = NULL, /* not required */
	[MCTX_ALIGNED_REDIRECT_ID].stats = NULL,	/* not required */
#ifdef MEMORY_CONTEXT_CHECKING
	[MCTX_ALIGNED_REDIRECT_ID].check = NULL,	/* not required */
#endif


	/*
	 * Unused (as yet) IDs should have dummy entries here.  This allows us to
	 * fail cleanly if a bogus pointer is passed to pfree or the like.  It
	 * seems sufficient to provide routines for the methods that might get
	 * invoked from inspection of a chunk (see MCXT_METHOD calls below).
	 */

	[MCTX_UNUSED1_ID].free_p = BogusFree,
	[MCTX_UNUSED1_ID].realloc = BogusRealloc,
	[MCTX_UNUSED1_ID].get_chunk_context = BogusGetChunkContext,
	[MCTX_UNUSED1_ID].get_chunk_space = BogusGetChunkSpace,

	[MCTX_UNUSED2_ID].free_p = BogusFree,
	[MCTX_UNUSED2_ID].realloc = BogusRealloc,
	[MCTX_UNUSED2_ID].get_chunk_context = BogusGetChunkContext,
	[MCTX_UNUSED2_ID].get_chunk_space = BogusGetChunkSpace,

	[MCTX_UNUSED3_ID].free_p = BogusFree,
	[MCTX_UNUSED3_ID].realloc = BogusRealloc,
	[MCTX_UNUSED3_ID].get_chunk_context = BogusGetChunkContext,
	[MCTX_UNUSED3_ID].get_chunk_space = BogusGetChunkSpace,

	[MCTX_UNUSED4_ID].free_p = BogusFree,
	[MCTX_UNUSED4_ID].realloc = BogusRealloc,
	[MCTX_UNUSED4_ID].get_chunk_context = BogusGetChunkContext,
	[MCTX_UNUSED4_ID].get_chunk_space = BogusGetChunkSpace,
};
```

```c
typedef enum MemoryContextMethodID
{
	MCTX_UNUSED1_ID,			/* 000 occurs in never-used memory */
	MCTX_UNUSED2_ID,			/* glibc malloc'd chunks usually match 001 */
	MCTX_UNUSED3_ID,			/* glibc malloc'd chunks > 128kB match 010 */
	MCTX_ASET_ID,
	MCTX_GENERATION_ID,
	MCTX_SLAB_ID,
	MCTX_ALIGNED_REDIRECT_ID,
	MCTX_UNUSED4_ID				/* 111 occurs in wipe_mem'd memory */
} MemoryContextMethodID;
```

---

`mcxt_methods[]` 里面的这种语法 : `[MCTX_ALIGNED_REDIRECT_ID].realloc = AlignedAllocRealloc` 什么意思？？

你看到的这一句：

```c
[MCTX_ALIGNED_REDIRECT_ID].realloc = AlignedAllocRealloc,
```

实际上是 **C99 的指定初始化器（Designated Initializers）**，它不是 PostgreSQL 发明的，而是 **C99 标准语法**。

这里实际上用了两种指定初始化器叠加在一起：

1. **数组指定初始化**
2. **结构体成员指定初始化**

下面分别解释。

------

## 1. 普通的结构体初始化

假设有

```c
typedef struct
{
    int a;
    int b;
} Test;
```

最普通的初始化：

```c
Test t = {1, 2};
```

等价于

```c
t.a = 1;
t.b = 2;
```

但是这种方式要求：

**必须按成员声明顺序写。**

------

## 2. 结构体指定初始化

C99 可以写成：

```c
Test t = {
    .b = 2,
    .a = 1
};
```

这里

```c
.b = 2
```

就是指定初始化。

无论顺序如何：

```c
t.a == 1
t.b == 2
```

------

## 3. 数组指定初始化

例如：

```c
int a[10] = {
    [3] = 100,
    [7] = 200
};
```

表示

```c
a[3] = 100;
a[7] = 200;
```

其它元素全部初始化为 0。

所以

```text
index
0 -> 0
1 -> 0
2 -> 0
3 -> 100
4 -> 0
5 -> 0
6 -> 0
7 -> 200
8 -> 0
9 -> 0
```

------

## 4. PostgreSQL 这里两个一起用了

看看定义：

```c
static const MemoryContextMethods mcxt_methods[] =
{
    [MCTX_ASET_ID].alloc = AllocSetAlloc,
    [MCTX_ASET_ID].free_p = AllocSetFree,
    ...
};
```

这里可以拆成两层来看。

第一层：

```c
[MCTX_ASET_ID]
```

表示

```text
mcxt_methods[MCTX_ASET_ID]
```

第二层：

```c
.alloc
```

表示

```text
mcxt_methods[MCTX_ASET_ID].alloc
```

所以

```c
[MCTX_ASET_ID].alloc = AllocSetAlloc
```

等价于

```c
mcxt_methods[MCTX_ASET_ID].alloc = AllocSetAlloc;
```

只是这是**初始化阶段**，不是运行时赋值。

------

## 5. 你的例子

例如

```c
[MCTX_ALIGNED_REDIRECT_ID].realloc = AlignedAllocRealloc,
```

完全等价于：

```c
mcxt_methods[MCTX_ALIGNED_REDIRECT_ID].realloc =
    AlignedAllocRealloc;
```

不过这是在编译时初始化。

------

## 6. 为什么不用普通初始化？

假设没有指定初始化，只能写：

```c
MemoryContextMethods mcxt_methods[] =
{
    {
        AllocSetAlloc,
        AllocSetFree,
        AllocSetRealloc,
        ...
    },
    {
        GenerationAlloc,
        ...
    }
};
```

这样有几个问题：

### 第一，顺序必须一致

如果结构体定义改了：

```c
typedef struct
{
    alloc;
    realloc;
    free;
} ...
```

整个初始化都错了。

------

### 第二，可读性差

看到

```c
{
    AllocSetAlloc,
    AllocSetFree,
    AllocSetRealloc,
    ...
}
```

你很难知道：

哪个是 alloc？

哪个是 realloc？

哪个是 stats？

------

用了指定初始化以后：

```c
.alloc = AllocSetAlloc,
.free_p = AllocSetFree,
.realloc = AllocSetRealloc,
.stats = AllocSetStats,
```

一眼就知道。

------

### 第三，可以跳过成员

例如：

```c
[MCTX_ALIGNED_REDIRECT_ID].alloc = NULL,
[MCTX_ALIGNED_REDIRECT_ID].free_p = AlignedAllocFree,
```

没有写的成员：

```c
.reset
.delete_context
```

自动初始化为

```c
NULL
```

------

## 7. 为什么 PostgreSQL 特别喜欢这种写法？

再看看这个枚举：

```c
typedef enum
{
    MCTX_UNUSED1_ID,
    MCTX_UNUSED2_ID,
    MCTX_UNUSED3_ID,

    MCTX_ASET_ID,

    MCTX_GENERATION_ID,

    MCTX_SLAB_ID,

    MCTX_ALIGNED_REDIRECT_ID,

    MCTX_UNUSED4_ID
} MemoryContextMethodID;
```

假设以后作者新增：

```c
MCTX_MYPOOL_ID
```

只需要加：

```c
[MCTX_MYPOOL_ID].alloc = MyPoolAlloc,
[MCTX_MYPOOL_ID].free_p = MyPoolFree,
...
```

根本不用去数：

> "这是数组第几个元素？"

这也是 Linux Kernel、PostgreSQL、GCC 等大型 C 项目大量使用指定初始化器的原因。

------

## 8. 它最终相当于什么？

编译器最终会把它理解成类似下面的初始化（概念上）：

```c
mcxt_methods[MCTX_ASET_ID].alloc = AllocSetAlloc;
mcxt_methods[MCTX_ASET_ID].free_p = AllocSetFree;
mcxt_methods[MCTX_ASET_ID].realloc = AllocSetRealloc;

mcxt_methods[MCTX_GENERATION_ID].alloc = GenerationAlloc;
mcxt_methods[MCTX_GENERATION_ID].free_p = GenerationFree;

mcxt_methods[MCTX_SLAB_ID].alloc = SlabAlloc;
...
```

只是这些赋值都是**编译期静态初始化**，不会在程序启动时逐条执行。

------

### 总结

`[MCTX_ALIGNED_REDIRECT_ID].realloc = AlignedAllocRealloc` 是 **数组指定初始化** 和 **结构体成员指定初始化** 的组合：

- `[MCTX_ALIGNED_REDIRECT_ID]`：指定初始化数组中索引为 `MCTX_ALIGNED_REDIRECT_ID` 的元素。
- `.realloc`：指定初始化该结构体元素中的 `realloc` 成员。
- `= AlignedAllocRealloc`：把函数指针成员初始化为 `AlignedAllocRealloc`。

也就是说，它的含义就是：

```c
mcxt_methods[MCTX_ALIGNED_REDIRECT_ID].realloc = AlignedAllocRealloc;
```

不过这是**静态初始化语法**，由编译器在生成程序时完成，而不是运行时执行的赋值语句。