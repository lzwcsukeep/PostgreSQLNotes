# 什么是 DuckDB?

DuckDB 是一个面向分析（OLAP）的、嵌入式的 SQL 数据库，有自己的数据存储文件格式`.duckdb`, 同时也内置直接对 CSV、Parquet、JSON 等文件做高速分析的能力。

## 1. 嵌入式 Embedded

这是理解 DuckDB 最重要的一点。

PostgreSQL的是 C/S 架构，需要启动一个服务进程，然后等待客户端连接，而**DuckDB 默认不需要单独启动一个数据库服务器** ，可以以一个库的形式嵌入到你的应用程序中。

下面是 PostgreSQL 和 DuckDB 工作模式的介绍：

```
PostgreSQL
    ↓
客户端 psql
    ↓
PostgreSQL Server
    ↓
Buffer Pool
    ↓
磁盘上的数据库文件
```

而 DuckDB 更像：

```
你的程序
    ↓
DuckDB Library
    ↓
SQL Engine
    ↓
直接读取 CSV / Parquet / JSON / DuckDB 文件
```

> **DuckDB 默认不需要单独启动一个数据库服务器。**

例如：

```bash
$ duckdb
```

进入 CLI 后：

```
SELECT 1;
```

或者：

```bash
$ duckdb mydb.duckdb
```

这时 `mydb.duckdb` 就是一个 DuckDB 数据库文件。

### DuckDB 嵌入式程序代码结构

DuckDB 可以直接嵌入应用程序：

```
Application
 ├── Your code
 └── DuckDB
      ├── SQL Parser
      ├── Optimizer
      ├── Executor
      ├── Storage
      └── Transaction
```

比如 C++：

```c++
#include "duckdb.hpp"

int main()
{
    duckdb::DuckDB db(nullptr);
    duckdb::Connection con(db);

    auto result = con.Query(
        "SELECT 1 + 2"
    );

    result->Print();
}
```

你的 C++ 程序本身就包含 DuckDB。

不需要：

```bash
systemctl start postgresql
```

也不需要：

```
localhost:5432
```

### DuckDB Cli

DuckDB 直接提供了一个命令行程序，使用DuckDB 的功能。

1. 安装

```bash
$ curl https://install.duckdb.org | bash
```

2. 使用

```bash
$ duckdb
DuckDB v1.4.5 (Andium) f31be57c18
Enter ".help" for usage hints.
Connected to a transient in-memory database.
Use ".open FILENAME" to reopen on a persistent database.
D // 这里执行SQL 
```



**相关链接：**

- DuckDB client APIs (“drivers”): https://duckdb.org/docs/current/clients/overview

- DuckDB C 开发API: https://duckdb.org/docs/current/clients/c/api

- CLI SQL 示例： https://duckdb.org/docs/current/sql/introduction



## 2. DuckDB 的分析型数据库(OLAP)

- **OLTP 事务型**

例如：

```
INSERT INTO users VALUES (...);
UPDATE users SET ...;
SELECT * FROM users WHERE id = 100;
```

这种数据库通常关注：

> **少量数据、快速定位、频繁修改。** PostgreSQL、MySQL、Oracle 都主要用于这种场景。

- **OLAP 分析型**

分析场景则经常是：

```
SELECT
    country,
    SUM(amount),
    AVG(amount),
    COUNT(*)
FROM orders
GROUP BY country;
```

假设orders表1 亿行你可能需要扫描几千万甚至全部数据。这种情况下随机访问某一行并不是重点，重点变成：快速扫描大量数据、过滤、聚合、排序。

这就是 DuckDB 擅长的场景。

- **DuckDB 适合 OLAP 的原因**

列式存储+Columnar Execution ：DuckDB 的执行引擎高度面向列式分析。

## 3. 直接查询文件CSV,Parquet

