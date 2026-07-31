>  问： postgresql 里的 catalog 哪些是整个集群共享的，哪些是一个数据库专有的？

这是个非常关键的问题，已经开始触及 PostgreSQL 的**系统目录架构设计核心**了。

先给你一个结论，然后再解释👇

------

# 一、结论（必须记住）

👉 PostgreSQL 的 catalog 分两类：

### ✅ 1️⃣ 集群级共享（shared catalog）

存放在：

```
global/
```

👉 **所有数据库都能看到**

------

### ✅ 2️⃣ 数据库级（per-database catalog）

存放在：

```
base/<db_oid>/
```

👉 **每个数据库独立一份**

------

# 二、哪些是“共享 catalog”？（重点🔥）

这些表在所有数据库之间共享：

| catalog            | 说明           |
| ------------------ | -------------- |
| `pg_database`      | 数据库列表     |
| `pg_authid`        | 用户（真实表） |
| `pg_roles`         | 用户视图       |
| `pg_tablespace`    | 表空间         |
| `pg_shdepend`      | 跨数据库依赖   |
| `pg_shdescription` | 共享对象注释   |

------

### 📌 特点

- 只有一份（全局）
- 存在 `global/` 目录
- 关系 `relisshared = true`

------

# 三、哪些是“数据库私有 catalog”？（更常见）

绝大多数 catalog 都是**数据库级别的**：

| 分类   | catalog                    |
| ------ | -------------------------- |
| 表结构 | `pg_class`, `pg_attribute` |
| 类型   | `pg_type`                  |
| schema | `pg_namespace`             |
| 约束   | `pg_constraint`            |
| 索引   | `pg_index`                 |
| 函数   | `pg_proc`                  |
| 依赖   | `pg_depend`                |
| 统计   | `pg_stat_*`                |

------

### 📌 特点

- 每个数据库一份
- 互相隔离
- 存在 `base/<db_oid>/`

------

# 四、为什么要这样设计？

## 🔹 1. 数据隔离

不同数据库可以有：

- 同名表
- 同名函数
- 不同 schema

👉 所以必须独立 catalog

------

## 🔹 2. 全局对象必须共享

例如：

- 用户（login）
- 数据库本身
- 表空间

👉 不可能每个数据库一套

------

# 五、如何判断一个 catalog 是否共享？

## 方法 1️⃣：看 `pg_class.relisshared`

```sql
SELECT relname, relisshared
FROM pg_class
WHERE relname LIKE 'pg_%';
```

👉 `relisshared = true` 就是共享

------

## 方法 2️⃣：源码（更底层🔥）

在源码目录：

```
src/include/catalog/*.h
```

例如：

```c
CATALOG(pg_database,1262,DatabaseRelationId) BKI_SHARED_RELATION
```

👉 `BKI_SHARED_RELATION` = 共享表

------

# 六、一个非常容易误解的点⚠️

### ❗ `pg_proc` 不是共享的

很多人以为函数是全局的，其实：

👉 ❌ 不是
👉 ✅ 每个数据库独立

------

# 七、总结（面试级答案）

👉 PostgreSQL catalog 分为两类：

### ✔ 共享 catalog（cluster-wide）

- 存在 `global/`
- 所有数据库可见
- 如：
  - `pg_database`
  - `pg_authid`
  - `pg_tablespace`

------

### ✔ 数据库 catalog（per-database）

- 存在 `base/<db_oid>/`
- 每个数据库独立
- 如：
  - `pg_class`
  - `pg_attribute`
  - `pg_proc`

------

# 八、一句话理解

👉 **“用户和数据库是全局的，其它对象基本都是数据库私有的”**

------

如果你愿意再深入一层，我可以带你看：

👉 shared catalog 在 buffer manager 里的特殊处理
👉 relisshared 如何影响文件路径（非常经典源码点🔥）
👉 initdb 时这些表是怎么初始化的

这些就是 PostgreSQL 内核设计的关键细节了。