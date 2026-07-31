# PG 判断可见性规则

这是 PostgreSQL MVCC 里最核心的机制之一。

你如果真正理解：

- `xmin`
- `xmax`
- snapshot

那 PostgreSQL 的：

- UPDATE
- DELETE
- VACUUM
- HOT
- REPEATABLE READ

基本都会突然变清晰。

------

# 一、先理解：PostgreSQL 的 UPDATE 不是真修改

在 PostgreSQL 里：

```sql
UPDATE t SET x = 2 WHERE id = 1;
```

并不是：

```text
把旧值覆盖
```

而是：

```text
旧 tuple 标记删除
+
插入一个新 tuple
```

所以：

> PostgreSQL 里一个逻辑行，可能同时存在多个 tuple 版本

------

# 二、tuple 里的两个核心字段

每个 heap tuple 都有：

| 字段 | 含义                   |
| ---- | ---------------------- |
| xmin | 创建该 tuple 的事务 ID |
| xmax | 删除该 tuple 的事务 ID |

------

# 三、最基础例子

假设：

事务 100 插入：

```sql
INSERT INTO t VALUES (1);
```

heap tuple：

| value | xmin | xmax |
| ----- | ---- | ---- |
| 1     | 100  | 0    |

意思：

- 事务 100 创建
- 还没被删除

------

# 四、DELETE 发生了什么

事务 101：

```sql
DELETE FROM t WHERE value = 1;
```

并不会删物理数据。

而是：

| value | xmin | xmax |
| ----- | ---- | ---- |
| 1     | 100  | 101  |

意思：

- tuple 是 100 创建的
- 被 101 删除了

------

# 五、UPDATE 的真实本质

```sql
UPDATE t SET value = 2 WHERE value = 1;
```

实际：

## 旧 tuple：

| value | xmin | xmax |
| ----- | ---- | ---- |
| 1     | 100  | 101  |

------

## 新 tuple：

| value | xmin | xmax |
| ----- | ---- | ---- |
| 2     | 101  | 0    |

------

所以：

> UPDATE = DELETE + INSERT

这是 PostgreSQL MVCC 的根基。

------

# 六、那到底谁能看到哪个版本？

这就要引入：

# Snapshot（事务快照）

事务启动时，会得到一个 snapshot。

snapshot 里最重要的是：

| 字段  | 含义                 |
| ----- | -------------------- |
| xmin  | 当前系统最老活跃事务 |
| xmax  | 下一个可分配事务 ID  |
| xip[] | 当前活跃事务列表     |

------

比如：

当前：

| 事务 | 状态   |
| ---- | ------ |
| 100  | 已提交 |
| 101  | 活跃中 |
| 102  | 活跃中 |

新事务启动时 snapshot：

```text
xmin = 101
xmax = 103
xip = [101,102]
```

意思：

- 101/102 正在运行

- <101 的事务已经结束

- > =103 的事务还没开始

------

# 七、PostgreSQL 判断 tuple 可见性的核心逻辑

真正核心来了。

读取 tuple 时：

------

# 第一步：检查 xmin（创建者）

## 规则：

### 情况1：xmin 对应事务未提交

```text
不可见
```

因为：

> 创建者自己都没 commit

------

### 情况2：xmin 在 snapshot 的 active list 里

比如：

```text
xip = [101,102]
```

tuple：

```text
xmin = 101
```

说明：

> 创建 tuple 的事务，在你开始时还没提交

所以：

```text
不可见
```

------

### 情况3：xmin < snapshot xmin

说明：

```text
tuple 在你事务开始前就已提交
```

所以：

```text
可见
```

------

# 第二步：检查 xmax（删除者）

如果：

```text
xmax = 0
```

说明：

```text
没人删
```

所以：

```text
可见
```

------

如果：

```text
xmax = 某事务
```

还要判断：

------

## 情况1：xmax 事务未提交

说明：

```text
删除操作还没生效
```

所以：

```text
tuple 仍可见
```

------

## 情况2：xmax 在 snapshot active list

说明：

> 删除事务在你开始时还没提交

所以：

```text
tuple 仍可见
```

------

## 情况3：xmax 已提交且早于 snapshot

说明：

```text
tuple 在你事务开始前就被删了
```

所以：

```text
不可见
```

------

# 八、最经典场景（一定要理解）

## 事务 A

```sql
BEGIN;
UPDATE t SET x = 2 WHERE x = 1;
```

产生：

旧 tuple：

| x    | xmin | xmax |
| ---- | ---- | ---- |
| 1    | 100  | 200  |

新 tuple：

| x    | xmin | xmax |
| ---- | ---- | ---- |
| 2    | 200  | 0    |

事务 A 还没提交。

------

## 事务 B 查询

它的 snapshot 看到：

```text
200 still active
```

所以：

### 旧 tuple

```text
xmax=200 但未提交
```

=> 仍可见

------

### 新 tuple

```text
xmin=200 未提交
```

=> 不可见

------

所以事务 B 仍看到：

```text
x = 1
```

这就是：

> PostgreSQL 如何避免 dirty read

------

# 九、VACUUM 为什么存在

因为旧 tuple 不会立刻删除：

| x    | xmin | xmax |
| ---- | ---- | ---- |
| 1    | 100  | 200  |

仍在 heap page。

VACUUM 判断：

```text
是否还有事务可能看到它？
```

如果：

```text
所有 snapshot 都不可能再访问
```

才真正回收。

------

# 十、一句话总结整个 MVCC

PostgreSQL 的可见性本质：

> tuple 是否可见 = xmin/xmax 与当前事务 snapshot 的比较结果

------

# 十一、最核心理解（真正底层）

PostgreSQL 根本不是：

```text
“表里只有一行数据”
```

而是：

```text
“表里有多个历史版本”
```

而 snapshot 决定：

> 当前事务应该看到哪个历史时间点。