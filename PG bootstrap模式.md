# PostgreSQL `BootstrapModeMain` 函数超清晰解释
这是 **PostgreSQL 初始化（引导）模式的主入口函数**，**只在数据库第一次初始化（initdb）时运行**，作用是**从零创建系统表、初始化 WAL、创建模板数据库**，让一个全新的空数据目录变成可用的 PostgreSQL 数据库。

一句话总结：
**它是 initdb 背后真正干活的“数据库创世程序”。**

---

## 一、这个函数是干嘛的？
1. **运行时机**
   只在执行 `initdb` 时调用，**正常启动数据库不会跑这个函数**。

2. **核心任务**
   - 从零创建系统 catalog（`pg_class`、`pg_attribute` 等系统表）
   - 初始化 WAL（事务日志）
   - 建立共享内存、信号量
   - 加载并执行特殊的 **bootstrap 指令**（不是普通 SQL）
   - 生成一个干净的 `template1` 模板数据库

3. **模式特点**
   引导模式**不支持标准 SQL**，只执行一套专门用于初始化的**bootstrap 语言指令**。

---

## 二、关键步骤逐行解释

### 1. 基础初始化
```c
InitStandaloneProcess(argv[0]);
InitializeGUCOptions();
```
- 把进程设为**独立后台进程**
- 初始化所有配置参数（GUC）

---

### 2. 解析命令行参数
处理 `initdb` 传进来的参数：
- `-D`：数据目录
- `-B`：共享缓冲区
- `-c`：配置参数
- `-X`：WAL段大小
- `-F`：关闭 fsync（初始化时加速）
- `-k`：开启数据校验

这一步只是**读取参数，不做实际工作**。

---

### 3. 进入数据目录 + 创建锁文件
```c
checkDataDir();
ChangeToDataDir();
CreateDataDirLockFile(false);
```
- 检查数据目录是否合法
- 切换工作目录到数据目录
- **创建锁文件**（防止多个初始化进程同时运行）

---

### 4. 设置运行模式
```c
SetProcessingMode(BootstrapProcessing);
IgnoreSystemIndexes = true;
```
- 进入**引导模式**
- 初始化阶段**暂时忽略系统索引**（系统表还没建好）

---

### 5. 创建共享内存 & 信号量
```c
InitializeMaxBackends();
CreateSharedMemoryAndSemaphores();
```
和正常启动数据库一样：
- 计算最大连接数
- 创建共享内存
- 创建进程间同步信号量

---

### 6. 如果只是检查配置（check_only）
```c
if (check_only) {
	SetProcessingMode(NormalProcessing);
	CheckerModeMain();
	abort();
}
```
如果只是**测试参数是否合法**（不真正初始化），到这里就退出。

---

### 7. 后端进程初始化
```c
InitProcess();
BaseInit();
bootstrap_signals();
BootStrapXLOG();
```
- 初始化进程结构
- 初始化基础子系统
- 设置信号处理
- **最重要：创建并初始化 WAL 事务日志系统**

---

### 8. 初始化系统表结构
```c
InitPostgres(NULL, InvalidOid, NULL, InvalidOid, false, false, NULL);
```
加载最核心的系统表结构，让数据库能认识系统 catalog。

---

### 9. 执行引导指令（创世核心！）
```c
boot_yyparse();
```
这是**整个函数的灵魂**：
- 读取 `initdb` 提供的 bootstrap 指令
- **创建所有系统表、系统视图、内置函数、类型**
- 完成模板数据库的构建

这一步执行完，数据库就真正“诞生”了。

---

### 10. 收尾工作
```c
RelationMapFinishBootstrap();
cleanup();
proc_exit(0);
```
- 写入系统表映射文件
- 清理资源
- 退出，完成初始化

---

## 三、最关键的 3 个核心点
1. **`boot_yyparse()`**
   执行初始化指令，**创建所有系统表**，是真正的“创世逻辑”。

2. **`BootStrapXLOG()`**
   初始化 WAL 日志，没有它数据库无法运行。

3. **`CreateDataDirLockFile(false)`**
   防止同时运行多个 initdb，保证数据目录安全。

---

## 四、用最简单的话总结
`BootstrapModeMain` 就是：

✅ **PostgreSQL 的初始化创世主函数**
✅ 只在 `initdb` 时运行一次
✅ 从零创建系统表、WAL、模板数据库
✅ 不支持普通 SQL，只执行专用引导指令
✅ 完成后，一个全新的数据库就诞生了

---

### 最终一句话记忆
**这个函数 = PostgreSQL 的“盘古开天”程序。**

