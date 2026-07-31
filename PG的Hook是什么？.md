问： 解释一下 postgresql 的 Hook

---

答：PostgreSQL 的 **Hook（钩子）** 是 PostgreSQL 提供给扩展（Extension）或者二次开发者的一种**插入点（extension point）**。

简单来说：

> **Hook = PostgreSQL 在某个关键流程执行时，主动调用你注册的一段函数。**

这样，你就可以**在不修改（或者少修改）PostgreSQL 原有代码**的情况下，插入自己的逻辑。

------

# 为什么需要 Hook？

假设 PostgreSQL 原来的流程是：

```text
客户端SQL
      │
      ▼
Parser
      │
      ▼
Analyzer
      │
      ▼
Planner
      │
      ▼
Executor
```

如果你想：

- 在 SQL 执行前打印日志
- 修改 SQL
- 修改查询计划
- 拦截某些语句
- 增加权限检查
- 实现 MySQL 兼容

如果没有 Hook，就只能：

> 修改 PostgreSQL 源码。

这样以后升级 PostgreSQL 会非常痛苦。

于是 PostgreSQL 在很多地方留了：

> "这里如果有人注册了 Hook，我先让 Hook 执行。"

例如：

```c
if (ProcessUtility_hook)
    ProcessUtility_hook(...);
else
    standard_ProcessUtility(...);
```

这就是 Hook。

------

# Hook 的本质

本质上就是一个 **全局函数指针（Function Pointer）**。

例如：

```c
ProcessUtility_hook_type ProcessUtility_hook = NULL;
```

实际上就是

```c
typedef void (*ProcessUtility_hook_type)(
    PlannedStmt *pstmt,
    const char *queryString,
    ...
);
```

所以

```c
ProcessUtility_hook
```

其实就是：

```c
void (*ProcessUtility_hook)(...)
```

一个函数指针。

默认：

```c
ProcessUtility_hook == NULL
```

没人注册。

扩展加载以后：

```c
ProcessUtility_hook = my_ProcessUtility;
```

以后 PostgreSQL 就会调用：

```c
my_ProcessUtility(...)
```

------

# 一个真实例子

例如：

```c
void
ProcessUtility(...)
{
    if (ProcessUtility_hook)
        (*ProcessUtility_hook)(...);
    else
        standard_ProcessUtility(...);
}
```

假设你写了一个插件：

```c
static ProcessUtility_hook_type prev_hook = NULL;

static void
my_ProcessUtility(...)
{
    printf("SQL is coming!\n");

    if (prev_hook)
        prev_hook(...);
    else
        standard_ProcessUtility(...);
}
```

初始化：

```c
void
_PG_init(void)
{
    prev_hook = ProcessUtility_hook;
    ProcessUtility_hook = my_ProcessUtility;
}
```

于是执行流程变成：

```text
ProcessUtility()
        │
        ▼
my_ProcessUtility()
        │
        ▼
printf()
        │
        ▼
standard_ProcessUtility()
```

------

# 为什么要保存 prev_hook？

你可能会问：

为什么这样写：

```c
prev_hook = ProcessUtility_hook;
ProcessUtility_hook = my_hook;
```

而不是：

```c
ProcessUtility_hook = my_hook;
```

因为：

可能别人已经注册了 Hook。

例如：

插件A：

```text
ProcessUtility_hook
        │
        ▼
pluginA
```

后来插件B：

```text
ProcessUtility_hook
        │
        ▼
pluginB
```

如果 B 不保存：

```text
pluginA
```

那 pluginA 永远不会执行。

正确应该是：

```text
pluginB
      │
      ▼
pluginA
      │
      ▼
standard_ProcessUtility()
```

所以都会写：

```c
prev_hook = ProcessUtility_hook;
ProcessUtility_hook = my_hook;
```

然后：

```c
if (prev_hook)
    prev_hook(...);
else
    standard_ProcessUtility(...);
```

形成 Hook 链。

------

# Hook 的执行流程

假设：

有三个插件。

```text
pluginA
pluginB
pluginC
```

加载顺序：

```text
pluginA
pluginB
pluginC
```

最终：

```text
ProcessUtility_hook
        │
        ▼
pluginC
        │
        ▼
pluginB
        │
        ▼
pluginA
        │
        ▼
standard_ProcessUtility
```

所以：

> **最后加载的插件最先执行。**

------

# PostgreSQL 常见 Hook

PostgreSQL 有几十种 Hook，覆盖了解析、规划、执行、认证等多个阶段。