DuckDB 可以直接读取和查询CSV，Parquet  格式的文件，参见官方数据[导入导出](https://duckdb.org/docs/current/data/overview)文档。

- 读 CSV 文件

```bash
SELECT * FROM 'flights.csv';
CREATE TABLE ontime (
    FlightDate DATE,
    UniqueCarrier VARCHAR,
    OriginCityName VARCHAR,
    DestCityName VARCHAR
);
COPY ontime FROM 'flights.csv';
```

- 读 Parquet  文件

```bash
SELECT * FROM 'test.parquet';
SELECT * FROM read_parquet('test.parquet');
COPY tbl FROM 'test.parquet';
```

- DuckDB 、IceBerg 、Parquet

Parquet 是开放列式存储文件格式，IceBerg  是开放表格式。DuckDB 结合Parquet, IceBerg 可以实现高效的数据分析功能。

其中：

- Parquet负责高效存储数据

- Iceberg负责管理大规模数据表的元数据、Snapshot、Schema、Partition 等**

- DuckDB负责执行 SQL 分析

例如：

```
IceBerg 
 │
 ├── data/part-0001.parquet
 ├── data/part-0002.parquet
 ├── data/part-0003.parquet
 │
 ↓
DuckDB
 │
 ↓
SELECT ...
```

这也是现代 Data Lake / Lakehouse 架构里 DuckDB 很有意思的地方。

## 4. DuckDB 的核心技术：Vectorized Execution

这个尤其值得关注，传统执行方式可以想象成：

```
Tuple-at-a-time

tuple 1
   ↓
filter
   ↓
aggregate

tuple 2
   ↓
filter
   ↓
aggregate

tuple 3
   ↓
filter
   ↓
aggregate
```

而 DuckDB 使用非常重要的：**Vectorized Execution**， 一次处理一批数据：

```
Vector
┌─────────────────────────┐
│ 10 20 30 40 50 ...      │
└─────────────────────────┘
             ↓
          Filter
             ↓
┌─────────────────────────┐
│ 20 40 50 ...             │
└─────────────────────────┘
             ↓
         Aggregate
```

比如一次处理几百/几千个 tuple，而不是一个 tuple 一个 tuple 地处理。这能够更好地利用：

```
CPU Cache
SIMD
Pipeline
Memory Bandwidth
```

## 5. DuckDB 还可以作为数据转换工具

CSV < --> DuckDB <-->Parquet

比如：

```bash
COPY (
    SELECT *
    FROM 'users.csv'
)
TO 'users.parquet'
(FORMAT PARQUET);
```

```bash
COPY (
    SELECT *
    FROM 'users.parquet'
)
TO 'users.csv'
(FORMAT CSV);
```

因此 DuckDB 不只是数据库。它也经常被当作：数据处理 / ETL engine

## 6. DuckDB 和 PostgreSQL 的关系

它们都是 SQL 数据库，但设计目标不同。

|               | PostgreSQL       | DuckDB               |
| ------------- | ---------------- | -------------------- |
| 类型          | Server DBMS      | Embedded DBMS        |
| 主要方向      | OLTP + 通用      | OLAP                 |
| Server        | 有               | 通常没有             |
| Client/Server | 是               | 通常不是             |
| 行存储        | 主要是           | 不同于传统行存储思路 |
| 列式分析      | 不是核心         | 核心能力             |
| CSV           | 支持             | 非常方便             |
| Parquet       | 可通过扩展等方式 | 原生重点能力         |
| 并发事务      | 很强             | 有，但目标不同       |
| 单机分析      | 可以             | 非常强               |
| 分布式数据库  | 否               | 否                   |
| 嵌入 C/C++    | 可以但复杂       | 核心设计之一         |

可以粗略理解：

```
PostgreSQL
    ↓
通用数据库服务器
    ↓
业务系统 / OLTP
```

而：

```
DuckDB
    ↓
嵌入式分析引擎
    ↓
数据分析 / ETL / Data Science / OLAP
```

## 7. DuckDB 和 SQLite 很像，但方向完全不同

如果你了解 SQLite，会更容易理解 DuckDB。

SQLite：

```
Application
     │
     ↓
SQLite
     │
     ↓
xxx.sqlite
```

DuckDB：

```
Application
     │
     ↓
DuckDB
     │
     ↓
xxx.duckdb
```

所以两者都是：

> Embedded Database

但是设计目标不同：

```
SQLite
   ↓
OLTP / Embedded Application
   ↓
行式访问

DuckDB
   ↓
OLAP / Analytics
   ↓
列式分析 + 向量化执行
```

可以简单记：

> **SQLite ≈ 嵌入式 OLTP**

> **DuckDB ≈ 嵌入式 OLAP**

当然这是一个帮助理解的简化，并不是严格定义。

# 总结

DuckDB 是一个**嵌入式**，**列式**，**OLAP数据库**，支持**向量化执行**。既可以读取自己的数据库文件.duckdb，也可以直接读取外部CSV,Parquet文件。可以把DuckDB 嵌入到自己的程序中使用，官方也提供了一个 CLI 命令行使用。



参考链接

- DuckDB 官网：https://duckdb.org/

- DuckDB 文档：https://duckdb.org/docs/current/

- DuckDB GitHub: https://github.com/duckdb/duckdb

