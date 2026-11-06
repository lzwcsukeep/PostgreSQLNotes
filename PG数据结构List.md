**现在 PostgreSQL 的 List 已经不是链表了，而是一个动态数组（Dynamic Array，也就是类似 C++ 的 std::vector）。**

# 1. List 的整体结构

```c
typedef struct List
{
    NodeTag     type;

    int         length;

    int         max_length;

    ListCell   *elements;   // 动态数组起始位置

    ListCell    initial_elements[FLEXIBLE_ARRAY_MEMBER];

} List;
```

画出来就是

```
                List
+----------------------------------+
| type = T_List                    |
| length = 3                       |
| max_length = 8                   |
| elements -----------+            |
+---------------------|------------+
                      |
                      ▼
             +------------------+
             | cell0            |
             | cell1            |
             | cell2            |
             | ...              |
             +------------------+
```

里面真正保存元素的是

```c
elements
```

它指向一块**连续内存**。

------

# 2. ListCell 是什么？

它不是链表节点。

看看定义（简化）：

```c
typedef union ListCell
{
    void *ptr_value;
    int int_value;
    Oid oid_value;
    TransactionId xid_value;
} ListCell;
```

可以看到

**里面根本没有**

```c
next
```

也没有

```c
prev
```

因此它不是

```
node -> next -> next
```

这种链表。

它只是一个 union。

例如：

```
ListCell

+-------------+
| ptr_value   |
+-------------+
```

或者

```
+-------------+
| int_value   |
+-------------+
```

根据 List 的类型解释。

例如

```
T_List

cell[0]
    ptr_value -----> Node

cell[1]
    ptr_value -----> Node

cell[2]
    ptr_value -----> Node
```

------

# 3. length

```
length
```

表示当前元素数量。

例如

```
A
B
C
```

那么

```
length = 3
```

------

# 4. max_length

这是容量（capacity）。

类似

```cpp
std::vector
```

例如

```
length = 3

capacity = 8
```

说明

```
+------------------------+
A
B
C
空
空
空
空
空
+------------------------+
```

还能继续插入。

------

# 5. elements

真正的数据放这里。

例如

```
elements
```

指向

```
+---------+
cell0
cell1
cell2
cell3
...
+---------+
```

访问第 i 个元素就是

```c
list->elements[i]
```

所以

随机访问复杂度

```
O(1)
```

不像以前链表

```
O(n)
```

------

# 6. initial_elements 是什么？

这是 PostgreSQL 一个非常巧妙的优化。

```
List
```

最后有

```c
initial_elements[FLEXIBLE_ARRAY_MEMBER]
```

这是 C99 的 Flexible Array Member。

意思就是

> List 对象后面可以紧跟一块数组。

例如

```
malloc(sizeof(List)+8*sizeof(ListCell))
```

内存实际上长这样：

```
+-------------------+
List Header
+-------------------+
cell0
cell1
cell2
cell3
...
+-------------------+
```

所以

```
elements
```

直接指向

```
initial_elements
```

完全不用再次 malloc。

源码里也有注释：

```c
If elements == initial_elements,
it's not a separate allocation.
```

意思就是：

```
elements
        │
        ▼
initial_elements
```

没有额外分配。

------

## 总结

可以把 PostgreSQL 当前的 `List` 理解成一个轻量级的 `std::vector<ListCell>`：

| PostgreSQL `List` 成员 | 类比 C++ `std::vector`                  | 作用                                        |
| ---------------------- | --------------------------------------- | ------------------------------------------- |
| `length`               | `size()`                                | 当前元素数量                                |
| `max_length`           | `capacity()`                            | 已分配容量                                  |
| `elements`             | `data()`                                | 指向连续元素数组                            |
| `initial_elements`     | 小对象优化（Small Buffer Optimization） | 小列表无需再次 `malloc`，减少内存分配       |
| `ListCell`             | `union` 元素                            | 保存 `void*`、`int`、`Oid` 等不同类型的数据 |

因此，在你前面看到的代码：

```c
List *elemlist;
ListCell *l;

foreach(l, elemlist)
{
    char *curhost = (char *) lfirst(l);
}
```

实际上就是在遍历一个**动态数组**，`l` 并不是传统链表节点，而是指向 `elements` 数组中当前 `ListCell` 的指针，每次循环都通过指针加一移动到下一个元素，而不是通过 `next` 指针跳转。这也是 PostgreSQL 新版 `List` 相比旧版链表实现性能提升的重要原因之一。