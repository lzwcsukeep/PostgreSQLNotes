# 第一章

## 1.1 PostgreSQL 简介及发展历程

PostgreSQL 是一种先进的对象-关系数据库管理系统(ORDBMS),它不仅支持关系数据库的各种功能,而且还具备类、继承等对象数据库的特征。它是目前功能最强大、特性最丰富和结构最复杂的开源数据库管理系统,其中有些特性甚至连商业数据库都不具备。

这个起源于加州大学伯克 利分校(UCB)的数据库研究计划,现在已经衍生成一项国际开发项目,并且拥有广泛的用户群。

PostgreSQL 主要运行在Unix 和 Linux 操作系统上(从8.0版本开始推出了 Windows 平台上的版本),并且免费开放源代码,用户可以在其官方网站`www. postgresql. org`上下载各种安装程序和文档。 PostgreSQL的发展历程见证了数据库理论和技术的发展历程,PostgreSQL 由 UCB 计算机科学教 授 Michael Stonebraker 于1986年创建。在此之前, Stonebraker 教授领导了关系数据库 Ingres 研究项 目(Ingres 项目的源代码可以从 www. ingres.com 免费获取),1982年他离开 UCB 并将 Ingres 商业化,使之成为Relational Technologies (RT)公司的一个产品。而后该公司被 Computer Associations (CA) 公司收购。2004年,CA 在开源许可下发布了 Ingres release 3,并继续开发销售 Ingres。而Stonebraker 教授在返回 UCB 后开始了一项post-Ingres 计划,该计划致力于解决基于关系模型的数据库管理系 统产品的局限性,这即是 Postgres (但还不是现在的 PostgreSQL)的开端。 从1986年开始,Stonebraker 教授发表了一系列论文,引入对象关系理念,探讨了新的数据库 的结构设计和扩展设计。1988年,他提出了Postgres 的第一个原型设计,1989年6月发布了版本1, 1990年6月发布了带有重写后的规则系统的版本2。1991年发布了版本3,在版本3里改进了规则 系统,增加了对多种存储系统支持的能力,并且改进了查询引擎。1993年,Postgres 用户开始剧增, 并且特性需求急剧增加。在做了一些代码清理后发布了版本4,之后 Postgres 项目正式终止。随后, Stonebraker 再次创业,成立 Illustra 公司提供对Postgres 的商业支持,llustra 在1997年被 Informix 收 购,而Stonebraker 成为 Informix 的CTO, Informix 由于财务问题在2001年被IBM 收购。

尽管 Postgres 计划终止了,但BSD 许可证(UCB 在其下发行的 Postgres)却使开放源代码开发者获得副本并进一步开发系统。1994年,两个UCB 的研究生, Andrew Yu 和 Jolly Chen,增加了一个SQL 语言解释器来替代早先的基于Ingres 的QUEL 系统,建立了 Postgres95。代码随后被发行到互联网上。1996年,该计划被重新命名为 PostgreSQL,以反映数据库的新查询语言 SQL,来自世界各地的数据库开发者和志愿者通过互联网协作起来,发行了 PostgreSQL 的第一个版本6.0,并且一直维护着这套软件。自此以后,PostgreSQL 开始持续稳定地发布新版本,在新版本中有很多改进。 2005年1月19日,版本8.0发行,从这个版本开始,PostgreSQL 以原生的方式(即不需要模拟中间层的支持)开始支持 Windows 操作系统。从 Michael Stonebraker 教授开始,全世界无数著名的数据库专家和优秀的黑客为 PostgreSQL 的 发展做出了杰出的贡献,使 PostgreSQL 项目充满活力,不断向前发展,并使得 PostgreSQL 成为目前 最好的开源数据库管理系统之一。

## 1.2 PostgreSQL 的特性

PostgreSQL 是一种几乎可以运行在各种平台上的免费的开放源码的对象关系数据库管理系统, 拥有与企业级数据库相媲美的特性,如**完善的SQL 标准支持**、**多版本并发控制**、**时间点恢复**、**表空间机制**、**异步复制**、**嵌套事务**、**在线/热备份**、**一个复杂的查询优化器**、**预写日志容错技术**。它支持**国际字符集**、**多字节字符编码**、**Unicode**,并且对格式化、排序、大小写敏感提供**本地化支持**。 PostgreSQL 在管理大数据量方面有良好的可扩展性,对并发用户管理具有自适应性。现在已经出现 具有管理超过4万亿字节数据能力的实用版本产品。 

