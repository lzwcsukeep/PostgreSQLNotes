# PostgreSQL 项目基本信息

# 代码统计

下面是`cloc` 对 postgresql16项目源代码的统计信息：

```bash
lzw@ubuntu:~/workspace/ext_projs/postgresql-16.0/src$ cloc .
    5297 text files.
    5146 unique files.                                          
    1427 files ignored.

github.com/AlDanial/cloc v 1.90  T=3.49 s (1116.5 files/s, 820993.4 lines/s)
---------------------------------------------------------------------------------------
Language                             files          blank        comment           code
---------------------------------------------------------------------------------------
C                                     1332         181072         348329        1002040
PO File                                438         165588         203647         495107
C/C++ Header                           978          18185          59979          98289
SQL                                    376          21736          16843          83051
Perl                                   255          10237           8872          47577
yacc                                     9           4610           3188          37180
XML                                      2             10             18          30952
make                                   234           1734           2079           5750
Meson                                  202            852            410           5525
lex                                     11           1073           2138           5238
Bourne Shell                            25            210            307            935
C++                                      3            164            277            713
Python                                   6             84             40            238
DOS Batch                                6             21             26            141
D                                        1             14             14             66
Assembly                                 2             14             38             44
Windows Resource File                    2              3              1             33
Bourne Again Shell                       1             21              6             32
sed                                      2              3             17             19
Prolog                                   1              2              0             17
IDL                                      8              0              0              8
Windows Message File                     1              0              0              5
Windows Module Definition                1              0              1              4
---------------------------------------------------------------------------------------
SUM:                                  3896         405633         646230        1812964
---------------------------------------------------------------------------------------

```

重点信息：

- C语言源代码文件有1332 个，总共100万行代码。
- C/C++ 头文件978个，总共9万8千行代码
- 还有一些其他代码，所有的加起来总共一百八十一万行代码

# 目录解释

```bash
lzw@ubuntu:~/workspace/ext_projs/postgresql-16.0$ ls
aclocal.m4  configure     contrib    doc             HISTORY  Makefile     meson_options.txt  src
config      configure.ac  COPYRIGHT  GNUmakefile.in  INSTALL  meson.build  README
```

- `src/` 是内核，最核心目录：`src/`

  > PostgreSQL 的 95% 核心代码都在这里

- `contrib/` 是官方扩展

- `doc/` 是 文档源码（SGML）最终生成 PostgreSQL 官方文档。一般内核开发不常改。

- `configure / configure.ac / aclocal.m4`

  > 是 Autoconf 构建系统。
  >
  > 作用：
  >
  > - 检测系统特性
  > - 生成 Makefile
  > - 判断是否支持 ICU、SSL、LLVM
  >
  > 你运行：
  >
  > ```
  > ./configure
  > ```
  >
  > 就是用的这个。

- `Makefile / GNUmakefile.in` 是 构建入口，定义

  - make
  - make world
  - make check
  - make install

- `meson.build / meson_options.txt` : 从 PG 16 开始支持 Meson 构建系统，未来 Meson 会逐渐替代 Autoconf。

- `config/` 是 平台适配相关。例如：各种 OS 差异，编译选项，一般很少动。

下面讲 **必须了解的核心目录结构**，按“内核开发视角”讲，而不是简单文件说明。

重点讲：
👉 如果你要读源码 / 改内核 / 写扩展，哪些目录一定要熟

------

## 一、最核心目录：`src/`

> PostgreSQL 的 95% 核心代码都在这里

------

### 1️⃣ `src/backend/` ⭐⭐⭐⭐⭐（最重要）

这是数据库服务器本体：

```text
postgres 进程的核心代码
```

如果你理解这个目录，就理解了 PostgreSQL 的内核。

主要子目录：

