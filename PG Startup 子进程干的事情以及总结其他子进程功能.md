#  StartupProcess 进程扫盲

这是 PostgreSQL 启动流程中会运行的一个子进程。

**结论先说：**

> **StartupProcess 不是一个新的程序（program），也不是执行了另外一个可执行文件。**
>
> **它只是 postmaster fork 出来的一个子进程，这个子进程最终执行 `StartupProcessMain()`。**

也就是说：

- 父进程：`postmaster`
- 子进程：`startup process`
- **二者运行的都是同一个 postgres 可执行文件，只不过 fork 以后走了不同的代码路径。**

------

## 整个过程

假设执行的是

```bash
postgres -D data
```

操作系统只启动了一个程序：

```
postgres
```

进入

```
main()
    ↓
PostmasterMain()
```

随后来到你贴的代码：

```c
StartupPID = StartupDataBase();
```

实际上就是

```c
StartupPID = StartChildProcess(StartupProcess);
```

继续：

```c
pid = fork_process();
```

这里发生了

```
               postgres(postmaster)
                     PID=1000
                        |
                     fork()
                        |
        --------------------------------
        |                              |
 parent (1000)                  child (1001)
```

注意：

**这里没有 exec()。**

所以：

两个进程

```
PID 1000
PID 1001
```

共享同一个程序映像（code segment）。

区别只是：

- PID 不一样
- 数据页 COW（Copy On Write）
- 执行流不同

------

## 为什么执行流不同？

因为

```c
if (pid == 0)
{
    ...
    AuxiliaryProcessMain(type);
}
```

父进程

```
fork()
```

返回

```
pid = 1001
```

继续执行

```c
return pid;
```

然后

```
ServerLoop();
```

开始监听客户端。

而子进程

```
fork()
```

返回

```
pid = 0
```

于是进入

```c
AuxiliaryProcessMain(type);
```

这里

```
type == StartupProcess
```

于是

```c
switch(type)
{
    case StartupProcess:
        StartupProcessMain();
        break;

    case BgWriterProcess:
        BackgroundWriterMain();
        break;

    case CheckpointerProcess:
        CheckpointerMain();
        break;
}
```

因此 Startup 子进程最后进入

```
StartupProcessMain()
```

------

## StartupProcessMain 做什么？

最后真正干活的是

```c
StartupXLOG();
```

几乎所有数据库启动恢复工作都在这里。

包括：

```
读取 control file

↓

找到 checkpoint

↓

读取 WAL

↓

Redo

↓

恢复共享缓冲

↓

恢复 clog

↓

恢复 multixact

↓

恢复事务状态

↓

一致性检查

↓

数据库进入一致状态
```

恢复完成以后

```c
proc_exit(0);
```

Startup Process 就退出了。

然后 postmaster 收到

```
SIGCHLD
```

知道 Startup Process 已完成恢复。

随后

```
PM_STARTUP
        ↓
PM_RECOVERY
        ↓
PM_HOT_STANDBY（如果是 standby）
        ↓
PM_RUN
```

开始允许普通 Backend 登录。

------

## 为什么不用 exec()？

这是 PostgreSQL 一个非常经典的设计。

Linux 下：

```
fork()
```

几乎不要钱（Copy-on-Write）。

因为：

```
代码段
```

直接共享。

例如

```
postgres
```

加载以后：

```
.text
.rodata
```

都不用重新加载。

fork 出来的子进程立即可以运行：

```
StartupProcessMain()

CheckpointerMain()

WalWriterMain()

BgWriterMain()

BackendMain()
```

如果用了

```
fork()

↓

execve()
```

则需要重新：

```
重新加载 ELF

重新定位

重新初始化 libc

重新建立 MemoryContext

重新 Attach Shared Memory

...
```

效率低很多。

因此 PostgreSQL 在 Unix/Linux 上采用

```
fork

↓

直接调用不同 Main()
```

而不是

```
exec()
```

------

## 为什么 Windows 又用了 EXEC_BACKEND？

你注意到了代码里的

```c
#ifdef EXEC_BACKEND
```

Windows 没有真正意义上的

```
fork()
```

所以只能

```
CreateProcess()
```

重新启动一个

```
postgres.exe
```

命令行类似

```
postgres.exe --forkaux 3
```

然后新的 postgres.exe 从

```
main()
```

开始运行。

根据

```
--forkaux
```

参数知道自己应该变成

```
Startup Process
```

于是最后仍然进入

```
StartupProcessMain()
```

所以：

Linux

```
postgres
    │
 fork()
    │
StartupProcessMain()
```

Windows

```
postgres.exe
      │
CreateProcess()
      │
postgres.exe --forkaux StartupProcess
      │
main()
      │
StartupProcessMain()
```

最终执行的函数一样，只是创建子进程的方法不同。

------

## StartupProcess 与普通 Backend 的区别

可以把 PostgreSQL 的进程分为两类：

| 进程                  | 是否接受客户端连接 | 主要职责                                 | 生命周期                                                     |
| --------------------- | ------------------ | ---------------------------------------- | ------------------------------------------------------------ |
| **Postmaster**        | 否                 | 管理所有子进程、监听连接                 | 数据库整个运行期间                                           |
| **Startup Process**   | 否                 | 执行启动恢复（WAL Redo、恢复数据库状态） | 启动阶段，恢复完成后通常退出（主库）；在备用库上可能长期运行以持续回放 WAL |
| **Checkpointer**      | 否                 | 执行 checkpoint，将脏页刷盘              | 长期存在                                                     |
| **Background Writer** | 否                 | 后台写脏页，减少后端刷盘压力             | 长期存在                                                     |
| **WAL Writer**        | 否                 | 定期将 WAL Buffer 写入 WAL 文件          | 长期存在                                                     |
| **Backend**           | 是                 | 为单个客户端执行 SQL                     | 每个连接对应一个进程                                         |

因此，**Startup Process 本质上是 postmaster fork 出来的一个辅助进程（auxiliary process），不是独立的程序，也不是新的可执行文件，而是同一个 `postgres` 进程映像中的另一条执行路径，专门负责数据库启动恢复。**

