# Bison 介绍

`Bison` 是 GNU 提供的一个 **语法分析器生成器（parser generator）**，它可以根据你编写的文法规则（Grammar），自动生成一个 LR 系列语法分析器。

PostgreSQL 的 `gram.y` 文件就是语法规则描述文件。实际上 PostgreSQL 的 SQL 解析器就是基于 Bison 生成的。

------

# 1. Bison 是干什么的

编译器或解释器通常分为：

```text
源代码
   │
   ▼
词法分析(Lex)
   │
   ▼
Token流
   │
   ▼
语法分析(Bison)
   │
   ▼
语法树(AST)
   │
   ▼
后续处理
```

例如：

```sql
SELECT id FROM user;
```

经过 Flex（scan.l）后：

```text
SELECT
IDENT(id)
FROM
IDENT(user)
';'
```

然后 Bison 根据 SQL 文法规则进行归约：

```text
SelectStmt
 ├── targetList
 │     └── id
 └── fromClause
       └── user
```

最终生成 AST。

------

# 2. 学习 Bison 时最重要的几个概念

按重要程度排序：

1. Token（终结符）

```yacc
%token SELECT
```

1. 非终结符

```yacc
stmt
expr
```

1. 产生式

```yacc
expr:
    expr '+' expr
```

1. 语义动作

```yacc
{
    $$ = $1 + $3;
}
```

1. `%union`

定义语义值类型。

1. Shift / Reduce

理解 LR 解析器核心机制。

1. 优先级

```yacc
%left
%right
```

1. AST 构造

PostgreSQL 的 `gram.y` 基本上就是：

```yacc
SELECT ...
{
    $$ = makeNode(SelectStmt);
}
```

不断构造语法树节点。

------

如果你正在研究 PostgreSQL 的 `gram.y`，下一步最值得深入的是 **Bison 底层的 LR 状态机原理（Shift、Reduce、State、Item、Goto 表）**。理解了这些，再看 PostgreSQL 中几十万行 SQL 文法规则，会清楚为什么一个 `SELECT` 语句最终能被归约成一个 `SelectStmt` 节点。

# 3. 一个最简单的例子

假设要解析：

```text
1+2
```

创建：

```c
/* calc.y */
%{
#include <stdio.h>
%}

%%

expr:
      expr '+' expr
    | NUMBER
    ;

%%

int main()
{
    yyparse();
}

int yyerror(char *s)
{
    printf("error\n");
}
```

生成代码：

```bash
bison -d calc.y
```

产生：

```text
calc.tab.c
calc.tab.h
```

------

# 4. Bison 文件结构

通常：

```yacc
%{
C代码
%}

声明区

%%

语法规则区

%%

用户代码区
```

例如：

```yacc
%{
#include <stdio.h>
%}

%token SELECT FROM IDENT

%%

stmt:
      SELECT IDENT FROM IDENT
    ;

%%

int main()
{
    yyparse();
}
```

------

# 5. 语法规则

形式：

```yacc
非终结符:
      产生式1
    | 产生式2
    | 产生式3
    ;
```

例如：

```yacc
expr:
      expr '+' expr
    | expr '-' expr
    | NUMBER
    ;
```

表示：

```text
expr -> expr + expr
expr -> expr - expr
expr -> NUMBER
```

------

# 6. Token 声明

词法分析器返回 Token。

例如：

```yacc
%token SELECT
%token FROM
%token IDENT
```

等价于：

```c
#define SELECT 258
#define FROM   259
#define IDENT  260
```

Flex 中：

```c
"select" return SELECT;
"from"   return FROM;
[a-z]+   return IDENT;
```

------

# 7. 语义值 YYSTYPE

Bison 不仅关心结构，还关心值。

例如：

```text
1+2
```

需要得到：

```text
3
```

定义：

```yacc
%union
{
    int ival;
}
```

然后：

```yacc
%token <ival> NUMBER
%type  <ival> expr
```

------

## 语义动作

```yacc
expr:
      expr '+' expr
      {
          $$ = $1 + $3;
      }
    | NUMBER
      {
          $$ = $1;
      }
    ;
```

这里：

```text
$1  第一项
$2  第二项
$3  第三项

$$ 当前非终结符结果
```

例如：

```text
2 + 3

$1 = 2
$3 = 3

$$ = 5
```

------

# 8. 运算符优先级

下面的文法有歧义：

```yacc
expr:
      expr '+' expr
    | expr '*' expr
    ;
```

对于：

```text
1+2*3
```

可能：

```text
(1+2)*3
```

也可能：

```text
1+(2*3)
```

因此需要指定优先级：

```yacc
%left '+'
%left '*'
```

后声明的优先级更高：

```text
* > +
```

------

## PostgreSQL 中

经常看到：

```yacc
%left OR
%left AND
%right NOT
```

用于定义 SQL 运算优先级。

------

# 9. Shift/Reduce 冲突

Bison 最著名的问题之一。

例如：

```yacc
if_stmt:
      IF expr THEN stmt
    | IF expr THEN stmt ELSE stmt
    ;
```

输入：

```text
if a then
    if b then s1
else
    s2
```

问题：

```text
else 属于哪个 if ?
```

产生：

```text
shift/reduce conflict
```

Bison 输出：

```text
1 shift/reduce conflict
```

------

## 查看冲突

```bash
bison -v gram.y
```

生成：

```text
gram.output
```

里面详细记录：

```text
State 123

shift/reduce conflict
```

可以分析具体原因。

------

# 10. `%expect`

PostgreSQL 的 `gram.y` 开头有：

```yacc
%expect 0
```

意思：

```text
预计冲突数量为 0
```

如果产生冲突：

```text
error
```

确保 SQL 文法没有意外变化。

有些项目：

