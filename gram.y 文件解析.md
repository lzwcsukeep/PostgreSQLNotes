## gram.y 文件结构

一个标准的 `.y` 文件结构是：

```bison
声明区
%%

语法规则区

%%

用户代码区
```

在 Bison/Yacc 文件中，`%%` 是**分隔符（delimiter）**，用于划分文件的不同区域。

举例：PostgreSQL gram.y 的结构

简化后类似：

```c
%{
    C代码
%}

%token ...
%type ...
%left ...
%right ...

%%

stmtblock:
      stmtmulti
;

stmtmulti:
      stmtmulti ';' toplevel_stmt
    | toplevel_stmt
;

SelectStmt:
      ...
;

%%

static Node *
makeSetOp(...)
{
    ...
}
```

总结： `.y` 文件顶层结构其实非常简单，就三层：%% 是分隔符

```c
┌─────────────────────┐
│ 声明区               │
│ %token              │
│ %type               │
│ %union              │
└─────────┬───────────┘
          │
         %%
          │
┌─────────▼───────────┐
│ 语法规则区            │
│ SelectStmt: ...     │
│ a_expr: ...         │
└─────────┬───────────┘
          │
         %%
          │
┌─────────▼───────────┐
│ C辅助函数区           │
│ makeSetOp()         │
│ check_func_name()   │
└─────────────────────┘
```

