背景： sh 脚本里面有些变量依赖一些其他变量而有不同的值。比如下面的 sh 脚本部分内容:

```bash
#!/usr/bin/env bash

set -uo pipefail

suite_dir=$(CDPATH= cd -- "$(dirname -- "$0")" && pwd)
mysql_bin=${MYSQL_BIN:-mysql}
mysql_host=${MYSQL_HOST:-127.0.0.1}
mysql_port=${MYSQL_PORT:-3306}
mysql_user=${MYSQL_USER:-test}
mysql_password=${MYSQL_PASSWORD:-}
mysql_database=${MYSQL_DATABASE:-unvdb_mysqldb}
test_timeout=${MYSQL_TEST_TIMEOUT:-30}
actual_dir=${MYSQL_COMPAT_ACTUAL_DIR:-"$suite_dir/actual"}
expected_dir="$suite_dir/expected"
update_expected=${MYSQL_COMPAT_UPDATE_EXPECTED:-0}

```

我运行的时候想传递 变量 `MYSQL_COMPAT_UPDATE_EXPECTED` 的值进去，那要怎么做呢？

答案： 这里 提到的 `MYSQL_COMPAT_UPDATE_EXPECTED` 是环境变量，就设置环境变量就行。

----

# 方法

这些变量分两类，传法不同：

- **前 7 个带默认值的**（`${VAR:-default}`）：通过**环境变量**传，即 `VAR=值 bash script.sh` 或 `export VAR=值`。
- **后 3 个不带默认值的**（`suite_dir` / `expected_dir`）：是脚本**内部计算**出来的，正常不用传，硬要传也只能靠改脚本或用环境变量间接影响。

## 一、带默认值的变量 —— 用环境变量传

`${MYSQL_BIN:-mysql}` 的意思是：**如果环境里已有 `MYSQL_BIN` 就用它，否则用 `mysql`**。所以你在外面设同名环境变量即可覆盖。

### 方式 1：命令前置（最常用，只对这条命令生效）

```bash
MYSQL_HOST=192.168.1.10 MYSQL_PORT=3307 MYSQL_USER=root \
MYSQL_PASSWORD=secret MYSQL_DATABASE=mydb \
bash script.sh
```

### 方式 2：先 export，再运行（对当前 shell 后续命令都生效）

```bash
export MYSQL_HOST=192.168.1.10
export MYSQL_PORT=3307
export MYSQL_USER=root
export MYSQL_PASSWORD=secret
bash script.sh
```

### 方式 3：用 env 命令

```bash
env MYSQL_HOST=192.168.1.10 MYSQL_PORT=3307 bash script.sh
```

### 各变量对应关系

| 脚本变量          | 环境变量名                     | 默认值              |
| ----------------- | ------------------------------ | ------------------- |
| `mysql_bin`       | `MYSQL_BIN`                    | `mysql`             |
| `mysql_host`      | `MYSQL_HOST`                   | `127.0.0.1`         |
| `mysql_port`      | `MYSQL_PORT`                   | `3306`              |
| `mysql_user`      | `MYSQL_USER`                   | `test`              |
| `mysql_password`  | `MYSQL_PASSWORD`               | 空                  |
| `mysql_database`  | `MYSQL_DATABASE`               | `unvdb_mysqldb`     |
| `test_timeout`    | `MYSQL_TEST_TIMEOUT`           | `30`                |
| `actual_dir`      | `MYSQL_COMPAT_ACTUAL_DIR`      | `$suite_dir/actual` |
| `update_expected` | `MYSQL_COMPAT_UPDATE_EXPECTED` | `0`                 |

> 注意：`mysql_password=${MYSQL_PASSWORD:-}` 这里默认是**空字符串**，所以你传空也等于没传。`update_expected` 传 `1` 才会开启更新。

## 二、不带默认值的变量 —— 内部计算，通常不用传

```bash
suite_dir=$(CDPATH= cd -- "$(dirname -- "$0")" && pwd)
expected_dir="$suite_dir/expected"
```

- `suite_dir`：脚本所在目录（由 `$0` 推导），**写死在逻辑里**，没有对应的环境变量入口。
- `expected_dir`：固定是 `$suite_dir/expected`。

想改这两个，只有两条路：

1. **改脚本**，比如给它们也加默认值语法：
   
   ```bash
   suite_dir=${SUITE_DIR:-$(CDPATH= cd -- "$(dirname -- "$0")" && pwd)}
   expected_dir=${EXPECTED_DIR:-"$suite_dir/expected"}
   ```
然后就能 `SUITE_DIR=/tmp/foo bash script.sh` 了。
   
2. **不改脚本**：`expected_dir` 可以通过把脚本放到别的目录来间接改变（因为它是相对 `suite_dir` 的）。

## 三、几种传参方式的区别（重要）

| 写法                           | 子进程能否看到 | 说明                                              |
| ------------------------------ | -------------- | ------------------------------------------------- |
| `VAR=1 bash script.sh`         | ✅ 能           | 只对这条命令生效，最干净                          |
| `export VAR=1; bash script.sh` | ✅ 能           | 对当前终端后续命令都生效                          |
| `bash script.sh VAR=1`         | ❌ 不能         | 这是**位置参数** `$1`，不是环境变量               |
| `VAR=1`（单独一行）            | ❌ 不能         | 只是当前 shell 的普通变量，没 export 子进程看不到 |

**最常见的坑**：写成 `bash script.sh MYSQL_HOST=1.2.3.4`。这不是设环境变量，而是把 `MYSQL_HOST=1.2.3.4` 当成 `$1` 传给脚本，脚本里的 `${MYSQL_HOST:-...}` 依然用默认值。

## 四、验证变量是否传进去了

配合你上一个问题里的 `-x` 一起用最直观：

```bash
MYSQL_HOST=192.168.1.10 MYSQL_USER=root bash -x script.sh 2>&1 | head -20
```

你会看到类似：

```
+ mysql_host=192.168.1.10
+ mysql_user=root
```

说明覆盖成功；如果显示的是默认值，就是没传对。

## 小结

- 要覆盖的变量，**名字必须和 `${...}` 里的一致**，用 `VAR=值 bash script.sh` 或 `export`。
- 没写 `:-默认值` 的（`suite_dir` / `expected_dir`）是脚本内部算的，外部传不进去，除非改脚本。
- 别用 `bash script.sh VAR=值`，那是位置参数，不是环境变量。