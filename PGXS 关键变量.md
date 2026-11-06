PGXS 的全称是 **PostgreSQL Extension Building Infrastructure**，你可以把它理解为 PostgreSQL 官方提供的一套“**Makefile 模板库**”。

它的核心作用是**自动化**编译和安装 PostgreSQL 扩展模块的繁琐过程。你只需要写一个极其简单的 `Makefile`，剩下的事情（比如去哪找头文件、用什么编译参数、装到哪个目录）都交给 PGXS 处理。

### 🛠️ 怎么用 PGXS？

你只需要在扩展的源码目录下，创建一个名为 `Makefile` 的文件，内容通常像这样：

```makefile
# 1. 定义扩展的基本信息
MODULES = your_extension_name      # 编译出的共享库名（不带 .so 后缀）
EXTENSION = your_extension_name    # 扩展名，需对应 your_extension_name.control 文件
DATA = your_extension_name--1.0.sql # 安装的 SQL 脚本

# 2. 引入 PGXS 基础设施（这三行通常是固定的）
PG_CONFIG = pg_config
PGXS := $(shell $(PG_CONFIG) --pgxs)
include $(PGXS)
```

### 📦 关键变量解释

在 `include $(PGXS)` 之前，你可以设置一些变量来告诉 PGXS 你要做什么：

*   **指定构建内容（三选一）**：
    *   `MODULES`：从**单个**源文件（如 `foo.c`）编译一个共享库。最常用。
    *   `MODULE_big`：从**多个**源文件编译一个共享库（需配合 `OBJS` 变量列出所有 `.o` 文件）。
    *   `PROGRAM`：编译一个独立的可执行程序（较少用于普通扩展）。
*   **指定安装内容**：
    *   `EXTENSION`：扩展名，PGXS 会自动安装同名的 `.control` 文件到 `share/extension` 目录。
    *   `DATA`：要安装的 SQL 脚本文件。
    *   `HEADERS_$MODULE`：要安装的头文件。

### 🚀 编译与安装

写完 `Makefile` 后，只需两条命令：

1.  **编译**：`make`
2.  **安装**：`make install`

> **注意**：PGXS 会默认使用你 `PATH` 环境变量里找到的第一个 `pg_config`。如果你想针对另一个 PostgreSQL 版本编译，可以设置 `PG_CONFIG` 变量指向对应的 `pg_config` 路径。

### 💡 总结

PGXS 就是 PostgreSQL 官方给的“**开箱即用**”编译方案，让你不用去研究复杂的编译链接细节，专注于写扩展代码本身。



# PGXS 体系里 在插件目录里执行 make 不指定目标 ，实际会编译什么？

在 PGXS 体系里，插件目录下直接执行 `make`（不指定目标），实际执行的**默认目标就是 `all`**，也就是**编译你的插件，但不会安装**。

### 🎯 默认目标是什么？

PGXS 遵循 Make 的标准约定：**Makefile 中第一个出现的规则，就是默认目标**。PGXS 的主文件（`pgxs.mk`）里，第一个定义的目标正是 `all`，它的作用就是构建你指定的模块（`MODULES` 或 `MODULE_big`）或程序（`PROGRAM`）。



链接：

[PostgreSQL: Documentation: 15: 38.18. Extension Building Infrastructure](https://www.postgresql.org/docs/15/extend-pgxs.html)