# WAL record 源码分析

```c
author  李志文
```

# 一条 WAL record 的统一布局

>  pg16.0 : src/include/access/xlogrecord.h

```c
/*
 * The overall layout of an XLOG record is:
 *		Fixed-size header (XLogRecord struct)
 *		XLogRecordBlockHeader struct
 *		XLogRecordBlockHeader struct
 *		...
 *		XLogRecordDataHeader[Short|Long] struct
 *		block data
 *		block data
 *		...
 *		main data
 *
 * There can be zero or more XLogRecordBlockHeaders, and 0 or more bytes of
 * rmgr-specific data not associated with a block.  XLogRecord structs
 * always start on MAXALIGN boundaries in the WAL files, but the rest of
 * the fields are not aligned.
 *
 * The XLogRecordBlockHeader, XLogRecordDataHeaderShort and
 * XLogRecordDataHeaderLong structs all begin with a single 'id' byte. It's
 * used to distinguish between block references, and the main data structs.
 */
typedef struct XLogRecord
{
	uint32		xl_tot_len;		/* total len of entire record */
	TransactionId xl_xid;		/* xact id */
	XLogRecPtr	xl_prev;		/* ptr to previous record in log */
	uint8		xl_info;		/* flag bits, see below */
	RmgrId		xl_rmid;		/* resource manager for this record */
	/* 2 bytes of padding here, initialize to zero */
	pg_crc32c	xl_crc;			/* CRC for this record */

	/* XLogRecordBlockHeaders and XLogRecordDataHeader follow, no padding */

} XLogRecord;
```

从 pg16.0 源码 `src/include/access/xlogrecord.h` 中描述可知，所有的 WAL record 都有一个统一的布局结构，依次是：

- Fixed-size header (XLogRecord struct)
 *		XLogRecordBlockHeader struct
 *		XLogRecordBlockHeader struct
 *		...
 *		XLogRecordDataHeader[Short|Long] struct
 *		block data
 *		block data
 *		...
 *		main data

下面依次解释：

1. `struct XLogRecord` 是 WAL record的固定部分，也是 头部信息，每个字段如注释。 

```c
typedef struct XLogRecord
{
	uint32		xl_tot_len;		/* total len of entire record */
	TransactionId xl_xid;		/* xact id */
	XLogRecPtr	xl_prev;		/* ptr to previous record in log */
	uint8		xl_info;		/* flag bits, see below */
	RmgrId		xl_rmid;		/* resource manager for this record */
	/* 2 bytes of padding here, initialize to zero */
	pg_crc32c	xl_crc;			/* CRC for this record */

	/* XLogRecordBlockHeaders and XLogRecordDataHeader follow, no padding */

} XLogRecord;
```

2. `XLogRecordBlockHeader` struct ,zero or more

```c
/*
 * Header info for block data appended to an XLOG record.
 *
 * 'data_length' is the length of the rmgr-specific payload data associated
 * with this block. It does not include the possible full page image, nor
 * XLogRecordBlockHeader struct itself.
 *
 * Note that we don't attempt to align the XLogRecordBlockHeader struct!
 * So, the struct must be copied to aligned local storage before use.
 */
typedef struct XLogRecordBlockHeader
{
	uint8		id;				/* block reference ID */
	uint8		fork_flags;		/* fork within the relation, and flags */
	uint16		data_length;	/* number of payload bytes (not including page
								 * image) */

	/* If BKPBLOCK_HAS_IMAGE, an XLogRecordBlockImageHeader struct follows */
	/* If BKPBLOCK_SAME_REL is not set, a RelFileLocator follows */
	/* BlockNumber follows */
} XLogRecordBlockHeader;
```

紧跟着 XLogRecord后面的块数据的头部信息。

- id : 块引用ID ，标识一个数据块
- fork_flags： flags 信息和 fork 
- data_length: 负载数据占用的字节数

**fork_flags 常用的 flag bit 定义如下：**

```c
/*
 * The fork number fits in the lower 4 bits in the fork_flags field. The upper
 * bits are used for flags.
 */
#define BKPBLOCK_FORK_MASK	0x0F
#define BKPBLOCK_FLAG_MASK	0xF0
#define BKPBLOCK_HAS_IMAGE	0x10	/* block data is an XLogRecordBlockImage */
#define BKPBLOCK_HAS_DATA	0x20
#define BKPBLOCK_WILL_INIT	0x40	/* redo will re-init the page */
#define BKPBLOCK_SAME_REL	0x80	/* RelFileLocator omitted, same as
									 * previous */
```

其中 如果fork_flags 的 `BKPBLOCK_HAS_IMAGE` bit 位被设置了，就代表 block data 是一个 full-page-write 数据。

`BKPBLOCK_HAS_DATA` 用来判断 block 有没有数据。

3. `XLogRecordDataHeaderShort` 和 `XLogRecordDataHeaderLong`

```c
/*
 * XLogRecordDataHeaderShort/Long are used for the "main data" portion of
 * the record. If the length of the data is less than 256 bytes, the short
 * form is used, with a single byte to hold the length. Otherwise the long
 * form is used.
 *
 * (These structs are currently not used in the code, they are here just for
 * documentation purposes).
 */
typedef struct XLogRecordDataHeaderShort
{
	uint8		id;				/* XLR_BLOCK_ID_DATA_SHORT */
	uint8		data_length;	/* number of payload bytes */
}			XLogRecordDataHeaderShort;

#define SizeOfXLogRecordDataHeaderShort (sizeof(uint8) * 2)

typedef struct XLogRecordDataHeaderLong
{
	uint8		id;				/* XLR_BLOCK_ID_DATA_LONG */
	/* followed by uint32 data_length, unaligned */
}			XLogRecordDataHeaderLong;
```

