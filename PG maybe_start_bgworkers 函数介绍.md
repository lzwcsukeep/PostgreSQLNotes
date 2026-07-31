位置：

该函数处在 Postmaster 启动流程中，在 ServerLoop 之前：

```c
void
PostmasterMain(int argc, char *argv[])
{
	/* Start bgwriter and checkpointer so they can help with recovery */
	if (CheckpointerPID == 0)
		CheckpointerPID = StartCheckpointer();
	if (BgWriterPID == 0)
		BgWriterPID = StartBackgroundWriter();

	StartupPID = StartupDataBase();
	Assert(StartupPID != 0);
	StartupStatus = STARTUP_RUNNING;
	pmState = PM_STARTUP;

	/* Some workers may be scheduled to start now */
	maybe_start_bgworkers();         ////  <<<< here

	status = ServerLoop();
}
```

介绍：

`maybe_start_bgworkers()` 可以理解成 **Postmaster 的 Background Worker 调度器（scheduler）**。

它**不是创建 Checkpointer、BgWriter、Startup 这些核心进程**（这些已经前面启动了），它负责启动的是 **Background Worker Framework** 注册的后台进程，例如：

- logical replication launcher
- logical replication worker
- extension 注册的 worker（例如 pg_cron、TimescaleDB、pg_stat_statements 某些 worker）
- 自定义 extension 的后台进程
- 其它调用 `RegisterBackgroundWorker()` 注册的 worker

------

## 先看调用位置

你的代码顺序是：

```c
StartCheckpointer();
StartBackgroundWriter();

StartupDataBase();

maybe_start_bgworkers();

ServerLoop();
```

启动顺序实际上是：

```
Postmaster
│
├── Checkpointer
├── BgWriter
├── Startup Process（恢复 WAL）
│
├── maybe_start_bgworkers()
│
└── ServerLoop()
```

注意：

这里启动 Startup Process 之后，数据库**还没有 Ready**。

因此：

> **不是所有 Background Worker 都会立即启动。**

为什么？

因为 Background Worker 可以指定：

```c
bgw_start_time
```

例如：

```c
BgWorkerStart_PostmasterStart
BgWorkerStart_ConsistentState
BgWorkerStart_RecoveryFinished
```

所以 maybe_start_bgworkers() 会判断：

> **这个 worker 现在是否允许启动。**

------

# 它整体做了什么？

一句话：

> **扫描所有已经注册的 Background Worker，根据当前数据库状态，决定哪些应该启动，哪些等待，哪些删除，哪些稍后重启。**

整个流程如下：

```
BackgroundWorkerList
        │
        ▼
遍历每一个 worker
        │
        ├── 已经运行？
        │        │
        │        └──跳过
        │
        ├── 已经要求删除？
        │        │
        │        └──ForgetBackgroundWorker()
        │
        ├── 崩溃过？
        │        │
        │        ├──永不重启？
        │        │       │
        │        │       └──删除
        │        │
        │        └──还没到重启时间？
        │
        ├── 是否应该现在启动？
        │
        └── do_start_bgworker()
```

下面逐段解释。

------

# 第一部分：Crash Recovery 期间直接退出

```c
if (FatalError)
{
    StartWorkerNeeded = false;
    HaveCrashedWorker = false;
    return;
}
```

这里表示：

```
如果数据库正在 crash recovery
```

例如：

```
startup process
    ↓
正在 replay WAL
```

很多 worker 根本不能运行。

例如：

```
logical replication launcher
autovacuum
extension worker
```

因此：

```
FatalError == true

↓

什么都不启动
```

------

# 第二部分：清空两个调度标志

```c
StartWorkerNeeded = false;
HaveCrashedWorker = false;
```

这是两个 Postmaster 全局变量。

意思分别是：

```
StartWorkerNeeded
```

表示：

> ServerLoop 下一轮还要不要再调用 maybe_start_bgworkers()

例如：

```
还有 worker 没启动
```

就设置：

```
StartWorkerNeeded = true;
```

------

另一个：

```
HaveCrashedWorker
```

表示：

```
有没有 worker 正在等待 restart timeout
```

例如：

worker：

```
restart_time = 10 秒
```

刚崩溃：

```
现在不能立即启动
```

于是：

```
HaveCrashedWorker = true;
```

ServerLoop 会定时检查。

------

# 第三部分：遍历所有 Worker

```c
slist_foreach_modify(iter, &BackgroundWorkerList)
```

遍历：

```
BackgroundWorkerList

↓

RegisteredBgWorker
```

每一个节点就是：

```c
RegisteredBgWorker
```

里面包括：

```
PID

启动时间

重启策略

worker 参数

worker 状态
```

------

# 第四部分：已经运行？

```c
if (rw->rw_pid != 0)
    continue;
```

说明：

```
rw_pid != 0
```

worker 已经 fork 出来了。

例如：

```
LogicalLauncher

pid=12345
```

