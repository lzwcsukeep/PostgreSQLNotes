一句话： PG 的复合类型类似 C 语言的结构体类型。

---

# 复合类型创建语法

创建PostgreSQL 的 **复合类型（composite type）**的语法是：

```sql
CREATE TYPE name AS (
    attribute_name data_type [ COLLATE collation ]
    [, ...]
);
```

可以把它理解成：

> **定义一个“由多个字段组成的新类型”，类似于 C 语言中的 `struct`。**

例如我们定义一个员工信息类型：

```sql
CREATE TYPE employee_info AS (
    id       integer,
    name     text,
    salary   numeric(10,2)
);
```

这就创建了一个叫 `employee_info` 的 PostgreSQL 类型。

------

# 1. 最简单的创建方式

例如：

```sql
CREATE TYPE person AS (
    name text,
    age  integer
);
```

此时：

```text
person
├── name → text
└── age  → integer
```

你可以通过：

```sql
\dT person
```

查看这个类型。

也可以直接查询 `pg_type`：

```sql
SELECT
    oid,
    typname,
    typnamespace,
    typtype
FROM pg_type
WHERE typname = 'person';
```

对于这种类型：

```text
typtype = 'c'
```

其中：

```text
c = composite type
```

------

# 2. 如何创建这种类型的值？

这是最重要的部分。

有了：

```sql
CREATE TYPE person AS (
    name text,
    age integer
);
```

可以这样构造一个 `person`：

```sql
SELECT ROW('Alice', 20)::person;
```

结果类似：

```text
      row
----------------
 (Alice,20)
```

这里：

```sql
ROW('Alice', 20)
```

创建的是一个行值，然后：

```sql
::person
```

把它转换成 `person` 类型。

也可以写成：

```sql
SELECT ('Alice', 20)::person;
```

------

# 3. 可以把它理解成 C 的 struct

例如 C：

```c
struct Person
{
    char *name;
    int age;
};
```

PostgreSQL：

```sql
CREATE TYPE person AS (
    name text,
    age integer
);
```

于是：

```text
C：

Person
├── name
└── age


PostgreSQL：

person
├── name : text
└── age  : integer
```

而：

```sql
ROW('Alice', 20)::person
```

类似于：

```c
Person p = {
    .name = "Alice",
    .age = 20
};
```

------

# 4. 如何访问复合类型的字段？

假设：

```sql
CREATE TYPE person AS (
    name text,
    age integer
);
```

先构造一个值：

```sql
SELECT ROW('Alice', 20)::person;
```

如果想访问 `name`：

```sql
SELECT (ROW('Alice', 20)::person).name;
```

结果：

```text
 name
-------
 Alice
```

访问 `age`：

```sql
SELECT (ROW('Alice', 20)::person).age;
```

结果：

```text
 age
-----
 20
```

这里要特别注意括号：

```sql
(ROW('Alice', 20)::person).name
```

而不是：

```sql
ROW('Alice', 20)::person.name
```

因为 `.` 在这里是访问复合类型字段的语法。

------

# 5. 更实际的例子：把它作为表字段

复合类型最常见的用途之一，就是作为另一个表的字段。

例如：

```sql
CREATE TYPE address AS (
    city    text,
    street  text,
    zipcode text
);
```

然后：

```sql
CREATE TABLE employee (
    id      integer,
    name    text,
    address address
);
```

现在：

```text
employee
├── id
├── name
└── address
      │
      ├── city
      ├── street
      └── zipcode
```

插入数据：

```sql
INSERT INTO employee VALUES (
    1,
    'Alice',
    ROW('Singapore', 'Orchard Road', '238823')
);
```

查询：

```sql
SELECT *
FROM employee;
```

可能得到：

```text
 id | name  |             address
----+-------+--------------------------------
  1 | Alice | (Singapore,"Orchard Road",238823)
```

------

# 6. 如何访问嵌套字段？

例如：

```sql
SELECT (address).city
FROM employee;
```

得到：

```text
   city
-----------
 Singapore
```

或者：

```sql
SELECT (address).zipcode
FROM employee;
```

得到：

```text
 zipcode
---------
 238823
```

所以：

```text
employee
   │
   ▼
address
   │
   ├── city
   ├── street
   └── zipcode
```

SQL：

```sql
(address).city
```

就是：

```text
先取 address
      ↓
再取 address.city
```

------

# 7. 可以直接修改复合类型的某个字段吗？

可以。

例如：

```sql
UPDATE employee
SET address.city = 'Tokyo'
WHERE id = 1;
```

不过 PostgreSQL 对复合字段更新有一些语法细节。更明确的写法是：

```sql
UPDATE employee
SET address = ROW(
    (address).city,
    (address).street,
    (address).zipcode
)
WHERE id = 1;
```

实际使用中，如果经常需要单独更新字段，通常还是把这些字段直接设计成表列。

------

# 8. 复合类型可以作为函数参数

这也是非常有用的地方。

例如：

```sql
CREATE TYPE person AS (
    name text,
    age  integer
);
```

创建一个函数：

```sql
CREATE FUNCTION print_person(p person)
RETURNS text
AS $$
BEGIN
    RETURN p.name || ' is ' || p.age || ' years old';
END;
$$ LANGUAGE plpgsql;
```

然后：

```sql
SELECT print_person(ROW('Alice', 20)::person);
```

得到：

```text
Alice is 20 years old
```

这里就体现了 PostgreSQL 类型系统的一个特点：

```text
person
   │
   ├── 可以作为表字段
   ├── 可以作为函数参数
   ├── 可以作为函数返回值
   ├── 可以创建变量
   └── 可以作为数组元素
```

------

# 9. 可以作为函数返回值

例如：