| Hook                        | 作用                                             | 调用时机        |
| --------------------------- | ------------------------------------------------ | --------------- |
| `ProcessUtility_hook`       | 拦截 DDL、事务控制、COPY、VACUUM 等 Utility 语句 | Utility 执行前  |
| `ExecutorStart_hook`        | Executor 初始化                                  | 执行开始        |
| `ExecutorRun_hook`          | Executor 真正运行                                | 查询执行        |
| `ExecutorFinish_hook`       | Executor 收尾                                    | 查询结束前      |
| `ExecutorEnd_hook`          | Executor 释放资源                                | 查询结束        |
| `planner_hook`              | 修改 Planner                                     | 生成执行计划前  |
| `post_parse_analyze_hook`   | Parser + Analyzer 完成后                         | Query 树生成后  |
| `object_access_hook`        | 对象访问控制                                     | 创建/删除对象时 |
| `emit_log_hook`             | 日志输出                                         | 写日志前        |
| `ExecutorCheckPerms_hook`   | 权限检查                                         | Executor 前     |
| `ClientAuthentication_hook` | 用户认证                                         | 登录认证时      |

------

# 以 planner_hook 为例

Planner 原来：

```text
Query
    │
    ▼
standard_planner()
    │
    ▼
Plan
```

有 Hook：

```text
Query
    │
    ▼
planner_hook()
    │
    ▼
standard_planner()
    │
    ▼
Plan
```

你的 Hook 可以：

```c
PlannedStmt *
my_planner(Query *query, ...)
{
    printf("planning...\n");

    PlannedStmt *plan;

    if (prev_hook)
        plan = prev_hook(...);
    else
        plan = standard_planner(...);

    return plan;
}
```

甚至可以：

```c
plan = standard_planner(...);

modify_plan(plan);

return plan;
```

直接修改执行计划。

------

# openHalo 为什么大量使用 Hook？

你最近一直在研究 **openHalo（基于 PostgreSQL 14.18 实现 MySQL 兼容）**。

对于这种项目，Hook 非常有价值，因为它可以在 PostgreSQL 的关键阶段插入 MySQL 兼容逻辑，而尽量减少对原始代码的侵入。例如：

- 在 `post_parse_analyze_hook` 中补充或调整语义分析结果。
- 在 `ProcessUtility_hook` 中改写或兼容 MySQL 特有的 DDL 行为（如某些 `ALTER TABLE` 语法）。
- 在 `planner_hook` 中针对 MySQL 特性调整查询计划。
- 在执行阶段的 Hook 中实现兼容行为或统计信息。

当然，并不是所有兼容功能都能靠 Hook 完成。像你之前研究过的 `guc-file.l`（Flex 词法分析器）或 MySQL 专用语法解析器，属于 **Parser/Lexer** 层面的修改，这些通常仍需要直接修改 PostgreSQL 源码，因为标准 Hook 无法改变词法和语法规则。

------

# Hook 的设计思想

可以把 PostgreSQL 的 Hook 理解成面向对象语言里的“事件回调”或“拦截器”：

```text
PostgreSQL
     │
     ▼
这里要执行 Planner
     │
     ├── 没有 Hook
     │       │
     │       ▼
     │ standard_planner()
     │
     └── 有 Hook
             │
             ▼
      my_planner()
             │
             ▼
      standard_planner()
```

因此，**Hook 的本质就是一个全局函数指针，PostgreSQL 在关键执行节点主动调用它，让扩展能够在不大量修改核心代码的前提下扩展或改变数据库行为。**

------

结合你目前在阅读 PostgreSQL 源码的进度，如果继续深入，我建议下一步研究 **Hook 的完整生命周期**：从扩展的 `_PG_init()` 如何注册 Hook，到一次 SQL 执行过程中各个 Hook（`post_parse_analyze_hook`、`planner_hook`、`ExecutorStart_hook`、`ExecutorRun_hook` 等）按时间顺序依次被触发，这样你会对 PostgreSQL 的整体执行流程建立非常清晰的认识。

# 自己观察到的一个Hook应用位置

```c
// src/backend/utils/misc/guc.c : InitializeOneGUCOption
switch (gconf->vartype)
	{
		case PGC_BOOL:
			{
				struct config_bool *conf = (struct config_bool *) gconf;
				bool		newval = conf->boot_val;
				void	   *extra = NULL;

				if (!call_bool_check_hook(conf, &newval, &extra,
										  PGC_S_DEFAULT, LOG))
					elog(FATAL, "failed to initialize %s to %d",
						 conf->gen.name, (int) newval);
				if (conf->assign_hook)
					conf->assign_hook(newval, extra);
				*conf->variable = conf->reset_val = newval;
				conf->gen.extra = conf->reset_extra = extra;
				break;
			}
        ...
	 }

// 定义
struct config_bool
{
	struct config_generic gen;
	/* constant fields, must be set correctly in initial value: */
	bool	   *variable;
	bool		boot_val;
	GucBoolCheckHook check_hook;
	GucBoolAssignHook assign_hook;
	GucShowHook show_hook;
	/* variable fields, initialized at runtime: */
	bool		reset_val;
	void	   *reset_extra;
};


// 
typedef void (*GucBoolAssignHook) (bool newval, void *extra);

```

上述GUC 相关代码，给 GUC 变量赋值的时候：

```c
if (conf->assign_hook)
	conf->assign_hook(newval, extra);
```

这部分代码就是一个 Hook, 判断这个函数指针是否有值，在原有的流程上插入自己的一段逻辑。

这就是一个最简单的Hook。

