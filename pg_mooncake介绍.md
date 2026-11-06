# pg_mooncake 介绍

# 1. 什么是 pg_mooncake？

**pg_mooncake** 是一个 PostgreSQL 扩展，用于在 PostgreSQL 中提供面向分析场景的列式存储能力。

它的核心思路是：

> 将 PostgreSQL 中的表实时同步到基于 **Iceberg + Parquet** 的列式存储中，再使用 **DuckDB** 进行高性能分析查询。

整体架构可以简单理解为：

```text
                    PostgreSQL
                        |
                        |
                  pg_mooncake
                        |
                        v
                    moonlink
                        |
             +----------+----------+
             |                     |
             v                     v
        Iceberg Table         实时数据同步
             |
             v
      Parquet Data Files
             |
             v
           DuckDB
             |
             v
       Analytics Query
```

因此，pg_mooncake 并不是简单地把 PostgreSQL 的 Heap Table 改成列存储，而是在 PostgreSQL 数据库之外维护一个**列式数据镜像（columnstore mirror）**。

官方地址：https://github.com/Mooncake-Labs/pg_mooncake

------

# 2. 为什么需要 pg_mooncake？

PostgreSQL 的默认存储方式是行式存储。

例如：

```sql
CREATE TABLE orders (
    id       BIGINT,
    user_id  BIGINT,
    price    DOUBLE PRECISION,
    status   TEXT
);
```

PostgreSQL Heap 中的数据可以抽象理解成：

```text
Row 1:
id | user_id | price | status

Row 2:
id | user_id | price | status

Row 3:
id | user_id | price | status
```

这种存储方式非常适合 OLTP：

```sql
SELECT *
FROM orders
WHERE id = 100;
```

因为一个订单的数据通常集中在一起。

但是分析查询经常是：

```sql
SELECT
    user_id,
    SUM(price)
FROM orders
GROUP BY user_id;
```

或者：

```sql
SELECT
    COUNT(*),
    AVG(price),
    MAX(price)
FROM orders;
```

这种查询可能需要扫描大量数据，但实际上只需要访问少数几个列：

```text
user_id
price
```

列式存储在这种场景下更加适合。

------

# 3. pg_mooncake 的核心思想

pg_mooncake 的核心思想可以概括为：

```text
PostgreSQL
    |
    | 事务 / INSERT / UPDATE / DELETE
    |
    v
moonlink
    |
    | 实时同步
    v
Iceberg Table
    |
    +----------------------+
    |                      |
    v                      v
Parquet File 1       Parquet File 2
    |                      |
    +----------+-----------+
               |
               v
             DuckDB
               |
               v
         Analytical Query
```

因此，同一份业务数据实际上可以同时拥有：

```text
PostgreSQL Heap
        +
Iceberg / Parquet Columnstore
```

前者适合事务处理，后者适合分析查询。

------

# 4. pg_mooncake 使用到的几个核心组件

理解 pg_mooncake，需要先理解下面几个组件：

```text
PostgreSQL
    |
    +---- pg_mooncake
              |
              +---- moonlink
              |
              +---- Iceberg
              |
              +---- Parquet
              |
              +---- DuckDB
```

它们承担不同的职责。

------

## 4.1 PostgreSQL

PostgreSQL 是原始数据的数据库系统。

例如：

```sql
CREATE TABLE orders (
    id BIGINT,
    user_id BIGINT,
    price DOUBLE PRECISION
);
```

业务应用仍然可以正常执行：

```sql
INSERT
UPDATE
DELETE
SELECT
```

------

## 4.2 pg_mooncake

pg_mooncake 是 PostgreSQL Extension。

它负责把 PostgreSQL 和 Mooncake 的分析存储体系连接起来。

可以理解成：

```text
PostgreSQL
    |
    v
pg_mooncake
    |
    v
Mooncake analytical storage
```

