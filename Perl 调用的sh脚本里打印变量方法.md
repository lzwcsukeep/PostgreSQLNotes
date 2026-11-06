背景：MySQL 兼容性测试框架分几层。Perl 驱动层，.sh 脚本层。 Perl 语言会调用 sh 脚本。且 sh 脚本的标准输出会被 perl 层捕获。因此当我需要

查看 sh 脚本里某个变量的实际值时，使用 echo 命令打印的变量值会被 Perl 层捕获，无法输出到终端直接显示。

> 具体为 `t/001_mysql_compat.pl` 使用 `IPC::Run` 捕获了 `run_mysql_compat.sh` 的标准输出和标准错误，而且只有测试失败时才通过 `diag()` 输出它们。

解决方法：在 sh 脚本里面将感兴趣的变量打印到文件，然后从文件中查看。

也就是下面的方法二。

---

如果你只是想在运行 `make check` 时查看 `run_mysql_compat.sh` 中某个变量的值，最简单的办法是：在 `t/001_mysql_compat.pl` 中调用 `run` 后，无条件打印捕获的输出。

不过，如果你只想查看 `suite_dir`，我推荐下面两种方法。

## 方法一：直接打印变量（推荐）

在 `run_mysql_compat.sh` 中：

```
suite_dir=$(CDPATH= cd -- "$(dirname -- "$0")" && pwd)
echo "DEBUG: suite_dir=$suite_dir"
```

然后在 `t/001_mysql_compat.pl` 中，找到：

```
ok($result, 'MySQL SQL compatibility suite passes');
```

在它后面临时增加：

```
diag("DEBUG: $stdout") if $stdout ne '';
diag("DEBUG: $stderr") if $stderr ne '';
```

这样，无论测试成功还是失败，只要捕获到了输出，就会打印出来。

## 方法二：只输出变量，不改 Perl 代码

如果你不想修改 Perl 测试脚本，可以在 Shell 脚本中将变量输出到一个临时文件：

```
suite_dir=$(CDPATH= cd -- "$(dirname -- "$0")" && pwd)
echo "DEBUG: suite_dir=$suite_dir" >> /tmp/mysql_compat_debug.log
```

运行测试后查看：

```
cat /tmp/mysql_compat_debug.log
```

这种方法不受 `IPC::Run` 捕获标准输出和标准错误的影响。

建议： 如果只是临时调试一个变量，用方法二最快；如果希望在 `make check` 的终端输出中直接看到调试信息，用方法一更合适。调试完成后记得删除临时代码。



