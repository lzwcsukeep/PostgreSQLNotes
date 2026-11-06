按照官方描述，最值得测试的不是只做几个 `SELECT`，而是把 **PostgreSQL 原表 → Mooncake columnstore mirror → 数据同步 → UPDATE/DELETE → 聚合/过滤/排序 → JOIN → 大量数据** 这一整条链路测一遍。

官方目前对 `pg_mooncake` 的定位就是：给 PostgreSQL 表创建一个 **Iceberg/列式存储 mirror**，通过 Moonlink 接收 `INSERT/UPDATE/DELETE` 变化，并让分析查询访问这个 mirror。([GitHub](https://github.com/Mooncake-Labs/pg_mooncake/blob/main/README.md))

下面给你一套比较完整的测试 SQL，建议直接在你现在的 `testdb` 里跑。

------

# 1. 基础测试：INSERT 同步

先建立测试环境：

```sql
DROP TABLE IF EXISTS trades CASCADE;

CREATE TABLE trades(
    id      bigint PRIMARY KEY,
    symbol  text,
    time    timestamp,
    price   real
);

CALL mooncake.create_table('trades_iceberg', 'trades');
```

插入数据：

```sql
INSERT INTO trades VALUES
    (1, 'AMD',  '2024-06-05 10:00:00', 119),
    (2, 'AMZN', '2024-06-05 10:05:00', 207),
    (3, 'AAPL', '2024-06-05 10:10:00', 203),
    (4, 'AMZN', '2024-06-05 10:15:00', 210),
    (5, 'AMD',  '2024-06-05 10:20:00', 121),
    (6, 'AAPL', '2024-06-05 10:25:00', 205);
```

然后：

```sql
SELECT *
FROM trades_iceberg
ORDER BY id;
```

预期：

```text
 id | symbol |        time         | price
----+--------+---------------------+-------
  1 | AMD    | 2024-06-05 10:00:00 |   119
  2 | AMZN   | 2024-06-05 10:05:00 |   207
  3 | AAPL   | 2024-06-05 10:10:00 |   203
  4 | AMZN   | 2024-06-05 10:15:00 |   210
  5 | AMD    | 2024-06-05 10:20:00 |   121
  6 | AAPL   | 2024-06-05 10:25:00 |   205
```

这个测试验证：

> `trades` 的 INSERT 是否进入 `trades_iceberg`。

------

# 2. 测试单条 INSERT 的实时同步

继续插入：

```sql
INSERT INTO trades VALUES
    (7, 'NVDA', '2024-06-05 10:30:00', 140);
```

然后：

```sql
SELECT *
FROM trades_iceberg
WHERE id = 7;
```

预期：

```text
 id | symbol |        time         | price
----+--------+---------------------+-------
  7 | NVDA   | 2024-06-05 10:30:00 |   140
```

这个测试非常重要，因为它更接近：

```text
INSERT
   ↓
PostgreSQL WAL
   ↓
logical replication / CDC
   ↓
Moonlink
   ↓
columnstore / Iceberg
```

官方也明确把 `INSERT/UPDATE/DELETE` 的实时 ingestion 作为 pg_mooncake 的核心能力。([GitHub](https://github.com/Mooncake-Labs/pg_mooncake/blob/main/README.md?utm_source=chatgpt.com))

------

# 3. 测试 UPDATE 同步

修改 AMD：

```sql
UPDATE trades
SET price = 130
WHERE id = 1;
```

检查：

```sql
SELECT *
FROM trades_iceberg
WHERE id = 1;
```

预期：

```text
 id | symbol |        time         | price
----+--------+---------------------+-------
  1 | AMD    | 2024-06-05 10:00:00 |   130
```

再测试多个字段：

```sql
UPDATE trades
SET symbol = 'AMD2',
    price = 135,
    time = '2024-06-05 11:00:00'
WHERE id = 1;
```

然后：

```sql
SELECT *
FROM trades_iceberg
WHERE id = 1;
```

检查整个 tuple 是否同步。

------

# 4. 测试 DELETE 同步

删除：

```sql
DELETE FROM trades
WHERE id = 7;
```

然后：

```sql
SELECT *
FROM trades_iceberg
WHERE id = 7;
```

预期：

```text
(0 rows)
```

再看看数量：

```sql
SELECT count(*) FROM trades;
SELECT count(*) FROM trades_iceberg;
```

两个结果应该一致。

------

# 5. 最重要的：源表和 Mirror 一致性测试

这个我非常建议你做。

```sql
SELECT
    (SELECT count(*) FROM trades) AS postgres_count,
    (SELECT count(*) FROM trades_iceberg) AS mooncake_count;
```

进一步：

```sql
SELECT
    (SELECT count(*) FROM trades)
    =
    (SELECT count(*) FROM trades_iceberg)
    AS count_equal;
```

预期：

```text
 count_equal
-------------
 t
```

------

# 6. 测试聚合

这是 Mooncake 的主要使用场景。

### AVG

```sql
SELECT avg(price)
FROM trades_iceberg;
```

### SUM

```sql
SELECT sum(price)
FROM trades_iceberg;
```

### MIN / MAX

```sql
SELECT
    min(price),
    max(price)
FROM trades_iceberg;
```

### COUNT

```sql
SELECT count(*)
FROM trades_iceberg;
```

------

# 7. GROUP BY 测试

例如统计每个股票：

```sql
SELECT
    symbol,
    count(*) AS trade_count,
    avg(price) AS avg_price,
    min(price) AS min_price,
    max(price) AS max_price
FROM trades_iceberg
GROUP BY symbol
ORDER BY symbol;
```

这类查询就比较典型：

```text
        trades_iceberg
              │
              ↓
       Columnar Scan
              │
              ↓
        GROUP BY symbol
              │
              ↓
       aggregation
```

------

# 8. WHERE + 聚合

例如：

```sql
SELECT
    symbol,
    avg(price) AS avg_price
FROM trades_iceberg
WHERE price > 200
GROUP BY symbol
ORDER BY avg_price DESC;
```

再测试时间范围：

```sql
SELECT
    symbol,
    count(*),
    avg(price)
FROM trades_iceberg
WHERE time >= '2024-06-05 10:10:00'
GROUP BY symbol;
```

------

# 9. ORDER BY + LIMIT

```sql
SELECT *
FROM trades_iceberg
ORDER BY price DESC
LIMIT 3;
```

这是很常见的 OLAP 查询：

```sql
SELECT symbol, price
FROM trades_iceberg
ORDER BY price DESC
LIMIT 10;
```

------

# 10. 多条件过滤

```sql
SELECT *
FROM trades_iceberg
WHERE symbol IN ('AMD', 'AMZN')
  AND price > 120
ORDER BY time;
```

再测试：

```sql
SELECT *
FROM trades_iceberg
WHERE price BETWEEN 120 AND 210
  AND time >= '2024-06-05 10:10:00';
```

------

# 11. 测试 NULL

这个也建议测。

先插入：

```sql
INSERT INTO trades VALUES
    (8, 'TSLA', '2024-06-05 11:00:00', NULL);
```

查询：

```sql
SELECT *
FROM trades_iceberg
WHERE id = 8;
```

再：

```sql
SELECT
    count(*) AS total,
    count(price) AS non_null_price
FROM trades_iceberg;
```

测试：

```sql
SELECT *
FROM trades_iceberg
WHERE price IS NULL;
```

------

# 12. 测试 UPDATE → DELETE 连续变化

这个比较适合测试 CDC。

```sql
INSERT INTO trades VALUES
    (100, 'TEST', '2024-06-05 12:00:00', 100);
```

检查：

```sql
SELECT * FROM trades_iceberg WHERE id = 100;
```

然后：

```sql
UPDATE trades
SET price = 200
WHERE id = 100;
```

检查：

```sql
SELECT * FROM trades_iceberg WHERE id = 100;
```

然后：

```sql
DELETE FROM trades
WHERE id = 100;
```

最后：

```sql
SELECT * FROM trades_iceberg WHERE id = 100;
```

形成：

```text
INSERT
  ↓
id=100 price=100

UPDATE
  ↓
id=100 price=200

DELETE
  ↓
不存在
```

这个测试比单独 INSERT 更能体现 Moonlink/CDC 的作用。

------

# 13. 测试 JOIN

再创建一个普通 PostgreSQL 表：

```sql
CREATE TABLE symbols (
    symbol text PRIMARY KEY,
    company text,
    sector text
);
```

插入：

```sql
INSERT INTO symbols VALUES
    ('AMD',  'AMD',  'Semiconductor'),
    ('AMZN', 'Amazon', 'Cloud'),
    ('AAPL', 'Apple', 'Consumer'),
    ('NVDA', 'NVIDIA', 'Semiconductor'),
    ('TSLA', 'Tesla', 'Automotive');
```

然后：

```sql
SELECT
    t.symbol,
    s.company,
    s.sector,
    avg(t.price) AS avg_price
FROM trades_iceberg t
JOIN symbols s
    ON t.symbol = s.symbol
GROUP BY
    t.symbol,
    s.company,
    s.sector
ORDER BY avg_price DESC;
```

这个测试很有意思，因为它测试：

```text
                 ┌── trades_iceberg
                 │
PostgreSQL JOIN ─┤
                 │
                 └── symbols
```

也可以进一步观察 pg_duckdb / DuckDB 对这种查询的处理。

------

# 14. 测试 JOIN + GROUP BY + HAVING

例如：

```sql
SELECT
    s.sector,
    count(*) AS trades,
    avg(t.price) AS avg_price
FROM trades_iceberg t
JOIN symbols s
    ON t.symbol = s.symbol
GROUP BY s.sector
HAVING count(*) >= 2
ORDER BY avg_price DESC;
```

这已经比较接近真正的分析型 SQL。

------

# 15. 大量数据测试

前面的数据太少，基本测不出列式存储的优势。

可以生成 100 万行：

```sql
INSERT INTO trades
SELECT
    g AS id,
    CASE g % 5
        WHEN 0 THEN 'AAPL'
        WHEN 1 THEN 'AMD'
        WHEN 2 THEN 'AMZN'
        WHEN 3 THEN 'NVDA'
        ELSE 'TSLA'
    END AS symbol,
    '2024-01-01'::timestamp
        + (g || ' minutes')::interval AS time,
    100 + random() * 200 AS price
FROM generate_series(1000, 1000999) AS g;
```

然后：

```sql
SELECT count(*)
FROM trades;
```

以及：

```sql
SELECT count(*)
FROM trades_iceberg;
```

确认 mirror 已经同步。

------

# 16. 大数据聚合测试

然后跑：

```sql
SELECT
    symbol,
    count(*),
    avg(price),
    min(price),
    max(price)
FROM trades_iceberg
GROUP BY symbol
ORDER BY symbol;
```

这个时候才比较适合观察 Mooncake 的分析能力。

------

# 17. 时间范围分析

因为你的 `trades` 本身就是时间序列数据，可以测试：

```sql
SELECT
    date_trunc('day', time) AS day,
    symbol,
    count(*) AS cnt,
    avg(price) AS avg_price
FROM trades_iceberg
GROUP BY
    date_trunc('day', time),
    symbol
ORDER BY day, symbol;
```

再测试：

```sql
SELECT
    date_trunc('hour', time) AS hour,
    avg(price)
FROM trades_iceberg
WHERE time >= '2024-01-10'
  AND time <  '2024-01-20'
GROUP BY date_trunc('hour', time)
ORDER BY hour;
```

这种 workload 就很接近典型 OLAP。

------

# 18. 和普通 PostgreSQL Heap Table 对比

这个是我最推荐你做的测试。

你的数据现在同时存在：

```text
trades
   │
   └── PostgreSQL Heap Table

trades_iceberg
   │
   └── Mooncake Columnstore
```

所以可以直接比较：

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT
    symbol,
    count(*),
    avg(price),
    min(price),
    max(price)
FROM trades
GROUP BY symbol;
```

和：

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT
    symbol,
    count(*),
    avg(price),
    min(price),
    max(price)
FROM trades_iceberg
GROUP BY symbol;
```

再比较过滤：

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT
    avg(price)
FROM trades
WHERE symbol = 'NVDA'
  AND price > 200;
```

vs：

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT
    avg(price)
FROM trades_iceberg
WHERE symbol = 'NVDA'
  AND price > 200;
```

这就能把你前面研究的 **PostgreSQL executor / pg_duckdb / DuckDB vectorized execution / columnstore** 串起来。

------

# 19. 做一个“数据一致性”自动测试

可以写一个非常实用的 SQL：

```sql
SELECT
    (SELECT count(*) FROM trades)
        =
    (SELECT count(*) FROM trades_iceberg)
    AS count_match;
```

再比较聚合：

```sql
SELECT
    (SELECT sum(price) FROM trades)
        =
    (SELECT sum(price) FROM trades_iceberg)
    AS sum_match;
```

再比较：

```sql
SELECT
    (SELECT avg(price) FROM trades)
        =
    (SELECT avg(price) FROM trades_iceberg)
    AS avg_match;
```

不过 `real` 使用浮点数，`avg()` 做严格 `=` 比较时可能存在浮点精度问题。更稳妥可以：

```sql
SELECT
    abs(
        (SELECT avg(price) FROM trades)
        -
        (SELECT avg(price) FROM trades_iceberg)
    ) < 0.0001 AS avg_match;
```

------

# 20. 最后做一个综合压力测试

我比较建议你最终跑这一套：

```sql
-- 1. 插入
INSERT INTO trades
SELECT
    g,
    CASE g % 5
        WHEN 0 THEN 'AAPL'
        WHEN 1 THEN 'AMD'
        WHEN 2 THEN 'AMZN'
        WHEN 3 THEN 'NVDA'
        ELSE 'TSLA'
    END,
    '2024-01-01'::timestamp + (g || ' minutes')::interval,
    100 + random() * 200
FROM generate_series(2000000, 3000000) g;
```

然后：

```sql
-- 2. 一致性
SELECT
    (SELECT count(*) FROM trades),
    (SELECT count(*) FROM trades_iceberg);
-- 3. 聚合
SELECT
    symbol,
    count(*),
    avg(price),
    min(price),
    max(price)
FROM trades_iceberg
GROUP BY symbol;
-- 4. 时间过滤
SELECT
    date_trunc('day', time) AS day,
    symbol,
    count(*),
    avg(price)
FROM trades_iceberg
WHERE time >= '2024-01-10'
  AND time <  '2024-01-20'
GROUP BY day, symbol
ORDER BY day, symbol;
-- 5. Top-N
SELECT
    symbol,
    price,
    time
FROM trades_iceberg
ORDER BY price DESC
LIMIT 100;
-- 6. 复杂分析
SELECT
    symbol,
    count(*) AS cnt,
    avg(price) AS avg_price
FROM trades_iceberg
WHERE price > 150
GROUP BY symbol
HAVING count(*) > 10000
ORDER BY avg_price DESC;
```

------

## 我建议你按这个顺序测试

可以把整个实验分成 **5 个阶段**：

| 阶段 | 测试                              | 主要验证                   |
| ---- | --------------------------------- | -------------------------- |
| ①    | INSERT                            | PostgreSQL → Mooncake 同步 |
| ②    | UPDATE / DELETE                   | CDC 是否正确处理变化       |
| ③    | GROUP BY / AVG / SUM              | Columnstore 分析           |
| ④    | JOIN / ORDER BY / HAVING          | 复杂 OLAP                  |
| ⑤    | 百万/千万数据 + `EXPLAIN ANALYZE` | 性能                       |

```text
                  PostgreSQL
                      │
          ┌───────────┴───────────┐
          ↓                       ↓
       trades              trades_iceberg
      Heap Table             Columnstore
          │                       │
          ↓                       ↓
      PG Executor            DuckDB Executor
          │                       │
          └───────────┬───────────┘
                      ↓
                  性能比较
```

另外要注意一个容易混淆的点：**`trades_iceberg` 不是 `trades` 的一个普通 PostgreSQL heap 备份**。官方描述的是一个与源表保持同步的 columnstore mirror，并且底层数据采用 Iceberg/lakehouse 方向的存储；因此你做性能实验时，实际上是在比较 **Heap + PostgreSQL executor** 与 **Mooncake columnstore + DuckDB 执行路径**。([GitHub](https://github.com/Mooncake-Labs/pg_mooncake/blob/main/README.md))