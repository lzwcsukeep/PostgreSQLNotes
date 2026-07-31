# postgresql 底层存储格式笔记

postgresql 用来存储数据的文件叫做堆文件(heap file),堆文件有预定义的格式。本篇笔记就是讲解 一个堆文件内部的格式。

## 堆文件是 Page 的数组

下面是来自postgresql 官网的[一段话](https://www.postgresql.org/docs/current/storage-page-layout.html)：

> Every table and index is stored as an array of *pages* of a fixed size (usually 8 kB, although a different page size can be selected when compiling the server). In a table, all the pages are logically equivalent, so a particular item (row) can be stored in any page. 

也就是说一个 磁盘上一个堆文件的逻辑视图如下：

```text
[Page1][Page2]...[PageN]...
```

每个Page固定大小，一般是8KB。

## 帧(Page)内部固定格式

对于单个Page,其内部也是有固定格式的，一个Page 划分成五部分，分别是：

- PageHeaderData : 帧头，固定24字节大小
- ItemIdData : Item定位数组，这部分是一个数组，每个数组元素定位了一个 Item，每一个数组元素大小4字节，使用 (offset,length)对的方式定位Item.
- Free space : 未使用的空间，后续如果在此帧内保存数据，就是在Free space 里面存放新增的数据。
- Items : 实际的Item 数据
- Special space : 特殊空间，普通表Table 这部分为空

Item 的概念指的是关系中一个独立的数据，对普通表来说，一个 Item 就是表的一行(one row)。

---

**Table 66.2. Overall Page Layout**

| Item           | Description                                                  |
| -------------- | ------------------------------------------------------------ |
| PageHeaderData | 24 bytes long. Contains general information about the page, including free space pointers. |
| ItemIdData     | Array of item identifiers pointing to the actual items. Each entry is an (offset,length) pair. 4 bytes per item. |
| Free space     | The unallocated space. New item identifiers are allocated from the start of this area, new items from the end. |
| Items          | The actual items themselves.                                 |
| Special space  | Index access method specific data. Different methods store different data. Empty in ordinary tables. |

---

**Page Layout 图示**

![image-20260224155641326](Postgresql底层存储格式笔记.assets/image-20260224155641326.png)

## 帧头(PageHeader)内部格式

对于每个帧头，帧头内部也有固定的格式，这里只列举重要的：

- pd_lower : free space 的起始位置在帧内的偏移量
- pd_upper : free space 的结束位置在帧内的偏移量 
- pd_special : 特殊空间的起始位置在帧内的偏移量
- pd_pagesize_version : 帧大小和布局版本信息
- ...

完整帧头内部格式：

**Table 66.3. PageHeaderData Layout**

| Field               | Type           | Length  | Description                                                  |
| ------------------- | -------------- | ------- | ------------------------------------------------------------ |
| pd_lsn              | PageXLogRecPtr | 8 bytes | LSN: next byte after last byte of WAL record for last change to this page |
| pd_checksum         | uint16         | 2 bytes | Page checksum                                                |
| pd_flags            | uint16         | 2 bytes | Flag bits                                                    |
| pd_lower            | LocationIndex  | 2 bytes | Offset to start of free space                                |
| pd_upper            | LocationIndex  | 2 bytes | Offset to end of free space                                  |
| pd_special          | LocationIndex  | 2 bytes | Offset to start of special space                             |
| pd_pagesize_version | uint16         | 2 bytes | Page size and layout version number information              |
| pd_prune_xid        | TransactionId  | 4 bytes | Oldest unpruned XMAX on page, or zero if none                |

All the details can be found in `src/include/storage/bufpage.h`.

## 一个 Item 的内部格式(Tuple格式)

一个 Item 内部也有格式，Item 的底层存储表示就是 Tuple。

一个tuple 是表的一行在底层的存储形式，一个 tuple 有自己的内部格式。

```tex
+-------------------+
| HeapTupleHeader   |
+-------------------+
| null bitmap       |
+-------------------+
| user data         |
+-------------------+
```

一个 Item 的格式的文字描述如下：

> All table rows are structured in the same way. There is a fixed-size header (occupying 23 bytes on most machines), followed by an optional null bitmap, an optional object ID field, and the user data. The header is detailed in [Table 66.4](https://www.postgresql.org/docs/current/storage-page-layout.html#HEAPTUPLEHEADERDATA-TABLE). The actual user data (columns of the row) begins at the offset indicated by `t_hoff`, which must always be a multiple of the MAXALIGN distance for the platform.

### 元组头(TupleHeader) 内部格式

**Table 66.4. HeapTupleHeaderData Layout**

| Field       | Type            | Length  | Description                                           |
| ----------- | --------------- | ------- | ----------------------------------------------------- |
| t_xmin      | TransactionId   | 4 bytes | insert XID stamp                                      |
| t_xmax      | TransactionId   | 4 bytes | delete XID stamp                                      |
| t_cid       | CommandId       | 4 bytes | insert and/or delete CID stamp (overlays with t_xvac) |
| t_xvac      | TransactionId   | 4 bytes | XID for VACUUM operation moving a row version         |
| t_ctid      | ItemPointerData | 6 bytes | current TID of this or newer row version              |
| t_infomask2 | uint16          | 2 bytes | number of attributes, plus various flag bits          |
| t_infomask  | uint16          | 2 bytes | various flag bits                                     |
| t_hoff      | uint8           | 1 byte  | offset to user data                                   |

关键部分：

- t_hoff : 用户数据的偏移量

有用的一段话：

> Interpreting the actual data can only be done with information obtained from other tables, mostly `pg_attribute`. The key values needed to identify field locations are `attlen` and `attalign`. There is no way to directly get a particular attribute, except when there are only fixed width fields and no null values. All this trickery is wrapped up in the functions *heap_getattr*, *fastgetattr* and *heap_getsysattr*.
>
> To read the data you need to examine each attribute in turn. First check whether the field is NULL according to the null bitmap. If it is, go to the next. Then make sure you have the right alignment. If the field is a fixed width field, then all the bytes are simply placed. If it's a variable length field (attlen = -1) then it's a bit more complicated. All variable-length data types share the common header structure `struct varlena`, which includes the total length of the stored value and some flag bits. Depending on the flags, the data can be either inline or in a TOAST table; it might be compressed, too (see [Section 66.2](https://www.postgresql.org/docs/current/storage-toast.html)).

---

## 总结

堆文件内部的格式有好几个层级，首先意识到堆文件逻辑上就是Page的数组，即堆文件=连续的Pages。Page 内部分为了五个部分：PageHeaderData、ItemIdData、Free space、Items、Special space。其中[PageHeaderData](#帧头(PageHeader)内部格式)区域又有自己的内部格式。以及Page内保存用户实际数据的区域Items中一个Item，也就是[Tuple](#一个 Item 的内部格式(Tuple格式))也有自己的格式。

## 参考链接

- [Database Page Layout](https://www.postgresql.org/docs/current/storage-page-layout.html)

