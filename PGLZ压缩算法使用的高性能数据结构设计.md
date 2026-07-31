# PGLZ 压缩使用的高性能数据结构 

使用的技术叫做： intrusive linked list  +   ring buffer node pool

特点：

- 无 malloc
- 固定内存
- O(1) 插入删除
- cache friendly
- 自动淘汰旧历史

 `intrusive linked list`（侵入式链表）是系统编程里非常经典的一种链表设计。

# intrusive linked list

# 一、什么叫 intrusive（侵入式）

核心特点：

```
链表指针直接放在对象内部
```

即：

```
struct Node {
    struct Node *next;
    struct Node *prev;
    ...
};
```

对象本身：

```
自己就是链表节点
```

# 二、和普通链表的区别

## 1. 普通（non-intrusive）链表

普通写法：

```
struct ListNode {
    void *data;
    struct ListNode *next;
};

struct User {
    int id;
};
```

关系：

```
ListNode
   |
   +--> User
```

即：

```
链表节点
和
真实对象
是分开的
```

需要：

```
malloc(ListNode)
malloc(User)
```

两次分配。

------

## 2. intrusive 链表

侵入式：

```
struct User {
    int id;

    struct User *next;
    struct User *prev;
};
```

关系：

```
User 本身就是链表节点
```

不需要额外 ListNode。

------

# 三、PGLZ 就是 intrusive linked list

这里：

```
typedef struct PGLZ_HistEntry
{
    struct PGLZ_HistEntry *next;
    struct PGLZ_HistEntry *prev;

    int hindex;
    const char *pos;
} PGLZ_HistEntry;
```

注意：

```
next/prev
直接在对象内部
```

因此：

```
PGLZ_HistEntry 自己就是链表节点
```

不是：

```
ListNode -> PGLZ_HistEntry
```

这种二层结构。

------

# 四、为什么叫“侵入式”

因为：

```
链表逻辑“侵入”了业务对象
```

对象必须：

```
带 next/prev 字段
```

所以：

```
对象结构被链表需求修改了
```

这就叫 intrusive。

------

# 五、intrusive linked list 的巨大优势

这是系统软件最爱用的结构。

------

## 1. 不需要 malloc

PGLZ：

```
static PGLZ_HistEntry hist_entries[4097];
```

直接：

```
数组元素就是节点
```

无需：

```
malloc(node)
```

------

## 2. cache locality 极好

连续数组：

```
hist_entries[0..4096]
```

CPU cache 很友好。

普通 malloc 链表：

```
节点散落内存
```

cache miss 很严重。

------

## 3. 少一次间接访问

普通链表：

```
ListNode -> data
```

需要：

```
两次 pointer dereference
```

侵入式：

```
node 本身就是对象
```

少一次跳转。

------

## 4. 更省内存

普通链表：

```
ListNode + Object
```

侵入式：

```
只有 Object
```

# ring buffer node pool

上面所讲的链表节点，并不是使用 malloc 分配，而是在一个固定大小的 ring buffer node pool 里面取。当 buffer node pool 里节点全部用完之后,但是又需要新的节点时，会回收之前已经使用过的 node。

## ring buffer 数据结构

代码实现上 ring buffer node pool 实际就是一个固定数组。 但是结合算法，使用起来非常巧妙。

```c
#define PGLZ_HISTORY_SIZE		4096
static PGLZ_HistEntry hist_entries[PGLZ_HISTORY_SIZE + 1];
```

上面 `hist_entries` 数组，使用起来就是一个 ring buffer node pool, 里面 node 为 `PGLZ_HistEntry` , 定义如下： 

```c
typedef struct PGLZ_HistEntry
{
	struct PGLZ_HistEntry *next;	/* links for my hash key's list */
	struct PGLZ_HistEntry *prev;
	int			hindex;			/* my current hash key */
	const char *pos;			/* my input position */
} PGLZ_HistEntry;
```

## ring buffer 指针：hist_next

在 PGLZ 里面，会使用一个 整数变量 表示 ring buffer pool 的指针：

```c
int			hist_next = 1;
```