- **开放特性:** PostgreSQL 内置了丰富的数据类型,如任意精度的数值、无限制长度的文本、 几何图元、IP地址、数组等;同时还允许用户定义基于正规 SQL 类型的新类型,让数据库 自身理解复杂数据,自定义类型中还可以包含继承关系。用户可以为数据库内几乎所有的对象定义新的类型,如索引、操作符(可重载现有操作符)、聚集函数、数据域、数据类型转换、会话(编码转换)等。
- **可编程性:** PostgreSQL 同样拥有大量的编程接口供用户开发使用,如 ODBC、JDBC (Java)、 Libpq (C/C++)
- **可定制性:** PostgreSQL 拥有广泛的编程语言支持来实现函数功能,包括内置的PL/pgSQL 过程语言,PL/Perl、plPHP、PL/Python、PL/Ruby、PL/Tel 等脚本语言,以及 Java、C/C++ 等高级编程语言。函数的输出是一系列行类型的集合,可以在查询中当做表来使用,函数也可以被定义成以创建者或者调用者的身份运行。在其他的数据库产品中,函数也会被称为“存储过程”。
- **索引手段:** 用户可以自定义索引方法或者使用内置 B-Tree 索引、Hash 表索引、GiST 索引、 GIN 索引。GiST 索引不是某种特定的索引类型,而是一种通用索引基础结构,可以在这种结构上实现很多不同的索引策略。PostgreSQL 同时还支持如下功能:反向索引检索、表达式索引、部分索引、位图索引扫描。
- **多种身份认证方式:** PostgreSQL 中可以使用数据库用户/角色、操作系统、PAM、Kerberos 等方式,根据主机配置文件(pg_hba. conf) 中的设置执行对应的身份认证。

## 1.3 PostgreSQL 代码结构

PostgreSQL 是遵照一个和BSD开源协议类似的协议 PostgreSQL License 发布。该许可证可以在 PostgreSQL 源代码的COPYRIGHT 文件中找到。从该协议可以看到,开发人员或者商业组织只要遵循该协议,便可以自由地使用 PostgreSQL,可以完全控制这些第三方代码,在必要的时候可以加以修改或者二次开发。 PostgreSQL 源代码包含3400多个文件(截至8.4.1 版本),主要程序由 C语言编写,包括十几个大型模块,定义了几百个主要的数据结构和上万个函数。PostgreSQL 源代码结构清晰,每一个子目录都对应一个模块,其中主要目录(模块)及用途如下。

- Bootstrap:用于支持 Bootstrap 运行模式,该模式主要用来创建初始的模板数据库。
- Main:主程序模块,它负责将控制权转到 Postmaster 进程或 Postgres 进程。
- Postmaster: 监听用户请求的守护进程,并控制 Postgres 进程的启动和终止。
- Libpq: C/C++的库函数,处理与客户端间的通信,几乎所有的模块都依赖它。 
- Tcop: Postgres 服务进程的主要处理部分,它调用 Parser、Optimizer、Executor、和 Commands 中的函数来执行客户端提交的查询。
- Parser: 编译器,将 SQL 查询转化为内部查询树。
- Optimizer: 优化器,根据查询树创建最优的查询路径和查询计划。
- Executor: 执行器,执行来自 Optimizer 的查询计划。
- Commands: 执行非计划查询的 SQL 命令,如创建表命令等。
- Catalog: 系统表操作,包含用于操作系统表的专用函数。
- Storage: 管理各种类型的存储系统(如磁盘、闪存等)。
- Access: 提供各种存取方法,支持堆、索引等对数据的存取。
- Nodes: 定义系统内部所用到的节点、链表等结构,以及处理这些结构的函数。
- Utils: 各种支持函数,如错误报告、 各种初始化操作等。
- Rewrite: 查询重写,根据规则系统对查询进行重写。
- Initdb: 初始化数据库集簇。
- TSearch: 全文检索。
- Psql: 数据库交互工具。
- Port: 平台兼容性处理相关的函数。

## 1.4 Linux 下源码安装PostgreSQL