- WAL record 的 main data 的描述信息。

## 什么是 block data 和 main data?

在前面提到了整体布局最后面会有 block data 和 main data，那 block data 和 main data 分别是什么？ 区别是啥？

一条 WAL 日志 可能会涉及修改数据文件的 多个 Page , 一个 Page 也就是一个 block 。因此一条 WAL 日志可能会有多个 block data。

一个 block data 就是该条 WAL 日志对底层数据文件某block 的修改负载。比如 INSERT 语句插入一个元组，那么 block data 里面就会含有该元组。

main data 就是记录 WAL record 的操作信息，一般是一个结构体，比如 INSERT 语句的main data 是一个`xl_heap_insert` struct 。记录了插入的元组存放在Page里的哪个位置。以及元组的flag。

```c
typedef struct xl_heap_insert
{
	OffsetNumber offnum;		/* inserted tuple's offset */
	uint8		flags;

	/* xl_heap_header & TUPLE DATA in backup block 0 */
} xl_heap_insert;
```

还有其他操作的 main data 的定义在 `src/include/access/heapam_xlog.h`：

```c
// delete
typedef struct xl_heap_delete
{
	TransactionId xmax;			/* xmax of the deleted tuple */
	OffsetNumber offnum;		/* deleted tuple's offset */
	uint8		infobits_set;	/* infomask bits */
	uint8		flags;
} xl_heap_delete;

// update
typedef struct xl_heap_update
{
	TransactionId old_xmax;		/* xmax of the old tuple */
	OffsetNumber old_offnum;	/* old tuple's offset */
	uint8		old_infobits_set;	/* infomask bits to set on old tuple */
	uint8		flags;
	TransactionId new_xmax;		/* xmax of the new tuple */
	OffsetNumber new_offnum;	/* new tuple's offset */

	/*
		* If XLH_UPDATE_CONTAINS_OLD_TUPLE or XLH_UPDATE_CONTAINS_OLD_KEY flags
		* are set, xl_heap_header and tuple data for the old tuple follow.
		*/
} xl_heap_update;
```

# lsn、SegNo、WAL fileName 之间的关系

lsn (Log Sequence Number)的定义是 一条 WAL record 在 WAL 日志流中的定位符。定位了 WAL 在日志流中的位置。实际是一个 64 位整数。

WAL file：理论上日志流是一个无限的往后追加的流，实际上我们用 WAL file 把这个无限的流切割成相等的很多个部分，一个部分就是一个 WAL file,也叫 WAL Segment file, 通常一个 WAL file 大小为 16MB ,可以用参数 wal_segment_size控制(initdb --wal-segsize指定)。 

SegNo就是一条日志所在的 Segment 的序号，即这条日志在第几个 Segment 里面。

## 三者转化的宏或函数

- lsn -> SegNo

```c
/*
 * Compute a segment number from an XLogRecPtr.
 *
 * For XLByteToSeg, do the computation at face value.  For XLByteToPrevSeg,
 * a boundary byte is taken to be in the previous segment.  This is suitable
 * for deciding which segment to write given a pointer to a record end,
 * for example.
 */
#define XLByteToSeg(xlrp, logSegNo, wal_segsz_bytes) \
	logSegNo = (xlrp) / (wal_segsz_bytes)
```

- lsn -> 文件内偏移量

```c
#define XLogSegmentOffset(xlogptr, wal_segsz_bytes)	\
	((xlogptr) & ((wal_segsz_bytes) - 1))
```

- SegNo + offset -> lsn

```c
#define XLogSegNoOffsetToRecPtr(segno, offset, wal_segsz_bytes, dest) \
		(dest) = (segno) * (wal_segsz_bytes) + (offset)
```

- timeline ,segNo -> WAL filename

```c
static inline void
XLogFileName(char *fname, TimeLineID tli, XLogSegNo logSegNo, int wal_segsz_bytes)
{
	snprintf(fname, MAXFNAMELEN, "%08X%08X%08X", tli,
			 (uint32) (logSegNo / XLogSegmentsPerXLogId(wal_segsz_bytes)),
			 (uint32) (logSegNo % XLogSegmentsPerXLogId(wal_segsz_bytes)));
}
// SegNo 是一个 segment 的一维序号，真实文件还会对这个序号进行分组，每组组内有固定数量的 segments, 组号用 logId 
// 组内的序号用 seg 表示。然后将 logId 和 seg 编码进文件名
// SegNo = logId * (组内segments数) + seg
// 反之： 
//    logId = SegNo / (组内segments数)
//    seg = SegNo % (组内segments数)
// 可以认为 将一维的 segNo 升维变成了 二维(logId, seg)
```

- WAL filename -> segNo

```c
static inline void
XLogFromFileName(const char *fname, TimeLineID *tli, XLogSegNo *logSegNo, int wal_segsz_bytes)
{
	uint32		log;
	uint32		seg;

	sscanf(fname, "%08X%08X%08X", tli, &log, &seg);
	*logSegNo = (uint64) log * XLogSegmentsPerXLogId(wal_segsz_bytes) + seg;
}
// 文件名就编码了 log 和 seg ,直接从文件名中提取出来即可。
// 然后按照公式计算出 SegNo
```

