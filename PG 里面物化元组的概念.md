背景： 在 PostgreSQL 执行器源码下面 `tuptable.h` 里面有如下注释,里面提到了 物化(materialized)这个词，如何理解这个词？

```c
 * There's various different types of tuple table slots, each being able to
 * store different types of tuples. Additional types of slots can be added
 * without modifying core code. The type of a slot is determined by the
 * TupleTableSlotOps* passed to the slot creation routine. The builtin types
 * of slots are
 *
 * 1. physical tuple in a disk buffer page (TTSOpsBufferHeapTuple)
 * 2. physical tuple constructed in palloc'ed memory (TTSOpsHeapTuple)
 * 3. "minimal" physical tuple constructed in palloc'ed memory
 *    (TTSOpsMinimalTuple)
 * 4. "virtual" tuple consisting of Datum/isnull arrays (TTSOpsVirtual)
 *
 *
 * The first two cases are similar in that they both deal with "materialized"
 * tuples, but resource management is different.  For a tuple in a disk page
 * we need to hold a pin on the buffer until the TupleTableSlot's reference
 * to the tuple is dropped; while for a palloc'd tuple we usually want the
 * tuple pfree'd when the TupleTableSlot's reference is dropped.
```

---

答：在 PostgreSQL 的执行器源码中，`materialized` 可以理解为：元组已经以某种具体的数据形式存在于内存或存储页面中，而不是仅仅通过一组尚未物化的数据引用来表示。

你给出的注释里，`materialized tuples` 特指前两种 Slot 实现所处理的元组。这里要注意，`materialized` 不等于“已经写入磁盘”，也不等于“已经复制到一块独立的内存中”。

## 1. 先理解 materialized 和 virtual 的区别

PostgreSQL 的 `TupleTableSlot` 可以通过不同方式表示一行数据。

假设有一行数据：

```
id = 1, name = 'Alice'
```

Materialized tuple（物化元组）

这行数据已经具有具体的元组存储表示。例如：

- 元组位于表的数据页中，通过 buffer 引用它。
- 元组已经被复制到通过 `palloc()` 分配的内存中。

Virtual tuple（虚拟元组）

这行数据不一定有一个独立的、完整的物理元组表示，而是通过 `Datum` 数组和空值标记数组来组织各列的值。

例如：

```
slot->tts_values[0] = Int32GetDatum(1);
slot->tts_values[1] = CStringGetTextDatum("Alice");

slot->tts_isnull[0] = false;
slot->tts_isnull[1] = false;
```

这只是示意代码，用来说明虚拟元组的数据组织方式。实际实现还需要正确处理类型、内存上下文和数据生命周期。

## 2. 为什么注释强调 resource management？

你引用的原文是：

> The first two cases are similar in that they both deal with "materialized" tuples, but resource management is different.

这里的 `first two cases` 指的是前面提到的两种 Slot 实现：

1. `TTSOpsBufferHeapTuple`：元组位于 buffer 对应的数据页中。
2. `TTSOpsHeapTuple`：元组位于 `palloc()` 分配的内存中。

它们的共同点是：都有具体的物理元组表示。

但它们的资源管理方式不同。

### 情况一：元组位于 buffer 数据页中

```
磁盘数据页
    ↓ 读取到共享缓冲区
Buffer
    ↓ 引用
HeapTuple
    ↓
TupleTableSlot
```

Slot 引用这个元组时，需要保证对应的 buffer 仍然有效。因此，需要持有 buffer pin，直到 Slot 不再引用该元组。

这里的 pin 是缓冲区管理机制中的固定引用，防止缓冲区在仍被使用时被淘汰或复用。它不意味着数据页会一直被锁住。

当 Slot 放弃这个引用时，就需要释放相应的 pin。

### 情况二：元组位于 `palloc()` 分配的内存中

```
palloc()
   ↓
一块内存中的 HeapTuple
   ↓
TupleTableSlot 引用它
```

这种情况下，元组占用的是动态分配的内存。

如果这块内存由 Slot 拥有，那么当 Slot 不再需要该元组时，通常就要调用 `pfree()` 释放它。

不过，如果 Slot 只是借用其他地方分配的元组，而不拥有它，就不能随意释放。因此，PostgreSQL 需要区分“引用元组”和“拥有元组”。

这也对应 `TupleTableSlot` 中的 `TTS_FLAG_SHOULDFREE` 标记：它用于表示 Slot 是否应该在清理时释放所引用的元组。

## 3. 为什么 virtual tuple 不属于这里的前两种情况？

虚拟元组通常直接使用 Slot 内部的两个数组表示数据：

```
Datum *tts_values;
bool  *tts_isnull;
```

因此，它不一定需要一个单独分配的 `HeapTuple`，也不一定需要持有 buffer pin。

但这不代表虚拟元组完全没有内存管理问题。例如，某个 `Datum` 可能引用其他执行节点提供的内存。当底层内存被释放或复用时，这个引用也可能失效。

所以，虚拟元组同样需要正确管理数据的生命周期，只是管理方式不同。

## 4. 总结

在你引用的 PostgreSQL 源码语境中，可以这样理解：

- Materialized tuple：具有具体物理元组表示的元组，例如位于 buffer 数据页中的元组，或位于 `palloc()` 内存中的元组。
- Virtual tuple：通过 Slot 的 `Datum` 数组和空值数组组织列值的元组，不要求具有独立的完整物理元组表示。
- Resource management：决定元组的存储资源由谁持有，以及 Slot 不再引用它时应释放什么资源。

最重要的一点是：`materialized` 描述的是元组的表示形式，而不是简单地表示它有没有被加载到内存中。

物化根本不是啥新概念，之前学的 Page 格式里面，一个元组在 Page 里面有一个 ItemID , 这个ItemID 指示了这个元组数据在Page 里的起始位置和长度。那么把文件Page里面的元组数据放到内存里，这个元组就叫有具体物理元组的元组，也就是物化元组了。