| 目录       | 作用                                     |
| ---------- | ---------------------------------------- |
| access/    | 存储引擎（heap、btree、hash、gin、gist） |
| catalog/   | 系统表定义                               |
| commands/  | DDL/DML 实现                             |
| executor/  | 执行器                                   |
| optimizer/ | 查询优化器                               |
| parser/    | SQL 语法解析                             |
| planner/   | 查询计划生成                             |
| storage/   | buffer、WAL、锁                          |
| tcop/      | SQL 执行主入口                           |
| utils/     | 内存管理、GUC、错误处理                  |

📌 如果你想研究：

| 方向         | 看哪里                 |
| ------------ | ---------------------- |
| MVCC         | storage/ + access/heap |
| 锁机制       | storage/lmgr           |
| WAL          | access/transam         |
| 查询优化     | optimizer/             |
| SQL 执行流程 | tcop/ + executor/      |

------

### 2️⃣ `src/include/` ⭐⭐⭐⭐⭐

头文件目录。

里面定义：

- 核心数据结构
- 内部 API
- 系统 catalog 结构
- MemoryContext

你读 backend 必须配合 include。

------

### 3️⃣ `src/bin/`

客户端工具源码：

| 目录     | 工具         |
| -------- | ------------ |
| psql/    | psql 客户端  |
| pg_dump/ | 备份         |
| pg_ctl/  | 启停         |
| initdb/  | 初始化数据库 |

------

### 4️⃣ `src/interfaces/`

客户端库：

```text
libpq/
```

这是 C 客户端接口库。

------

### 5️⃣ `src/test/`

测试框架：

| 目录         | 作用           |
| ------------ | -------------- |
| regress/     | SQL regression |
| isolation/   | 并发测试       |
| recovery/    | 恢复测试       |
| replication/ | 复制测试       |

------

## 二、contrib/

官方扩展目录 ⭐⭐⭐

不是核心数据库的一部分，但很重要。

例如：

- pg_stat_statements
- hstore
- pg_trgm
- uuid-ossp

如果你想写扩展：

👉 这里是最佳参考代码

------

## 三、doc/

文档源码（SGML）

最终生成 PostgreSQL 官方文档。

一般内核开发不常改。

------

## 四、configure / configure.ac / aclocal.m4

这是 Autoconf 构建系统。

作用：

- 检测系统特性
- 生成 Makefile
- 判断是否支持 ICU、SSL、LLVM

你运行：

```bash
./configure
```

就是用的这个。

------

## 五、Makefile / GNUmakefile.in

构建入口。

定义：

- make
- make world
- make check
- make install

------

## 六、meson.build / meson_options.txt

从 PG 16 开始支持 Meson 构建系统。

可以：

```bash
meson setup build
ninja -C build
```

未来 Meson 会逐渐替代 Autoconf。

------

## 七、config/

平台适配相关。

例如：

- 各种 OS 差异
- 编译选项

一般很少动。

------

## 如果你是内核学习者，必须掌握的目录

### 第一阶段（SQL 执行流程）

- src/backend/parser
- src/backend/optimizer
- src/backend/executor
- src/backend/tcop

------

### 第二阶段（存储 & MVCC）

- src/backend/access/heap
- src/backend/storage
- src/backend/access/transam

------

### 第三阶段（系统 catalog）

- src/backend/catalog
- src/include/catalog

------

### 整个 PostgreSQL 执行流程文件位置

```text
tcop/postgres.c
    ↓
parser/
    ↓
optimizer/
    ↓
executor/
    ↓
storage/
```

入口函数：

```text
PostgresMain()
```

在：

```text
src/backend/tcop/postgres.c
```

------

🎯 如果只选 5 个必须理解的目录

1. src/backend/
2. src/include/
3. src/backend/access/
4. src/backend/storage/
5. src/backend/optimizer/

------

## 总结

- `src/` 是内核
- `backend/` 是服务器
- `include/` 是核心结构
- `contrib/` 是官方扩展
- `test/` 是测试系统

# Make 目标解释

## 编译构建相关

- make
- make -C contrib
- make world

在 PostgreSQL 源码目录下编译时，这几个 `make` 命令的含义和编译范围不同，主要区别在于“编译哪些组件”。

### 一、`make`

#### ✅ 含义

只编译 PostgreSQL **核心主程序部分**

