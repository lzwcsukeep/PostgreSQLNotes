本文介绍 Postgresql 哈希索引页面类型的基本知识。

如果看 PostgreSQL 16 的源码：

```c
src/include/access/hash.h
```

Hash Index 的页类型实际上定义在：

```c
#define LH_UNUSED_PAGE			(0)
#define LH_OVERFLOW_PAGE		(1 << 0)
#define LH_BUCKET_PAGE			(1 << 1)
#define LH_BITMAP_PAGE			(1 << 2)
#define LH_META_PAGE			(1 << 3)
```

所以一个 Hash 索引文件中总共就 **5 种 Page 类型**。

------

# 整体布局

假设一个 Hash 索引文件：

```text
Block 0     Meta Page

Block 1     Bucket 0
Block 2     Bucket 1
Block 3     Bucket 2
Block 4     Bucket 3

Block 5     Bitmap Page

Block 100   Overflow Page
Block 101   Overflow Page
```

逻辑结构：

```text
Hash Index

├── Meta Page
├── Bucket Pages
├── Overflow Pages
├── Bitmap Pages
└── Unused Pages
```

------

# 1. Meta Page（LH_META_PAGE）

固定：

```text
Block 0
```

永远是：

```c
LH_META_PAGE
```

保存整个 Hash Index 的元数据。

结构：

```c
HashMetaPageData
```

主要字段：

```c
hashm_maxbucket
hashm_highmask
hashm_lowmask
hashm_ffactor
hashm_ntuples
```

例如：

```text
当前最大Bucket号
当前Tuple数
填充因子
Split状态
```

------

## 类似于

BTree 的：

```text
Meta Page
```

或者 Heap 的：

```text
FSM/VM 元数据
```

------

总结：

```c
HashMetaPageData

hashm_ntuples
    当前记录数

hashm_ffactor
    每Bucket目标容量

hashm_maxbucket
    当前最大Bucket

hashm_highmask
hashm_lowmask
    Hash值映射Bucket

hashm_firstfree
    第一个空闲Overflow

hashm_nmaps
    Bitmap页数量

hashm_spares[]
    Bucket号→Block号转换

hashm_mapp[]
    Bitmap页位置
```



# 2. Bucket Page（LH_BUCKET_PAGE）

这是最核心的页。

例如：

```text
Bucket 0
```

对应：

```text
Block 1
```

页尾：

```c
HashPageOpaqueData
```

里面：

```c
hasho_flag = LH_BUCKET_PAGE
```

表示：

```text
这是某个Bucket的主页
```

------

## 存放内容

```text
Hash Index Tuple

HashValue
+
Heap TID
```

例如：

```text
hash(100)
→ TID(500,1)

hash(200)
→ TID(600,2)
```

------

# 3. Overflow Page（LH_OVERFLOW_PAGE）

Bucket 满了以后申请。

例如：

```text
Bucket 2

Block 3
    |
    v

Block 100
    |
    v

Block 101
```

其中：

```text
Block100
Block101
```

就是：

```c
LH_OVERFLOW_PAGE
```

------

页尾：

```c
hasho_prevblkno
hasho_nextblkno
```

形成双向链表：

```text
BucketPage
  ↕
Overflow1
  ↕
Overflow2
```

------

## 存放内容

和 Bucket Page 一模一样：

```text
Hash Index Tuple
```

区别只是：

```text
Bucket Page
=
链表头

Overflow Page
=
链表节点
```

------

# 4. Bitmap Page（LH_BITMAP_PAGE）

这个很多人第一次看源码容易忽略。

------

作用：

管理：

```text
Overflow Page 的分配和回收
```

------

因为：

```text
Overflow Page
```

会不断：

```text
申请
释放
再申请
```

例如：

```text
Block100
Block101
Block102
```

后来：

```text
Block101
```

空了。

PostgreSQL 不会：

```text
truncate文件
```

而是：

```text
放回空闲池
```

------

Bitmap Page 记录：

```text
100 -> used
101 -> free
102 -> used
```

以后需要新 Overflow Page：

```text
直接拿101
```

而不是扩展文件。

------

## 类似于

你熟悉的话：

```text
Heap中的FSM
```

思想很像。

------

# 5. Unused Page（LH_UNUSED_PAGE）

这种页目前没被使用。

标志：

```c
hasho_flag = LH_UNUSED_PAGE
```

表示：

```text
空闲页
```

------

例如：

```text
Overflow Page释放
```

后：

```text
Block101
```

可能变成：

```text
LH_UNUSED_PAGE
```

等待复用。

------

# 从源码判断页类型

页尾：

```c
HashPageOpaqueData
```

里面：

```c
uint16 hasho_flag;
```

取：

```c
HashPageOpaque opaque;

opaque = HashPageGetOpaque(page);
```

然后：

```c
switch (opaque->hasho_flag & LH_PAGE_TYPE)
{
    case LH_META_PAGE:
    case LH_BUCKET_PAGE:
    case LH_OVERFLOW_PAGE:
    case LH_BITMAP_PAGE:
    case LH_UNUSED_PAGE:
}
```

判断。

------

# 你可以把整个 Hash Index 文件理解成

```text
Block 0
└── Meta

Block 1
└── Bucket0

Block 2
└── Bucket1

Block 3
└── Bucket2

Block 4
└── Bucket3

Block 5
└── Bitmap

Block 100
└── Overflow(Bucket2)

Block 101
└── Overflow(Bucket2)

Block 102
└── Unused
```

------

# 和 B-Tree 对比

B-Tree 常见页类型：

```text
Meta
Internal
Leaf
Deleted
HalfDead
```

------

Hash Index 页类型则简单得多：

```text
Meta
Bucket
Overflow
Bitmap
Unused
```

其中真正存放索引 Tuple 的只有：

```text
LH_BUCKET_PAGE
LH_OVERFLOW_PAGE
```

而：

```text
LH_META_PAGE
LH_BITMAP_PAGE
LH_UNUSED_PAGE
```

都不存放用户索引数据，只负责管理整个 Hash 索引结构。