# WAL record 解码过程

## 数据结构准备

- XLogReaderState 结构体 ： 记录解码全过程信息

  ```c
  struct XLogReaderState
  {
  	XLogReaderRoutine routine;
  	uint64		system_identifier;
  
  	void	   *private_data;
  
  	XLogRecPtr	ReadRecPtr;		/* start of last record read */
  	XLogRecPtr	EndRecPtr;		/* end+1 of last record read */
  
  	XLogRecPtr	abortedRecPtr;
  	XLogRecPtr	missingContrecPtr;
  	XLogRecPtr	overwrittenRecPtr;
  
  	XLogRecPtr	DecodeRecPtr;	/* start of last record decoded */
  	// 下一条记录的开始位置
  	XLogRecPtr	NextRecPtr;		/* end+1 of last record decoded */
  	XLogRecPtr	PrevRecPtr;		/* start of previous record decoded */
  
  	// 当前用户正在使用的 record ，一定是队列头
  	DecodedXLogRecord *record;
  
  	// 这是个环形的缓冲区
  	char	   *decode_buffer;
  	size_t		decode_buffer_size;
  	bool		free_decode_buffer; /* need to free? */
  	char	   *decode_buffer_head; /* data is read from the head */
  	char	   *decode_buffer_tail; /* new data is written at the tail */
  
  	DecodedXLogRecord *decode_queue_head;	/* oldest decoded record */
  	DecodedXLogRecord *decode_queue_tail;	/* newest decoded record */
  
  	// 这里指向的内存大小是 一帧,一个Page
  	char	   *readBuf;
  	uint32		readLen;
  
  	// Segment 上下文信息： 所在目录， segment 大小
  	WALSegmentContext segcxt;
  	// 打开的Segment 信息： tli ,segNo, fd
  	WALOpenSegment seg;
  	uint32		segoff;
  
  	XLogRecPtr	latestPagePtr;
  	TimeLineID	latestPageTLI;
  
  	XLogRecPtr	currRecPtr;
  	TimeLineID	currTLI;
  
  
  	XLogRecPtr	currTLIValidUntil;
  	TimeLineID	nextTLI;
  	// 放 record 的buffer
  	char	   *readRecordBuf;
  	uint32		readRecordBufSize;
  	char	   *errormsg_buf;
  	bool		errormsg_deferred;
  	bool		nonblocking;
  };
  ```

- DecodedXLogRecord struct : 一条WAL record 的解码内容，连续保存

  ```c
  typedef struct DecodedXLogRecord
  {
  	/* Private member used for resource management. */
  	size_t		size;			/* total size of decoded record */
  	// 标记这个解码的记录是放在 decode buffer 还是另外使用 malloc 分配的buffer
  	bool		oversized;		/* outside the regular decode buffer? */
  	struct DecodedXLogRecord *next; /* decoded record queue link */
  
  	/* Public members. */
  	XLogRecPtr	lsn;			/* location */
  	XLogRecPtr	next_lsn;		/* location of next record */
  	XLogRecord	header;			/* header */
  	RepOriginId record_origin;
  	TransactionId toplevel_xid; /* XID of top-level transaction */
  	char	   *main_data;		/* record's main data portion */
  	uint32		main_data_len;	/* main data portion's length */
  	// 标记下面数组的实际元素个数
  	int			max_block_id;	/* highest block_id in use (-1 if none) */
  	DecodedBkpBlock blocks[FLEXIBLE_ARRAY_MEMBER];
  } DecodedXLogRecord;
  ```

- decode buffer

  真正开始解码时，会提前分配一个 decode buffer ，解码出来元组会尝试放在 decode buffer 中，减少反复申请内存带来的开销，提高解码效率。该 buffer 是一个环形缓冲区，缓冲区当前正在使用的部分由 [decode_buffer_head,decode_buffer_tail) 确定。

  定义：

  ```c
  struct XLogReaderState{
      ...
      // 这是个环形的缓冲区
  	char	   *decode_buffer;      /* buffer 起始位置 */
  	size_t		decode_buffer_size; /* buffer 的总大小 */
  	bool		free_decode_buffer; /* need to free? */
  	char	   *decode_buffer_head; /* data is read from the head */
  	char	   *decode_buffer_tail; /* new data is written at the tail */
      ...
  }
  ```

  初始化：

  ```c
  	/* Allocate a circular decode buffer if we don't have one already. */
  	if (unlikely(state->decode_buffer == NULL))
  	{
  		if (state->decode_buffer_size == 0)
  			state->decode_buffer_size = DEFAULT_DECODE_BUFFER_SIZE;
  		state->decode_buffer = malloc(state->decode_buffer_size);  // <<<< malloc 分配
  		state->decode_buffer_head = state->decode_buffer;
  		state->decode_buffer_tail = state->decode_buffer;
  		state->free_decode_buffer = true;
  	}
  ```