```sql
CREATE FUNCTION get_person()
RETURNS person
AS $$
    SELECT ROW('Alice', 20)::person;
$$ LANGUAGE SQL;
```

然后：

```sql
SELECT get_person();
```

得到：

```text
 get_person
------------
 (Alice,20)
```

还可以直接访问字段：

```sql
SELECT (get_person()).name;
```

结果：

```text
 Alice
```

------

# 10. 复合类型还可以创建数组

因为 `person` 本身是一个类型，所以它也可以成为数组元素类型。

例如：

```sql
CREATE TYPE person AS (
    name text,
    age integer
);
```

然后：

```sql
SELECT ARRAY[
    ROW('Alice', 20)::person,
    ROW('Bob', 30)::person
];
```

得到类似：

```text
{"(Alice,20)","(Bob,30)"}
```

这体现了 PostgreSQL 类型系统非常统一的一点：

```text
person
  │
  └── person[]
```

就像：

```text
integer
  │
  └── integer[]

text
  │
  └── text[]
```

------

# 11. 这个类型和 `CREATE TABLE` 有一个非常重要的关系

这和你上一轮问的 PostgreSQL 类型系统关系非常大。

你可能会问：

> `CREATE TYPE ... AS (...)` 创建了一个复合类型，那 `CREATE TABLE` 有没有类型？

实际上：

```sql
CREATE TABLE employee (
    id integer,
    name text
);
```

PostgreSQL **也会自动创建一个与这个表对应的复合类型**。

也就是说：

```text
CREATE TABLE employee
        │
        ├───────────────┐
        ▼               ▼
    pg_class          pg_type
     employee          employee
       表                 行类型
```

你可以验证：

```sql
CREATE TABLE employee (
    id integer,
    name text
);
```

然后：

```sql
SELECT
    c.oid AS table_oid,
    c.relname,
    c.reltype AS row_type_oid
FROM pg_class c
WHERE c.relname = 'employee';
```

再：

```sql
SELECT
    oid,
    typname,
    typtype
FROM pg_type
WHERE oid = (
    SELECT reltype
    FROM pg_class
    WHERE relname = 'employee'
);
```

你会看到：

```text
typtype = c
```

也就是：

```text
c = composite
```

------

# 12. 所以 `CREATE TYPE AS (...)` 和表的 row type 很像

例如：

```sql
CREATE TYPE person AS (
    name text,
    age integer
);
```

产生：

```text
person
   │
   ├── name text
   └── age  integer
```

而：

```sql
CREATE TABLE employee (
    name text,
    age integer
);
```

也会产生一个对应的 row type：

```text
employee row type
   │
   ├── name text
   └── age  integer
```

区别在于：

```text
CREATE TYPE
    ↓
只有类型
    ↓
没有表数据


CREATE TABLE
    ↓
创建表
    ↓
同时拥有对应的 row type
```

这是理解 PostgreSQL 类型系统非常关键的一点。

------

# 13. 如何查看复合类型的字段？

这里可以看：

```sql
pg_attribute
```

例如：

```sql
SELECT
    a.attrelid,
    a.attnum,
    a.attname,
    a.atttypid
FROM pg_attribute a
WHERE a.attrelid = 'person'::regclass;
```

不过这里需要注意一个非常容易混淆的地方：

对于独立创建的 composite type：

```sql
CREATE TYPE person AS (
    name text,
    age integer
);
```

它**没有对应的普通表**。

因此更推荐通过 `pg_type` 和相关系统目录/函数来观察它。

你可以先使用：

```sql
\dT+ person
```

以及：

```sql
\d person
```

来查看。

------

# 14. 一个完整的实验

建议你直接在 PostgreSQL 中执行下面这一套：

```sql
DROP TYPE IF EXISTS person;

CREATE TYPE person AS (
    name text,
    age  integer
);
```

构造一个值：

```sql
SELECT ROW('Alice', 20)::person;
```

访问字段：

```sql
SELECT (ROW('Alice', 20)::person).name;
SELECT (ROW('Alice', 20)::person).age;
```

创建表：

```sql
CREATE TABLE people (
    id integer,
    info person
);
```

插入：

```sql
INSERT INTO people
VALUES (
    1,
    ROW('Alice', 20)::person
);
```

查询：

```sql
SELECT *
FROM people;
```

访问嵌套字段：

```sql
SELECT info.name, info.age
FROM people;
```

如果出现解析问题，可以写得更明确：

```sql
SELECT (info).name, (info).age
FROM people;
```

------

# 15. 从 PostgreSQL 源码角度理解

如果你现在是在研究 PostgreSQL 源码，那么这个语句：

```sql
CREATE TYPE person AS (
    name text,
    age integer
);
```

本质上会建立一个：

```text
Composite Type
```

对应的类型信息最终会进入：

```text
pg_type
```

其中：

```text
pg_type
   │
   ├── oid
   ├── typname = person
   ├── typnamespace
   ├── typtype = 'c'
   └── ...
```

然后它的字段信息和类型之间的关系，会通过系统目录建立起来。

所以从你目前学习的 PostgreSQL 系统目录角度，可以把它串成：

```text
CREATE TYPE person AS (
    name text,
    age integer
);
             │
             ▼
        类型定义
             │
             ▼
          pg_type
             │
       ┌─────┴─────┐
       │           │
       ▼           ▼
    person       typtype='c'
       │
       │
       ├── name → text
       │
       └── age  → int4
```

**最值得记住的一点是：`CREATE TYPE ... AS (...)` 创建的不是一个表，而是一个真正的 PostgreSQL composite type。这个类型可以像 `integer`、`text` 一样成为表字段、函数参数、函数返回值，甚至数组的元素类型。**