#### 📦 编译内容

主要包括：

- `src/backend`（数据库服务器 postgres）
- `src/bin`（客户端工具，如 psql）
- `libpq`（客户端 C 接口库）
- 基础头文件

#### ❌ 不包括

- contrib 扩展
- 文档
- 测试套件

#### 📌 使用场景

最常用的方式：

```bash
./configure
make
make install
```

适合：

- 正常安装 PostgreSQL
- 开发数据库内核
- 修改 backend 源码调试

------

### 二、`make -C contrib`

#### ✅ 含义

进入 `contrib/` 目录编译官方扩展模块

`-C` 的意思是：

> Change directory before doing anything

等价于：

```bash
cd contrib
make
```

------

#### 📦 contrib 是什么？

`contrib/` 目录里是：

- 官方扩展模块
- 不属于核心数据库
- 可选安装

常见扩展例如：

- `pg_stat_statements`
- `pg_trgm`
- `hstore`
- `uuid-ossp`
- `intarray`

------

#### 📌 使用场景

如果你：

- 只想编译扩展
- 或者 core 已经编译完了

可以：

```bash
make -C contrib
make -C contrib install
```

------

### 三、`make world`

这是一个 **“一键全编译”目标**。

#### ✅ 含义

编译整个 PostgreSQL 世界（核心 + 扩展）

#### 📦 包含内容

- core
- contrib
- pl languages
- 其他附带模块

等价于：

```bash
make
make -C contrib
```

但内部是通过 Makefile 依赖组织好的。

------

#### 对应安装命令

如果用：

```bash
make world
make install-world
```

等价于：

```bash
make
make install
make -C contrib
make -C contrib install
```

------

### 四、源码结构角度理解

PostgreSQL 顶层 Makefile 里有很多 target：

| 目标          | 作用                |
| ------------- | ------------------- |
| all           | 默认，编译 core     |
| world         | 编译 core + contrib |
| install       | 安装 core           |
| install-world | 安装 core + contrib |

------

### 五、简单对比表

| 命令            | 编译 core | 编译 contrib | 推荐场景   |
| --------------- | --------- | ------------ | ---------- |
| make            | ✅         | ❌            | 最常用     |
| make -C contrib | ❌         | ✅            | 只编扩展   |
| make world      | ✅         | ✅            | 一次性全部 |

------

### 六、开发者常见使用习惯

### 🔹 内核开发

```bash
make -j8
```

### 🔹 内核 + 扩展一起调试

```bash
make world -j8
```

### 🔹 只改了某个扩展

```bash
make -C contrib/pg_stat_statements
```

------

### 七、一个小细节（很多人不知道）

`make world` 会比 `make` 慢很多，因为：

- contrib 有大量独立 extension
- 每个 extension 都要单独编译
- 还可能编译 PL/Perl、PL/Python（如果 configure 开启）

------

### 八、总结

- `make` → 编译数据库核心
- `make -C contrib` → 编译官方扩展
- `make world` → 编译整个 PostgreSQL 世界

------

## 测试相关

- make check
- make check-prove
- make -C src/test/isolation check
- make check-world

这些 `make` 目标都和 **测试（regression test）** 有关，是在 PostgreSQL 源码中用于验证构建是否正确的。

它们的区别主要在于：

- 测试范围
- 使用的测试框架
- 是否包含扩展

我给你系统拆开讲清楚。

------

### 一、`make check`

#### ✅ 含义

运行 **核心回归测试（regression tests）**

------

#### 📦 测试内容

- SQL 语句功能
- DDL / DML
- 事务
- 索引
- 并发基础行为
- 系统 catalog

测试目录：

```bash
src/test/regress
```

------

#### ⚙️ 工作方式

它会：

1. 在源码目录里初始化一个临时 PostgreSQL 实例
2. 启动一个测试数据库
3. 执行一系列 SQL 文件
4. 把输出和 `expected/` 目录做 diff
5. 输出结果

------

#### 📌 特点