1. 从 [ PostgreSQL 官方网站 ](https://www.postgresql.org/ftp/source/)下载最新源代码,通常是.tar. gz 的压缩格式。
2. 通过 tar 或者图形化软件解压源码包,假设解压到 `/usr/local/src/pgsrc`目录下。
3. 从源代码编译安装 PostgreSQL 需要用到C语言开发库(包括gcc、ld 等),默认配置的情况下还需要 libreadline、zlib 和xml2 这几个开发库的支持,在 Ubuntu 等 Linux 发行版下可以通过 apt-get 安装上述开发库。
4. 在源代码目录下运行 `configure` 脚本,默认情况下 PostgreSQL 将被安装在`/usr/local/pgsql`目录下。
5. 在源代码目录下分别执行 `make` 和 `make install` 命令即可从源代码编译 PostgreSQL 并将相关文件拷贝到目标目录下。
6. 初始化数据库集簇: 
   - cd /usr/local/pgsql 
   - mkdir data (创建数据库文件的存放目录) 
   - chown postgres:postgres data 
   - su postgres 
   - cd bin
   - ./initdb --no-locale -D ../data 
7. 启动数据库服务器: 
   - ./pg_ctl start -D ./data 
8. 创建和访问数据库 test: 
   - ./createdb test
   - ./psql test

以上，就从源码安装完毕 Postgresql 数据库，并且使用自带的`bin`工具创建了名为 test 的数据库,并且使用交互式客户端`psql`连接上了数据库test，从这开始就可以执行SQL命令了。

## 1.5 PostgreSQL 二进制程序介绍

数据库安装完成后,在安装目录的 bin 目录下包含了 PostgreSQL 的工具命令。这些命令包括数据库初始化, 创建、删除数据库,启动、停止数据库,连接数据库,备份、恢复数据库等丰富的工具。下面是源码中bin目录以及backend目录编译出来的二进制程序介绍。

|   命令名   | 常用参数                                                     | 功能介绍                                                     | 示例                                |
| :--------: | ------------------------------------------------------------ | ------------------------------------------------------------ | ----------------------------------- |
|   initdb   | -D 数据目录<br>-E 编码 <br>-X WAL 日志目录                   | 初始化数据库集簇，创建模板数据库（template0 / template1）及系统 catalog 表 | initdb -D ../data                   |
| createuser | -d 新用户可以创建数据库<br>\-c 指定用户的最大连接数<br>-U 连接的用户名(不是要创建的用户) | 创建 PostgreSQL 用户（等价 `CREATE USER`）                   | createuser -d -l -U postgres pguser |
|  dropuser  | -U 连接用户                                                  | 删除用户                                                     | dropuser -U postgres pguser         |
|  createdb  | -E 数据库的编码<br> -l 数据库 locale 设置 <br>-O 数据库所有者 | 创建一个新的 PostgreSQL 的数据库(和SQL语句 CREATE DATABASE 相同),如 createdb test | createdb test                       |
|   dropdb   | -i 删除之前提示确认操作<br>-U 连接数据库的用户名             | 删除数据库                                                   | dropdb test                         |
|  pg_dump   | -f 备份文件名称<br>-F 备份文件格式,custom、tar、plain text   | 将 PostgreSQL数据库导出到一个备份文件,如`pg_dump test > /home/pgsql/backup/1.bak` |                                     |
| pg_dumpall | -f 备份文件名称                                              | 将所有的PostgreSQL 数据库导出到一个备份文件                  |                                     |
| pg_restore | -d 连接数据库名称<br>-f 备份文件名称                         | 从一个由pg_dump 或pg_dumpall 程序导出的备份文件 中恢复 PostgreSQL 数据库 |                                     |
|  vacuumdb  |                                                              | 清理和分析一个 PostgreSQL 数据库,它是客户端程序 psql 环境下 SQL语句 VACUUM 的 shell 脚本封装,二者功能完全相同 |                                     |
|   pg_ctl   |                                                              | 启动、停止、重启 PostgreSQL 服务                             |                                     |
|  postgres  |                                                              | PostgreSQL 单用户模式的数据库服务                            |                                     |
| postmaster |                                                              | PostgreSQL 多用户模式的数据库服务                            |                                     |
|    psql    |                                                              | 一个基于命令行的PostgreSQL 交互式客户端程序。                |                                     |

