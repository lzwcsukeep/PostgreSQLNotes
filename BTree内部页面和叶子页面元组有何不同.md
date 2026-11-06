元组结构体定义没有任何不同，都是 `IndexTuple`

```c
typedef struct IndexTupleData
{
	ItemPointerData t_tid;		/* reference TID to heap tuple */

	/* ---------------
	 * t_info is laid out in the following fashion:
	 *
	 * 15th (high) bit: has nulls
	 * 14th bit: has var-width attributes
	 * 13th bit: AM-defined meaning
	 * 12-0 bit: size of tuple
	 * ---------------
	 */

	unsigned short t_info;		/* various info about tuple */

} IndexTupleData;				/* MORE DATA FOLLOWS AT END OF STRUCT */

typedef IndexTupleData *IndexTuple;
```

```c
typedef struct ItemPointerData
{
	BlockIdData ip_blkid;
	OffsetNumber ip_posid;
}

/* If compiler understands packed and aligned pragmas, use those */
#if defined(pg_attribute_packed) && defined(pg_attribute_aligned)
			pg_attribute_packed()
			pg_attribute_aligned(2)
#endif
ItemPointerData;

typedef ItemPointerData *ItemPointer;
```

区别在于，

当判断为内部节点时，只使用 `ItemPointerData` 的 `BlockIdData` 部分,即块号部分，用于定位一个子块号。

当判断 Page 是叶子节点时，使用 完整的 `ItemPointerData` 信息，即 块号+ Offset 都有用。用于定位一个堆元组。

**形象图示：**

```c
              Internal Page
      ┌──────────────────────────┐
      │ key1 | downlink → page 5 │
      │ key2 | downlink → page 9 │
      └──────────────────────────┘
                   ↓

              Leaf Page
      ┌──────────────────────────┐
      │ key1 | heap TID (1,3)    │
      │ key2 | heap TID (2,7)    │
      └──────────────────────────┘
```