不用再启动。

跳过。

------

# 第五部分：是否要求删除

```c
if (rw->rw_terminate)
{
    ForgetBackgroundWorker(&iter);
    continue;
}
```

有些 worker：

```
请求注销
```

例如 extension：

```
UnregisterBackgroundWorker()
```

就把：

```
rw_terminate = true
```

下一轮：

```
ForgetBackgroundWorker()
```

从链表删掉。

------

# 第六部分：Worker 是否崩溃过

```c
if (rw->rw_crashed_at != 0)
```

说明：

之前：

```
worker

↓

SIGSEGV

↓

退出
```

Postmaster 已记录：

```
rw_crashed_at
```

于是需要判断：

```
还能不能 restart？
```

------

## 永远不重启

```c
bgw_restart_time == BGW_NEVER_RESTART
```

说明：

这个 worker：

```
死一次

↓

永久结束
```

于是：

```
ForgetBackgroundWorker()
```

然后通知：

```
notify_pid
```

一般：

```
SIGUSR1
```

告诉注册者：

```
worker 已经结束
```

------

## 是否到 restart 时间

例如：

```
restart_time = 30 秒
```

崩溃：

```
10:00:00
```

现在：

```
10:00:05
```

只过去：

```
5 秒
```

于是：

```
continue;
```

暂时不启动。

同时：

```c
HaveCrashedWorker = true;
```

告诉：

```
以后还要回来检查。
```

------

# 第七部分：现在是否允许启动？

这一句最重要：

```c
bgworker_should_start_now(...)
```

它检查：

```
bgw_start_time
```

例如：

worker 注册：

```c
bgw_start_time =
    BgWorkerStart_PostmasterStart;
```

那：

```
Postmaster 一启动

↓

立即启动
```

另一种：

```c
BgWorkerStart_ConsistentState
```

只有：

```
WAL Replay

↓

达到 consistent state

↓

允许启动
```

另一种：

```
RecoveryFinished
```

必须：

```
Recovery 完全结束
```

才能启动。

所以：

```
maybe_start_bgworkers()

↓

根据 pmState

↓

判断是否允许启动。
```

------

# 第八部分：真正启动 Worker

```c
do_start_bgworker(rw)
```

这里实际上就是：

```
fork()

↓

exec BackgroundWorkerMain()
```

成功以后：

```
rw_pid

=

child pid
```

以后：

```
Postmaster
```

就管理这个 worker。

------

# 为什么启动失败立即 return？

如果：

```c
do_start_bgworker()
```

失败：

例如：

```
fork失败

资源不足

进程数达到上限
```

代码：

```c
StartWorkerNeeded = true;
return;
```

为什么不是 continue？

因为：

当前系统资源已经不足。

继续启动其它 worker：

```
也大概率失败。
```

所以：

```
ServerLoop

↓

下一轮再尝试
```

------

# 为什么最多启动 100 个？

```c
MAX_BGWORKERS_TO_LAUNCH
```

避免：

```
1000 个 worker

↓

一次 fork 完

↓

ServerLoop 长时间无法处理 socket
```

因此：

一次：

```
最多启动100个
```

剩余：

```
下一轮继续。
```

------

# 整个函数流程图

```text
                maybe_start_bgworkers()
                        │
                        ▼
             FatalError ?
                  │
          yes ───► return
                  │
                 no
                  │
                  ▼
      遍历 BackgroundWorkerList
                  │
      ┌───────────┼──────────────┐
      │           │              │
 已运行？     terminate?      crash?
      │           │              │
 continue      删除worker      判断restart
                               │
                               ▼
                    restart时间到了？
                               │
                     no ─────► 等待
                               │
                              yes
                               │
                               ▼
                 bgworker_should_start_now?
                               │
                      no ─────► continue
                               │
                              yes
                               │
                               ▼
                    do_start_bgworker()
                               │
                 成功          失败
                  │             │
                  ▼             ▼
             PID记录     StartWorkerNeeded=true
```

## 总结

`maybe_start_bgworkers()` 的职责可以概括为四点：

1. **遍历所有已注册的 Background Worker**（`BackgroundWorkerList`）。
2. **根据当前数据库状态决定是否可以启动**（例如 `PostmasterStart`、`ConsistentState`、`RecoveryFinished` 等启动时机）。
3. **处理 Worker 的生命周期**：跳过已运行的 Worker、删除被标记终止的 Worker、按重启策略处理崩溃的 Worker（立即重启、延迟重启或永不重启）。
4. **调用 `do_start_bgworker()` 创建新的后台 Worker**，并通过 `StartWorkerNeeded`、`HaveCrashedWorker` 两个标志通知 `ServerLoop` 是否需要继续调度。

因此，可以把它理解为 **Postmaster 中专门负责管理和调度 Background Worker 的调度器**，而真正的事件循环和长期管理工作则由后面的 `ServerLoop()` 持续完成。