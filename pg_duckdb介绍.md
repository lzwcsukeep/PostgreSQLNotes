官方介绍：

# pg_duckdb: Official PostgreSQL Extension for DuckDB

**pg_duckdb** integrates DuckDB's columnar-vectorized analytics engine into PostgreSQL, enabling high-performance analytics and data-intensive applications. Built in collaboration with [Hydra](https://hydra.so/) and [MotherDuck](https://motherduck.com/).

## Key Features

- **Execute analytics queries without changes**: run your existing SQL analytics queries as you normally would, and `pg_duckdb` will automatically use DuckDB's SQL engine to execute them when you set `duckdb.force_execution=true`.
- **Read/write data from data lakes**: Read/write* Parquet, CSV, JSON, Iceberg & Delta Lake from S3, GCS, Azure & R2.
- **Integration with cloud analytics**: Out-of-the-box support of [MotherDuck](https://motherduck.com/) as compute provider.

## How `pg_duckdb` works

`pg_duckdb` automatically accelerates your existing analytical queries.

- **No special syntax needed:** for queries on your regular PostgreSQL tables, you don't need to change your SQL. Just run your `SELECT` statements as you normally would, and `pg_duckdb` will use DuckDB's engine to execute them.
- **No data export required:** You do not need to export your data to Parquet or any other format. `pg_duckdb` works directly with your existing PostgreSQL tables.

---

理解：

`pg_duckdb` 是一个 `PostgreSQL` 插件，把 `DuckDB` 的向量化执行能力集成进了 `PostgreSQL`, 可以直接使用向量化执行能力查询 PG 原生的表。表的存储还是使用的PG 的 heap 存储方式。注意，没有把PG的行式表变成列式表。中间应该会把 Tuples 转化层 `DuckDB` 的 `DataChunk` 和 `Vector` 。