- decode queue

  所有解码出来的 record 会 形成一个队列，这个队列使用链表实现，最新解码出来的record 放在队尾，最旧的处在队头，XLogReaderState 结构体里保存指向队头和队尾的指针。

  ```c
  struct XLogReaderState{
      ...
  	DecodedXLogRecord *decode_queue_head;	/* oldest decoded record */
  	DecodedXLogRecord *decode_queue_tail;	/* newest decoded record */
      ...
  }	
  ```

  ```c
  typedef struct DecodedXLogRecord
  {
  	/* Private member used for resource management. */
  	size_t		size;			/* total size of decoded record */
  	bool		oversized;		/* outside the regular decode buffer? */
  	struct DecodedXLogRecord *next; /* decoded record queue link */
  	...
  } DecodedXLogRecord;
  ```

## 解码流程

- 根据 LSN 定位 Segment file ，定位 文件内偏移量。然后读取一帧页面
- 分配XLogReaderState ，分配 decode buffer
- 执行解码
- 将解码出来的记录放入队列尾部

解码流程真正工作函数 `DecodeXLogRecord`：

`DecodeXLogRecord` 思路：

- 使用的是 memcpy 将特定部分的数据拷贝到指定的buffer 位置或者 malloc 分配的内存位置，然后记录该地址
- 解码头部信息，根据头部信息的标志位确定实际的WAL record 的内容

```c
/**
 * DecodeXLogRecord - Decode an XLog record (PG > 14)
 *
 * @state:    XLog reader state
 * @decoded:  Output decoded record structure
 * @record:   Raw XLog record to decode
 * @lsn:      Log sequence number
 * @errormsg: Output error message on failure
 *
 * Parses the XLog record header and data fragments, copying block
 * images and data to properly aligned locations.
 *
 * Returns: true on success, false on error
 */
// record 指示了记录开始的位置
// decoded 指示了要拷贝的起始位置
bool DecodeXLogRecord(XLogReaderState *state,
				 DecodedXLogRecord *decoded,
				 XLogRecord *record,
				 XLogRecPtr lsn,
				 char **errormsg)
{
	/*
	 * read next _size bytes from record buffer, but check for overrun first.
	 */
#define COPY_HEADER_FIELD(_dst, _size)			\
	do {										\
		if (remaining < _size)					\
			goto shortdata_err;					\
		memcpy(_dst, ptr, _size);				\
		ptr += _size;							\
		remaining -= _size;						\
	} while(0)

	char	   *ptr;
	char	   *out;
	uint32		remaining;
	uint32		datatotal;
	RelFileNode *rnode = NULL;
	uint8		block_id;

	decoded->header = *record;
	decoded->lsn = lsn;
	decoded->next = NULL;
	decoded->record_origin = InvalidRepOriginId;
	decoded->toplevel_xid = InvalidTransactionId;
	decoded->main_data = NULL;
	decoded->main_data_len = 0;
	decoded->max_block_id = -1;
	ptr = (char *) record;
	ptr += SizeOfXLogRecord;
	remaining = record->xl_tot_len - SizeOfXLogRecord;

	/* Decode the headers */
	datatotal = 0;
	while (remaining > datatotal)
	{
		COPY_HEADER_FIELD(&block_id, sizeof(uint8));

		if (block_id == XLR_BLOCK_ID_DATA_SHORT)
		{
			/* XLogRecordDataHeaderShort */
			uint8		main_data_len;

			COPY_HEADER_FIELD(&main_data_len, sizeof(uint8));

			decoded->main_data_len = main_data_len;
			datatotal += main_data_len;
			break;				/* by convention, the main data fragment is
								 * always last */
		}
		else if (block_id == XLR_BLOCK_ID_DATA_LONG)
		{
			/* XLogRecordDataHeaderLong */
			uint32		main_data_len;

			COPY_HEADER_FIELD(&main_data_len, sizeof(uint32));
			decoded->main_data_len = main_data_len;
			datatotal += main_data_len;
			break;				/* by convention, the main data fragment is
								 * always last */
		}
		else if (block_id == XLR_BLOCK_ID_ORIGIN)
		{
			COPY_HEADER_FIELD(&decoded->record_origin, sizeof(RepOriginId));
		}
		else if (block_id == XLR_BLOCK_ID_TOPLEVEL_XID)
		{
			COPY_HEADER_FIELD(&decoded->toplevel_xid, sizeof(TransactionId));
		}
		else if (block_id <= XLR_MAX_BLOCK_ID)
		{
			/* XLogRecordBlockHeader */
			DecodedBkpBlock *blk;
			uint8		fork_flags;

			/* mark any intervening block IDs as not in use */
			for (int i = decoded->max_block_id + 1; i < block_id; ++i)
				decoded->blocks[i].in_use = false;

			if (block_id <= decoded->max_block_id)
			{
				printf(
									  "out-of-order block_id %u at %X/%X",
									  block_id,
									  LSN_FORMAT_ARGS(state->ReadRecPtr));
				goto err;
			}
			decoded->max_block_id = block_id;

			blk = &decoded->blocks[block_id];
			blk->in_use = true;
			blk->apply_image = false;

			COPY_HEADER_FIELD(&fork_flags, sizeof(uint8));
			blk->forknum = fork_flags & BKPBLOCK_FORK_MASK;
			blk->flags = fork_flags;
			blk->has_image = ((fork_flags & BKPBLOCK_HAS_IMAGE) != 0);
			blk->has_data = ((fork_flags & BKPBLOCK_HAS_DATA) != 0);

			blk->prefetch_buffer = InvalidBuffer;

			COPY_HEADER_FIELD(&blk->data_len, sizeof(uint16));

			datatotal += blk->data_len;

			if (blk->has_image)
			{
				COPY_HEADER_FIELD(&blk->bimg_len, sizeof(uint16));
				COPY_HEADER_FIELD(&blk->hole_offset, sizeof(uint16));
				COPY_HEADER_FIELD(&blk->bimg_info, sizeof(uint8));

				blk->apply_image = ((blk->bimg_info & BKPIMAGE_APPLY) != 0);

				if (BKPIMAGE_COMPRESSED(blk->bimg_info))
				{
					if (blk->bimg_info & BKPIMAGE_HAS_HOLE)
						COPY_HEADER_FIELD(&blk->hole_length, sizeof(uint16));
					else
						blk->hole_length = 0;
				}
				else
					blk->hole_length = BLCKSZ - blk->bimg_len;
				datatotal += blk->bimg_len;

			}
			COPY_HEADER_FIELD(&blk->blkno, sizeof(BlockNumber));
		}
	}

	if (remaining != datatotal)
		goto shortdata_err;

	/*
	 * Ok, we've parsed the fragment headers, and verified that the total
	 * length of the payload in the fragments is equal to the amount of data
	 * left.  Copy the data of each fragment to contiguous space after the
	 * blocks array, inserting alignment padding before the data fragments so
	 * they can be cast to struct pointers by REDO routines.
	 */
	out = ((char *) decoded) +
		offsetof(DecodedXLogRecord, blocks) +     // 非柔性数组成员
		sizeof(decoded->blocks[0]) * (decoded->max_block_id + 1);   // 柔性数组成员所占位置


	/* block data first */
	for (block_id = 0; block_id <= decoded->max_block_id; block_id++)
	{
		// 开始拷贝
		DecodedBkpBlock *blk = &decoded->blocks[block_id];

		if (!blk->in_use)
			continue;

		Assert(blk->has_image || !blk->apply_image);

		if (blk->has_image)
		{
			/* no need to align image */
			blk->bkp_image = out;
			memcpy(out, ptr, blk->bimg_len);
			ptr += blk->bimg_len;
			out += blk->bimg_len;
		}
		if (blk->has_data)
		{
			out = (char *) MAXALIGN(out);
			blk->data = out;
			memcpy(blk->data, ptr, blk->data_len);
			ptr += blk->data_len;
			out += blk->data_len;
		}
	}

	/* and finally, the main data */
	if (decoded->main_data_len > 0)
	{
		out = (char *) MAXALIGN(out);
		decoded->main_data = out;
		memcpy(decoded->main_data, ptr, decoded->main_data_len);
		ptr += decoded->main_data_len;
		out += decoded->main_data_len;
	}

	/* Report the actual size we used. */
	decoded->size = MAXALIGN(out - (char *) decoded);
	Assert(DecodeXLogRecordRequiredSpace(record->xl_tot_len) >=
		   decoded->size);

	return true;

shortdata_err:
	printf(
						  "record with invalid length at %X/%X",
						  LSN_FORMAT_ARGS(state->ReadRecPtr));
err:
	*errormsg = state->errormsg_buf;

	return false;
}
```