- 不需要安装（不需要 `make install`）
- 在 build 目录里运行
- 使用 Unix domain socket
- 默认端口 5432（可改）

------

#### 🎯 使用场景

你修改了：

- executor
- optimizer
- storage
- catalog

一般跑：

```bash
make check
```

------

### 二、`make check-prove`

这个比 `make check` 更高级一点。

------

#### ✅ 含义

使用 **Perl TAP 测试框架（prove）** 运行测试。

------

#### 📦 TAP 是什么？

TAP = Test Anything Protocol

它允许：

- 更复杂的测试逻辑
- 多实例测试
- replication 测试
- 并发控制
- 测试脚本用 Perl 写

------

#### 测试目录

```bash
src/test/recovery
src/test/replication
src/test/subscription
src/bin/pg_basebackup/t
...
```

这些目录都有：

```
t/*.pl
```

------

#### 📌 区别于 check

| 特性         | check           | check-prove |
| ------------ | --------------- | ----------- |
| 框架         | 传统 regression | Perl TAP    |
| 支持多实例   | ❌               | ✅           |
| 支持复杂测试 | ❌               | ✅           |
| 并发测试     | 有限            | 强          |

------

#### 🎯 使用场景

如果你改了：

- replication
- logical decoding
- WAL
- recovery
- 物理复制
- 逻辑复制

必须跑：

```bash
make check-prove
```

------

### 三、`make -C src/test/isolation check`

这是一个 **特定测试子系统**。

------

#### ✅ 含义

运行 isolation 测试框架。

------

#### 📦 isolation 是干什么的？

专门测试：

- 并发行为
- 锁机制
- MVCC
- 事务冲突
- 可重复读
- 死锁

目录：

```bash
src/test/isolation
```

------

#### 它和普通 regression 不同

普通 regression 是：

```
顺序执行 SQL
```

isolation 测试是：

```
多个 session 并发执行
控制执行顺序
验证结果
```

------

#### 示例测试场景

- 两个事务并发更新
- 一个事务持锁，另一个等待
- 验证是否出现 serialization failure

------

#### 🎯 使用场景

如果你修改：

- 锁管理
- MVCC
- snapshot
- transaction manager
- SSI

必须跑：

```bash
make -C src/test/isolation check
```

------

### 四、`make check-world`

这是测试界的 “world”。

------

#### ✅ 含义

运行：

- core regression tests
- TAP tests
- isolation tests
- contrib 扩展测试

等价于：

```bash
make check
make check-prove
make -C contrib check
make -C src/test/isolation check
```

------

#### 📦 包含内容

✔ core
✔ TAP
✔ isolation
✔ contrib extension tests

------

#### 🎯 使用场景

- 你改了很多底层代码
- 要提交 patch
- 要发邮件到 pgsql-hackers
- CI 测试

通常会跑：

```bash
make check-world
```

------

### 五、结构对比图

```
check
 └── src/test/regress

check-prove
 └── src/test/* (TAP)

isolation
 └── src/test/isolation

check-world
 └── 上面全部 + contrib
```

------

### 六、执行时间对比

| 命令             | 时间 |
| ---------------- | ---- |
| make check       | 快   |
| make check-prove | 中等 |
| isolation        | 快   |
| check-world      | 最慢 |

------

### 七、开发者经验建议

如果你是做内核开发（看你前面一直在研究 PG 内部）：

#### 修改 executor / planner

```bash
make check
```

#### 修改锁 / MVCC / 事务

```bash
make check
make -C src/test/isolation check
```

#### 修改 WAL / replication

```bash
make check-prove
```

#### 提交 patch 前

```bash
make check-world
```

------

### 八、一个容易踩的坑

`make check` 不会测试：

- replication
- logical replication
- basebackup
- pg_rewind

这些只在 TAP 里。

很多人只跑 `make check`，结果 patch 进 CI 挂掉。

------

### 九、总结

- `check` → 核心 SQL 回归
- `check-prove` → TAP 复杂测试
- `isolation` → 并发事务测试
- `check-world` → 全部测试 + 扩展

