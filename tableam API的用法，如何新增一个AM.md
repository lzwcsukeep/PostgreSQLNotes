新增一个 存储引擎就是新增一个 TableAM ，PostgreSQL 将 tableam API 开放出来了， 插件可以实现自己的表访问方法。

最主要的结构体就是 `TableAmRoutine`。实现 `TableAmRoutine` 结构体里面的方法就是实现自己的存储引擎。

除此之外还有一些 SQL 层面的工作。因为 `Postgresql` 对 访问方法 am 也有一个系统表 `pg_am`. 

如何编写自己的存储引擎？

- 定义一个全局的 `TableAmRoutine` 对象，并实现里面的方法。
- 还有一些 SQL 层面的工作，让用户建表的时候使用你的存储引擎存储表(USING ...)





SQL 层面的工作阅读一下官方参考链接：

- https://www.postgresql.org/docs/current/tableam.html

- https://www.postgresql.org/docs/current/catalog-pg-am.html