## 分配 buffer

```c
static DecodedXLogRecord *
XLogReadRecordAlloc(XLogReaderState *state, size_t xl_tot_len, bool allow_oversized)
{
	size_t		required_space = DecodeXLogRecordRequiredSpace(xl_tot_len);
	DecodedXLogRecord *decoded = NULL;

	/* Allocate a circular decode buffer if we don't have one already. */
	if (unlikely(state->decode_buffer == NULL))
	{
		if (state->decode_buffer_size == 0)
			state->decode_buffer_size = DEFAULT_DECODE_BUFFER_SIZE;
		state->decode_buffer = malloc(state->decode_buffer_size);
		state->decode_buffer_head = state->decode_buffer;
		state->decode_buffer_tail = state->decode_buffer;
		state->free_decode_buffer = true;
	}

	/* Try to allocate space in the circular decode buffer. */
	if (state->decode_buffer_tail >= state->decode_buffer_head)
	{
		/* Empty, or tail is to the right of head. */
		if (required_space <=
			state->decode_buffer_size -
			(state->decode_buffer_tail - state->decode_buffer))
		{
			/*-
			 * There is space between tail and end.
			 *
			 * +-----+--------------------+-----+
			 * |     |////////////////////|here!|
			 * +-----+--------------------+-----+
			 *       ^                    ^
			 *       |                    |
			 *       h                    t
			 */
			decoded = (DecodedXLogRecord *) state->decode_buffer_tail;
			decoded->oversized = false;
			return decoded;
		}
		else if (required_space <
				 state->decode_buffer_head - state->decode_buffer)
		{
			/*-
			 * There is space between start and head.
			 *
			 * +-----+--------------------+-----+
			 * |here!|////////////////////|     |
			 * +-----+--------------------+-----+
			 *       ^                    ^
			 *       |                    |
			 *       h                    t
			 */
			decoded = (DecodedXLogRecord *) state->decode_buffer;
			decoded->oversized = false;
			return decoded;
		}
	}
	else
	{
		/* Tail is to the left of head. */
		if (required_space <
			state->decode_buffer_head - state->decode_buffer_tail)
		{
			/*-
			 * There is space between tail and head.
			 *
			 * +-----+--------------------+-----+
			 * |/////|here!               |/////|
			 * +-----+--------------------+-----+
			 *       ^                    ^
			 *       |                    |
			 *       t                    h
			 */
			decoded = (DecodedXLogRecord *) state->decode_buffer_tail;
			decoded->oversized = false;
			return decoded;
		}
	}

	/* Not enough space in the decode buffer.  Are we allowed to allocate? */
	if (allow_oversized)
	{
		decoded = malloc(required_space);
		decoded->oversized = true;
		return decoded;
	}

	return NULL;
}
```

