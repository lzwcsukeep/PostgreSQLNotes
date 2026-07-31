## 一、什么是WAL ？

PostgreSQL 的 **WAL (Write-Ahead Logging，预写式日志)** 是保障事务 **ACID** 特性、尤其是崩溃恢复与数据一致性的核心机制。它的核心规则是：**数据文件的变更必须在其对应的 WAL 日志写入磁盘后，才能写入数据文件**。

---

### 1. 核心概念与原理
- **定义**：WAL 是一种标准的事务日志技术，通过“日志先行”确保数据安全。事务提交时，仅需将 WAL 刷盘即可，无需同步刷所有脏数据页，大幅降低随机 I/O 开销。
- **核心流程**：
  1.  执行 `INSERT/UPDATE/DELETE` 时，先生成 WAL 记录，写入 **WAL 缓冲区**（内存）。
  2.  事务提交时，WAL 缓冲区被强制刷盘（依赖 `fsync` 参数控制）。
  3.  数据页修改仅留在共享缓冲区，后续由后台进程异步刷入数据文件（脏页写回）。
  4.  崩溃恢复时，从重放最近检查点开始的 WAL，重做所有未刷盘的变更，实现一致性恢复。

### 2. 关键作用
1.  **崩溃恢复（REDO）**：异常重启后，通过重放 WAL 完成“前滚恢复”，保证数据不丢失。
2.  **高性能写入**：WAL 为**顺序写入**，远快于数据文件的随机写，显著提升事务吞吐量。
3.  **高可用与备份**：支持流复制、逻辑复制，结合基础备份可实现 **任意时间点恢复（PITR）**。
4.  **原子性与一致性**：未刷盘的 WAL 对应的事务变更会被丢弃，确保事务原子性。

### 3. 核心概念详解
| 概念 | 说明 |
|:--- |:--- |
| **LSN (Log Sequence Number)** |  WAL 记录的唯一位置标识，全局递增，用于定位日志位置（如 `0/15DFB708`）。 |
| **WAL 缓冲区** |  内存中的临时存储，批量刷盘减少 I/O，提升效率。 |
| **检查点 (Checkpoint)** |  定期触发的操作，将所有脏数据页刷盘，并写入 WAL 检查点记录。崩溃恢复从此点开始重放 WAL。 |
| **WAL 归档** |  启用 `archive_mode` 后，WAL 文件写满或检查点时归档至指定目录，用于备份与恢复。 |

### 4. 关键配置参数（postgresql.conf）
| 参数 | 作用 | 建议值 |
|:--- |:--- |:--- |
| **wal_level** |  WAL 详细程度：`minimal`（最小）、`replica`（默认，支持归档/复制）、`logical`（支持逻辑解码）。 | 按需调整，生产常用 `replica` |
| **fsync** |  是否启用 WAL 刷盘与数据一致性校验。关闭可提升性能但存在风险。 | `on`（生产环境必须开启） |
| **wal_sync_method** |  WAL 刷盘方式（如 `fsync`/`fdatasync`）。 | 系统默认即可 |
| **max_wal_size** |  WAL 最大尺寸，触发检查点与切换 WAL 文件。 | 1GB–4GB（依业务调整） |
| **archive_mode** |  启用 WAL 归档，配合 `archive_command` 实现 PITR。 | `on`（需备份时） |
| **archive_command** |  归档 WAL 的命令（如 `cp %p /archive/%f`）。 | 配置专属归档目录 |


postgresql.conf 文件WAL 相关默认配置：

```
#------------------------------------------------------------------------------
# WRITE-AHEAD LOG
#------------------------------------------------------------------------------

# - Settings -

#wal_level = replica			# minimal, replica, or logical
					# (change requires restart)
#fsync = on				# flush data to disk for crash safety
					# (turning this off can cause
					# unrecoverable data corruption)
#synchronous_commit = on		# synchronization level;
					# off, local, remote_write, remote_apply, or on
#wal_sync_method = fsync		# the default is the first option
					# supported by the operating system:
					#   open_datasync
					#   fdatasync (default on Linux and FreeBSD)
					#   fsync
					#   fsync_writethrough
					#   open_sync
#full_page_writes = on			# recover from partial page writes
#wal_log_hints = off			# also do full page writes of non-critical updates
					# (change requires restart)
#wal_compression = off			# enables compression of full-page writes;
					# off, pglz, lz4, zstd, or on
#wal_init_zero = on			# zero-fill new WAL files
#wal_recycle = on			# recycle WAL files
#wal_buffers = -1			# min 32kB, -1 sets based on shared_buffers
					# (change requires restart)
#wal_writer_delay = 200ms		# 1-10000 milliseconds
#wal_writer_flush_after = 1MB		# measured in pages, 0 disables
#wal_skip_threshold = 2MB

# - Checkpoints -

#checkpoint_timeout = 5min		# range 30s-1d
#checkpoint_completion_target = 0.9	# checkpoint target duration, 0.0 - 1.0
#checkpoint_flush_after = 256kB		# measured in pages, 0 disables
#checkpoint_warning = 30s		# 0 disables
max_wal_size = 1GB
min_wal_size = 80MB

```