因此用户仍然可以从 PostgreSQL 的 SQL 接口访问分析数据。

------

## 4.3 moonlink

**moonlink** 负责实时数据摄取（ingestion）。

它可以接收 PostgreSQL 中的数据变化：

```text
INSERT
UPDATE
DELETE
```

并将这些变化同步到分析存储。

可以抽象成：

```text
PostgreSQL

INSERT
UPDATE
DELETE
   |
   v
CDC / Change Stream
   |
   v
moonlink
   |
   v
Iceberg / Parquet
```

因此 pg_mooncake 不需要每次都重新扫描整个 PostgreSQL 表。

------

# 5. PostgreSQL CDC

pg_mooncake 的实时同步涉及 **CDC（Change Data Capture）**。

CDC 的意思是：

> 捕获数据库中的数据变化，而不是不断扫描整个数据库。

例如：

```sql
INSERT INTO orders VALUES
(1, 100, 50);
```

随后：

```text
PostgreSQL
     |
     | INSERT event
     v
    CDC
     |
     v
 moonlink
     |
     v
Parquet / Iceberg
```

如果执行：

```sql
UPDATE orders
SET price = 100
WHERE id = 1;
```

也会产生对应的数据变化事件。

这样就可以实现比较低延迟的数据同步。

------

# 6. Iceberg 是什么？

Apache Iceberg 是一种 **Table Format（表格式）**。

它不是数据库，也不是单个数据文件格式。

Iceberg 主要负责：

> 管理一张逻辑表由哪些数据文件组成，以及这些文件对应的表版本、Schema、Snapshot 等信息。

例如：

```text
orders/
│
├── metadata/
│   ├── metadata.json
│   ├── manifest-list
│   └── manifest files
│
└── data/
    ├── part-00001.parquet
    ├── part-00002.parquet
    └── part-00003.parquet
```

可以理解为：

```text
Iceberg Table
      |
      +---- Metadata
      |
      +---- Parquet File 1
      |
      +---- Parquet File 2
      |
      +---- Parquet File 3
```

------

# 7. Parquet 是什么？

Parquet 是一种**列式数据文件格式**。

一个 Parquet 文件大致可以理解为：

```text
    4-byte magic number "PAR1"
    <Column 1 Chunk 1>
    <Column 2 Chunk 1>
    ...
    <Column N Chunk 1>
    <Column 1 Chunk 2>
    <Column 2 Chunk 2>
    ...
    <Column N Chunk 2>
    ...
    <Column 1 Chunk M>
    <Column 2 Chunk M>
    ...
    <Column N Chunk M>
    File Metadata
    4-byte length in bytes of file metadata (little endian)
    4-byte magic number "PAR1"
```

例如：

```text
orders.parquet

id column
user_id column
price column
status column
```

与 PostgreSQL Heap 的行式存储相比，Parquet 更适合分析型扫描。

------

# 8. Iceberg 和 Parquet 的关系

这是理解 pg_mooncake 时非常重要的一点。

它们不是竞争关系。

可以理解为：

```text
Iceberg
  |
  | 管理
  v
Parquet Files
```

也就是：

- **Parquet：文件格式**
- **Iceberg：表格式**

例如：

```text
Iceberg Table
       |
       +---- part-001.parquet
       |
       +---- part-002.parquet
       |
       +---- part-003.parquet
```

每一个 `.parquet` 文件本身都是一个完整的 Parquet 文件。

Iceberg 负责把这些文件组织成一张逻辑表。

------

# 9. DuckDB 是什么？

DuckDB 是一个面向分析场景的数据库系统，同时也可以作为嵌入式 SQL 分析引擎使用。

它最大的特点之一是：

> 可以直接查询外部数据文件。

例如：

```sql
SELECT *
FROM 'orders.parquet';
```

甚至不需要先把 Parquet 导入 DuckDB 数据库。

DuckDB 可以直接：

