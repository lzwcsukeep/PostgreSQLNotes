目前已了解到的知识1：

PG 可以管理函数对象。函数对象也有 Oid, 内部使用 Oid 唯一识别函数。此处说的函数指的是 SQL 层的函数，SQL 语句 Create Function 创建的函数。

SQL 函数的元数据信息保存在系统表 `pg_proc` 

`pg_proc` 的结构如下：

```c
                   Table "pg_catalog.pg_proc"
     Column      |     Type     | Collation | Nullable | Default 
-----------------+--------------+-----------+----------+---------
 oid             | oid          |           | not null |               <<< 函数唯一标识: Oid
 proname         | name         |           | not null |               <<< 函数名称
 pronamespace    | oid          |           | not null | 
 proowner        | oid          |           | not null | 
 prolang         | oid          |           | not null | 
 procost         | real         |           | not null | 
 prorows         | real         |           | not null | 
 provariadic     | oid          |           | not null | 
 prosupport      | regproc      |           | not null | 
 prokind         | "char"       |           | not null | 
 prosecdef       | boolean      |           | not null | 
 proleakproof    | boolean      |           | not null | 
 proisstrict     | boolean      |           | not null | 
 proretset       | boolean      |           | not null | 
 provolatile     | "char"       |           | not null | 
 proparallel     | "char"       |           | not null | 
 pronargs        | smallint     |           | not null |                   <<< 函数参数个数
 pronargdefaults | smallint     |           | not null | 
 prorettype      | oid          |           | not null |                   <<< 函数返回值类型 
 proargtypes     | oidvector    |           | not null |                   <<< 函数参数类型
 proallargtypes  | oid[]        |           |          | 
 proargmodes     | "char"[]     |           |          | 
 proargnames     | text[]       | C         |          | 
 proargdefaults  | pg_node_tree | C         |          | 
 protrftypes     | oid[]        |           |          | 
 prosrc          | text         | C         | not null | 
 probin          | text         | C         |          | 
 prosqlbody      | pg_node_tree | C         |          | 
 proconfig       | text[]       | C         |          | 
 proacl          | aclitem[]    |           |          | 

```



目前已了解的知识2：

- Oid 是持久化到系统表 `pg_proc` 里面的 

- PG 内核使用一个结构体来来记录函数的 Oid 和实际执行的 C 函数等信息，这个结构体是 `FmgrBuiltin` .

```c
/*
 * This table stores info about all the built-in functions (ie, functions
 * that are compiled into the Postgres executable).
 */

typedef struct
{
	Oid			foid;			/* OID of the function */
	short		nargs;			/* 0..FUNC_MAX_ARGS, or -1 if variable count */
	bool		strict;			/* T if function is "strict" */
	bool		retset;			/* T if function returns a set */
	const char *funcName;		/* C name of the function */
	PGFunction	func;			/* pointer to compiled function */
} FmgrBuiltin;
```

- 内核里面还有上面的结构体数组，记录了 PG 内置的函数。如下：

```c

const FmgrBuiltin fmgr_builtins[] = {
  { 3, 1, true, false, "heap_tableam_handler", heap_tableam_handler },
  { 31, 1, true, false, "byteaout", byteaout },
  { 33, 1, true, false, "charout", charout },
  { 34, 1, true, false, "namein", namein },
  { 35, 1, true, false, "nameout", nameout },
  { 38, 1, true, false, "int2in", int2in },
  { 39, 1, true, false, "int2out", int2out },
  { 40, 1, true, false, "int2vectorin", int2vectorin },
  { 41, 1, true, false, "int2vectorout", int2vectorout },
  { 42, 1, true, false, "int4in", int4in },
  { 43, 1, true, false, "int4out", int4out },
  { 44, 1, true, false, "regprocin", regprocin },
  { 45, 1, true, false, "regprocout", regprocout },
  { 46, 1, true, false, "textin", textin },
  { 47, 1, true, false, "textout", textout },
  { 48, 1, true, false, "tidin", tidin },
  { 49, 1, true, false, "tidout", tidout },
  { 50, 1, true, false, "xidin", xidin },
  { 51, 1, true, false, "xidout", xidout },
  { 52, 1, true, false, "cidin", cidin },
  { 53, 1, true, false, "cidout", cidout },
  { 54, 1, true, false, "oidvectorin", oidvectorin },
  { 55, 1, true, false, "oidvectorout", oidvectorout },
  { 56, 2, true, false, "boollt", boollt },
  { 57, 2, true, false, "boolgt", boolgt },
  { 60, 2, true, false, "booleq", booleq },
  { 61, 2, true, false, "chareq", chareq },
  { 62, 2, true, false, "nameeq", nameeq },
  { 63, 2, true, false, "int2eq", int2eq },
  { 64, 2, true, false, "int2lt", int2lt },
  { 65, 2, true, false, "int4eq", int4eq },
  ...
  ...
  ...
  { 6308, 1, true, false, "window_ntile_support", window_ntile_support },
  { 6309, 1, true, false, "pg_stat_get_db_conflict_logicalslot", pg_stat_get_db_conflict_logicalslot },
  { 6310, 1, true, false, "pg_stat_get_lastscan", pg_stat_get_lastscan },
  { 6311, 0, true, false, "system_user", system_user }
};  // 该结构体数组总共2989 个元素。
// 6311 是 Oid , "system_user" 是名称字符串， system_user 是 函数指针
```

- PG 函数在 `psql` 执行的时候，应该会查 `pg_proc` 系统表，然后拿着 `Oid`  查这个函数记录表。就能找到实际要执行的 C 函数。函数的结果可以传递给内核其他模块。



// 函数管理了解到此为止