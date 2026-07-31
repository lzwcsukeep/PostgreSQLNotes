问： 解释一下 checkpoint 的原理

---

在 PostgreSQL 里，**checkpoint（检查点）**本质上是 WAL 体系里的一个“断点标记 + 刷盘同步点”，用来控制崩溃恢复的成本。

可以用一句话先抓住核心：

> 👉 **Checkpoint = 把内存中的脏数据页“批量落盘”，并在 WAL 中记录一个恢复起点**

------

# 一、为什么需要 checkpoint？

如果没有 checkpoint：

- 数据修改一直写 WAL（顺序写）
- 数据页一直在内存（dirty pages）
- 崩溃恢复时必须从“数据库启动以来所有 WAL”开始重放

👉 问题：

- WAL 会无限增长
- 恢复时间会越来越长

------

## checkpoint 的作用

👉 **把“恢复范围”切断**

让恢复变成：

```text
从 checkpoint LSN → crash LSN
```

而不是：

```text
从数据库创建 → crash
```

------

# 二、checkpoint 在做什么（核心三件事）

一次 checkpoint 实际做三件关键事情：

------

## 1️⃣ 刷脏页（最重要）

把 shared buffers 中的 dirty pages：

👉 **全部或部分写入磁盘**

函数：

```c
BufferSync()
```

------

## 2️⃣ 记录 checkpoint LSN

写入控制文件（control file）：

- checkpoint LSN
- checkpoint REDO point

👉 表示：

> “恢复从这里开始”

------

## 3️⃣ 生成 checkpoint WAL 记录

写一条 WAL：

- 标记 checkpoint
- 记录一致性点

------

# 三、checkpoint 的本质模型

可以这样理解：

> 👉 checkpoint = “一致性快照点 + WAL 分界线”

------

## 结构图：

```text
WAL:   A ---- B ---- C ---- CHECKPOINT ---- D ---- E
                          ↑
                    recovery start point
```

------

# 四、checkpoint 解决的核心问题

## 1️⃣ 控制恢复时间

恢复只需要：

- checkpoint → crash

而不是全部 WAL

------

## 2️⃣ 防止 WAL 无限增长

如果没有 checkpoint：

👉 WAL 不能删除

------

## 3️⃣ 允许回收 WAL

checkpoint 完成后：

👉 checkpoint 之前的 WAL 可以 recycle

------

# 五、checkpoint 的执行流程（源码级）

入口：

```c
CreateCheckPoint()
```

------

## Step 1：收集信息

- 当前 WAL LSN
- 当前 buffer 状态

------

## Step 2：刷 buffer（关键）

```c
BufferSync()
```

逻辑：

- 遍历所有 buffer
- 找 dirty page
- 写回磁盘

------

## Step 3：写 checkpoint WAL

```c
XLogInsert(RM_XLOG_ID, XLOG_CHECKPOINT)
```

------

## Step 4：更新 control file

```c
UpdateControlFile()
```

写入：

- redo LSN
- checkpoint LSN

------

# 六、checkpoint 和 WAL 的关系（核心）

非常关键的一点：

> ❗ checkpoint 不替代 WAL，它只是“标记 WAL 的恢复边界”

------

## WAL 仍然必须存在

因为：

- checkpoint 只保证“数据页落盘”
- 但 crash 可能发生在 checkpoint 之后

------

# 七、checkpoint 发生时发生了什么（直觉）

可以把系统想成：

------

## checkpoint 前：

```text
内存：dirty pages 很多
磁盘：旧数据
WAL：不断增长
```

------

## checkpoint 进行中：

```text
开始刷 dirty pages
记录 checkpoint LSN
```

------

## checkpoint 完成：

```text
磁盘：数据一致
WAL：可以回收旧部分
```

------

# 八、checkpoint 的代价

👉 checkpoint 是性能敏感操作

因为：

- 需要刷大量磁盘 IO
- 会造成 IO spike

------

# 九、checkpoint 的触发机制

不是一直执行，而是：

------

## 1️⃣ 时间触发

```conf
checkpoint_timeout = 5min
```

------

## 2️⃣ WAL 大小触发

```conf
max_wal_size
```

------

## 3️⃣ 手动触发

```sql
CHECKPOINT;
```

------

# 十、checkpoint vs WAL recovery（对比）

| 机制       | 作用                  |
| ---------- | --------------------- |
| WAL        | 记录所有变化          |
| checkpoint | 刷数据 + 切断恢复起点 |
| redo       | 崩溃后重放 WAL        |

------

# 十一、一个完整生命周期

```text
修改数据
   ↓
写 WAL
   ↓
checkpoint 周期触发
   ↓
刷脏页
   ↓
记录 checkpoint LSN
   ↓
崩溃
   ↓
从 checkpoint LSN redo WAL
```

------

# 十二、一句话总结（最重要）

👉 **checkpoint = “把内存数据落盘 + 在 WAL 中打一个恢复起点，从而缩短崩溃恢复范围”**

------

# 如果你想继续深入（推荐下一步）

我可以带你继续拆：

- checkpoint 在源码中的完整调用链（bgwriter / checkpointer 进程）
- 或画一张 **WAL + checkpoint + buffer + redo 的完整时序图**
- 或讲“为什么 checkpoint 会造成 IO 峰值（工程优化点）”

这些都是 PostgreSQL 存储引擎的核心难点。