```text
Parquet
   |
   v
Parquet Scanner
   |
   v
DuckDB Query Engine
   |
   v
Result
```

------

# 10. DuckDB 的列式执行

DuckDB 使用向量化执行（Vectorized Execution）。

传统的逐行执行可以抽象为：

```text
row 1 -> expression -> result
row 2 -> expression -> result
row 3 -> expression -> result
...
```

向量化执行则会一次处理一批数据：

```text
         Vector
+----------------------+
| 100 | 200 | 300 | ...|
+----------------------+
          |
          v
      Expression
          |
          v
         Vector
```

这样可以减少函数调用、分支等开销，并充分利用 CPU。

------

# 11. DuckDB 为什么适合读取 Parquet？

Parquet 本身就是列式格式。

例如查询：

```sql
SELECT SUM(price)
FROM 'orders.parquet';
```

DuckDB 主要需要读取：

```text
price
```

而不一定需要读取：

```text
id
user_id
status
```

这就是典型的 **Column Projection**。

此外，Parquet 的 Row Group 还保存统计信息，例如：

```text
Row Group 1

price:
min = 1
max = 100


Row Group 2

price:
min = 1000
max = 5000
```

如果查询：

```sql
WHERE price > 1000
```

DuckDB 可以利用这些信息跳过不可能满足条件的 Row Group。

这属于 Predicate Pushdown / Row Group Pruning 等优化。

------

# 12. DuckDB 数据库文件和 Parquet 的区别

DuckDB 也有自己的数据库文件：

```bash
duckdb mydb.duckdb
```

此时：

```text
mydb.duckdb
```

是 DuckDB 自己管理的数据库文件。

而：

```text
orders.parquet
```

是 Parquet 数据文件。

两者不要混淆：

```text
mydb.duckdb
    |
    +---- DuckDB Database


orders.parquet
    |
    +---- Parquet Data File
```

pg_mooncake 的分析存储主要围绕：

```text
Iceberg + Parquet
```

而不是简单地把数据保存成：

```text
xxx.duckdb
```

------

# 13. DuckDB 读取外部文件

DuckDB 可以直接读取：

### CSV

```sql
SELECT *
FROM 'users.csv';
```

### Parquet

```sql
SELECT *
FROM 'orders.parquet';
```

### 多个 Parquet

```sql
SELECT *
FROM 'orders/*.parquet';
```

因此 DuckDB 可以作为数据湖上的查询引擎。

------

# 14. COPY 和 Parquet

DuckDB 还可以使用 `COPY` 将查询结果写入 Parquet。

例如：

```sql
COPY orders
TO 'orders.parquet'
(FORMAT PARQUET);
```

数据流：

```text
DuckDB Table
     |
     | COPY TO
     v
orders.parquet
```

反过来也可以：

```sql
SELECT *
FROM 'orders.parquet';
```

因此：

```text
Parquet
   ^
   |
COPY
   |
DuckDB
```

------

# 15. pg_mooncake 的整体数据流

将前面的内容组合起来：

```text
                         PostgreSQL
                              |
                              |
                         pg_mooncake
                              |
                              v
                           moonlink
                              |
                       PostgreSQL CDC
                              |
                              v
                     +----------------+
                     | Iceberg Table  |
                     +----------------+
                              |
                +-------------+-------------+
                |                           |
                v                           v
        part-00001.parquet          part-00002.parquet
                |                           |
                +-------------+-------------+
                              |
                              v
                           DuckDB
                              |
                              v
                       Vectorized Engine
                              |
                              v
                         SQL Result
```

------

# 16. 一个简单的例子

假设 PostgreSQL 中存在：

```sql
CREATE TABLE orders (
    id BIGINT,
    user_id BIGINT,
    price DOUBLE PRECISION
);
```

插入数据：

```sql
INSERT INTO orders VALUES
(1, 100, 10),
(2, 100, 20),
(3, 200, 50);
```

