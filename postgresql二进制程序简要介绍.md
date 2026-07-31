# PG 自带的二进制程序列表如下

解释一下这些二进制程序

```bash
lzw@ubuntu:/usr/lib/postgresql/16/bin$ ls
clusterdb   dropuser    pg_archivecleanup  pg_config       pg_dumpall      pg_resetwal    pg_test_timing   postgres   vacuumlo
createdb    initdb      pg_basebackup      pg_controldata  pg_isready      pg_restore     pg_upgrade       psql
createuser  oid2name    pgbench            pg_ctl          pg_receivewal   pg_rewind      pg_verifybackup  reindexdb
dropdb      pg_amcheck  pg_checksums       pg_dump         pg_recvlogical  pg_test_fsync  pg_waldump       vacuumdb
```

---

你现在看到的是 **PostgreSQL 安装后的核心工具集（bin 目录）**。
可以把它们理解为三大类：

```text
1️⃣ 客户端工具（连数据库干活）
2️⃣ 管理工具（控制数据库）
3️⃣ 运维/底层工具（备份、WAL、校验）
```

我帮你按功能分组讲，比逐个背更清晰 👇

------

# 一、客户端工具（最常用）

## ⭐ `psql`

👉 最核心工具

```bash
psql -d mydb
```

✔ 交互式 SQL 客户端
✔ 你平时写 SQL 就靠它

------

## 其他简单管理命令（本质是 psql 封装）

| 命令         | 作用       |
| ------------ | ---------- |
| `createdb`   | 创建数据库 |
| `dropdb`     | 删除数据库 |
| `createuser` | 创建用户   |
| `dropuser`   | 删除用户   |

👉 本质：

```text
调用 psql 执行 SQL
```

------

## `clusterdb`

👉 对整个数据库做操作（如 VACUUM）

------

## `reindexdb`

👉 重建索引（非常重要）

------

## `vacuumdb`

👉 执行 VACUUM（清理死元组）

------

## `vacuumlo`

👉 清理“大对象”（large objects）

------

# 二、数据库服务相关（核心）

## ⭐ `postgres`

👉 数据库服务器进程

```bash
postgres -D data_dir
```

✔ 真正的数据库内核
✔ 一般不会手动直接启动（用 pg_ctl）

------

## ⭐ `pg_ctl`

👉 控制数据库服务

```bash
pg_ctl start
pg_ctl stop
```

✔ 启动 / 停止 / 重启
✔ 查看状态

------

## ⭐ `initdb`

👉 初始化数据库集群

```bash
initdb -D /data
```

✔ 创建数据目录
✔ 初始化系统表

------

# 三、备份 & 恢复工具（非常重要）

## ⭐ `pg_dump`

👉 备份单个数据库

```bash
pg_dump mydb > dump.sql
```

------

## ⭐ `pg_dumpall`

👉 备份整个实例（所有数据库）

------

## ⭐ `pg_restore`

👉 恢复备份（配合 pg_dump）

------

## ⭐ `pg_basebackup`

👉 物理备份（复制整个数据目录）

✔ 用于主从复制
✔ 比 pg_dump 更底层

------

# 四、WAL / 复制 / 恢复相关（进阶重点）

👉 这是 PostgreSQL 最核心的机制之一

------

## `pg_receivewal`

👉 实时接收 WAL（日志流复制）

------

## `pg_recvlogical`

👉 逻辑复制（logical replication）

------

## `pg_archivecleanup`

👉 清理归档 WAL

------

## `pg_waldump`

👉 查看 WAL 内容（调试神器）

------

## `pg_resetwal`

👉 重置 WAL（⚠️ 非常危险）

✔ 用于灾难恢复
❌ 正常情况不要用

------

## `pg_rewind`

👉 主从切换后“追赶数据”

------

# 五、检查 / 诊断工具

## `pg_isready`

👉 检查数据库是否可连接

```bash
pg_isready
```

------

## `pg_controldata`

👉 查看数据库控制文件信息

------

## `pg_amcheck`

👉 检查索引/表结构一致性（新版本很重要）

------

## `pg_checksums`

👉 校验数据页 checksum

------

## `pg_verifybackup`

👉 验证备份是否完整

------

## `pg_test_fsync`

👉 测试磁盘 fsync 性能

------

## `pg_test_timing`

👉 测试系统时间精度

------

# 六、性能 / 测试工具

## ⭐ `pgbench`

👉 压测工具（TPC-B 风格）

```bash
pgbench -i
pgbench -c 10 -T 60
```

✔ 测试吞吐量
✔ 模拟并发事务

------

# 七、升级工具

## ⭐ `pg_upgrade`

👉 跨版本升级数据库

✔ 比 dump/restore 快很多

------

# 八、其他工具

## `oid2name`

👉 把 OID 转成对象名（调试用）

------

# 九、总结一张图（强烈建议记住）

```text
数据库核心：
    postgres     → 服务器
    pg_ctl       → 启停控制
    initdb       → 初始化

客户端：
    psql         → 交互
    createdb     → 创建库

备份恢复：
    pg_dump / pg_restore
    pg_basebackup

WAL/复制：
    pg_receivewal
    pg_waldump
    pg_rewind

维护：
    vacuumdb
    reindexdb

诊断：
    pg_isready
    pg_amcheck

性能：
    pgbench
```

------

# 十、结合你当前学习（建议重点掌握）

你现在在研究 PG 内核 + 构建系统，建议优先熟悉：

👉 必会：

- `psql`
- `pg_ctl`
- `initdb`
- `pg_dump / pg_restore`

👉 进阶：

- `pg_basebackup`
- `pg_waldump`
- `pgbench`

------

# 如果你想更深入（非常推荐）

我可以帮你讲这些“底层视角”：

- `postgres` 进程启动后做了什么（源码级）
- `pg_ctl` 是怎么控制 postmaster 的
- WAL 是怎么被 pg_waldump 解析的
- pg_dump vs pg_basebackup 本质区别（逻辑 vs 物理）

这些会直接把你带进 PostgreSQL 内核世界 👍