### 5. 常见操作与运维
1.  **查看 WAL 信息**
    -   查看当前 WAL 位置：`SELECT pg_current_wal_lsn();`
    -   查看当前 WAL 位置对应的WAL 文件：`SELECT pg_walfile_name(pg_current_wal_lsn());` 
    -   pg_switch_wal () : 强制服务器切换WAL文件，使得当前WAL 文件可以被归档
2.  **手动触发检查点**
    -   `CHECKPOINT;`（需超级用户权限）
3.  **WAL 清理**
    -   自动回收：完成归档或不再需要的 WAL 段会被自动删除。
    -   手动清理：仅在确认安全时使用，避免数据丢失。

## 二、LSN(Log Sequence Number)

定义：LSN = WAL 日志中的“位置指针 / 偏移量”，它是一个 64 位整数，用来唯一标识：

> **某一条 WAL 记录在日志流中的位置**

因为用户的操作是不断的，可以把 WAL 看成一条不断追加的日志流，lsn 就是用来记录某条日志在这个流中的位置。

LSN 是一个不断递增的64位整数，被分为高 32 位和 低32位。

**PostgreSQL 中的表示形式**

通常看到的是：

```
0/16B6C50
```

结构：

```
[高32位]/[低32位]
```

## 三、Checkpoint

checkpoint 概念: 把内存中的“脏数据页”刷到磁盘，并在 WAL 中记录一个“恢复起点”的过程。

作用：把脏数据页刷到磁盘，并记录一个“恢复起点”

好处：减少恢复时需要 replay 的 WAL 数量

### 1. checkpoint 做的事情

- 刷脏页（最重要）： 把 shared buffers 中的 dirty pages 写入磁盘
- 记录 checkpoint LSN ： 将checkpoint LSN 写入控制文件（control file),表示“恢复从这里开始”
- 生成 checkpoint WAL 记录 ： 写一条 WAL：
  - 标记 checkpoint
  - 记录一致性点

### 2.  checkpoint 解决的核心问题

- 控制恢复时间

  ​	恢复只需要：checkpoint → crash

  ​	而不是全部 WAL

- 防止 WAL 无限增长

  如果没有 checkpoint： WAL 不能删除

- 允许回收 WAL

  checkpoint 完成后：checkpoint 之前的 WAL 可以 recycle

### 3. checkpoint 和 WAL 的关系

checkpoint 不替代 WAL，它只是“标记 WAL 的恢复边界”。WAL 仍然必须存在

因为：

- checkpoint 只保证“数据页落盘”
- 但 crash 可能发生在 checkpoint 之后

### 4. checkpoint 的触发机制

- 时间触发

```
checkpoint_timeout = 5min
```

- WAL 大小触发

```
max_wal_size
```

- 手动触发

```
CHECKPOINT;
```

## 四、WAL 日志 dump 实操

借助 `pg_waldump` 工具，我们可以用文本的方式读取 WAL 日志。

1.   使用 psql 客户端在表 t1 中插入一条数据，记录插入前后的 LSN：

![image-20260423141737871](WAL 技术介绍-双周技术总结版.assets/image-20260423141737871.png)

2.  使用 `pg_waldump` 查看该插入操作的日志信息：

![image-20260423141912717](WAL 技术介绍-双周技术总结版.assets/image-20260423141912717.png)

从 WAL 日志中可以看到 一条插入语句实际上会产生两条日志，一条对对文件的修改日志，一条事务提交的日志。解释一下这个日志的输出信息。

- rmgr ： 资源管理器，表示这条日志操作的对象，Heap 表示对堆文件的修改
- len(rec/tot) : 表示这条日志的长度
- tx: 这条日志对应的事务号
- lsn: 本条日志的位置标识，是一个64位不断增长的整数。分为高 32 位 和 低32位两部分
- desc: 日志描述信息
- off : 这条数据被插入的元组偏移量，也就是被插入到了Page 的第十七个元组的位置。

## 五、WAL 重放（redo）恢复机制

这是 PostgreSQL 崩溃恢复的核心。

### 1. 崩溃场景

假设：

- WAL 已写
- 数据页还没刷

此时宕机：数据页是“旧的”，但 WAL 是“新的”

### 2. 恢复思路

**用 WAL 把数据“重新做一遍”**

这就是 redo：

> 再执行一次已经记录的修改

### 3. 恢复流程（非常关键）

启动数据库时：

- 第一步：找到 checkpoint

从控制文件读取：

```
checkpoint LSN
```

**从这里开始恢复（而不是从头）**

- 第二步：从 checkpoint 开始扫描 WAL

- 第三步：逐条 redo WAL

伪流程：

```
for each WAL record:
    if (page.pd_lsn >= record.lsn)
        skip   // 已经应用过
    else
        apply redo
```

- 第四步：恢复完成
  - 数据页恢复到一致状态
  - 数据库可用