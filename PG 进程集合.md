# PG 进程集合

```bash
lzw@ubuntu:~/workspace/UDB-TX/unvdb-data$ ps -ef | grep postgres
postgres    1049       1  0 09:42 ?        00:00:00 /usr/lib/postgresql/16/bin/postgres -D /var/lib/postgresql/16/main -c config_file=/etc/postgresql/16/main/postgresql.conf
postgres    1069    1049  0 09:42 ?        00:00:00 postgres: 16/main: checkpointer 
postgres    1070    1049  0 09:42 ?        00:00:00 postgres: 16/main: background writer 
postgres    1072    1049  0 09:42 ?        00:00:00 postgres: 16/main: walwriter 
postgres    1073    1049  0 09:42 ?        00:00:00 postgres: 16/main: autovacuum launcher 
postgres    1074    1049  0 09:42 ?        00:00:00 postgres: 16/main: archiver 
postgres    1075    1049  0 09:42 ?        00:00:00 postgres: 16/main: logical replication launcher 
```

                    ┌──────────────┐
                    │  postmaster  │
                    └──────┬───────┘
                           │
       ┌────────────┬──────┼──────────────┬────────────┐
       │            │      │              │            │
    checkpointer  bgwriter walwriter autovacuum  archiver ...


你这条 `ps -ef | grep postgres` 输出，其实就是一个典型的 **PostgreSQL 多进程架构**。我按每个进程给你讲清楚它们在干嘛（这块非常重要，属于数据库内核核心知识）。

------

## 🔴 1. 主进程（postmaster / postgres）

```
/usr/lib/postgresql/16/bin/postgres -D ...
```

👉 这是 **主进程（Postmaster）**

**作用：**

- 启动整个数据库实例
- 监听客户端连接（TCP / Unix socket）
- fork 出子进程（每个连接一个 backend）
- 管理所有后台进程
- 崩溃恢复（crash recovery）

📌 可以理解为：**“总调度 + 管理者”**

------

## 🟡 2. checkpointer（检查点进程）

```
postgres: 16/main: checkpointer
```

**作用：**

- 定期触发 **checkpoint**
- 把内存（shared buffers）里的脏页刷到磁盘

**为什么需要？**

- 避免崩溃恢复时重放太多 WAL
- 控制恢复时间

📌 核心点：
👉 checkpoint = “把数据库状态落盘的一个时间点”

------

## 🔵 3. background writer（后台写进程）

```
postgres: 16/main: background writer
```

**作用：**

- 持续、温和地把脏页写入磁盘
- 减轻 checkpoint 时的 IO 峰值

📌 和 checkpointer 区别：

| 进程              | 特点       |
| ----------------- | ---------- |
| background writer | 平时慢慢写 |
| checkpointer      | 到点强制写 |

👉 一个是“日常打扫”，一个是“定期大扫除”

------

## 🟢 4. walwriter（WAL 写进程）

```
postgres: 16/main: walwriter
```

**作用：**

- 把 WAL buffer 写入 WAL 文件（磁盘）

📌 为什么重要？
PostgreSQL 的核心机制：
👉 **先写 WAL，再写数据（WAL ahead logging）**

作用：

- 保证事务持久性（Durability）
- 崩溃恢复依赖 WAL

------

## 🟣 5. autovacuum launcher（自动清理调度器）

```
postgres: 16/main: autovacuum launcher
```

**作用：**

- 启动多个 autovacuum worker
- 自动做：
  - VACUUM（清理死元组）
  - ANALYZE（更新统计信息）

📌 为什么必须有？
因为 PostgreSQL 使用：
👉 **MVCC（多版本并发控制）**

会产生：

- dead tuples（垃圾数据）

如果不清理：

- 表会膨胀（bloat）
- 查询变慢

------

## 🟠 6. archiver（归档进程）

```
postgres: 16/main: archiver
```

**作用：**

- 把 WAL 文件归档到指定位置

📌 用于：

- 数据库备份
- PITR（时间点恢复）

👉 没开归档的话，这个进程可能是 idle

------

## 🟤 7. logical replication launcher（逻辑复制调度器）

```
postgres: 16/main: logical replication launcher
```

**作用：**

- 管理逻辑复制 worker
- 支持：
  - 发布（publisher）
  - 订阅（subscriber）

📌 用于：

- 数据同步
- 微服务数据分发
- 跨库复制