## hist_next 的变化规律：

这里：

```c
int hist_next = 1;
```

每次：

```c
hist_next++
```

超过：

```
4096
```

就：

```c
hist_next = 1;
hist_recycle = true;
```

因此：

```
旧节点会被覆盖
```

这不是 bug。

这是设计目标。

**因此：hist_next 本质是 ring buffer 指针。**

每次需要一个新的节点，就使用 hist_next 所指示的 `PGLZ_HistEntry` 元素。 当走到 `PGLZ_HistEntry hist_entries[]` 尽头，重新从 1 开始。

# 代码实现(非常重要！！！)

PGLZ 压缩算法使用的完整数据结构实际是：

```c
hash table
    +
intrusive doubly linked list
    +
ring buffer node pool
```

---

## 完整数据结构

```c
/* ----------
 * PGLZ_HistEntry -
 *
 *		Linked list for the backward history lookup
 *
 * All the entries sharing a hash key are linked in a doubly linked list.
 * This makes it easy to remove an entry when it's time to recycle it
 * (because it's more than 4K positions old).
 * ----------
 */
typedef struct PGLZ_HistEntry
{
	struct PGLZ_HistEntry *next;	/* links for my hash key's list */
	struct PGLZ_HistEntry *prev;
	int			hindex;			/* my current hash key */
	const char *pos;			/* my input position */
} PGLZ_HistEntry;

/* ----------
 * Statically allocated work arrays for history
 * ----------
 */
static int16 hist_start[PGLZ_MAX_HISTORY_LISTS];
static PGLZ_HistEntry hist_entries[PGLZ_HISTORY_SIZE + 1];

/* ----------
 * Local definitions
 * ----------
 */
#define PGLZ_MAX_HISTORY_LISTS	8192	/* must be power of 2 */
#define PGLZ_HISTORY_SIZE		4096

/** 由起始地址 _s 和后面的四个字节 计算出一个 index 的函数，
 *  然后可得 start = hist_start[index] ，就是 hist_entries[start] 就是链表的头节点。
 *  然后记录 _s 和后面四个字节计算出来的 index 到 hist_entries[hist_next] 
 *  hist_entries[hist_next].index = index
 *  hist_entries[hist_next].pos = s
 *  最后更新 hist_start[index] = hist_next ;
 *  然后 hist_next++
 */
// 所以 所有计算得到的相同的 index 的节点，都会串在同一个链表里面
// index 不变，但是 hist_start[index] 的值会不断更新，表示从 hist_entries里哪个元素开始
// 是 相同 index 的链表的头
// 结论： 这个函数功能很纯粹： 从 _s 和 后面四个字节，计算出一个整数 index
#define pglz_hist_idx(_s,_e, _mask) (										\
			((((_e) - (_s)) < 4) ? (int) (_s)[0] :							\
			 (((_s)[0] << 6) ^ ((_s)[1] << 4) ^								\
			  ((_s)[2] << 2) ^ (_s)[3])) & (_mask)				\
		)

// 往历史表中添加一个元素， 这个元素的组成其实就是一个 (_s,_e)
// 甚至基本可以当成就是 _s, 只是实际会计算 _s 开始，往后四个字节
// 所以这个历史表里 存放的就是 PGLZ_HistEntry: >> char* pos 和 int hindex
#define pglz_hist_add(_hs,_he,_hn,_recycle,_s,_e, _mask)	\
do {									\
			int __hindex = pglz_hist_idx((_s),(_e), (_mask));				\
			int16 *__myhsp = &(_hs)[__hindex];								\
			PGLZ_HistEntry *__myhe = &(_he)[_hn];							\
			if (_recycle) {													\
				if (__myhe->prev == NULL)									\
					(_hs)[__myhe->hindex] = __myhe->next - (_he);			\
				else														\
					__myhe->prev->next = __myhe->next;						\
				if (__myhe->next != NULL)									\
					__myhe->next->prev = __myhe->prev;						\
			}																\
			__myhe->next = &(_he)[*__myhsp];								\
			__myhe->prev = NULL;											\
			__myhe->hindex = __hindex;										\
			__myhe->pos  = (_s);											\
			/* If there was an existing entry in this hash slot, link */	\
			/* this new entry to it. However, the 0th entry in the */		\
			/* entries table is unused, so we can freely scribble on it. */ \
			/* So don't bother checking if the slot was used - we'll */		\
			/* scribble on the unused entry if it was not, but that's */	\
			/* harmless. Avoiding the branch in this critical path */		\
			/* speeds this up a little bit. */								\
			/* if (*__myhsp != INVALID_ENTRY) */							\
				(_he)[(*__myhsp)].prev = __myhe;							\
			*__myhsp = _hn;													\
			if (++(_hn) >= PGLZ_HISTORY_SIZE + 1) {							\
				(_hn) = 1;													\
				(_recycle) = true;											\
			}																\
} while (0)

```

