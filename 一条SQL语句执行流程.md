# 一条 SQL 从输入到输出的完整执行流程

------

# 一、完整流程概览

一条 SQL 语句执行，一般经过四大阶段：

```
SQL 文本 → 解析器 → 优化器 → 执行器 → 返回结果
```

每个阶段内部还有 MVCC、WAL、事务管理等机制。

------

# 二、阶段一：SQL 解析（Parsing）

**目的**：把 SQL 文本转换成数据库内部可理解的结构。

SQL 示例：

```sql
SELECT name, salary FROM emp WHERE dept = 'IT';
```

## 步骤：

1. **词法分析**（Lexical Analysis）
   - 文本分割成 token：`[SELECT] [name] [,] [salary] [FROM] [emp] [WHERE] [dept] ['IT']`
2. **语法分析**（Syntax Analysis）
   - 生成语法树（Parse Tree / AST）
   - 检查 SQL 是否符合语法
3. **语义分析**（Semantic Analysis）
   - 表和列是否存在
   - 类型是否匹配
   - 权限是否允许访问

**输出**：Parse Tree / AST，内部结构化表示。

------

# 三、阶段二：查询优化器（Optimizer）

**目的**：根据 Parse Tree 生成最优执行计划。

1. **逻辑优化**
   - 消除冗余操作（如多余的 join）
   - 下推过滤条件（Predicate Pushdown）
2. **物理优化**
   - 选择访问表的方法：顺序扫描 / 索引扫描
   - 选择 join 算法：Nested Loop / Hash Join / Merge Join
   - 排序、聚合方式优化
3. **生成 Query Plan**
   - 逻辑树 → 物理执行树
   - 包含算子（Scan, Filter, Join, Aggregate）

**输出**：Query Plan（执行计划）

------

# 四、阶段三：执行器（Executor）

**目的**：按照执行计划实际操作数据，产生结果或修改。

1. **算子迭代执行**
   - 每个算子提供 `Next()` 接口
   - 数据从下层算子拉取，经过上层算子处理
2. **访问数据 & MVCC**
   - 扫描表或索引
   - 对每条 tuple 检查 **可见性（xmin/xmax + 当前事务快照）**
   - 只处理对当前事务可见的版本
3. **执行算子操作**
   - Filter: 筛选条件
   - Project: SELECT 的列
   - Join / Aggregate / Sort
4. **写 WAL（如果是修改操作）**
   - INSERT/UPDATE/DELETE 写日志保证持久性
5. **返回结果**
   - SELECT: tuple stream
   - UPDATE/DELETE/INSERT: 影响行数

------

# 五、阶段四：事务管理 & 提交

1. **事务 ID 分配**
   - 每个事务分配 XID
   - UPDATE/DELETE 新版本 tuple xmin = XID，旧版本 xmax = XID
2. **提交或回滚**
   - COMMIT → 改变事务状态，其他事务可见
   - ROLLBACK → 丢弃修改，释放 XID
3. **垃圾回收（VACUUM）**
   - 清理旧版本 tuple，防止版本链无限增长

------

# 六、完整示例流程（SELECT）

```sql
SELECT name, salary FROM emp WHERE dept='IT';
```

### 执行步骤：

1. **解析器**
   - Parse Tree: SELECT(name, salary) FROM emp WHERE dept='IT'
2. **优化器**
   - 执行计划: Seq Scan on emp → Filter(dept='IT') → Project(name, salary)
3. **执行器**
   - 扫描 emp 表
   - 对每条 tuple 做 MVCC 可见性判断
   - 筛选 dept='IT'
   - 返回 name, salary

------

# 七、完整示例流程（UPDATE）

```sql
UPDATE emp SET salary = salary+100 WHERE dept='IT';
```

1. **解析器**
   - Parse Tree: UPDATE emp SET salary=salary+100 WHERE dept='IT'
2. **优化器**
   - 执行计划: Seq Scan on emp → Filter(dept='IT') → Update(salary=salary+100)
3. **执行器**
   - 扫描 emp
   - MVCC 可见性判断
   - 对可见行：
     - 创建新 tuple（xmin = XID，xmax = NULL）
     - 标记旧 tuple xmax = XID
     - 写 WAL 日志
4. **事务提交**
   - 改变 XID 状态 → 对其他事务可见
5. **垃圾回收**
   - 旧 tuple 等待 VACUUM 清理

------

# 八、总结执行流程图

```
SQL 文本
   │
解析器（Parsing） → Parse Tree / AST
   │
优化器（Optimizer） → Query Plan（算子树）
   │
执行器（Executor）：
   ├─ 扫描表 / 索引
   ├─ MVCC 可见性判断
   ├─ Filter / Project / Join / Aggregate
   ├─ 修改操作写 WAL
   └─ 返回结果
   │
事务管理：
   ├─ 分配 XID
   ├─ 提交 / 回滚
   └─ 清理旧版本（VACUUM）
```

------

# 九、核心要点

1. **解析器**：理解 SQL
2. **优化器**：生成最优计划
3. **执行器**：真正访问数据 + MVCC + WAL
4. **事务管理**：保证 ACID
5. **MVCC + 版本链**：支持高并发读取
6. **WAL**：保证持久性