## 放入队列

```c
XLogPageReadResult XLogDecodeNextRecord(XLogReaderState *state, bool nonblocking)
{
	XLogRecPtr	RecPtr;
	XLogRecord *record;
	XLogRecPtr	targetPagePtr;
	bool		randAccess;
	uint32		len,
				total_len;
	uint32		targetRecOff;
	uint32		pageHeaderSize;
	bool		assembled;
	bool		gotheader;
	int			readOff;
	DecodedXLogRecord *decoded;
	char	   *errormsg;		/* not used */

	/*
	 * randAccess indicates whether to verify the previous-record pointer of
	 * the record we're reading.  We only do this if we're reading
	 * sequentially, which is what we initially assume.
	 */
	randAccess = false;

	/* reset error state */
	state->errormsg_buf[0] = '\0';
	decoded = NULL;

	state->abortedRecPtr = InvalidXLogRecPtr;
	state->missingContrecPtr = InvalidXLogRecPtr;

	RecPtr = state->NextRecPtr;

	if (state->DecodeRecPtr != InvalidXLogRecPtr)
	{
		/* read the record after the one we just read */

		/*
		 * NextRecPtr is pointing to end+1 of the previous WAL record.  If
		 * we're at a page boundary, no more records can fit on the current
		 * page. We must skip over the page header, but we can't do that until
		 * we've read in the page, since the header size is variable.
		 */
	}
	else
	{
		/*
		 * Caller supplied a position to start at.
		 *
		 * In this case, NextRecPtr should already be pointing either to a
		 * valid record starting position or alternatively to the beginning of
		 * a page. See the header comments for XLogBeginRead.
		 */
		Assert(RecPtr % XLOG_BLCKSZ == 0 || XRecOffIsValid(RecPtr));
		randAccess = true;
	}

restart:
	state->nonblocking = nonblocking;
	state->currRecPtr = RecPtr;
	assembled = false;

	// 下面这行等价于 向下对齐到 page 边界
	targetPagePtr = RecPtr - (RecPtr % XLOG_BLCKSZ);
	// targetRecOff 为相对于 Page 起始位置的偏移量
	targetRecOff = RecPtr % XLOG_BLCKSZ;

	/*
	 * Read the page containing the record into state->readBuf. Request enough
	 * byte to cover the whole record header, or at least the part of it that
	 * fits on the same page.
	 */
	// 1. 读取一帧
	readOff = ReadPageInternal(state, targetPagePtr,
							   Min(targetRecOff + SizeOfXLogRecord, XLOG_BLCKSZ));
	if (readOff == XLREAD_WOULDBLOCK)
		return XLREAD_WOULDBLOCK;
	else if (readOff < 0)
		goto err;

	/*
	 * ReadPageInternal always returns at least the page header, so we can
	 * examine it now.
	 */
	pageHeaderSize = XLogPageHeaderSize((XLogPageHeader) state->readBuf);
	if (targetRecOff == 0)
	{
		/*
		 * At page start, so skip over page header.
		 */
		RecPtr += pageHeaderSize;
		targetRecOff = pageHeaderSize;
	}
	else if (targetRecOff < pageHeaderSize)
	{
		printf("invalid record offset at %X/%X (expected >=%u, got %u)\n",
							  LSN_FORMAT_ARGS(RecPtr),
							  pageHeaderSize, targetRecOff);
		goto err;
	}

	if ((((XLogPageHeader) state->readBuf)->xlp_info & XLP_FIRST_IS_CONTRECORD) &&
		targetRecOff == pageHeaderSize)
	{
		printf("contrecord is requested by %X/%X\n",
							  LSN_FORMAT_ARGS(RecPtr));
		goto err;
	}

	/* ReadPageInternal has verified the page header */
	Assert(pageHeaderSize <= readOff);

	/*
	 * Read the record length.
	 *
	 * NB: Even though we use an XLogRecord pointer here, the whole record
	 * header might not fit on this page. xl_tot_len is the first field of the
	 * struct, so it must be on this page (the records are MAXALIGNed), but we
	 * cannot access any other fields until we've verified that we got the
	 * whole header.
	 */
	// RecPtr % XLOG_BLCKSZ 就能计算出 record 在当前 page 内的偏移
	// 2. 计算 RecPtr 对应的记录在帧内的偏移量，强制转化成 record
	record = (XLogRecord *) (state->readBuf + RecPtr % XLOG_BLCKSZ);
	total_len = record->xl_tot_len;

	/*
	 * If the whole record header is on this page, validate it immediately.
	 * Otherwise do just a basic sanity check on xl_tot_len, and validate the
	 * rest of the header after reading it from the next page.  The xl_tot_len
	 * check is necessary here to ensure that we enter the "Need to reassemble
	 * record" code path below; otherwise we might fail to apply
	 * ValidXLogRecordHeader at all.
	 */
	if (targetRecOff <= XLOG_BLCKSZ - SizeOfXLogRecord)
	{
		if (!ValidXLogRecordHeader(state, RecPtr, state->DecodeRecPtr, record,
								   randAccess))
			goto err;
		gotheader = true;
	}
	else
	{
		/* There may be no next page if it's too small. */
		if (total_len < SizeOfXLogRecord)
		{

			printf(
#ifdef CN
								  "读取到最后的日志段： %X/%X\n",
#else
								  "read until FINAL WAL SEGMENT: %X/%X\n",
#endif
								  LSN_FORMAT_ARGS(RecPtr),
								  (uint32) SizeOfXLogRecord, total_len);
			goto err;
		}
		/* We'll validate the header once we have the next page. */
		gotheader = false;
	}

	/*
	 * Try to find space to decode this record, if we can do so without
	 * calling palloc.  If we can't, we'll try again below after we've
	 * validated that total_len isn't garbage bytes from a recycled WAL page.
	 */
	decoded = XLogReadRecordAlloc(state,
								  total_len,
								  false /* allow_oversized */ );
	if (decoded == NULL && nonblocking)
	{
		/*
		 * There is no space in the circular decode buffer, and the caller is
		 * only reading ahead.  The caller should consume existing records to
		 * make space.
		 */
		return XLREAD_WOULDBLOCK;
	}

	len = XLOG_BLCKSZ - RecPtr % XLOG_BLCKSZ;
	if (total_len > len)
	{
		/* Need to reassemble record */
		char	   *contdata;
		XLogPageHeader pageHeader;
		char	   *buffer;
		uint32		gotlen;

		assembled = true;

		/*
		 * We always have space for a couple of pages, enough to validate a
		 * boundary-spanning record header.
		 */
		Assert(state->readRecordBufSize >= XLOG_BLCKSZ * 2);
		Assert(state->readRecordBufSize >= len);

		/* Copy the first fragment of the record from the first page. */
		memcpy(state->readRecordBuf,
			   state->readBuf + RecPtr % XLOG_BLCKSZ, len);
		buffer = state->readRecordBuf + len;
		gotlen = len;

		do
		{
			/* Calculate pointer to beginning of next page */
			targetPagePtr += XLOG_BLCKSZ;

			/* Wait for the next page to become available */
			readOff = ReadPageInternal(state, targetPagePtr,
									   Min(total_len - gotlen + SizeOfXLogShortPHD,
										   XLOG_BLCKSZ));

			if (readOff == XLREAD_WOULDBLOCK)
				return XLREAD_WOULDBLOCK;
			else if (readOff < 0)
				goto err;

			Assert(SizeOfXLogShortPHD <= readOff);

			pageHeader = (XLogPageHeader) state->readBuf;

			/*
			 * If we were expecting a continuation record and got an
			 * "overwrite contrecord" flag, that means the continuation record
			 * was overwritten with a different record.  Restart the read by
			 * assuming the address to read is the location where we found
			 * this flag; but keep track of the LSN of the record we were
			 * reading, for later verification.
			 */
			if (pageHeader->xlp_info & XLP_FIRST_IS_OVERWRITE_CONTRECORD)
			{
				state->overwrittenRecPtr = RecPtr;
				RecPtr = targetPagePtr;
				goto restart;
			}

			/* Check that the continuation on next page looks valid */
			if (!(pageHeader->xlp_info & XLP_FIRST_IS_CONTRECORD))
			{
				printf(
									  "there is no contrecord flag at %X/%X",
									  LSN_FORMAT_ARGS(RecPtr));
				goto err;
			}

			/*
			 * Cross-check that xlp_rem_len agrees with how much of the record
			 * we expect there to be left.
			 */
			if (pageHeader->xlp_rem_len == 0 ||
				total_len != (pageHeader->xlp_rem_len + gotlen))
			{
				printf(
									  "invalid contrecord length %u (expected %lld) at %X/%X",
									  pageHeader->xlp_rem_len,
									  ((long long) total_len) - gotlen,
									  LSN_FORMAT_ARGS(RecPtr));
				goto err;
			}

			/* Append the continuation from this page to the buffer */
			pageHeaderSize = XLogPageHeaderSize(pageHeader);

			if (readOff < pageHeaderSize)
				readOff = ReadPageInternal(state, targetPagePtr,
										   pageHeaderSize);

			Assert(pageHeaderSize <= readOff);

			contdata = (char *) state->readBuf + pageHeaderSize;
			len = XLOG_BLCKSZ - pageHeaderSize;
			if (pageHeader->xlp_rem_len < len)
				len = pageHeader->xlp_rem_len;

			if (readOff < pageHeaderSize + len)
				readOff = ReadPageInternal(state, targetPagePtr,
										   pageHeaderSize + len);

			memcpy(buffer, (char *) contdata, len);
			buffer += len;
			gotlen += len;

			/* If we just reassembled the record header, validate it. */
			if (!gotheader)
			{
				record = (XLogRecord *) state->readRecordBuf;
				if (!ValidXLogRecordHeader(state, RecPtr, state->DecodeRecPtr,
										   record, randAccess))
					goto err;
				gotheader = true;
			}

			/*
			 * We might need a bigger buffer.  We have validated the record
			 * header, in the case that it split over a page boundary.  We've
			 * also cross-checked total_len against xlp_rem_len on the second
			 * page, and verified xlp_pageaddr on both.
			 */
			if (total_len > state->readRecordBufSize)
			{
				char		save_copy[XLOG_BLCKSZ * 2];

				/*
				 * Save and restore the data we already had.  It can't be more
				 * than two pages.
				 */
				Assert(gotlen <= lengthof(save_copy));
				Assert(gotlen <= state->readRecordBufSize);
				memcpy(save_copy, state->readRecordBuf, gotlen);
				allocate_recordbuf(state, total_len);
				memcpy(state->readRecordBuf, save_copy, gotlen);
				buffer = state->readRecordBuf + gotlen;
			}
		} while (gotlen < total_len);
		Assert(gotheader);

		record = (XLogRecord *) state->readRecordBuf;

		pageHeaderSize = XLogPageHeaderSize((XLogPageHeader) state->readBuf);
		state->DecodeRecPtr = RecPtr;
		state->NextRecPtr = targetPagePtr + pageHeaderSize
			+ MAXALIGN(pageHeader->xlp_rem_len);
	}
	else
	{
		/* Wait for the record data to become available */
		readOff = ReadPageInternal(state, targetPagePtr,
								   Min(targetRecOff + total_len, XLOG_BLCKSZ));
		if (readOff == XLREAD_WOULDBLOCK)
			return XLREAD_WOULDBLOCK;
		else if (readOff < 0)
			goto err;

		/* Record does not cross a page boundary */

		state->NextRecPtr = RecPtr + MAXALIGN(total_len);

		state->DecodeRecPtr = RecPtr;
	}

	/*
	 * Special processing if it's an XLOG SWITCH record
	 */
	if (record->xl_rmid == 0 &&
		(record->xl_info & ~XLR_INFO_MASK) == XLOG_SWITCH)
	{
		/* Pretend it extends to end of segment */
		state->NextRecPtr += state->segcxt.ws_segsize - 1;
		state->NextRecPtr -= XLogSegmentOffset(state->NextRecPtr, state->segcxt.ws_segsize);
	}

	/*
	 * If we got here without a DecodedXLogRecord, it means we needed to
	 * validate total_len before trusting it, but by now now we've done that.
	 */
	if (decoded == NULL)
	{
		Assert(!nonblocking);
		decoded = XLogReadRecordAlloc(state,
									  total_len,
									  true /* allow_oversized */ );
		/* allocation should always happen under allow_oversized */
		Assert(decoded != NULL);
	}

	// 3. 解码到 decode buffer 中
	if (DecodeXLogRecord(state, decoded, record, RecPtr, &errormsg))
	{
		/* Record the location of the next record. */
		decoded->next_lsn = state->NextRecPtr;

		/*
		 * If it's in the decode buffer, mark the decode buffer space as
		 * occupied.
		 */
		if (!decoded->oversized)
		{
			/* The new decode buffer head must be MAXALIGNed. */
			Assert(decoded->size == MAXALIGN(decoded->size));
			if ((char *) decoded == state->decode_buffer)
				state->decode_buffer_tail = state->decode_buffer + decoded->size;
			else
				state->decode_buffer_tail += decoded->size;
		}

		/* Insert it into the queue of decoded records. */
		Assert(state->decode_queue_tail != decoded);
		// 4. 成功解码一个记录，添加到队列中，在队列尾部追加，并且更新尾部指针
		if (state->decode_queue_tail)
			state->decode_queue_tail->next = decoded;
		state->decode_queue_tail = decoded;
		// 头部空说明是第一个
		if (!state->decode_queue_head)
			state->decode_queue_head = decoded;
		return XLREAD_SUCCESS;
	}

err:
	if (assembled)
	{
		/*
		 * We get here when a record that spans multiple pages needs to be
		 * assembled, but something went wrong -- perhaps a contrecord piece
		 * was lost.  If caller is WAL replay, it will know where the aborted
		 * record was and where to direct followup WAL to be written, marking
		 * the next piece with XLP_FIRST_IS_OVERWRITE_CONTRECORD, which will
		 * in turn signal downstream WAL consumers that the broken WAL record
		 * is to be ignored.
		 */
		state->abortedRecPtr = RecPtr;
		state->missingContrecPtr = targetPagePtr;

		/*
		 * If we got here without reporting an error, make sure an error is
		 * queued so that XLogPrefetcherReadRecord() doesn't bring us back a
		 * second time and clobber the above state.
		 */
		state->errormsg_deferred = true;
	}

	if (decoded && decoded->oversized)
		free(decoded);

	/*
	 * Invalidate the read state. We might read from a different source after
	 * failure.
	 */

	/*
	 * If an error was written to errmsg_buf, it'll be returned to the caller
	 * of XLogReadRecord() after all successfully decoded records from the
	 * read queue.
	 */

	return XLREAD_FAIL;
}
```

# 总结

本篇文章讲解了 postgresql WAL 的相关知识，分析了 WAL record 的源码。

- WAL 日志里面 一条 record 的布局，每个 record 都分为固定部分 XLogRecord 头部、0个或多个 BlockHeader、DataHeader、0个或多个 block data 和 main data。
- 如何从 lsn 定位到 Segment file 和 Segment file 文件内部的偏移。以及 lsn 到 segNo, WAL file Name 的计算和转化。
- WAL 一条 record 如何解码，涉及到的数据结构，解码的函数等细节。具体如下。
- record 解码用到的 decode buffer , decode queue 如何工作的。
- record 的 XLogReaderState 结构体各个字段含义，工作流程。

