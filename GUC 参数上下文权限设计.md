直接读这个ChatGPT 回答：GUC参数的上下文字段初始化之后可以修改吗？

guc 参数通用字段:

```c
/*
 * Generic fields applicable to all types of variables
 *
 * The short description should be less than 80 chars in length. Some
 * applications may use the long description as well, and will append
 * it to the short description. (separated by a newline or '. ')
 *
 * srole is the role that set the current value, or BOOTSTRAP_SUPERUSERID
 * if the value came from an internal source or the config file.  Similarly
 * for reset_srole (which is usually BOOTSTRAP_SUPERUSERID, but not always).
 *
 * Variables that are currently of active interest for maintenance
 * operations are linked into various lists using the xxx_link fields.
 * The link fields are unused/garbage in variables not currently having
 * the specified properties.
 *
 * Note that sourcefile/sourceline are kept here, and not pushed into stacked
 * values, although in principle they belong with some stacked value if the
 * active value is session- or transaction-local.  This is to avoid bloating
 * stack entries.  We know they are only relevant when source == PGC_S_FILE.
 */
struct config_generic
{
	/* constant fields, must be set correctly in initial value: */
	const char *name;			/* name of variable - MUST BE FIRST */
	GucContext	context;		/* context required to set the variable */
	enum config_group group;	/* to help organize variables by function */
	const char *short_desc;		/* short desc. of this variable's purpose */
	const char *long_desc;		/* long desc. of this variable's purpose */
	int			flags;			/* flag bits, see guc.h */
	/* variable fields, initialized at runtime: */
	enum config_type vartype;	/* type of variable (set only at startup) */
	int			status;			/* status bits, see below */
	GucSource	source;			/* source of the current actual value */
	GucSource	reset_source;	/* source of the reset_value */
	GucContext	scontext;		/* context that set the current value */
	GucContext	reset_scontext; /* context that set the reset value */
	Oid			srole;			/* role that set the current value */
	Oid			reset_srole;	/* role that set the reset value */
	GucStack   *stack;			/* stacked prior values */
	void	   *extra;			/* "extra" pointer for current actual value */
	dlist_node	nondef_link;	/* list link for variables that have source
								 * different from PGC_S_DEFAULT */
	slist_node	stack_link;		/* list link for variables that have non-NULL
								 * stack */
	slist_node	report_link;	/* list link for variables that have the
								 * GUC_NEEDS_REPORT bit set in status */
	char	   *last_reported;	/* if variable is GUC_REPORT, value last sent
								 * to client (NULL if not yet sent) */
	char	   *sourcefile;		/* file current setting is from (NULL if not
								 * set in config file) */
	int			sourceline;		/* line in source file */
};
```

---

答： 可以，但是**几乎不会修改，也不应该修改**。

`config_generic` 中的 `context` 字段属于**常量字段（constant fields）**，从 PostgreSQL 的设计来看，它在初始化之后就是固定的。

你贴出来的注释已经说明了这一点：

```c
/* constant fields, must be set correctly in initial value: */

const char *name;
GucContext context;
enum config_group group;
const char *short_desc;
const char *long_desc;
int flags;
```

这里的 **must be set correctly in initial value** 就是在告诉开发者：

> 这些字段在静态初始化时就必须填写正确，之后整个运行过程中不会再改变。

------

## 为什么 context 要保持不变？

`context` 描述的是 **"这个 GUC 最低需要什么权限才能修改"**。

例如

```c
{"shared_buffers",
 PGC_POSTMASTER,
 ...
}
```

意味着

- 只能 postmaster 启动时设置
- `SET shared_buffers=...` 永远不允许

而

```c
{"work_mem",
 PGC_USERSET,
 ...
}
```

表示

任何用户都可以

```sql
SET work_mem='64MB';
```

这些权限是数据库设计的一部分。

如果运行过程中允许修改

```c
record->context = PGC_USERSET;
```

那么

原来只能 postmaster 设置的参数突然任何用户都能改，

整个权限系统都会失效。

因此 PostgreSQL 根本没有提供修改它的接口。

------

## PostgreSQL 在哪里使用它？

几乎所有设置 GUC 的地方都会检查

```c
record->context
```

例如

```
set_config_option()
        ↓
set_config_with_handle()
        ↓
check_GUC_context()
```

里面有类似逻辑（简化）：

```c
switch (record->context)
{
    case PGC_POSTMASTER:
        ...
    case PGC_SIGHUP:
        ...
    case PGC_SUSET:
        ...
    case PGC_USERSET:
        ...
}
```

也就是说

**context 是每次 SET 时都会读取的元数据。**

------

## 与 scontext 的区别

很多人第一次看 GUC 时都会混淆这两个字段。

### context（固定）

```c
record->context
```

表示

> 这个 GUC **允许在什么环境修改**。

例如

```
PGC_POSTMASTER
```

永远不会变。

------

### scontext（可变）

```c
record->scontext
```

表示

> **当前值** 是在哪种 Context 下设置进去的。

例如

启动时：

```conf
shared_buffers = 2GB
```

那么

```
context      = PGC_POSTMASTER
scontext     = PGC_POSTMASTER
source       = PGC_S_FILE
```

如果

```
SET work_mem='32MB';
```

那么

```
context      = PGC_USERSET   // 没变
scontext     = PGC_USERSET   // 当前值来源
source       = PGC_S_SESSION
```

事务结束以后

```
RESET work_mem;
```

又可能变成

```
scontext = PGC_S_FILE
source   = PGC_S_FILE
```

所以：

- `context` 不变；
- `scontext` 会随着当前生效值的来源变化。

------

## 能不能强行改？

技术上当然可以。

例如

```c
ConfigureNamesBool[i].gen.context = PGC_USERSET;
```

因为它不是 `const` 成员，编译器不会阻止。

但是 PostgreSQL **没有任何代码会这样做**。

原因包括：

1. GUC 的权限模型会被破坏。
2. 很多检查逻辑假定 `context` 永远不变。
3. `context` 属于参数定义（metadata），而不是参数状态（runtime state）。

因此，这属于**违反 PostgreSQL 的设计约定**，不应在扩展或核心代码中修改。

------

### 总结

`config_generic` 的字段可以分为两类：

| 字段                       | 是否运行时变化   | 作用                                       |
| -------------------------- | ---------------- | ------------------------------------------ |
| `name`                     | ❌ 固定           | 参数名                                     |
| `context`                  | ❌ 固定           | 参数允许修改的上下文（权限级别）           |
| `group`                    | ❌ 固定           | 参数分类                                   |
| `flags`                    | ❌ 固定           | 参数特性标志                               |
| `vartype`                  | 初始化时设置一次 | 参数类型，启动阶段确定后保持不变           |
| `source`                   | ✅ 可变           | 当前值来自哪里（配置文件、会话、命令行等） |
| `scontext`                 | ✅ 可变           | 当前值是在什么上下文下设置的               |
| `reset_source`             | ✅ 可变           | `RESET` 后恢复值的来源                     |
| `stack`                    | ✅ 可变           | 事务级/会话级设置栈                        |
| `extra`                    | ✅ 可变           | 当前值对应的附加数据                       |
| `sourcefile`、`sourceline` | ✅ 可变           | 当前值来自哪个配置文件及其行号             |

因此，**`context` 在设计上属于 GUC 的静态元数据，初始化后应视为不可修改；真正随着运行时变化的是 `source`、`scontext`、`stack` 等状态字段。**