```yacc
%expect 1
```

表示：

```text
允许 1 个已知冲突
```

------

# 11. 纯解析器（Pure Parser）

PostgreSQL：

```yacc
%pure-parser
```

表示：

```text
解析器无全局状态
```

旧版：

```c
YYSTYPE yylval;
```

是全局变量。

纯解析器会把状态放到参数里：

```c
yyparse(...)
```

从而：

```text
线程安全
支持多个解析器实例
```

------

# 12. `%parse-param`

PostgreSQL：

```yacc
%parse-param {core_yyscan_t yyscanner}
```

生成：

```c
yyparse(core_yyscan_t yyscanner)
```

这样语法分析器就能访问词法分析器状态。

------

# 13. 与 Flex 配合

通常：

### Flex

```c
int yylex()
{
    return token;
}
```

### Bison

```c
int yyparse()
{
    token = yylex();
}
```

关系：

```text
Flex -----> Token
              │
              ▼
            Bison
              │
              ▼
             AST
```

PostgreSQL 中：

```text
scan.l
  ↓
core_yylex()

gram.y
  ↓
base_yyparse()
```

------

# 14. PostgreSQL 中的 Bison

PostgreSQL 源码：

```text
src/backend/parser/

gram.y
scan.l
```

构建时：

```bash
bison gram.y
```

生成：

```text
gram.c
gram.h
```

最终：

```text
raw_parser()
    ↓
base_yyparse()
    ↓
gram.y规则
    ↓
生成Node树
```

例如：

```sql
SELECT * FROM t;
```

最终生成：

```c
SelectStmt
```

节点对象。

------

# 15. 安装 Bison

Ubuntu：

```bash
sudo apt install bison
```

查看版本：

```bash
bison --version
```

例如：

```text
bison (GNU Bison) 3.8.2
```

------

# gram.y 语法规则描述文件

**`gram.y` 本身并不是语法分析器，它是语法规则描述文件，而 Bison 会根据 `gram.y` 自动生成语法分析器代码。**

关系可以理解为：

```text
gram.y
  │
  │ Bison
  ▼
gram.c (或 gram.tab.c)
  │
  │ gcc
  ▼
可执行程序中的语法分析器(// 这一步才是真正有语法解析功能的语法分析器， gram.y 文件只是描述语法)
```

------

## PostgreSQL 中的实际情况

PostgreSQL 的源码里有：

```text
src/backend/parser/

gram.y
scan.l
```

构建时：

```bash
bison gram.y
```

生成：

```text
gram.c
gram.h
```

（不同版本可能叫 `gram.c` 或 `gram.tab.c`）

然后再编译：

```bash
gcc gram.c scan.c ...
```

最终链接进 PostgreSQL 服务进程。

------

## gram.y 里面写的是什么？

例如：

```yacc
stmt:
      SELECT target_list FROM relation_expr
      {
          SelectStmt *n = makeNode(SelectStmt);

          n->targetList = $2;
          n->fromClause = $4;

          $$ = (Node *) n;
      }
;
```

这里描述了：

```sql
SELECT a FROM t
```

应该如何识别。

Bison 会把这条规则转换成状态机。

------

## Bison 实际生成了什么？

假设：

```yacc
expr:
      expr '+' expr
    | NUM
;
```

Bison 生成的大致代码类似：

```c
switch (state)
{
    case 10:
        if (token == '+')
            shift();
        break;

    case 20:
        reduce(expr -> NUM);
        break;
}
```

当然真实代码远比这复杂。

实际上生成的是：

```text
ACTION表
GOTO表
状态机
归约规则
```

例如：

```text
state 0
    NUM -> shift 1

state 1
    '+' -> shift 2
    EOF -> reduce expr
```

这就是 LR 语法分析器。

------

## 最终得到的函数

Bison 会生成：

```c
int yyparse(void);
```

或者 PostgreSQL：

```c
int base_yyparse(core_yyscan_t yyscanner);
```

调用：

```c
base_yyparse(scanner);
```

它会不断：

```text
调用词法分析器
      ↓
获得Token
      ↓
状态转移
      ↓
归约
      ↓
执行语义动作
      ↓
构造AST
```

------

## PostgreSQL 中的调用链

对于：

```sql
SELECT * FROM test;
```

执行流程：

```text
raw_parser()
    │
    ▼
base_yyparse()
    │
    ▼
core_yylex()
    │
    ▼
scan.l
```

获得：

```text
SELECT
'*'
FROM
IDENT(test)
';'
```

然后 `base_yyparse()` 根据 `gram.y` 的规则不断归约：

```text
IDENT
 ↓
RangeVar

FROM clause
 ↓
fromClause

SELECT clause
 ↓
SelectStmt
```

最后得到：

```c
SelectStmt *
```

语法树节点。

------

## 为什么不手写语法分析器？

理论上可以手写：

```c
if (token == SELECT)
{
    ...
}
```

但 SQL 文法非常复杂。

PostgreSQL 的 `gram.y`：

```text
约 1.8 万行以上
数百个 Token
上千条产生式
```

手写维护几乎不现实。

因此：

```text
gram.y
    ↓
Bison
    ↓
LR状态机
    ↓
自动生成语法分析器
```

这是编译器和数据库最常见的做法。

------

如果继续研究 PostgreSQL，我建议下一步看看：

```text
gram.y
   ↓
Bison
   ↓
生成的 gram.c
   ↓
yyparse()
```

然后结合一个简单 SQL：

```sql
SELECT 1;
```

跟踪：

```text
token -> state -> shift -> reduce -> SelectStmt
```

你就能真正理解 **Bison 如何把文法规则变成一个可执行的 LR 状态机解析器**。这也是 PostgreSQL 解析器工作的核心原理。