> 为什么 pg_type 是数据库级别的?

这个问题问得很到位，其实背后是 PostgreSQL 的一个**核心设计权衡：类型系统是否需要“跨数据库一致”**。

先给你结论👇

------

# 一、结论

👉 `pg_type` 是**数据库级别（per-database）**，因为：

> **类型不仅仅是“名字”，而是和具体对象、OID、依赖关系强绑定的本地元数据**

------

# 二、如果把 `pg_type` 做成共享，会出什么问题？

我们反过来想👇

## ❌ 问题 1：OID 冲突 / 不一致

PostgreSQL 内部大量使用 **OID** 来标识类型：

- 表列：`pg_attribute.atttypid`
- 函数参数：`pg_proc.proargtypes`

👉 如果 `pg_type` 是全局的：

- 不同数据库创建类型
- OID 需要全局唯一

📌 这会导致：

- 全局 OID 分配复杂化
- dump / restore 更困难

------

## ❌ 问题 2：用户自定义类型（UDT）

PostgreSQL 支持：

```sql
CREATE TYPE my_type AS (a int, b text);
```

👉 如果 `pg_type` 是共享的：

- A 数据库创建的类型
- B 数据库也能看到？

这会带来问题：

- 命名冲突
- 权限混乱
- schema 隔离被破坏

------

## ❌ 问题 3：类型依赖“本地对象”

很多类型依赖本数据库对象：

### 例子：复合类型（composite type）

```sql
CREATE TABLE t1 (id int, name text);
```

👉 系统自动创建一个类型：

- 类型名 = 表名
- 存在 `pg_type`

📌 这个类型依赖：

```id="h7lqg6"
pg_class (表)
pg_attribute (列)
```

👉 如果 `pg_type` 是共享的：

- 这个类型引用哪个数据库的表？❌

------

## ❌ 问题 4：namespace（schema）隔离失效

类型属于 schema：

- `pg_type.typnamespace → pg_namespace`

而 `pg_namespace` 是数据库级的

👉 如果 `pg_type` 共享：

- namespace 怎么解析？❌

------

## ❌ 问题 5：扩展（extension）机制

扩展（如 PostGIS）会创建类型：

👉 这些类型是：

- 安装在某个数据库中
- 只在该数据库可见

如果共享：

👉 扩展会变成“全局污染”

------

# 三、设计原则总结

PostgreSQL 的一个隐含原则👇

> **凡是“依赖具体数据库对象”的 catalog，一定是数据库级别**

------

# 四、对比一下为什么这些是共享的

| catalog         | 为什么共享         |
| --------------- | ------------------ |
| `pg_database`   | 数据库本身就是全局 |
| `pg_authid`     | 用户是全局         |
| `pg_tablespace` | 文件路径是全局     |

------

而 `pg_type`：

👉 ❌ 依赖本地对象
👉 ❌ 依赖 schema
👉 ❌ 包含用户定义内容

所以必须是：

👉 ✅ per-database

------

# 五、一个非常关键的理解（建议记住🔥）

👉 PostgreSQL **没有跨数据库查询能力**

```sql
SELECT * FROM db1.table1; -- ❌ 不支持
```

📌 这意味着：

> **每个数据库就是一个“逻辑隔离的 universe”**

所以：

- 表 ✔ 独立
- 函数 ✔ 独立
- 类型 ✔ 必须独立

------

# 六、从源码角度再补一刀（你这个阶段很关键🔥）

在源码中：

- `pg_type` **没有**：

```c
BKI_SHARED_RELATION
```

而：

```c
pg_database → BKI_SHARED_RELATION
```

👉 这就是最直接的证据

------

# 七、一句话总结

👉 **pg_type 之所以是数据库级别，是因为类型本质上是“本地对象结构的一部分”，而不是全局资源**

------

如果你想继续深入，我可以带你看一个更底层的点👇

👉 `pg_type` 和 `pg_class`、`pg_attribute` 的三者关系（类型系统核心）
👉 数组类型、toast 类型是如何自动生成的
👉 OID 在类型系统中的作用（超级关键🔥）

这些是理解 PostgreSQL 内核最有价值的一部分。