解释一下下述源代码中

- if (header->xlp_info & XLP_FIRST_IS_CONTRECORD)
- 以及continuation 是什么？

```c
XLogRecPtr XLogFindNextRecord(XLogReaderState *state, XLogRecPtr RecPtr)
{
		XLogPageHeader header;
		/* Read the page containing the record */
		readLen = ReadPageInternal(state, targetPagePtr, targetRecOff);
		header = (XLogPageHeader) state->readBuf;
		...
		/* skip over potential continuation data */
		if (header->xlp_info & XLP_FIRST_IS_CONTRECORD)
		{
			/*
			 * If the length of the remaining continuation data is more than
			 * what can fit in this page, the continuation record crosses over
			 * this page. Read the next page and try again. xlp_rem_len in the
			 * next page header will contain the remaining length of the
			 * continuation data
			 *
			 * Note that record headers are MAXALIGN'ed
			 */
			if (MAXALIGN(header->xlp_rem_len) >= (XLOG_BLCKSZ - pageHeaderSize))
				tmpRecPtr = targetPagePtr + XLOG_BLCKSZ;
			else
			{
				/*
				 * The previous continuation record ends in this page. Set
				 * tmpRecPtr to point to the first valid record
				 */
				tmpRecPtr = targetPagePtr + pageHeaderSize
					+ MAXALIGN(header->xlp_rem_len);
				break;
			}
		}
		else
		{
			tmpRecPtr = targetPagePtr + pageHeaderSize;
			break;
		}
}
```

----

答：

这里的核心是：**WAL record 可能跨页（cross page）存储**。
一个 WAL page（通常 8KB）放不下完整 record 时，record 会被拆成多段：

- 前一页放 record 的前半部分
- 下一页开头放剩余部分

这种“某条 record 的剩余部分”就叫：

> continuation record（续页数据 / continuation data）

所以：

```c
if (header->xlp_info & XLP_FIRST_IS_CONTRECORD)
```

的意思是：

> 当前 WAL page 的“第一页数据”并不是一个新 record 的开始，
> 而是上一页那个 record 的 continuation（剩余部分）。

------

# 一、先理解 WAL page 的布局

一个 WAL page 大概长这样：

```text
+-------------------+
| Page Header       |
+-------------------+
| Record A          |
| Record B (partial)|  <-- 放不下了
+-------------------+
```

下一页：

```text
+-------------------+
| Page Header       |
| continuation of B |
| Record C          |
| Record D          |
+-------------------+
```

注意：

下一页最开始不是新 record。

而是：

```text
Record B 的剩余数据
```

这就是：

```text
continuation data
```

------

# 二、XLP_FIRST_IS_CONTRECORD 是什么

这是 WAL page header 的一个 flag。

定义类似：

```c
#define XLP_FIRST_IS_CONTRECORD 0x0001
```

含义：

> 当前 page 的第一个数据不是新 record，
> 而是上一页 record 的 continuation。

即：

```text
this page starts with a fragment
```

------

# 三、这一段代码在干什么

代码：

```c
if (header->xlp_info & XLP_FIRST_IS_CONTRECORD)
```

是在判断：

```text
当前 page 开头是不是 continuation data
```

------

# 四、为什么需要这个判断

因为：

```c
XLogFindNextRecord()
```

的目标是：

> 找到“真正 record 的起始位置”

但用户给的 `RecPtr`：

```text
可能正好落在 continuation 中间
```

例如：

```text
Page1:
  Record A
  Record B(partial)

Page2:
  continuation of B
  Record C
```

如果：

```c
RecPtr 指向 Page2 开头
```

那么：

```text
它实际上位于 Record B 的中间
```

不是合法 record 起点。

因此必须：

```text
跳过 continuation
```

找到：

```text
Record C
```

的真正开始位置。

------

# 五、这一段逻辑的意思

核心：

```c
if (header->xlp_info & XLP_FIRST_IS_CONTRECORD)
```

说明：

```text
page 开头是 continuation
```

于是：

------

## 情况1：continuation 占满整页

```c
if (MAXALIGN(header->xlp_rem_len) >=
    (XLOG_BLCKSZ - pageHeaderSize))
```

意思：

```text
剩余 continuation 数据太大，
这一整页都还在 continuation
```

于是：

```c
tmpRecPtr = targetPagePtr + XLOG_BLCKSZ;
```

跳到下一页继续看。

------

## 情况2：continuation 在本页结束

例如：

```text
+-------------------+
| page header       |
+-------------------+
| continuation      |
| continuation end  |
+-------------------+
| Record C          |
| Record D          |
+-------------------+
```

那么：

```c
tmpRecPtr =
    targetPagePtr +
    pageHeaderSize +
    MAXALIGN(header->xlp_rem_len);
```

意思：

```text
跳过 continuation 数据
```

最终：

```text
tmpRecPtr 指向 Record C
```

------

# 六、xlp_rem_len 是什么

这是 page header 里的字段。

含义：

```text
这一页开头 continuation 还剩多少字节
```

例如：

Record B 总共还差 1200 字节。

那么：

```c
header->xlp_rem_len = 1200
```

表示：

```text
page header 后面前 1200 字节属于 continuation
```

不是新 record。

------

# 七、为什么注释说：

```c
we know now that tmpRecPtr is an address pointing to a valid XLogRecord
```

意思：

经过前面的跳转以后：

```text
tmpRecPtr 已经不在 continuation 里面了
```

因此它一定：

- 要么位于 page header 后第一个 record
- 要么位于 continuation 后第一个完整 record

即：

```text
一定是合法 XLogRecord 起点
```

------

# 八、举一个完整例子

假设：

```text
page size = 8192
header = 24
```

第一页：

```text
24B header
7000B Record A
1000B Record B(partial)
```

第二页：

```text
24B header
500B continuation of B
2000B Record C
```

第二页 header：

```c
xlp_info |= XLP_FIRST_IS_CONTRECORD
xlp_rem_len = 500
```

那么代码：

```c
tmpRecPtr =
    page_start +
    pageHeaderSize +
    MAXALIGN(500)
```

结果：

```text
tmpRecPtr -> Record C
```

------

# 九、为什么 WAL 要允许 continuation

因为：

```text
一个 WAL record 可以很大
```

比如：

- full page image
- 大事务
- 大 tuple
- heap rewrite

都可能超过：

```text
8KB WAL page
```

所以 PostgreSQL 的 WAL record：

```text
天然支持跨页存储
```

------

# 十、总结一句话

```c
if (header->xlp_info & XLP_FIRST_IS_CONTRECORD)
```

表示：

> 当前 WAL page 开头不是新 record，
> 而是上一页 record 的 continuation（剩余部分）。

而这整个循环的目的就是：

```text
跳过 continuation data，
找到真正的 WAL record 起点。
```