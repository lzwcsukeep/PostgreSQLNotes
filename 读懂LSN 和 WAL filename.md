### LSN

The LSN is a 64-bit integer, representing a position in the write-ahead log stream. This 64-bit integer is split into two segments (high 32 bits and low 32 bits). It is printed as two hexadecimal numbers separated by a slash (XXXXXXXX/YYZZZZZZ). The 'X' represents the high 32-bits of the LSN and ‘Y’ is the high 8 bits of the lower 32-bits section. The 'Z' represents the offset position in the file. Each element is a hexadecimal number. The 'X' and 'Y' values are used in the second part of the WAL file on a default PostgreSQL deployment.

理解： LSN 后24位bits 表示该LSN在文件内的偏移量。2^24 次方刚好可以寻址 16MB 的大小。



### WAL File Name

The WAL file name is in the format TTTTTTTTXXXXXXXXYYYYYYYY. Here 'T' is the timeline, 'X' is the high 32-bits from the LSN, and 'Y' is the low 32-bits of the LSN.

解释： 一个 WAL 文件 表示一段连续的 LSN 区间， WAL 文件名包含这段区间的起始LSN 号。

WAL 文件名中的 `XXXXXXXXYYYYYYYY` 来自这个 WAL segment 的起始 LSN