pg_mooncake 将数据同步到分析存储：

```text
Iceberg Table
      |
      +---- orders-001.parquet
```

然后分析查询：

```sql
SELECT
    user_id,
    SUM(price)
FROM orders
GROUP BY user_id;
```

分析引擎可以从：

```text
Parquet
```

中读取：

```text
user_id
price
```

然后进行：

```text
Scan
  |
  v
Vectorized Execution
  |
  v
Aggregation
  |
  v
Result
```

结果：

```text
user_id | sum
--------+-----
100     | 30
200     | 50
```

------

# 17. pg_mooncake 和传统 PostgreSQL 查询的区别

传统 PostgreSQL：

```text
PostgreSQL Heap
       |
       v
PostgreSQL Executor
       |
       v
Result
```

pg_mooncake：

```text
PostgreSQL
    |
    +---- OLTP / 原生查询
    |
    +---- pg_mooncake
             |
             v
        Iceberg / Parquet
             |
             v
           DuckDB
             |
             v
        Analytics Query
```

因此，两者的核心区别之一是：

```text
PostgreSQL Heap
    VS
Columnar analytical mirror
```

------

# 18. pg_mooncake 的主要价值

pg_mooncake 将几个技术结合起来：

```text
PostgreSQL
     +
CDC
     +
moonlink
     +
Iceberg
     +
Parquet
     +
DuckDB
```

分别解决不同问题：

| 组件        | 主要作用                        |
| ----------- | ------------------------------- |
| PostgreSQL  | 事务数据库 / 原始数据           |
| pg_mooncake | PostgreSQL 与分析存储之间的连接 |
| CDC         | 捕获数据变化                    |
| moonlink    | 实时数据摄取与同步              |
| Iceberg     | 管理分析表                      |
| Parquet     | 列式数据存储                    |
| DuckDB      | 高性能分析查询                  |

------

# 19. 最核心的几个概念

学习 pg_mooncake 时，可以记住下面几个关系：

```text
PostgreSQL
    |
    | 数据变化
    v
   CDC
    |
    v
moonlink
    |
    v
Iceberg Table
    |
    v
Parquet Files
    |
    v
DuckDB
    |
    v
Analytics
```

其中：

```text
Parquet = 文件格式

Iceberg = 表格式

DuckDB = 查询/执行引擎

moonlink = 数据摄取/同步组件

pg_mooncake = PostgreSQL 扩展
```

------

# 20. 与数据仓库 / 数据湖的关系

这种架构与现代 Lakehouse 架构比较接近：

```text
                 PostgreSQL
                     |
                     v
                    CDC
                     |
                     v
                  Iceberg
                     |
                     v
                  Parquet
                     |
        +------------+------------+
        |            |            |
        v            v            v
      DuckDB        Spark       Trino
```

Iceberg + Parquet 提供开放的数据存储层。

不同分析引擎可以读取相同的数据。

因此 Parquet 数据不一定只能由 DuckDB 使用。

------

# 21. 一句话总结

pg_mooncake 可以理解为：

> **在 PostgreSQL 的基础上，通过实时数据同步，将 PostgreSQL 表维护成一个基于 Iceberg + Parquet 的列式数据镜像，并利用 DuckDB 对这个列式数据进行高性能分析查询。**

它把：

```text
PostgreSQL 的事务能力
+
CDC 实时同步
+
Iceberg 表管理
+
Parquet 列式存储
+
DuckDB 向量化分析
```

组合到了一起。

最重要的架构关系是：

```text
                 PostgreSQL
                     |
                     | CDC
                     v
                 pg_mooncake
                     |
                     v
                  moonlink
                     |
                     v
               Iceberg Table
                     |
                     v
              Parquet Files
                     |
                     v
                   DuckDB
                     |
                     v
             Analytical Query
```

# 22. pg_mooncake 测试用例

[pg_mooncake测试用例](./pg_mooncake 测试用例.md)