上述数据结构互相配合的地方：

## PGLZ_HistEntry hist_entries

```c
// 就是一个 ring buffer node pool , 所有链表节点的内存不需要再malloc,链表节点都从下面这个数组中
// 获得。
// 随着代码的运行，这个数组内部实际上相当于保存了多条链表。只是链表节点的总数不会超过 PGLZ_HISTORY_SIZE
// 如果所有节点都被使用了，但是又有新的数据需要保存，会把当前定位到的节点，假设是 hist_entries[hn]，从当前
// 链表中摘出来，然后使用。
// 这样就实现了滑动窗口，固定长度的机制。
static PGLZ_HistEntry hist_entries[PGLZ_HISTORY_SIZE + 1];
```

## int hist_next;

```c
// 实际使用时需要配上一个全局的整数， 这个是 ring buffer node pool 指针
// 这个指针指到的 元素 就是下一个待使用的 链表节点。 hist_entries[hist_next]
// hist_next 使用了一个节点之后，只会 hist_next++。 当达到最大边界 PGLZ_HISTORY_SIZE + 1 时，重新从 1 开始
// 就是下面代码所演示：
// 			if (++(_hn) >= PGLZ_HISTORY_SIZE + 1) {							\
//				(_hn) = 1;													\
//				(_recycle) = true;											\
//			}
// 结论： 记住 hist_next 实际是 ring buffer node pool 指针。指向下一个待用节点位置。会返回到旧节点去，不过没关系
//       返回到旧节点去，会先把旧节点从原来的链表中摘出来，然后就可以继续使用了。
int hist_next;
```

## hist_start[PGLZ_MAX_HISTORY_LISTS]

上述 hist_entries虽然是个数组，但是内部其实已经构成了许多条链表了，现在不知道的就是链表的起始位置在哪里，下面这个数组就是用来记录，链表的起始位置。hist_entries 数组的链表元素的特点：处于同一个链表里所有元素的 PGLZ_HistEntry.hindex 值都一样。

下面这个数组，用这种方式来记录同一链表的头节点位置： 

- 给定 hindex 有：

- start = hist_start[hindex]

- 则 hist_entries[start] 就是头节点

```c
// 上述 hist_entries 内部已经构成了多条链表了，但是不知道链表的起始位置在哪里
// 给一个整数 index 
// 结合下面这个数组，  start = hist_start[index]
// 那么 hist_entries[start] 就是 index 所在链表的头节点。
static int16 hist_start[PGLZ_MAX_HISTORY_LISTS];
```

**总结： 这种设计实际用上的数据结构就下面这些。**

```c
typedef struct PGLZ_HistEntry
{
	struct PGLZ_HistEntry *next;	/* links for my hash key's list */
	struct PGLZ_HistEntry *prev;
	int			hindex;			/* my current hash key */
	const char *pos;			/* my input position */
} PGLZ_HistEntry;

static int16 hist_start[PGLZ_MAX_HISTORY_LISTS];
static PGLZ_HistEntry hist_entries[PGLZ_HISTORY_SIZE + 1];
int hist_next;
```

但这套结构：

- 无 malloc
- O(1) 删除
- O(1) 插入
- 固定内存
- cache friendly

非常的高效，非常的amazing.