# MySQL 8.0.41 源码阅读：条件优化与索引选择

> 本文基于 MySQL 8.0.41 源码，系统讲解一条 `SELECT` 语句的 WHERE 条件如何经过优化器的层层处理，最终选定最优索引并定位到存储引擎中的 B+ 树位置。全文以**条件优化**和**索引选择**为主线，辅以数据结构、代价模型、执行细节等内容帮助理解。

## 目录

- [第一章 全景概览](#第一章-全景概览)
- [第二章 WHERE 条件的内部表示](#第二章-where-条件的内部表示)
- [第三章 条件规范化与等值传播](#第三章-条件规范化与等值传播)
  - [3.6 optimize_cond 之外的其他条件优化](#36-optimize_cond-之外的其他条件优化)
  - [3.7 MySQL 未实现的条件优化](#37-mysql-未实现的条件优化)
  - [3.8 条件优化能力总结](#38-条件优化能力总结)
- [第四章 条件下推到各表](#第四章-条件下推到各表)
  - [4.3 条件下推到派生表](#43-条件下推到派生表derived-condition-pushdown)
  - [4.4 Index Condition Pushdown（ICP）](#44-index-condition-pushdownicp)
- [第五章 Range Optimizer：从条件构建范围树](#第五章-range-optimizer从条件构建范围树)
  - [5.6 OR 条件的处理局限与 Index Merge](#56-or-条件的处理局限与-index-merge)
- [第六章 范围扫描的执行：从 SEL_ARG 到存储引擎索引定位](#第六章-范围扫描的执行从-sel_arg-到存储引擎索引定位)
- [第七章 Range 分析的两阶段设计](#第七章-range-分析的两阶段设计)
  - [7.6 跨表不等式条件的传递缺失](#76-跨表不等式条件的传递缺失)
- [第八章 候选索引评估与代价模型](#第八章-候选索引评估与代价模型)
- [第九章 最优访问路径选择 best_access_path](#第九章-最优访问路径选择-best_access_path)
- [第十章 JOIN 顺序优化：贪心搜索算法](#第十章-join-顺序优化贪心搜索算法)
- [第十一章 端到端完整示例](#第十一章-端到端完整示例)
- [附录 A：关键数据结构速查](#附录-a关键数据结构速查)
- [附录 B：关键源码位置索引](#附录-b关键源码位置索引)
- [附录 C：子查询优化与去关联的能力边界](#附录-c子查询优化与去关联的能力边界)
- [附录 D：关键函数源码注释](#附录-d关键函数源码注释)

---

## 第一章 全景概览

一条 SQL 从文本到执行，在优化器阶段要经历以下核心步骤。本文聚焦的**条件优化**和**索引选择**贯穿其中：

```
SQL 文本
  │
  ▼
Parser（解析器）
  │  生成 Item 表达式树
  ▼
JOIN::optimize()                           ← sql/sql_optimizer.cc:337
  │
  ├─ optimize_cond()                       ← 【条件优化】规范化（等值传播、常量传播、消除恒真/假）
  │   ├─ build_equal_items()               ← 构建多重等值谓词 Item_equal
  │   ├─ propagate_cond_constants()        ← 常量传播 (field=field → field=const)
  │   └─ remove_eq_conds()                 ← 移除恒真/恒假条件
  │
  ├─ make_join_plan()                      ← 制定 JOIN 计划
  │   ├─ update_ref_and_keys()             ← 收集可用于 ref 访问的 Key_use 结构
  │   │
  │   ├─ estimate_rowcount()               ← 【索引选择·第一阶段】Range 分析（仅常量条件）
  │   │   └─ get_quick_record_count()
  │   │       └─ test_quick_select(prev_tables=0, read_tables=0)
  │   │           ├─ get_mm_tree()         ← 将条件转换为范围树
  │   │           ├─ get_key_scans_params()← 评估各索引的范围扫描代价
  │   │           └─ → 输出: tab->range_scan(), tab->found_records, tab->needed_reg
  │   │
  │   └─ choose_table_order()              ← 【索引选择·核心】确定表的 JOIN 顺序
  │       └─ greedy_search()
  │           └─ best_extension_by_limited_search()
  │               └─ best_access_path()    ← 为每个表选择最优访问方式
  │                   ├─ find_best_ref()    ← 评估 ref 访问
  │                   └─ calculate_scan_cost()  ← 评估 range/全表扫描
  │
  ├─ make_join_query_block()               ← 将条件下推到各表
  │   ├─ make_cond_for_table()             ← 提取适用于特定表的条件
  │   └─ 【索引选择·第二阶段】若 needed_reg 非空，重新运行 Range 分析
  │
  └─ make_join_readinfo()                  ← 最终确定各表的访问方法
```

**关键源文件**：

| 文件 | 职责 |
|------|------|
| `sql/sql_optimizer.cc` | JOIN 优化主入口，条件优化、等值传播 |
| `sql/sql_planner.cc` | 访问路径选择、JOIN 顺序搜索 |
| `sql/range_optimizer/range_optimizer.cc` | Range Optimizer 入口 |
| `sql/range_optimizer/range_analysis.cc` | 范围树构建（get_mm_tree） |
| `sql/range_optimizer/tree.h` / `tree.cc` | SEL_TREE / SEL_ARG 数据结构及合并 |
| `sql/range_optimizer/index_range_scan_plan.cc` | 范围扫描代价评估 |
| `sql/opt_costmodel.h` | 代价模型 |
| `sql/item_cmpfunc.h` | 条件 Item 类层次 |
| `sql/handler.h` / `handler.cc` | 存储引擎接口（代价估算、统计信息） |

---

## 第二章 WHERE 条件的内部表示

> **注：本章为辅助内容，帮助理解后续章节中条件如何被处理和变换。**

MySQL 将 SQL 中的每个表达式都表示为 `Item` 对象。WHERE 条件由一棵 `Item` 树表示。

### 2.1 核心类层次

```
Item                                     ← sql/item.h:853 （所有表达式基类）
  ├─ Item_func                           ← sql/item_func.h:171 （函数基类）
  │   ├─ Item_bool_func                  ← sql/item_cmpfunc.h:295 （布尔函数基类）
  │   │   ├─ Item_func_eq               ← 等值比较 (=)
  │   │   ├─ Item_func_lt               ← 小于 (<)
  │   │   ├─ Item_func_ge               ← 大于等于 (>=)
  │   │   ├─ Item_func_between          ← BETWEEN
  │   │   ├─ Item_func_in               ← IN (...)
  │   │   └─ ...
  │   │
  │   └─ Item_cond                       ← sql/item_cmpfunc.h:2427 （AND/OR 基类）
  │       ├─ Item_cond_and              ← AND 条件节点
  │       │   └─ cond_equal: COND_EQUAL ← 存储当前 AND 级别的多重等值
  │       └─ Item_cond_or               ← OR 条件节点
  │
  └─ Item_equal                          ← sql/item_cmpfunc.h:2567 （多重等值谓词）
      ├─ fields: List<Item_field>        ← 等值字段列表 (f1=f2=...=fk)
      └─ m_const_arg: Item*             ← 可选的常量值
```

### 2.2 关键类型说明

- **Item_cond_and / Item_cond_or**：AND/OR 逻辑运算节点，持有子 Item 列表 (`List<Item> list`)。嵌套的 AND/OR 会在 `fix_fields()` 阶段被**扁平化**——例如 `(a AND (b AND c))` 变为 `(a AND b AND c)`。

- **Item_equal**：MySQL 独有的多重等值谓词。例如 `a=b AND b=c` 会被合并为一个 `Item_equal(a, b, c)`。若等值中包含常量（如 `a=5`），则变为 `Item_equal(5, a, b, c)`。

- **COND_EQUAL**：等值条件容器，内嵌在 `Item_cond_and` 中。`current_level` 存储该 AND 层级提取出的所有 `Item_equal` 对象，`upper_levels` 指向外层。

```cpp
// sql/item_cmpfunc.h:2707
struct COND_EQUAL {
  List<Item_equal> current_level;  // 当前 AND 层的 Item_equal 列表
  COND_EQUAL *upper_levels;        // 外层 AND 的 COND_EQUAL 指针
};
```

---

## 第三章 条件规范化与等值传播

这是条件优化的**第一个核心环节**。优化器在进入索引选择之前，先将 WHERE 条件进行规范化处理，使其更利于后续的索引匹配。

### 3.1 入口：optimize_cond()

```cpp
// sql/sql_optimizer.cc:10338
bool optimize_cond(THD *thd, Item **cond, COND_EQUAL **cond_equal,
                   mem_root_deque<Table_ref *> *join_list,
                   Item::cond_result *cond_value);
```

该函数执行三个关键转换步骤：

### 3.2 步骤一：等值传播（equality_propagation）

```cpp
// sql/sql_optimizer.cc:10374
build_equal_items(thd, *cond, cond, nullptr, true, join_list, cond_equal);
```

**目标**：识别形如 `a=b` 的等值条件，将它们合并为 `Item_equal` 对象。

**过程**（`build_equal_items_for_cond()`）：

1. 递归遍历条件树
2. 在每个 AND 层级，通过 `check_equality()` 检测等值谓词
3. `check_simple_equality()` 将 `a=b` 和 `b=c` 合并为 `Item_equal(a,b,c)`
4. 如果等值中包含常量（如 `a=5`），则变为 `Item_equal(5, a, b, c)`
5. 等值谓词从原来的 AND 列表中移除，放入 `COND_EQUAL.current_level`

**示例**：

```sql
WHERE a = b AND b = c AND c = 5 AND d > 10
```

转换后：

```
Item_cond_and:
  ├─ cond_equal.current_level: [Item_equal(5, a, b, c)]
  └─ list: [d > 10]
```

**这一步对索引选择的意义**：等值传播使得原本散落的 `a=b, b=c, c=5` 被统一识别为 `a=b=c=5`。这意味着 Range Optimizer 后续可以对 a、b、c 三个字段都应用 `=5` 的范围条件，大大增加了可利用的候选索引。

### 3.3 步骤二：常量传播（constant_propagation）

```cpp
// sql/sql_optimizer.cc:10390
propagate_cond_constants(thd, nullptr, *cond, *cond);
```

将 `field = const` 的知识传播到其他使用该 field 的条件中。例如：

- `a = 5 AND a + b > 10` → `5 + b > 10`（简化后续条件）

### 3.4 步骤三：消除恒真/恒假条件（trivial_condition_removal）

```cpp
// sql/sql_optimizer.cc:10408
remove_eq_conds(thd, *cond, cond, cond_value);
```

- 移除 `1=1` 这种恒真条件
- 如果检测到 `1=0` 这种恒假条件，标记整个查询返回空集

### 3.5 后续：字段替换

在确定 JOIN 顺序后，`substitute_for_best_equal_field()` 会将 `Item_equal` 对象转换回普通的等值谓词 (`Item_func_eq`)，并选择最优的字段作为比较基准——优先选择已经通过索引定位的表的字段。

### 3.6 optimize_cond 之外的其他条件优化

除了 `optimize_cond()` 中的三步转换，MySQL 在优化流程的其他阶段还实现了以下条件优化：

#### 常量折叠（Constant Folding）

```cpp
// sql/sql_const_folding.cc
fold_condition(thd, cond, &retcond, &cond_value);
```

基于**字段类型的值域**判断条件恒真/恒假。例如：
- `unsigned_tinyint_col < 0` → `FALSE`（unsigned 不可能小于 0）
- `signed_tinyint_col < 128` → `TRUE`（signed tinyint 最大值 127）
- `decimal_col >= 5.0` 当存储精度截断后值恰好为 5 时 → 简化为 `decimal_col = 5`

#### 外连接转内连接（Outer-to-Inner Join Conversion）

```cpp
// sql/sql_resolver.cc:768
simplify_joins(thd, &m_table_nest, true, false, &m_where_cond);
```

当 WHERE 条件隐含了空值拒绝（null-rejecting）语义时，LEFT JOIN 可以安全地转换为 INNER JOIN。例如：

```sql
-- 原始：LEFT JOIN + WHERE 过滤右表
SELECT * FROM t1 LEFT JOIN t2 ON t1.a = t2.b WHERE t2.c > 10;
-- t2.c > 10 拒绝了 t2 为 NULL 的行，等价于 INNER JOIN
SELECT * FROM t1 INNER JOIN t2 ON t1.a = t2.b WHERE t2.c > 10;
```

转换为 INNER JOIN 后，优化器获得了更大的表排序自由度——t2 可以排在 t1 前面，可能产生更优的执行计划。

#### 表达式索引替换（Generated Column Index Substitution）

```cpp
// sql/sql_optimizer.cc:1217
where_cond->compile(&Item::gc_subst_analyzer, &dummy,
                    &Item::gc_subst_transformer, ...);
```

当 WHERE 中出现 `f(col)` 且表上有基于相同表达式的函数索引（generated column），优化器自动将表达式替换为 generated column 字段，从而使用对应索引。例如：

```sql
-- 表上有 INDEX idx_upper ((UPPER(name)))
SELECT * FROM users WHERE UPPER(name) = 'ALICE';
-- 优化器将 UPPER(name) 替换为对应的 generated column 字段，走 idx_upper 索引
```

#### 分区裁剪（Partition Pruning）

```cpp
// sql/sql_optimizer.cc:2826
prune_partitions(thd, tbl->table, query_block, cond);
```

基于 WHERE 条件判断哪些分区可能包含匹配行，只扫描相关分区。

#### 公共子表达式提取（CSE，仅 Hypergraph 优化器）

```cpp
// sql/join_optimizer/common_subexpression_elimination.cc
Item *CommonSubexpressionElimination(Item *cond);
```

将 `(a AND b) OR (a AND c)` 重写为 `a AND (b OR c)`，使提取出的 `a` 可以独立下推或用于 hash join。**注意：此优化仅在 MySQL 8.0 的新 Hypergraph 优化器中实现，传统优化器不支持。**

### 3.7 MySQL 未实现的条件优化

与 PostgreSQL、Oracle、SQL Server 等数据库相比，MySQL 在条件优化方面存在以下显著缺失。理解这些缺失有助于在实际使用中规避性能陷阱。

#### 不等式传递推导（Transitive Closure for Inequalities）

MySQL 只做**等值传播**（`a=b AND b=5` → `a=5`），**不做不等式的传递推导**。

```sql
-- 表上有索引 INDEX idx_a(a)
SELECT * FROM t1 WHERE a = b AND b > 10;
```

- **PostgreSQL/Oracle**：从 `a = b AND b > 10` 推导出 `a > 10`，自动为 `a` 生成范围条件，走 `idx_a` 索引
- **MySQL**：不推导 `a > 10`。`idx_a` 索引无法利用，如果 `b` 上没有索引则退化为全表扫描
- **Workaround**：手动添加 `AND a > 10`

同样，`a > b AND b > 10` 不会推导出 `a > 10`。

#### 冗余谓词消除（Redundant Predicate Elimination）

MySQL 不做条件层面的蕴含关系识别和冗余消除。

```sql
SELECT * FROM orders WHERE amount > 100 AND amount > 50 AND amount < 1000;
```

- **Oracle**：识别 `amount > 100` 蕴含 `amount > 50`，删除后者
- **MySQL**：三个条件都保留。Range Optimizer 在 SEL_ARG 层面通过区间交集隐式得到 `(100, 1000)`，索引扫描范围正确；但条件求值时仍逐个评估三个谓词，`amount > 50` 白费 CPU

更常见于 ORM 或动态拼接 SQL 的场景：

```sql
-- 业务层和权限层各自加了过滤条件
SELECT * FROM orders
WHERE created_at > '2024-01-01'       -- 业务层
  AND created_at > '2023-06-01'       -- 权限层（已被上面覆盖，冗余）
  AND status = 'paid'
  AND status IN ('paid', 'shipped');   -- status='paid' 已蕴含在 IN 中，冗余
```

#### OR→IN 合并

```sql
SELECT * FROM orders
WHERE status = 'paid' OR status = 'shipped' OR status = 'delivered';
```

- **PostgreSQL/Oracle**：自动合并为 `status IN ('paid', 'shipped', 'delivered')`，IN 可用排序二分或 hash 查找
- **MySQL**：不做条件层面的合并。Range Optimizer 在 SEL_ARG 层通过 `tree_or()` 得到三个等值区间，索引扫描等价；但在其他代码路径（如 `update_ref_and_keys()` 构建 Key_use）中，一条 IN 和多条 OR 的处理效率有差异

#### 通用布尔代数简化

MySQL 只做 AND/OR 的扁平化（消除不必要的嵌套），不做通用的布尔代数变换。

**吸收律**：`A OR (A AND B) = A`

```sql
SELECT * FROM orders WHERE status = 'paid' OR (status = 'paid' AND amount > 100);
-- 应简化为 WHERE status = 'paid'
-- MySQL 不做此简化，(status='paid' AND amount>100) 分支白白评估 amount>100
```

**分配律**：`(A OR B) AND (A OR C) = A OR (B AND C)`

```sql
SELECT * FROM orders
WHERE (customer_id = 100 OR customer_id = 200)
  AND (customer_id = 100 OR status = 'paid');
-- 应简化为 WHERE customer_id = 100 OR (customer_id = 200 AND status = 'paid')
-- 简化后 customer_id = 100 可独立下推，对索引利用更友好
-- MySQL 传统优化器不做此变换（Hypergraph 优化器的 CSE 仅处理反向模式）
```

**德摩根定律**：`NOT (A OR B) = NOT A AND NOT B`

```sql
SELECT * FROM orders WHERE NOT (customer_id <> 100 OR status <> 'paid');
-- 等价于 WHERE customer_id = 100 AND status = 'paid'
-- 后者可直接用于索引匹配，前者的 NOT+OR 结构 MySQL 无法利用索引
```

#### 基于约束的条件简化

MySQL 的 `fold_condition()` 利用字段类型值域做判断（如 unsigned < 0 恒假），但**不利用 CHECK 约束和 UNIQUE 约束**做条件简化。

```sql
CREATE TABLE measurements (
    id INT PRIMARY KEY,
    temperature DECIMAL(5,2),
    CHECK (temperature >= -273.15)
);
SELECT * FROM measurements WHERE temperature > -300;
```

- **PostgreSQL**：利用 CHECK 约束推断 `temperature > -300` 恒真，消除该条件
- **MySQL**：不利用 CHECK 约束语义。虽然 MySQL 8.0 支持 CHECK 约束语法，但优化器不用它来推断条件

```sql
CREATE TABLE users (id INT PRIMARY KEY, email VARCHAR(200) UNIQUE NOT NULL);
SELECT COUNT(DISTINCT email) FROM users;
```

- **Oracle**：利用 UNIQUE 约束消除不必要的 DISTINCT 操作，`COUNT(DISTINCT email)` → `COUNT(email)` → `COUNT(*)`
- **MySQL**：不做此推断，仍执行完整的 DISTINCT 去重逻辑

### 3.8 条件优化能力总结

| 优化类型 | MySQL 8.0 | PostgreSQL | Oracle |
|---------|-----------|------------|--------|
| 等值传播 | ✅ 完善 | ✅ | ✅ |
| 常量传播 | ✅ | ✅ | ✅ |
| 常量折叠（类型值域） | ✅ | ✅ | ✅ |
| 恒真/恒假消除 | ✅ | ✅ | ✅ |
| 外连接→内连接 | ✅ | ✅ | ✅ |
| 表达式索引替换 | ✅ | ✅ | ✅ |
| 分区裁剪 | ✅ | ✅ | ✅ |
| 不等式传递推导 | ❌ | ✅ | ✅ |
| 冗余谓词消除 | ❌ 仅范围层隐式 | ✅ | ✅ |
| OR→IN 合并 | ❌ | ✅ | ✅ |
| 通用布尔代数简化 | ❌ 仅扁平化 | ✅ 部分 | ✅ |
| 基于约束简化 | ❌ 仅类型值域 | ✅ CHECK/NOT NULL | ✅ |
| CSE（公共子表达式） | ⚠️ 仅 Hypergraph | ✅ | ✅ |

---

## 第四章 条件下推到各表

条件规范化之后，优化器需要将全局 WHERE 条件拆分，将各部分"下推"到它们最早能被评估的表上。

### 4.1 核心函数：make_cond_for_table()

```cpp
// sql/sql_optimizer.cc:9496
static Item *make_cond_for_table(THD *thd, Item *cond, table_map tables,
                                 table_map used_table,
                                 bool exclude_expensive_cond);
```

此函数从整体 WHERE 条件中提取**只依赖于指定表集合**的子条件：

- **AND 条件**：递归处理每个子条件，收集适用的部分
- **OR 条件**：所有分支都必须适用，否则整个 OR 都不能下推
- 检查每个条件引用的表 (`used_tables()`)，只保留依赖表都已在计划前缀中的条件

### 4.2 条件下推对索引选择的影响

条件下推决定了每张表在评估时有哪些 WHERE 子句可用。只有下推到当前表的条件才能参与该表的索引选择——无论是 ref 访问还是 range 扫描。

### 4.3 条件下推到派生表（Derived Condition Pushdown）

```cpp
// sql/sql_derived.cc
Query_block::push_conditions_to_derived_tables();
```

MySQL 8.0 支持将外层查询的条件穿透到派生表（子查询/视图）的内部查询中。优化器会将条件中的列引用替换为派生表内部的对应表达式，然后下推到内层 WHERE 中。

```sql
-- 原始查询
SELECT * FROM (SELECT a + 7 AS i FROM t1) AS dt WHERE dt.i > 100;
-- 条件下推后等价于
SELECT * FROM (SELECT a + 7 AS i FROM t1 WHERE a + 7 > 100) AS dt;
```

这使得内层查询可以利用 `t1.a` 上的索引进行范围扫描，而不是先物化整个派生表再过滤。

### 4.4 Index Condition Pushdown（ICP）

```cpp
// sql/sql_select.cc:2905
void QEP_TAB::push_index_cond(const JOIN_TAB *join_tab, uint keyno, ...);
```

ICP 是一种将部分 WHERE 条件下推到**存储引擎层**的优化。在索引扫描过程中，存储引擎可以直接评估那些只涉及索引列的条件，提前过滤不满足的行，减少回表次数。

```sql
-- 索引 idx_name_age(name, age)
SELECT * FROM users WHERE name LIKE 'A%' AND age > 30;
```

- **无 ICP**：存储引擎通过 `name LIKE 'A%'` 定位索引范围，对每条匹配记录回表读取完整行，再由 Server 层评估 `age > 30`
- **有 ICP**：存储引擎在索引扫描时直接评估 `age > 30`（age 在索引中可见），不满足的行直接跳过，不回表

EXPLAIN 中 `Extra: Using index condition` 表示启用了 ICP。

### 4.5 条件下推的层次总结

MySQL 的条件下推在多个层次上发挥作用：

| 下推层次 | 机制 | 效果 |
|---------|------|------|
| 全局 → 各表 | `make_cond_for_table()` | 条件在最早可评估的表上过滤 |
| 外层 → 派生表内层 | `push_conditions_to_derived_tables()` | 避免物化全部数据再过滤 |
| Server 层 → 存储引擎层 | ICP (`push_index_cond()`) | 减少回表次数 |

---

## 第五章 Range Optimizer：从条件构建范围树

Range Optimizer 是索引选择的**核心组件**。它将 WHERE 条件转换为索引上的**区间范围**，评估各种范围扫描的代价，为后续的索引选择提供基础数据。

### 5.1 入口：test_quick_select()

```cpp
// sql/range_optimizer/range_optimizer.cc:524
int test_quick_select(THD *thd, MEM_ROOT *return_mem_root,
                      MEM_ROOT *temp_mem_root, Key_map keys_to_use,
                      table_map prev_tables, table_map read_tables,
                      ha_rows limit, bool force_quick_range,
                      const enum_order interesting_order, TABLE *table,
                      bool skip_records_in_range, Item *cond,
                      Key_map *needed_reg, bool ignore_table_scan,
                      Query_block *query_block, AccessPath **path);
```

**整体流程**：

```
test_quick_select()
  │
  ├─ 1. 计算全表扫描代价作为基线
  │
  ├─ 2. 若有覆盖索引，计算最短覆盖索引扫描代价
  │
  ├─ 3. 调用 get_mm_tree() 构建范围树 ← 核心步骤
  │
  ├─ 4. 若 tree->type == IMPOSSIBLE → 返回 -1（不可能有结果）
  │     若 tree->type != KEY → tree = nullptr（无法用于范围扫描）
  │
  ├─ 5. 评估 Group Index Skip Scan
  │
  ├─ 6. 评估 Index Skip Scan
  │
  ├─ 7. 评估各索引的范围扫描 → get_key_scans_params()
  │     对每个索引调用 check_quick_select() 估算行数和代价
  │
  ├─ 8. 评估 ROR 索引交集
  │
  ├─ 9. 评估 Index Merge Union
  │
  └─ 10. 返回代价最低的 AccessPath
```

### 5.2 构建范围树：get_mm_tree()

```cpp
// sql/range_optimizer/range_analysis.cc:845
SEL_TREE *get_mm_tree(THD *thd, RANGE_OPT_PARAM *param, table_map prev_tables,
                      table_map read_tables, table_map current_table,
                      bool remove_jump_scans, Item *cond);
```

这是 Range Optimizer 的核心函数，递归地将 WHERE 条件转换为范围树。

**算法**：

```
get_mm_tree(cond):
  │
  ├─ 若 cond 是 Item_cond (AND/OR)：
  │   for each child_item in cond.list:
  │     new_tree = get_mm_tree(child_item)   // 递归
  │     if AND: tree = tree_and(tree, new_tree)
  │     if OR:  tree = tree_or(tree, new_tree)
  │   return tree
  │
  ├─ 若 cond 是常量：
  │   return SEL_TREE(ALWAYS) 或 SEL_TREE(IMPOSSIBLE)
  │
  └─ 若 cond 是 Item_func（比较操作）：
      根据 functype 分派：
      ├─ BETWEEN / IN / EQ / LT / LE / GT / GE / NE:
      │   → get_full_func_mm_tree()
      │       → get_mm_parts()     ← 为字段找到所有匹配的索引键部分
      │           → get_mm_leaf()  ← 创建单个 SEL_ARG 范围节点
      └─ 其他: return nullptr
```

### 5.3 叶子节点构造：get_mm_leaf()

`get_mm_leaf()`（`sql/range_optimizer/range_analysis.cc:1370`）是**设置范围标志位的关键函数**，它根据比较运算符类型创建 `SEL_ARG` 节点：

| SQL 条件 | 生成的范围 | min_flag | max_flag |
|----------|-----------|----------|----------|
| `col = 5` | `[5, 5]` | `0` | `0` |
| `col > 5` | `(5, +∞)` | `NEAR_MIN` | `NO_MAX_RANGE` |
| `col >= 5` | `[5, +∞)` | `0` | `NO_MAX_RANGE` |
| `col < 5` | `(-∞, 5)` | `NO_MIN_RANGE` | `NEAR_MAX` |
| `col <= 5` | `(-∞, 5]` | `NO_MIN_RANGE` | `0` |
| `col BETWEEN 3 AND 7` | `[3, 7]` | `0` | `0` |
| `col IN (1,3,5)` | `[1,1] ∪ [3,3] ∪ [5,5]` | — | — |
| `col IS NULL` | `[NULL, NULL]` | `0` | `0` |
| `col IS NOT NULL` | `(NULL, +∞)` | `NEAR_MIN` | `NO_MAX_RANGE` |

> **注：`key_range_flags` 标志位**定义在 `include/my_base.h:1078-1122`，是贯穿整个范围优化过程的关键，它们在 `SEL_ARG` 的 `min_flag`/`max_flag` 中设置，传递到 `QUICK_RANGE` 的 `flag` 中，最终决定存储引擎 `ha_rkey_function` 的取值（详见第六章）。

### 5.4 SEL_TREE / SEL_ARG 数据结构

> **注：本节为辅助内容，帮助理解范围树的内部结构。**

#### SEL_TREE

```cpp
// sql/range_optimizer/tree.h:871
class SEL_TREE {
  enum Type { IMPOSSIBLE, ALWAYS, KEY } type;
  Mem_root_array<SEL_ROOT *> keys;   // 每个候选索引一个 SEL_ROOT
  Key_map keys_map;                   // 非 NULL 的 keys 位图
  List<SEL_IMERGE> merges;           // Index Merge 备选方案
};
```

- `keys[i]` 表示第 i 个索引上的范围条件
- 多个索引的 AND 关系隐含在 SEL_TREE 中（取行的交集）

#### SEL_ARG（范围区间的红黑树节点）

```cpp
// sql/range_optimizer/tree.h:465
class SEL_ARG {
  uint8 min_flag, max_flag;       // 范围边界标志
  uint8 part;                      // 索引键部分编号 (0-based)
  uchar *min_value, *max_value;   // 范围端点值

  SEL_ARG *left, *right;          // 红黑树子节点（组织同层区间）
  SEL_ARG *next, *prev;           // 双向链表（同层区间的 OR 关系）
  SEL_ROOT *next_key_part;        // 下一个键部分的范围（AND 语义）
};
```

**图语义**：
- `next` / `prev` → **OR 关系**（同一键部分上的不相交区间）
- `next_key_part` → **AND 关系**（连续键部分上的条件）

**示例**：`WHERE (kp1=1 AND kp2=10) OR (kp1=2 AND kp2=20)`

```
SEL_ROOT (kp1)
   │
   ├─ SEL_ARG [kp1=1] ──next──> SEL_ARG [kp1=2]
   │     │                           │
   │     └─ next_key_part            └─ next_key_part
   │           │                           │
   │        SEL_ROOT (kp2)              SEL_ROOT (kp2)
   │           │                           │
   │        SEL_ARG [kp2=10]           SEL_ARG [kp2=20]
```

### 5.5 范围树合并

范围树合并是 AND/OR 条件转换的关键操作：

**tree_and() — AND 合并**（`sql/range_optimizer/tree.cc:503`）：
- 对每个索引：调用 `key_and()` 合并对应的 SEL_ROOT
- 在**相同键部分**上做**区间交集**，在**不同键部分**上通过 `next_key_part` 做 AND 连接
- AND 可以保留所有索引的范围（交集越多越精确）

**tree_or() — OR 合并**（`sql/range_optimizer/tree.cc:680`）：
- 对每个**共有索引**：调用 `key_or()` 合并对应的 SEL_ROOT
- 在**相同键部分**上做**区间并集**
- 不共有的索引会被丢弃（OR 要求两边都满足，不能只用一边的索引）
- OR 只能保留两边都有范围的索引（并集可能退化为全表扫描）

### 5.6 OR 条件的处理局限与 Index Merge

OR 条件是 MySQL 范围优化的一个薄弱环节。`tree_or()` 要求 OR 的每个分支在**同一个索引**上都有范围条件，否则该索引被丢弃。当 OR 的不同分支涉及不同索引时，MySQL 依赖 **Index Merge** 机制：

```
test_quick_select()
  ├─ 7. get_key_scans_params()   ← 单索引范围扫描
  ├─ 8. get_best_ror_intersect() ← ROR 索引交集（AND 场景）
  └─ 9. get_best_disjunct_quick()← Index Merge Union（OR 场景）
```

Index Merge Union 的工作方式是：对 OR 的每个分支分别走各自的索引取回 rowid 集合，对 rowid 求并集，再回表。但它有严格的限制：

**限制一：OR 中任一分支不可索引则整体失效**

```sql
-- amount 上没有索引
SELECT * FROM orders WHERE customer_id = 100 OR amount > 10000;
```

`amount > 10000` 无法产生范围扫描，Index Merge 无法使用，整条查询退化为全表扫描。

**限制二：嵌套 AND/OR 组合时很难触发**

```sql
SELECT * FROM orders
WHERE (customer_id = 100 AND amount > 500)
   OR (status = 'refunded' AND amount < 100);
```

每个 OR 分支是两个条件的 AND，其中 `amount` 无索引，导致 Index Merge 无法使用。

#### MySQL 未实现的 OR 展开（OR Expansion / OR-to-UNION）

Oracle 和 PostgreSQL 采用了一种更强大的策略——**OR 展开**：将 OR 条件转换为等价的 UNION ALL，每个分支独立优化。

```sql
-- 原始查询
SELECT * FROM orders WHERE customer_id = 100 OR status = 'refunded';

-- Oracle/PostgreSQL 自动转换为
SELECT * FROM orders WHERE customer_id = 100
UNION ALL
SELECT * FROM orders WHERE status = 'refunded' AND customer_id <> 100;
```

两个分支分别走 `idx_customer` 和 `idx_status`，各自扫少量行再合并。即使某个分支不可索引，另一个分支仍然可以走索引：

```sql
-- Oracle 对此查询的处理
SELECT * FROM orders WHERE customer_id = 100    -- 走 idx_customer
UNION ALL
SELECT * FROM orders WHERE amount > 10000       -- 全表扫描，但结果集可能很小
  AND customer_id <> 100;
```

**MySQL 不支持 OR 展开**。用户遇到此类场景需要手动改写为 UNION ALL。

---

## 第六章 范围扫描的执行：从 SEL_ARG 到存储引擎索引定位

> **注：本章为辅助内容，详细讲解范围优化器的结果如何最终转化为存储引擎的 B+ 树定位操作。理解这一流程有助于深入理解索引选择的"最后一公里"。**

### 6.1 整体执行链路

```
SEL_ARG 树
    │
    ▼
get_ranges_from_tree()    ──→  QUICK_RANGE 数组（物化后的索引范围）
    │
    ▼
IndexRangeScanIterator::Read()
    │
    ▼
handler::multi_range_read_next()
    │
    ▼
quick_range_seq_next()    ──→  将 QUICK_RANGE 转换为 key_range（含 ha_rkey_function）
    │
    ▼
handler::read_range_first()
    │
    ▼
handler::ha_index_read_map(buf, key, keypart_map, ha_rkey_function)
    │
    ▼
存储引擎 (InnoDB) B+ 树定位
```

### 6.2 QUICK_RANGE：物化后的索引范围

`SEL_ARG` 树经过 `get_ranges_from_tree()` 递归遍历后，被物化为 `QUICK_RANGE` 数组：

```cpp
// sql/range_optimizer/range_optimizer.h:69-170
class QUICK_RANGE {
  uchar *min_key, *max_key;       // 范围的最小/最大键值
  uint16 min_length, max_length;  // 键值长度
  uint16 flag;                    // key_range_flags 标志位的组合
  key_part_map min_keypart_map, max_keypart_map;  // 使用的 keypart 位图
};
```

### 6.3 从 QUICK_RANGE 到 ha_rkey_function 的转换

`QUICK_RANGE::make_min_endpoint()` 决定起始位置的索引定位方式：

```cpp
// sql/range_optimizer/range_optimizer.h:121-128
void make_min_endpoint(key_range *kr) {
    kr->flag = ((flag & NEAR_MIN)   ? HA_READ_AFTER_KEY
                : (flag & EQ_RANGE) ? HA_READ_KEY_EXACT
                                    : HA_READ_KEY_OR_NEXT);
}
```

**起始键的三路判断**：

| QUICK_RANGE::flag 条件 | ha_rkey_function | 含义 |
|------------------------|------------------|------|
| `flag & NEAR_MIN` | `HA_READ_AFTER_KEY` | 不含左端点，定位到 > min_key 的第一条 |
| `flag & EQ_RANGE` | `HA_READ_KEY_EXACT` | 等值条件，精确定位 |
| 其他 | `HA_READ_KEY_OR_NEXT` | 含左端点，定位到 >= min_key 的第一条 |

**结束键的二路判断**：

| QUICK_RANGE::flag 条件 | ha_rkey_function | 含义 |
|------------------------|------------------|------|
| `flag & NEAR_MAX` | `HA_READ_BEFORE_KEY` | 不含右端点，在 < max_key 处停止 |
| 其他 | `HA_READ_AFTER_KEY` | 含右端点，扫描到 max_key 之后停止 |

### 6.4 InnoDB 层的 B+ 树定位

`handler::read_range_first()` 最终调用 `ha_index_read_map()`，将 `ha_rkey_function` 传递给 InnoDB。InnoDB 在 `convert_search_mode_to_innobase()` 中做最终转换：

```cpp
// storage/innobase/handler/ha_innodb.cc:10096
switch (find_flag) {
    case HA_READ_KEY_EXACT:
    case HA_READ_KEY_OR_NEXT:  return PAGE_CUR_GE;   // B+ 树 >= 定位
    case HA_READ_AFTER_KEY:    return PAGE_CUR_G;     // B+ 树 > 定位
    case HA_READ_BEFORE_KEY:   return PAGE_CUR_L;     // B+ 树 < 定位
    case HA_READ_KEY_OR_PREV:
    case HA_READ_PREFIX_LAST:  return PAGE_CUR_LE;    // B+ 树 <= 定位
}
```

### 6.5 执行示例

**`WHERE key_col = 5`：**

```
SEL_ARG: min=5, max=5, flags=0
→ QUICK_RANGE: flag = EQ_RANGE
→ start_key.flag = HA_READ_KEY_EXACT
→ InnoDB: 精确定位到 key=5 的第一条记录
```

**`WHERE key_col > 10`：**

```
SEL_ARG: min=10, min_flag=NEAR_MIN, max_flag=NO_MAX_RANGE
→ QUICK_RANGE: flag = NEAR_MIN | NO_MAX_RANGE
→ start_key.flag = HA_READ_AFTER_KEY
→ InnoDB: 定位到严格大于 10 的第一条记录
```

**`WHERE a = 3 AND b >= 5`（复合索引 (a, b)）：**

```
SEL_ARG(a): part=0, min=3, max=3
  └─ next_key_part → SEL_ARG(b): part=1, min=5, max_flag=NO_MAX_RANGE
→ QUICK_RANGE: min_key=[3,5], flag=NO_MAX_RANGE
→ start_key.flag = HA_READ_KEY_OR_NEXT
→ InnoDB: 在复合索引中定位到 (a=3, b>=5) 的第一条记录
```

> **注：`ha_rkey_function` 和 `key_range_flags` 是两套不同层次的标志位体系。**`key_range_flags` 是范围优化器的内部语言，以 bitmask 描述完整区间形状；`ha_rkey_function` 是跨存储引擎的操作指令，以枚举值描述单次索引定位操作。前者只在范围扫描路径中存在，后者是所有索引访问方式的通用接口。这是经典的**关注点分离**设计。

---

## 第七章 Range 分析的两阶段设计

Range Optimizer 在优化过程中需要被调用两次。这是因为 `choose_table_order` 尚未执行时，跨表条件（如 `t1.a = t2.b`）无法确定依赖关系是否满足。MySQL 采用**"预计算 + 延迟精化"**的两阶段设计来解决这一循环依赖。

### 7.1 第一阶段：仅常量条件的 Range 分析

**调用位置**：`make_join_plan()` → `estimate_rowcount()` → `get_quick_record_count()`

```cpp
// sql/sql_optimizer.cc:6236
int error = test_quick_select(
    thd, ...,
    0,    // prev_tables = 0：不假设任何表已被读取
    0,    // read_tables = 0：不假设任何表的值可用
    ..., condition, &tab->needed_reg, ...);
```

由于 `prev_tables | read_tables = 0`，任何引用了其他表的条件（如 `t1.a = t2.b`）在 `get_mm_parts()` 中都会被跳过：

```cpp
// sql/range_optimizer/range_analysis.cc:1087
if (value && value->used_tables() & ~(prev_tables | read_tables))
    return nullptr;  // 跨表条件，跳过
```

**MAYBE_KEY 机制**：对于"字段属于当前表但值依赖其他表"的条件，不会直接丢弃，而是标记为 `MAYBE_KEY`，记录到 `needed_reg` 位图中，通知第二阶段：该索引在确定 join 顺序后可能变得可用。

**本阶段的输出**保存在 `JOIN_TAB` 上：
- `tab->range_scan()`：最优范围扫描的 AccessPath（含代价）
- `tab->found_records`：估算的行数
- `tab->needed_reg`：依赖其他表的候选索引位图

### 7.2 choose_table_order 如何复用第一阶段结果

`choose_table_order()` 的搜索过程通过 `best_access_path()` 为每个候选位置的表评估代价。在搜索循环内部，`best_access_path` **直接读取**第一阶段的预计算结果，而非重新运行 Range Optimizer：

```cpp
// sql/sql_planner.cc:821-836  calculate_scan_cost() 内部
if (tab->range_scan()) {
    // 直接使用第一阶段存储的 range 代价和行数
    scan_and_filter_cost = prefix_rowcount * (tab->range_scan()->cost + ...);
}
```

### 7.3 第二阶段：join 顺序确定后的 Range 精化

**调用位置**：`make_join_query_block()`，在 join 顺序已确定之后。

若 `needed_reg` 为空（无跨表依赖），保留第一阶段结果；否则重新运行 Range Optimizer：

```cpp
// sql/sql_optimizer.cc:9905-9916
test_quick_select(
    thd, ..., usable_keys,
    used_tables & ~tab->table_ref->map(),  // prev_tables = 当前表之前的所有表
    0, ..., tab->condition(), &tab->needed_reg, ...);
```

此时 `prev_tables` 包含了 join 顺序中当前表之前的所有表。若 `t2` 排在 `t1` 前面，条件 `t1.a = t2.b` 中的 `t2.b` 在运行时将可用，Range Optimizer 就能为 `t1.a` 构建有效的范围节点。

### 7.4 两阶段对比

| | 第一阶段（`estimate_rowcount`） | 第二阶段（`make_join_query_block`） |
|---|---|---|
| **时机** | `choose_table_order` **之前** | `choose_table_order` **之后** |
| **`prev_tables`** | `0`（无前置表） | 已确定的前置表集合 |
| **可用条件** | 仅常量条件（如 `col = 5`） | 含跨表条件（如 `t1.a = t2.b`） |
| **跨表条件处理** | 跳过或标记为 `MAYBE_KEY` | 可构建真正的范围节点 |
| **调用次数** | 每个非 const 表各 1 次 | 仅对 `needed_reg` 非空的表调用 |
| **结果用途** | 被 `best_access_path` 在搜索中反复引用 | 直接用于最终执行计划 |

### 7.5 为何不合并为一次

将 Range 分析推迟到 `choose_table_order` 内部理论上可行，但工程代价极高：

1. **搜索循环中调用次数极多**：7 个表穷举搜索 = 5,040 次调用 `best_access_path`，每次重新运行 Range 分析不可承受
2. **Range 分析涉及存储引擎 I/O**：`handler::records_in_range()` 是 InnoDB B+ 树 dive 操作，不是纯内存计算
3. **ref 访问已覆盖跨表条件的评估**：使用内存中的 `rec_per_key` 统计信息，可以在 O(N!) 循环内高效执行
4. **粗略估算已足够引导搜索**：纯常量条件通常已足以区分"大表"与"小表"

### 7.6 跨表不等式条件的传递缺失

两阶段设计解决了跨表**等值条件**（如 `t1.a = t2.b`）的利用问题——第二阶段可以为其构建范围节点。但对于跨表**不等式条件的传递推导**，MySQL 完全不支持。

```sql
-- t1 上有索引 idx_a(a)，t2 上有索引 idx_b(b)
SELECT * FROM t1 JOIN t2 ON t1.a = t2.b WHERE t2.b > 10;
```

- **PostgreSQL/Oracle**：从 `t1.a = t2.b AND t2.b > 10` 推导出 `t1.a > 10`。无论 t1 和 t2 的 join 顺序如何，t1 都可以利用 `idx_a` 进行范围扫描 `a > 10`
- **MySQL**：不做此推导。如果 t1 排在 t2 前面，t1 上只能全表扫描（`t2.b > 10` 此时不可用，`t1.a > 10` 未被推导出）

这个缺失在多表 JOIN 中影响尤为显著：

```sql
SELECT * FROM t1
JOIN t2 ON t1.a = t2.b
JOIN t3 ON t2.c = t3.d
WHERE t3.d BETWEEN 100 AND 200;
```

理想情况下，优化器应该推导出 `t2.c BETWEEN 100 AND 200`（从 `t2.c = t3.d`）以及 `t1.a BETWEEN 100 AND 200`（从 `t1.a = t2.b` 和 `t2.b` 与 `t2.c` 的关系——如果 `b = c`）。MySQL 不做任何此类传递推导，导致如果 t1 或 t2 排在 t3 前面，它们无法利用范围条件过滤。

**Workaround**：在 WHERE 中手动为每张表添加冗余的过滤条件：

```sql
SELECT * FROM t1
JOIN t2 ON t1.a = t2.b
JOIN t3 ON t2.c = t3.d
WHERE t3.d BETWEEN 100 AND 200
  AND t2.c BETWEEN 100 AND 200   -- 手动添加
  AND t1.a BETWEEN 100 AND 200;  -- 手动添加
```

---

## 第八章 候选索引评估与代价模型

### 8.1 代价模型

MySQL 使用分层代价模型：

```cpp
// sql/opt_costmodel.h
class Cost_model_server {
  double row_evaluate_cost(double rows);   // CPU: 评估 WHERE 条件的代价
  double key_compare_cost(double keys);    // CPU: 比较 key 的代价
};

class Cost_model_table {
  double page_read_cost(double pages);            // IO: 读取数据页代价
  double page_read_cost_index(uint idx, double pages); // IO: 读取索引页代价
  double io_block_read_cost(double blocks);       // IO: 随机读取磁盘块代价
};
```

**代价组成** = **IO 代价**（从磁盘/缓冲池读取页面）+ **CPU 代价**（评估条件、比较 key）

### 8.2 索引统计信息：rec_per_key

```cpp
// sql/key.h:214-249
class KEY {
  rec_per_key_t records_per_key(uint key_part_no);  // 每个不同 key 值对应的平均行数
};
```

`rec_per_key` 是选择率估算的核心统计量：
- `records_per_key(0)` = 匹配第一个键部分的平均行数
- `records_per_key(N-1)` = 匹配全部 N 个键部分的平均行数
- 例如：索引 `(city, name)` 上 `records_per_key(0) = 1000`，`records_per_key(1) = 2`

### 8.3 Range 扫描行数估算

```cpp
// sql/range_optimizer/index_range_scan_plan.cc:568
ha_rows check_quick_select(THD *thd, RANGE_OPT_PARAM *param, uint idx,
                           bool index_only, SEL_ROOT *tree, ...);
```

1. 将 SEL_ARG 图转换为连续的 key 区间序列
2. 调用 `multi_range_read_info_const()` 估算每个区间的行数
3. 底层调用 `handler::records_in_range()` → InnoDB 通过 B+ 树 dive 估算
4. 汇总所有区间的行数和 IO 代价

### 8.4 Ref 访问代价估算

```cpp
// sql/sql_planner.cc:143
double find_cost_for_ref(const THD *thd, TABLE *table, unsigned keyno,
                         double num_rows, double worst_seeks);
```

| 场景 | 代价计算 |
|------|---------|
| 覆盖索引 | `index_scan_cost(keyno, 1, num_rows)` |
| 聚簇主键 | `read_cost(keyno, 1, num_rows)` |
| 普通二级索引 | `min(page_read_cost(keyno, num_rows), worst_seeks)` |

### 8.5 Ref 访问行数估算（fanout）

`find_best_ref()` 中行数估算的优先级：

1. 若索引为 UNIQUE 且所有键部分都有等值条件 → `fanout = 1`
2. 尝试复用 Range Optimizer 的估算（四个复用场景）
3. 使用 `rec_per_key` 统计信息
4. 若无统计信息，使用启发式线性插值公式

> **注：Range 估算的四个复用场景。**`find_best_ref()` 在多处尝试复用 range 优化器的行数估算，因为 range 优化器基于 InnoDB B+ 树 dive 通常更准确：
>
> | 场景 | 条件 | 说明 |
> |------|------|------|
> | ReuseRange-1 | ref(const) 且 range 使用同一索引 | range 的单点区间等价于 ref(const) |
> | ReuseRange-2 | range 在 ref 的前缀上有估算 | 取较低值作为调整 |
> | ReuseRange-3 | ref(const) 在部分键上 | 满足 const、相同 keypart、区间数匹配条件 |
> | ReuseRange-4 | range 在部分键上有更低估算 | 作为下限调整 |

### 8.6 条件过滤率（Condition Filtering）

`calculate_condition_filter()` 计算**不被访问方法覆盖的剩余条件**的过滤比例（0~1.0），这直接影响后续表看到的行数。

**三阶段计算流程**：

```
filter = 1.0

阶段1：排除已用列
  - ref/range 访问已覆盖的列不重复计算
  - 如果 cond_set ⊆ 已用列，直接返回 1.0

阶段2：利用 range 优化器对其他索引的估算（精度较高）
  - 遍历 quick_keys 中与已用列不重叠的索引
  - filter *= quick_rows[keyno] / records

阶段3：递归计算 WHERE 条件树（get_filtering_effect）
  - AND 条件：P(A AND B) = P(A) × P(B)（假设独立）
  - OR 条件：P(A OR B) = P(A) + P(B) - P(A)×P(B)（容斥原理）
```

**单个谓词的估算：直方图优先**。每个比较运算符先查直方图，没有再用硬编码默认值：

| 运算符 | 直方图 operator | 无直方图时的默认值 |
|-------|----------------|-------------------|
| `=` | `EQUALS_TO` | `0.1` |
| `>`, `>=` | `GREATER_THAN[_OR_EQUAL]` | `0.3333` |
| `<`, `<=` | `LESS_THAN[_OR_EQUAL]` | `0.3333` |
| `BETWEEN` | `BETWEEN` | `0.1111` |
| `IN (v1,..,vN)` | 逐值叠加 | `min(0.5, N × 0.1)` |

**与 EXPLAIN 的对应关系**：

| 代码中的值 | EXPLAIN 列 | 含义 |
|-----------|-----------|------|
| `rows_fetched` = 100 | `rows` = 100 | 访问方法预估返回行数 |
| `filter_effect` = 0.037 | `filtered` = 3.70 | 百分比 |
| `rows_fetched × filter_effect` = 3.7 | — | 该表向后传递的行数 |

---

## 第九章 最优访问路径选择 best_access_path

`best_access_path()` 是为单张表选择最优访问方法的**核心决策函数**。它将 ref 访问和 scan（range/index/全表扫描）进行对比，选择代价最低的方案。

### 9.1 ref 与 scan 两大阵营

在 `best_access_path()` 的语境中，所有访问方式被分为两大阵营：

**ref 阵营**（由 `find_best_ref()` 处理）：
- `eq_ref`：通过唯一索引的所有 keypart 做等值匹配，最多返回 1 行
- `ref`：通过非唯一索引做等值匹配，可能返回多行
- `ref_or_null`：同 ref，但额外匹配 `IS NULL`
- `fulltext`：全文索引查找

**scan 阵营**（由 `calculate_scan_cost()` 处理）：
- `range`：索引范围扫描
- `index`：索引全扫描
- `ALL`：全表扫描
- `index_merge`：多索引合并扫描

**关键点：range（索引范围扫描）属于 scan 阵营，不属于 ref。** ref 的本质是等值查找 (`=` / `<=>`），由 `Key_use` 数组驱动；range 是范围条件 (`>`, `<`, `BETWEEN`, `IN` 等），由前期的 Range Optimizer 独立分析。

### 9.2 决策流程

```
best_access_path()
  │
  ├─ 第一阶段：评估 ref 访问
  │   └─ find_best_ref()
  │       ├─ 遍历所有可用索引（通过 Key_use 列表）
  │       ├─ 对每个索引：
  │       │   ├─ 计算可用的键部分数（found_part）
  │       │   ├─ 分类索引类型：CLUSTERED_PK > UNIQUE > NOT_UNIQUE > FULLTEXT
  │       │   ├─ 估算 fanout（匹配行数）
  │       │   ├─ 计算 read_cost
  │       │   └─ 与当前最优 ref 比较，保留更优的
  │       └─ 返回最优 Key_use（或 nullptr）
  │
  ├─ 第二阶段：决定是否需要评估 scan（启发式规则）
  │   (1) ref 行数更少 且 ref 更便宜 → 跳过 scan
  │   (2) ref 使用了 ≥ range 的键部分数 → 跳过 scan
  │   (3) InnoDB + 覆盖索引 + ref 更优 → 跳过 scan
  │   (4) FORCE INDEX + 有 ref 但无 range → 跳过 scan
  │   否则进入第三阶段
  │
  ├─ 第三阶段：评估 scan 代价
  │   └─ calculate_scan_cost()
  │       ├─ 若有 range_scan: 使用 range 的代价和行数
  │       ├─ 否则: table_scan_cost() 或 index_scan_cost()
  │       ├─ 考虑 Join Buffer 的影响
  │       └─ 返回 scan_read_cost
  │
  ├─ 第四阶段：最终比较
  │   ref_total  = best_read_cost + row_evaluate_cost(prefix_rows × ref_rows)
  │   scan_total = scan_read_cost + row_evaluate_cost(prefix_rows × scan_rows)
  │   选择代价更低的方案
  │
  └─ 第五阶段：计算条件过滤效果
      filter_effect = calculate_condition_filter(tab, best_ref, ...)
```

### 9.3 find_best_ref() 的索引遍历逻辑

`find_best_ref()` 遍历 `Key_use` 数组中当前表的所有可用索引：

```
for each key of table:
    found_part = 0     // 可用的 keypart 位图
    const_part = 0     // 常量 keypart 位图

    for each keyuse of this key:
        // 过滤不可用的 keyuse：
        // 1. 引用了被排除的表（半连接外部表）
        // 2. 引用了尚未加入计划的表
        if excluded_tables & keyuse->used_tables → skip
        if remaining_tables & keyuse->used_tables → skip

        found_part |= keyuse->keypart_map
        if keyuse->val is const:
            const_part |= keyuse->keypart_map

    // 根据 found_part 确定索引类型和估算行数
    if all keyparts matched && UNIQUE:
        type = CLUSTERED_PK 或 UNIQUE, fanout = 1
    else:
        type = NOT_UNIQUE, fanout = records_per_key(...)

    // 代价计算并与当前最优比较
    cost = find_cost_for_ref(...)
    if cost < best_cost: update best
```

**索引选择优先级**：
1. `CLUSTERED_PK`：找到就立即返回（聚簇主键启发式最优）
2. `UNIQUE`：找到后不再考虑 NOT_UNIQUE（除非 `test_all_ref_keys`）
3. `NOT_UNIQUE`：同级选代价最低

### 9.4 WHERE/JOIN 条件如何转换为索引查找

条件到 `Key_use` 的转换发生在 `update_ref_and_keys()` 中（`find_best_ref()` 之前）：

| WHERE 条件形式 | Key_use 表示 | 访问类型 |
|---------------|-------------|---------|
| `t1.a = 5`（常量） | `const_part \|= keypart_map` | ref(const) |
| `t1.a = t2.b`（join 条件） | `used_tables = {t2}` | eq_ref / ref |
| `t1.a = 5 OR t1.a IS NULL` | `ref_or_null_part \|= keypart_map` | ref_or_null |
| `t1.a > 10`（范围条件） | 不产生 Key_use，由 Range Optimizer 处理 | range |

对于 join 条件 `t1.a = t2.b`：
- `keyuse->used_tables & remaining_tables` 必须为 0（t2 必须已在计划前缀中）
- 不同 join 顺序下，同一条件可用于不同表的 ref 访问

---

## 第十章 JOIN 顺序优化：贪心搜索算法

### 10.1 入口：choose_table_order()

```cpp
// sql/sql_planner.cc:1951
bool Optimize_table_order::choose_table_order();
```

- `STRAIGHT_JOIN`：按 FROM 指定顺序，调用 `optimize_straight_join()`
- 否则：调用 `greedy_search()` 进行代价驱动的搜索

### 10.2 表排序预处理

在贪心搜索之前，会对表进行预排序以提供良好的初始顺序：

1. **依赖关系**：被依赖的表排在前面
2. **键依赖**：如果表 B 需要表 A 的列做索引查找，表 A 排在前面
3. **记录数**：记录数少的表排在前面

### 10.3 贪心搜索算法

```
greedy_search(remaining_tables):
    while 还有剩余表:
        // 从当前位置向前搜索 search_depth 层
        best_extension_by_limited_search(remaining, idx, search_depth)

        // 选择搜索到的最优下一个表
        best_table = best_positions[idx]
        remaining -= best_table
        idx++
```

### 10.4 搜索深度

| 场景 | 搜索深度 | 复杂度 |
|------|---------|--------|
| 表数 ≤ 7 | N+1（穷举） | O(N!) |
| 表数 > 7 | 7（启发式） | O(N × N^7 / 7) |
| STRAIGHT_JOIN | 1（固定序） | O(N) |

### 10.5 深度优先搜索（带剪枝）

```
best_extension_by_limited_search(remaining, idx, depth):
    for each table T in remaining:
        // 为 T 选择最优访问路径
        best_access_path(T, remaining, idx, ...)

        // 代价剪枝：累积代价已超过当前最优
        if current_cost > best_cost_so_far: continue

        // 启发式剪枝：已有更好的表在此位置
        if prune_level == 1 && current_cost > best_at_this_position: continue

        if depth > 1 && remaining has more tables:
            // EQ_REF 优化：1:1 连接不需要穷举排列
            if rows_fetched <= 1.0:
                eq_ref_extension_by_limited_search(...)
            else:
                best_extension_by_limited_search(remaining - T, idx+1, depth-1)
        else:
            consider_plan(idx)  // 评估完整计划
```

### 10.6 累积代价模型

对于 N 张表的 join 计划 `[T1, T2, ..., TN]`：

```
prefix_rowcount[i] = prefix_rowcount[i-1] × rows_fetched[i] × filter_effect[i]
prefix_cost[i]     = prefix_cost[i-1] + read_cost[i] + row_evaluate_cost(prefix_rowcount[i-1] × rows_fetched[i])
```

搜索算法选择 `prefix_cost[N]`（总代价）最低的完整计划。

---

## 第十一章 端到端完整示例

### 建表与查询

```sql
CREATE TABLE orders (
  id INT PRIMARY KEY,
  customer_id INT,
  status VARCHAR(20),
  amount DECIMAL(10,2),
  created_at DATETIME,
  INDEX idx_customer (customer_id),
  INDEX idx_status_amount (status, amount),
  INDEX idx_created (created_at)
);

SELECT * FROM orders
WHERE customer_id = 100
  AND status = 'paid'
  AND amount > 500
  AND created_at > '2024-01-01';
```

### 阶段 1：条件规范化（optimize_cond）

四个简单比较条件，无等值传播的空间，条件保持原样。

### 阶段 2：Range Optimizer 构建范围树（get_mm_tree）

递归处理 AND 中的每个条件：

```
get_mm_tree("customer_id = 100") →
  SEL_TREE { keys[idx_customer] = SEL_ROOT([100, 100]) }

get_mm_tree("status = 'paid'") →
  SEL_TREE { keys[idx_status_amount] = SEL_ROOT(kp0: ['paid', 'paid']) }

get_mm_tree("amount > 500") →
  SEL_TREE { keys[idx_status_amount] = SEL_ROOT(kp1: (500, +∞)) }

get_mm_tree("created_at > '2024-01-01'") →
  SEL_TREE { keys[idx_created] = SEL_ROOT(('2024-01-01', +∞)) }
```

AND 合并后（`tree_and`）：

```
最终 SEL_TREE:
  keys[idx_customer]      = [100, 100]
  keys[idx_status_amount] = kp0: ['paid','paid'] → kp1: (500, +∞)
  keys[idx_created]       = ('2024-01-01', +∞)
```

### 阶段 3：评估各索引范围扫描代价（get_key_scans_params）

对每个非 NULL 索引调用 `check_quick_select()`：

| 索引 | 估算行数（InnoDB dive） | 代价 |
|------|---------|------|
| `idx_customer` | 50 | 51.1 |
| `idx_status_amount` | 20 | 21.1 |
| `idx_created` | 5000 | 2501.1 |

Range 最优选择：**`idx_status_amount`**（代价 21.1）

### 阶段 4：评估 ref 访问（find_best_ref）

通过 `Key_use` 列表评估：

| 索引 | 可用键部分 | fanout（rec_per_key） | read_cost |
|------|-----------|--------|-----------|
| `idx_customer`（kp0 = 100） | 1 | 50 | 25.5 |
| `idx_status_amount`（kp0 = 'paid'） | 1 | 200 | 101.1 |

ref 最优选择：**`idx_customer`**（fanout=50, cost=25.5）

> 注意：`amount > 500` 是范围条件，不产生 Key_use，所以 `idx_status_amount` 在 ref 阵营只能利用 kp0（status='paid'），无法利用 kp1（amount）。

### 阶段 5：best_access_path 最终对比

```
ref 总代价  = 25.5 + row_evaluate_cost(1 × 50)  = 25.5 + 5.0 = 30.5
range 总代价 = 21.1 + row_evaluate_cost(1 × 20) = 21.1 + 2.0 = 23.1
```

**最终选择：`idx_status_amount` 的范围扫描**（代价 23.1 < 30.5）

### 阶段 6：执行时索引定位

```
QUICK_RANGE: min_key=['paid', 500], flag=NEAR_MIN (amount > 500 的开区间)
→ start_key.flag = HA_READ_AFTER_KEY
→ InnoDB: 在 idx_status_amount 上定位到 (status='paid', amount > 500) 的第一条记录
→ 向后扫描直到 status 不再等于 'paid'
```

EXPLAIN 输出：

```
type: range, key: idx_status_amount, rows: 20, Extra: Using index condition
```

剩余条件 `customer_id = 100 AND created_at > '2024-01-01'` 作为 **filter condition** 在获取行后评估。

---

## 附录 C：子查询优化与去关联的能力边界

> **注：本附录为辅助内容，介绍 MySQL 在子查询条件优化方面的能力和局限。**

### C.1 MySQL 支持的子查询转换

MySQL 对 `IN` 子查询提供了多种优化策略：

| 策略 | 适用场景 | 效果 |
|------|---------|------|
| **Semi-join 转换** | `WHERE col IN (SELECT ...)` | 将子查询转换为 semi-join，参与正常的表排序优化 |
| **Subquery Materialization** | IN 子查询结果集较小 | 将子查询结果物化为临时表，用 hash 查找替代逐行执行 |
| **IN→EXISTS 转换** | 外表较小时 | 将 IN 转换为 EXISTS 相关子查询，利用内表索引 |

优化器会通过代价比较在这些策略中选择最优的一个。

### C.2 MySQL 的去关联局限

对于更复杂的相关子查询，MySQL 的去关联能力有限：

#### EXISTS + 聚合：无法去关联

```sql
-- 找出订单总额超过 10000 的客户
SELECT * FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders o
    WHERE o.customer_id = c.id
    GROUP BY o.customer_id
    HAVING SUM(o.amount) > 10000
);
```

- **SQL Server/Oracle**：自动去关联，转换为 semi-join + 派生表聚合，一次聚合 + 一次 JOIN
- **MySQL**：对 `customers` 表的每一行，都重新执行一次内部的 `GROUP BY + HAVING` 子查询。10 万行客户 = 10 万次子查询

**Workaround**：手动改写为 JOIN：

```sql
SELECT c.* FROM customers c
JOIN (
    SELECT customer_id FROM orders
    GROUP BY customer_id HAVING SUM(amount) > 10000
) o ON o.customer_id = c.id;
```

#### 标量相关子查询：无法去关联

```sql
-- 找出金额超过该客户平均订单金额的订单
SELECT * FROM orders o1
WHERE o1.amount > (
    SELECT AVG(o2.amount) FROM orders o2
    WHERE o2.customer_id = o1.customer_id
);
```

- **SQL Server**：去关联后转为 JOIN + 聚合派生表
- **MySQL**：对 `orders` 表中的每一行，都重新计算一次该客户的平均金额。100 万行订单 = 100 万次子查询

**Workaround**：手动改写：

```sql
SELECT o1.* FROM orders o1
JOIN (
    SELECT customer_id, AVG(amount) AS avg_amount
    FROM orders GROUP BY customer_id
) o2 ON o1.customer_id = o2.customer_id
WHERE o1.amount > o2.avg_amount;
```

#### NOT EXISTS / NOT IN 的反连接

MySQL 8.0 对 `NOT EXISTS` 和 `NOT IN` 支持 **anti-join** 转换（通过 `antijoin` 优化器开关控制），这是一个相对较新的改进。但对于含聚合的 NOT EXISTS 子查询，仍然无法去关联。

### C.3 子查询优化的实际影响

| 子查询类型 | MySQL 能力 | 性能影响 |
|-----------|-----------|---------|
| `IN (SELECT ...)` 简单子查询 | ✅ semi-join / 物化 / IN→EXISTS | 通常性能良好 |
| `EXISTS` 简单相关子查询 | ✅ 可利用内表索引 | 取决于外表行数和内表索引 |
| `EXISTS` + GROUP BY/HAVING | ❌ 逐行执行 | 外表行数多时性能灾难 |
| 标量相关子查询（含聚合） | ❌ 逐行执行 | 外表行数多时性能灾难 |
| `NOT IN` / `NOT EXISTS` 简单 | ✅ anti-join（8.0.17+） | 性能良好 |

---

## 附录 A：关键数据结构速查

> **注：以下数据结构在前文各章节中被引用，此处集中列出便于查阅。**

### POSITION — 记录一个表的最优访问方式

```cpp
// sql/sql_select.h:352
struct POSITION {
  double rows_fetched;      // 每次前缀组合获取的行数
  double read_cost;         // 访问代价（不含 row evaluate）
  float filter_effect;      // 非访问条件的过滤率 [0.0, 1.0]
  double prefix_rowcount;   // 累计输出行数（含之前所有表）
  double prefix_cost;       // 累计代价
  Key_use *key;             // ref 访问用的 Key_use（scan 为 nullptr）
  table_map ref_depend_map; // ref 依赖的表
  JOIN_TAB *table;          // 对应的表
  bool use_join_buffer;     // 是否使用 Join Buffer
};
```

### Key_use — 记录一个可用于 ref 访问的等值条件

```cpp
struct Key_use {
  TABLE_LIST *table_ref;    // 目标表
  Item *val;                // 等值右侧的值
  table_map used_tables;    // val 引用的表
  uint key;                 // 索引编号
  uint keypart;             // 键部分编号
  key_part_map bound_keyparts;  // 已绑定的键部分位图
  double fanout;                // 估算的匹配行数
  double read_cost;             // 估算的读取代价
};
```

### SEL_TREE / SEL_ARG / SEL_ROOT — Range Optimizer 的范围树

```cpp
class SEL_TREE {
  enum Type { IMPOSSIBLE, ALWAYS, KEY } type;
  Mem_root_array<SEL_ROOT *> keys;   // 每个候选索引一个 SEL_ROOT
};

class SEL_ROOT {
  enum class Type { IMPOSSIBLE, MAYBE_KEY, KEY_RANGE } type;
  SEL_ARG *root;                     // 指向红黑树根节点
};

class SEL_ARG {
  uint8 min_flag, max_flag;          // 范围边界标志
  uint8 part;                         // 索引键部分编号
  uchar *min_value, *max_value;      // 范围端点值
  SEL_ARG *next, *prev;              // 同层区间的 OR 链表
  SEL_ROOT *next_key_part;           // 下一键部分的 AND 连接
};
```

### QUICK_RANGE — 物化后的索引范围

```cpp
class QUICK_RANGE {
  uchar *min_key, *max_key;
  uint16 flag;                        // key_range_flags
  key_part_map min_keypart_map, max_keypart_map;
};
```

### AccessPath — 统一的访问路径描述

Range Optimizer 返回 `AccessPath` 对象来描述选定的范围扫描：

- `INDEX_RANGE_SCAN` — 单索引范围扫描
- `INDEX_MERGE` — Index Merge
- `ROWID_INTERSECTION` — ROR 交集
- `ROWID_UNION` — ROR 并集
- `GROUP_INDEX_SKIP_SCAN` — Group 松散索引扫描
- `INDEX_SKIP_SCAN` — 索引跳跃扫描

### key_range_flags — 范围标志位

```cpp
// include/my_base.h:1078-1122
enum key_range_flags {
  NO_MIN_RANGE      = 1 << 0,   // 无下界，从 -∞ 开始
  NO_MAX_RANGE      = 1 << 1,   // 无上界，到 +∞ 结束
  NEAR_MIN          = 1 << 2,   // 不含左端点（开区间）
  NEAR_MAX          = 1 << 3,   // 不含右端点（开区间）
  UNIQUE_RANGE      = 1 << 4,   // 唯一索引上的等值条件
  EQ_RANGE          = 1 << 5,   // 等值条件（min_key == max_key）
  NULL_RANGE        = 1 << 6,   // 含 NULL 的唯一范围
  GEOM_FLAG         = 1 << 7,   // R-tree 空间索引范围
  DESC_FLAG         = 1 << 10,  // 降序扫描
};
```

### ha_rkey_function — 存储引擎索引定位指令

```cpp
// include/my_base.h:78-93
enum ha_rkey_function {
  HA_READ_KEY_EXACT,       // = value
  HA_READ_KEY_OR_NEXT,     // >= value
  HA_READ_AFTER_KEY,       // > value
  HA_READ_BEFORE_KEY,      // < value（结束位置）
  HA_READ_KEY_OR_PREV,     // <= value
  HA_READ_PREFIX,          // 前缀匹配
  HA_READ_PREFIX_LAST,     // 前缀匹配的最后一条
};
```

---

## 附录 B：关键源码位置索引

| 功能 | 文件 | 函数 | 行号 |
|------|------|------|------|
| JOIN 优化入口 | `sql/sql_optimizer.cc` | `JOIN::optimize()` | 337 |
| 条件规范化 | `sql/sql_optimizer.cc` | `optimize_cond()` | 10338 |
| 等值传播 | `sql/sql_optimizer.cc` | `build_equal_items()` | 4468 |
| 常量传播 | `sql/sql_optimizer.cc` | `propagate_cond_constants()` | 4946 |
| 恒真/恒假消除 | `sql/sql_optimizer.cc` | `remove_eq_conds()` | 10454 |
| 常量折叠 | `sql/sql_const_folding.cc` | `fold_condition()` | — |
| 外连接转内连接 | `sql/sql_resolver.cc` | `simplify_joins()` | 768 |
| 表达式索引替换 | `sql/sql_optimizer.cc` | `gc_subst_transformer()` | 1217 |
| 分区裁剪 | `sql/sql_optimizer.cc` | `prune_partitions()` | 2826 |
| 公共子表达式提取 | `sql/join_optimizer/common_subexpression_elimination.cc` | `CommonSubexpressionElimination()` | — |
| 条件下推 | `sql/sql_optimizer.cc` | `make_cond_for_table()` | 9496 |
| 派生表条件下推 | `sql/sql_derived.cc` | `push_conditions_to_derived_tables()` | — |
| Index Condition Pushdown | `sql/sql_select.cc` | `push_index_cond()` | 2905 |
| Range Optimizer 入口 | `sql/range_optimizer/range_optimizer.cc` | `test_quick_select()` | 524 |
| 范围树构建 | `sql/range_optimizer/range_analysis.cc` | `get_mm_tree()` | 845 |
| 叶子节点构造 | `sql/range_optimizer/range_analysis.cc` | `get_mm_leaf()` | 1370 |
| 范围树合并 (AND) | `sql/range_optimizer/tree.cc` | `tree_and()` | 503 |
| 范围树合并 (OR) | `sql/range_optimizer/tree.cc` | `tree_or()` | 680 |
| 范围扫描代价估算 | `sql/range_optimizer/index_range_scan_plan.cc` | `check_quick_select()` | 568 |
| 范围物化 | `sql/range_optimizer/index_range_scan_plan.cc` | `get_ranges_from_tree()` | 774 |
| QUICK_RANGE 转 key_range | `sql/range_optimizer/range_optimizer.h` | `make_min_endpoint()` | 121 |
| 最优访问路径 | `sql/sql_planner.cc` | `best_access_path()` | 981 |
| ref 访问评估 | `sql/sql_planner.cc` | `find_best_ref()` | 207 |
| ref 代价计算 | `sql/sql_planner.cc` | `find_cost_for_ref()` | 143 |
| scan 代价计算 | `sql/sql_planner.cc` | `calculate_scan_cost()` | 770 |
| 条件过滤率 | `sql/sql_planner.cc` | `calculate_condition_filter()` | — |
| JOIN 顺序选择 | `sql/sql_planner.cc` | `choose_table_order()` | 1951 |
| 贪心搜索 | `sql/sql_planner.cc` | `greedy_search()` | 2328 |
| 深度搜索 | `sql/sql_planner.cc` | `best_extension_by_limited_search()` | 2719 |
| 代价模型 | `sql/opt_costmodel.h` | `Cost_model_server/Table` | 52/240 |
| 索引定位 | `sql/handler.cc` | `read_range_first()` | 7314 |
| InnoDB B+ 树定位 | `storage/innobase/handler/ha_innodb.cc` | `convert_search_mode_to_innobase()` | 10096 |
| SEL_TREE 定义 | `sql/range_optimizer/tree.h` | `SEL_TREE` | 871 |
| SEL_ARG 定义 | `sql/range_optimizer/tree.h` | `SEL_ARG` | 465 |
| key_range_flags | `include/my_base.h` | `enum key_range_flags` | 1078 |
| ha_rkey_function | `include/my_base.h` | `enum ha_rkey_function` | 78 |

---

## 附录 D：关键函数源码注释

> **注：本附录对附录 B 中列出的关键函数逐一给出精简后的核心代码及中文注释。重点标注难以理解的参数差异、调用时机、以及容易混淆的逻辑分支。数据结构/枚举定义已在附录 A 中列出，此处不再重复。**

---

### D.1 JOIN::optimize() — JOIN 优化总入口

```cpp
// sql/sql_optimizer.cc:337
// 整个 SELECT 优化的主驱动函数，在此依次调用各优化子阶段
bool JOIN::optimize(bool finalize_access_paths) {
  if (optimized) return false;          // 防止 EXPLAIN 时重复优化

  // ---- 阶段 0：派生表/视图递归优化 ----
  for (Table_ref *tl = query_block->leaf_tables; tl; tl = tl->next_leaf) {
    if (tl->is_view_or_derived()) {
      if (tl->optimize_derived(thd)) return true;
    }
  }

  // ---- 阶段 1：WHERE 条件优化 ----
  // ⚠️ 难点：optimize_cond 被调用两次，参数不同
  if (where_cond || query_block->outer_join) {
    // 第一次调用：处理 WHERE，传入 join_list（非 null）
    //   → 会执行等值传播 build_equal_items()
    //   → 会执行常量传播 propagate_cond_constants()
    //   → 会执行恒真/恒假消除 remove_eq_conds()
    if (optimize_cond(thd, &where_cond, &cond_equal,
                      &query_block->m_table_nest,    // ← WHERE 时传 join_list
                      &query_block->cond_value))
      return true;
    if (query_block->cond_value == Item::COND_FALSE) {
      zero_result_cause = "Impossible WHERE";
      create_access_paths_for_zero_rows();
      goto setup_subq_exit;
    }
  }
  if (having_cond) {
    // 第二次调用：处理 HAVING，join_list 传 nullptr
    //   → 跳过 build_equal_items()（HAVING 不做等值传播）
    //   → 仍执行常量传播和恒真/恒假消除
    if (optimize_cond(thd, &having_cond, &cond_equal,
                      nullptr,                        // ← HAVING 时传 nullptr
                      &query_block->having_value))
      return true;
    if (query_block->having_value == Item::COND_FALSE) {
      zero_result_cause = "Impossible HAVING";
      create_access_paths_for_zero_rows();
      goto setup_subq_exit;
    }
  }

  // ---- 阶段 2：外连接简化 ----
  if (query_block->outer_join) {
    if (query_block->simplify_joins(...)) return true;
  }

  // ---- 阶段 3：分区裁剪 ----
  if (query_block->partitioned_table_count) {
    if (prune_partitions(thd, ...)) return true;
  }

  // ---- 阶段 4：常量表检测与求值 ----
  // （将只有一行的表标记为 const table，提前读取）
  make_join_plan();  // 内部调用 best_access_path → choose_table_order 等

  // ---- 阶段 5：条件下推到各表 ----
  make_join_query_block(this, where_cond);  // 调用 make_cond_for_table()

  // ---- 阶段 6：Index Condition Pushdown ----
  // 为每个表调用 push_index_cond()

  // ---- 阶段 7：创建执行迭代器 ----
  create_access_paths();
  return false;
}
```

> **⚠️ 难点：`optimize_cond()` 两次调用的参数差异**
>
> | 调用场景 | `join_list` 参数 | 效果 |
> |---------|-----------------|------|
> | WHERE 条件 | `&query_block->m_table_nest`（非 null） | 执行等值传播 + 常量传播 + 恒真恒假消除 |
> | HAVING 条件 | `nullptr` | **跳过**等值传播，仅执行常量传播 + 恒真恒假消除 |
>
> 原因：HAVING 中的列引用的是聚合后的结果，不属于任何 join 表，做等值传播没有意义。

---

### D.2 optimize_cond() — 条件优化三步曲编排器

```cpp
// sql/sql_optimizer.cc:10338
// 依次调用三个子阶段对条件进行优化
bool optimize_cond(THD *thd, Item **cond, COND_EQUAL **cond_equal,
                   mem_root_deque<Table_ref *> *join_list,  // WHERE 时非 null，HAVING 时 null
                   Item::cond_result *cond_value) {
  // ---- 步骤 1：等值传播（仅 WHERE）----
  // ⚠️ join_list != nullptr 时才执行此步骤
  if (join_list) {
    // 将 a=b AND b=c 合并为 MULT_EQUAL(a,b,c)
    // 若含常量如 a=5，则替换 b→5, c→5
    if (build_equal_items(thd, *cond, cond, nullptr, true,
                          join_list, cond_equal))
      return true;
  }

  // ---- 步骤 2：常量传播（WHERE 和 HAVING 都执行）----
  if (*cond) {
    // 找到 field=const，把同一 AND 层内其他对 field 的引用替换为 const
    if (propagate_cond_constants(thd, nullptr, *cond, *cond))
      return true;
  }

  // ---- 步骤 3：恒真/恒假消除（WHERE 和 HAVING 都执行）----
  if (*cond) {
    // 移除 item=item 形式的恒等式
    // 折叠可在编译期求值的常量表达式
    if (remove_eq_conds(thd, *cond, cond, cond_value))
      return true;
  }
  return false;
}
```

> **⚠️ 难点：`cond_value` 的三个取值**
>
> | 值 | 含义 | 后续处理 |
> |----|------|---------|
> | `COND_OK` | 条件正常，不可在编译期确定 | 继续正常优化 |
> | `COND_TRUE` | 条件恒真，可移除 | `*cond` 被设为 nullptr |
> | `COND_FALSE` | 条件恒假，结果集为空 | 上层判断 "Impossible WHERE/HAVING" |

---

### D.3 build_equal_items() — 等值传播：构建多重等式

```cpp
// sql/sql_optimizer.cc:4468
// 将等值条件 (a=b, b=c) 转换为 MULT_EQUAL(a,b,c) 结构
bool build_equal_items(THD *thd, Item *cond, Item **retcond,
                       COND_EQUAL *inherited,   // 上层传下来的已知等式
                       bool do_inherit,          // 是否继承上层等式
                       mem_root_deque<Table_ref *> *join_list,
                       COND_EQUAL **cond_equal_ref) {
  COND_EQUAL *cond_equal = nullptr;

  if (cond) {
    // 核心：递归处理条件树，将 = 转换为 MULT_EQUAL
    if (build_equal_items_for_cond(thd, cond, &cond, inherited, do_inherit))
      return true;
    cond->update_used_tables();

    // 从处理后的条件中提取 COND_EQUAL 结构
    if (cond->type() == Item::COND_ITEM &&
        down_cast<Item_cond *>(cond)->functype() == Item_func::COND_AND_FUNC)
      cond_equal = &down_cast<Item_cond_and *>(cond)->cond_equal;
    else if (cond->type() == Item::FUNC_ITEM &&
             down_cast<Item_func *>(cond)->functype() == Item_func::MULT_EQUAL_FUNC) {
      cond_equal = new (thd->mem_root) COND_EQUAL;
      cond_equal->current_level.push_back(down_cast<Item_equal *>(cond));
    }
  }
  // 建立层次关系：当前层的 COND_EQUAL 指向上层
  if (cond_equal) {
    cond_equal->upper_levels = inherited;
    inherited = cond_equal;
  }
  *cond_equal_ref = cond_equal;

  // ⚠️ 难点：递归处理 JOIN ON 条件
  // 对 join_list 中每个表的 join_cond（ON 条件）也做等值传播
  // inherited 参数使得 ON 条件能继承 WHERE 中已知的等式
  if (join_list) {
    for (Table_ref *table : *join_list) {
      if (table->join_cond_optim()) {
        mem_root_deque<Table_ref *> *nested =
            table->nested_join ? &table->nested_join->m_tables : nullptr;
        Item *join_cond;
        if (build_equal_items(thd, table->join_cond_optim(), &join_cond,
                              inherited,   // ← WHERE 层的等式传递给 ON 条件
                              do_inherit, nested, &table->cond_equal))
          return true;
        table->set_join_cond_optim(join_cond);
      }
    }
  }
  *retcond = cond;
  return false;
}
```

> **⚠️ 难点：`inherited` 参数的层级传递**
>
> `inherited` 形成一条链表：WHERE 层 → 外层 ON → 内层 ON。这使得内层 ON 条件能利用 WHERE 中已知的等式（例如 WHERE 中有 `a=5`，则 ON 中的 `a` 也会被替换为 `5`）。但反过来不行——ON 条件的等式不会上传到 WHERE 层。

---

### D.4 propagate_cond_constants() — 常量传播

```cpp
// sql/sql_optimizer.cc:4946
// 在 AND 组内找到 field=const，将同层其他对 field 的引用替换为 const
static bool propagate_cond_constants(THD *thd,
                                     I_List<COND_CMP> *save_list,
                                     Item *and_father,  // 所属的 AND 节点
                                     Item *cond) {       // 当前处理的条件
  if (cond->type() == Item::COND_ITEM) {
    Item_cond *item_cond = down_cast<Item_cond *>(cond);
    bool and_level = item_cond->functype() == Item_func::COND_AND_FUNC;
    List_iterator_fast<Item> li(*item_cond->argument_list());
    Item *item;
    I_List<COND_CMP> save;
    while ((item = li++)) {
      // ⚠️ 难点：递归时 and_father 参数的传递规则
      // - 若当前是 AND 节点：传 cond 本身作为 and_father
      // - 若当前是 OR 节点：传 item（子节点）作为 and_father
      // 目的：只在同一个 AND 层内做替换，不跨 OR 边界
      if (propagate_cond_constants(thd, &save,
                                   and_level ? cond : item, item))
        return true;
    }
    if (and_level) {
      // AND 层处理完毕后，对收集到的 field=const 对做跨项替换
      I_List_iterator<COND_CMP> cond_itr(save);
      COND_CMP *cond_cmp;
      while ((cond_cmp = cond_itr++)) {
        Item **args = cond_cmp->cmp_func->arguments();
        if (!args[0]->const_item())
          change_cond_ref_to_const(thd, &save, cond_cmp->and_level,
                                   cond_cmp->and_level, args[0], args[1]);
      }
    }
  }
  // 叶子节点：找到 field = const 形式的等式
  else if (and_father != cond &&                      // 不是 AND 层本身
           cond->marker != Item::MARKER_CONST_PROPAG) // 未被处理过
  {
    Item_func *func;
    if (cond->type() == Item::FUNC_ITEM &&
        (func = down_cast<Item_func *>(cond)) &&
        (func->functype() == Item_func::EQ_FUNC ||
         func->functype() == Item_func::EQUAL_FUNC)) {
      Item **args = func->arguments();
      bool left_const = args[0]->const_item();
      bool right_const = args[1]->const_item();
      // 一边是常量，一边不是 → 可以传播
      if (!(left_const && right_const) &&
          args[0]->result_type() == args[1]->result_type()) {
        if (right_const) {
          // 将右侧常量传播到同层其他引用左侧字段的位置
          change_cond_ref_to_const(thd, save_list, and_father, and_father,
                                   args[0], args[1]);
        } else if (left_const) {
          change_cond_ref_to_const(thd, save_list, and_father, and_father,
                                   args[1], args[0]);
        }
      }
    }
  }
  return false;
}
```

> **⚠️ 难点：`and_father` 与 `cond` 的关系**
>
> ```
> WHERE a=5 AND b=a AND (c=1 OR d=2)
>       ↑          ↑         ↑
>    and_father = WHERE_AND   |
>                             在 OR 内部递归时，and_father = OR 节点
>                             所以 c=1 不会被传播到 OR 外面
> ```
> `and_father == cond` 表示当前节点就是 AND 层本身（不是其子项），此时不做处理。
> `and_father != cond` 表示当前是某个 AND 层的子项，可以收集等式或做替换。

---

### D.5 remove_eq_conds() — 恒真/恒假条件消除

```cpp
// sql/sql_optimizer.cc:10454
// 递归遍历条件树，移除恒真/恒假子条件，折叠常量表达式
bool remove_eq_conds(THD *thd, Item *cond, Item **retcond,
                     Item::cond_result *cond_value) {
  if (cond->type() == Item::COND_ITEM) {
    Item_cond *item_cond = down_cast<Item_cond *>(cond);
    bool and_level = item_cond->functype() == Item_func::COND_AND_FUNC;
    List_iterator<Item> li(*item_cond->argument_list());
    Item *item;
    *cond_value = Item::COND_UNDEF;

    while ((item = li++)) {
      Item *new_item;
      Item::cond_result tmp_cond_value;
      if (remove_eq_conds(thd, item, &new_item, &tmp_cond_value))
        return true;

      if (new_item == nullptr)
        li.remove();                         // 子条件被消除
      else if (item != new_item)
        li.replace(new_item);                // 子条件被替换

      // ⚠️ 难点：AND 与 OR 的短路逻辑完全相反
      switch (tmp_cond_value) {
        case Item::COND_FALSE:
          if (and_level) {
            // AND 中有一个 FALSE → 整体 FALSE（短路）
            *cond_value = Item::COND_FALSE;
            *retcond = nullptr;
            return false;
          }
          break;  // OR 中有一个 FALSE → 仅移除该分支
        case Item::COND_TRUE:
          if (!and_level) {
            // OR 中有一个 TRUE → 整体 TRUE（短路）
            *cond_value = Item::COND_TRUE;
            *retcond = nullptr;
            return false;
          }
          break;  // AND 中有一个 TRUE → 仅移除该分支
        case Item::COND_OK:
          if (and_level || *cond_value == Item::COND_FALSE)
            *cond_value = tmp_cond_value;
          break;
      }
    }
    // AND/OR 只剩一个子项 → 提升该子项
    if (item_cond->argument_list()->elements == 1) {
      *retcond = item_cond->argument_list()->head();
      return false;
    }
  }
  // 叶子节点：尝试编译期求值
  else if (can_evaluate_condition(thd, cond)) {
    bool value;
    if (eval_const_cond(thd, cond, &value)) return true;
    *cond_value = value ? Item::COND_TRUE : Item::COND_FALSE;
    *retcond = nullptr;  // 常量条件被消除
    return false;
  }
  // 比较函数：检查 left_item == right_item 的特殊情况
  else {
    *cond_value = cond->eq_cmp_result();
    // 两个相同的非 NULL 项比较 → 静态可知结果
    Item *left_item = down_cast<Item_func *>(cond)->arguments()[0];
    Item *right_item = down_cast<Item_func *>(cond)->arguments()[1];
    if (left_item->eq(right_item, true) && !cond->is_non_deterministic()) {
      if (!left_item->is_nullable() ||
          *cond_value == Item::COND_FALSE ||
          down_cast<Item_func *>(cond)->functype() == Item_func::EQUAL_FUNC) {
        *retcond = nullptr;
        return false;
      }
    }
  }
  // 最后尝试常量折叠
  return fold_condition_exec(thd, cond, retcond, cond_value);
}
```

> **⚠️ 难点：AND vs OR 的短路规则**
>
> | 子条件结果 | 在 AND 中的效果 | 在 OR 中的效果 |
> |-----------|----------------|---------------|
> | `COND_FALSE` | 整体立即返回 FALSE | 仅移除该分支 |
> | `COND_TRUE` | 仅移除该分支 | 整体立即返回 TRUE |
> | `COND_OK` | 保留 | 保留 |

---

### D.6 fold_condition() — 常量折叠（编译期比较求值）

```cpp
// sql/sql_const_folding.cc:1255
// 当比较的一侧是字段、另一侧是常量时，尝试在编译期确定比较结果
bool fold_condition(THD *thd, Item *cond, Item **retcond,
                    Item::cond_result *cond_value,
                    bool manifest_result) {  // 是否将结果物化为 Item_func_true/false
  *cond_value = Item::COND_OK;
  *retcond = cond;

  if (cond->type() == Item::COND_ITEM) {
    // AND/OR 节点：递归折叠子项
    return fold_arguments(thd, down_cast<Item_cond *>(cond));
  }

  Item_func *func = down_cast<Item_func *>(cond);
  switch (func->functype()) {
    case Item_func::ISNOTNULL_FUNC:
      if (!func->arguments()[0]->is_nullable()) {
        // NOT NULL 列的 IS NOT NULL → 恒真
        if (manifest_result) {
          thd->change_item_tree(retcond, new Item_func_true());
        } else {
          *cond_value = Item::COND_TRUE;
          *retcond = nullptr;
        }
        return false;
      }
      return fold_arguments(thd, func);

    case Item_func::EQ_FUNC:
    case Item_func::NE_FUNC:
    case Item_func::LT_FUNC: case Item_func::LE_FUNC:
    case Item_func::GE_FUNC: case Item_func::GT_FUNC:
    case Item_func::EQUAL_FUNC:
    case Item_func::MULT_EQUAL_FUNC:
      break;  // 进入下面的 field vs constant 分析
    default:
      return fold_arguments(thd, func);
  }

  // ⚠️ 核心：检查是否满足 field OP constant 的形式
  Item **args = func->arguments();
  bool seen_field = false, seen_constant = false;
  for (int i = 0; i < 2; ++i) {
    if (args[i]->real_item()->type() == Item::FIELD_ITEM)
      seen_field = true;
    else if (args[i]->const_for_execution() &&
             args[i]->type() != Item::SUBSELECT_ITEM)
      seen_constant = true;
  }

  if (!(seen_field && seen_constant))
    return fold_arguments(thd, func);  // 不满足条件，递归折叠参数

  // 分析常量相对于字段类型值域的位置
  // place 可能取值：OUTSIDE_HIGH, OUTSIDE_LOW, INSIDE, ON_MIN, ON_MAX 等
  Range_placement place;
  analyze_field_constant(thd, field_item, &constant, func,
                         left_has_field, &place, ...);

  // 根据 place 和比较类型决定是否可以折叠
  // 例如：tinyint 字段 < 300 → 恒真（300 超出 tinyint 范围）
  //       tinyint 字段 = 300 → 恒假
  switch (func_type) {
    case Item_func::EQ_FUNC:
      if (place == RP_OUTSIDE_HIGH || place == RP_OUTSIDE_LOW)
        fold_or_simplify(thd, ..., false, ...);  // → COND_FALSE
      break;
    case Item_func::LT_FUNC:
      if (place == RP_OUTSIDE_HIGH)
        fold_or_simplify(thd, ..., true, ...);   // → COND_TRUE
      break;
    // ... 其他比较类型类似
  }
  return false;
}
```

> **⚠️ 难点：`manifest_result` 参数**
>
> | 值 | 含义 | 使用场景 |
> |----|------|---------|
> | `true` | 将结果替换为 `Item_func_true()` / `Item_func_false()` | 条件在树中间被引用，需要保留 Item 节点 |
> | `false` | 设 `*retcond = nullptr`，由调用者移除 | 条件在 AND/OR 列表中，可以直接摘除 |

---

### D.7 simplify_joins() — 外连接转内连接

```cpp
// sql/sql_resolver.cc:1919
// 分析 WHERE/ON 中的 null-rejecting 谓词，将可简化的外连接转为内连接
bool Query_block::simplify_joins(
    THD *thd,
    mem_root_deque<Table_ref *> *join_list,
    bool top,      // 是否在最外层（WHERE 层）
    bool in_sj,    // 是否在 semi-join 嵌套内
    Item **cond,   // 当前层的过滤条件
    uint *changelog) {

  for (Table_ref *table : *join_list) {
    table_map used_tables, not_null_tables = 0;

    if (table->nested_join != nullptr) {
      // ⚠️ 难点：两次递归调用，参数不同
      if (table->join_cond() != nullptr) {
        Item *join_cond = table->join_cond();
        // 递归 1：用 ON 条件去 "对抗" 嵌套内的表
        //   top = false（因为 ON 不是 WHERE）
        //   cond = &join_cond（传 ON 条件）
        simplify_joins(thd, &nested_join->m_tables,
                       false,  // ← top=false
                       in_sj || table->is_sj_or_aj_nest(),
                       &join_cond, changelog);
      }
      // 递归 2：用外层条件去 "对抗" 嵌套内的表
      //   top 保持不变（如果外层是 WHERE 这里仍是 true）
      //   cond = 外层传入的 cond
      simplify_joins(thd, &nested_join->m_tables,
                     top,   // ← 保持原值
                     in_sj || table->is_sj_or_aj_nest(),
                     cond, changelog);

      used_tables = nested_join->used_tables;
      not_null_tables = nested_join->not_null_tables;
    } else {
      used_tables = table->map();
      if (*cond) not_null_tables = (*cond)->not_null_tables();
    }

    // 核心判断：内表列在 not_null_tables 中 → 外连接可转内连接
    if (!table->outer_join || (used_tables & not_null_tables)) {
      if (table->outer_join) {
        *changelog |= OUTER_JOIN_TO_INNER;
        table->outer_join = false;       // 转为内连接
      }
      if (table->join_cond()) {
        *changelog |= JOIN_COND_TO_WHERE;
        // ON 条件合并到 WHERE（或上层 ON）
        *cond = and_conds(*cond, table->join_cond());
        table->set_join_cond(nullptr);   // 清除 ON 条件
      }
    }

    // ⚠️ 只有 top=true（WHERE 层）才设置 dep_tables
    if (!top) continue;
    if (table->join_cond())
      table->dep_tables |= table->join_cond()->used_tables();
  }
  return false;
}
```

> **⚠️ 难点：`top` 参数的含义与递归变化**
>
> | 调用位置 | `top` 值 | 含义 |
> |---------|---------|------|
> | 从 `JOIN::optimize()` 首次调用 | `true` | 正在处理 WHERE 层 |
> | 用 ON 条件递归进嵌套 join | `false` | ON 不是 WHERE，不设置 dep_tables |
> | 用外层条件递归进嵌套 join | 继承外层值 | 如果外层是 WHERE，仍是 true |
>
> `top=true` 时的额外操作：设置 `table->dep_tables`（表依赖关系），用于后续 JOIN 排序。

---

### D.8 gc_subst_transformer() — 生成列索引替换

```cpp
// sql/item_func.cc:1300
// 检查表达式是否匹配某个生成列的定义，如果匹配则替换为对该生成列的引用
// 使得原本无法使用索引的表达式能通过生成列索引来加速
Item *Item_func::gc_subst_transformer(uchar *arg) {
  List<Field> *gc_fields = pointer_cast<List<Field> *>(arg);

  // 辅助 lambda：检查是否为常量或外部引用
  auto is_const_or_outer_reference = [](const Item *item) {
    return ((item->used_tables() & ~(OUTER_REF_TABLE_BIT | INNER_TABLE_BIT)) == 0);
  };

  switch (functype()) {
    // ⚠️ 情况 1：简单比较 (=, <, <=, >=, >)
    // 要求形式：<expr> OP <constant>，其中 <expr> 可被替换为生成列
    case EQ_FUNC: case LT_FUNC: case LE_FUNC: case GE_FUNC: case GT_FUNC: {
      Item **func = nullptr;
      Item *val = nullptr;
      if (args[0]->can_be_substituted_for_gc() &&
          is_const_or_outer_reference(args[1])) {
        func = args; val = args[1];           // 左侧是表达式，右侧是常量
      } else if (args[1]->can_be_substituted_for_gc() &&
                 is_const_or_outer_reference(args[0])) {
        func = args + 1; val = args[0];       // 右侧是表达式，左侧是常量
      } else break;

      // 在生成列列表中查找匹配的列，替换 func 指向的表达式
      substitute_gc_expression(func, nullptr, gc_fields, val->result_type(), this);
      break;
    }

    // ⚠️ 情况 2：BETWEEN / IN
    // 要求：args[0] 可被替换，args[1..n] 全部是相同类型的常量
    case BETWEEN: case IN_FUNC: {
      if (!args[0]->can_be_substituted_for_gc()) break;
      Item_result type = args[1]->result_type();
      // 检查所有右侧操作数类型一致且都是常量
      if (!std::all_of(args + 1, args + arg_count,
              [type, is_const_or_outer_reference](const Item *item) {
                return is_const_or_outer_reference(item) &&
                       item->result_type() == type;
              }))
        break;
      substitute_gc_expression(args, nullptr, gc_fields, type, this);
      break;
    }

    // ⚠️ 情况 3：MEMBER OF（JSON 多值索引）
    case MEMBER_OF_FUNC: {
      if (args[0]->const_for_execution() && !args[0]->is_null() &&
          args[1]->can_be_substituted_for_gc(/*array=*/true))
        substitute_gc_expression(args + 1, args, gc_fields,
                                 args[0]->result_type(), this);
      break;
    }

    // 情况 4、5：JSON_CONTAINS / JSON_OVERLAPS 类似处理
    default: break;
  }
  return this;
}
```

> **⚠️ 难点：什么样的表达式能被替换？**
>
> 例如表上有虚拟生成列 `gc_col AS (col_a + col_b)` 并建了索引，查询 `WHERE col_a + col_b > 10` 中的 `col_a + col_b` 会被替换为 `gc_col`，从而能使用该索引。`can_be_substituted_for_gc()` 检查表达式是否与某个生成列的定义表达式结构匹配。

---

### D.9 prune_partitions() — 分区裁剪

```cpp
// sql/range_optimizer/partition_pruning.cc:251
// 利用 WHERE 条件裁剪不可能包含匹配行的分区
bool prune_partitions(THD *thd, TABLE *table, Query_block *query_block,
                      Item *pprune_cond) {
  partition_info *part_info = table->part_info;

  if (part_info && part_info->is_pruning_completed) return false;  // prepare 阶段已完成
  table->all_partitions_pruned_away = false;
  if (!part_info) return false;           // 非分区表
  if (!pprune_cond) {                     // 无条件 → 所有分区都需要
    mark_all_partitions_as_used(part_info);
    return false;
  }

  // ⚠️ 难点：复用 Range Optimizer 的基础设施来分析分区键
  // 构造一个"虚拟索引"，key_parts 对应分区键列
  PART_PRUNE_PARAM prune_param;
  RANGE_OPT_PARAM *range_par = &prune_param.range_param;
  range_par->keys = 1;                   // 只有一个"虚拟索引"
  range_par->using_real_indexes = false;  // 不是真实索引

  // 用 get_mm_tree() 对分区键构建范围树
  // ⚠️ 与 test_quick_select() 中调用 get_mm_tree() 的区别：
  //   - using_real_indexes = false（分区裁剪用虚拟索引）
  //   - 不需要 remove_jump_scans（分区裁剪不关心跳跃扫描）
  SEL_TREE *tree = get_mm_tree(thd, range_par,
                               prev_tables, read_tables, current_table,
                               /*remove_jump_scans=*/false, pprune_cond);
  if (!tree) goto all_used;

  if (tree->type == SEL_TREE::IMPOSSIBLE) {
    part_info->is_pruning_completed = true;  // 没有分区匹配
    goto end;
  }
  if (tree->type != SEL_TREE::KEY) goto all_used;

  bitmap_clear_all(&part_info->read_partitions);

  if (tree->merges.is_empty()) {
    // 单一范围列表 → 直接遍历范围找匹配分区
    find_used_partitions(thd, &prune_param, tree->keys[0]);
  } else if (tree->merges.elements == 1) {
    // OR 产生的 imerge → 分别对每个分支找分区再取并集
    // 例如：partition_col=1 OR subpartition_col=2
    find_used_partitions_imerge(thd, &prune_param, tree->merges.head());
  } else {
    // 多个 imerge 的 AND → 分别处理再取交集
    find_used_partitions_imerge_list(thd, &prune_param, tree->merges);
  }

  // 判断是否需要在 execute 阶段再次裁剪
  if (pprune_cond->const_item() || !pprune_cond->const_for_execution() ||
      thd->lex->is_query_tables_locked())
    part_info->is_pruning_completed = true;
  // ...
}
```

> **⚠️ 难点：分区裁剪复用 Range Optimizer**
>
> 分区裁剪并不走真正的索引路径，而是构造一个 `using_real_indexes=false` 的虚拟索引，其 key_parts 对应分区键列。然后调用 `get_mm_tree()` 构建范围树，再用 `find_used_partitions()` 遍历范围树确定哪些分区可能包含数据。这就是为什么 `get_mm_tree()` 的 `param->using_real_indexes` 需要被检查的原因。

---

### D.10 CommonSubexpressionElimination() — 公共子表达式提取

```cpp
// sql/join_optimizer/common_subexpression_elimination.cc:178
// 从 OR 表达式中提取公共子条件
// 例如：(A AND B) OR (A AND C) → A AND (B OR C)
Item *CommonSubexpressionElimination(Item *cond) {
  if (!IsOr(cond)) return cond;  // 非 OR 表达式，无需处理

  Item_cond_or *or_item = down_cast<Item_cond_or *>(cond);
  if (or_item->argument_list()->is_empty())
    return new Item_func_false;  // 空 OR = FALSE

  // 取第一个 OR 分支中的所有 AND 子项作为候选
  List<Item> common_items;
  Item *first_group = or_item->argument_list()->head();

  if (IsAnd(first_group)) {
    // 第一个分支是 AND：检查其每个子项是否在所有其他分支中都出现
    Item_cond_and *and_group = down_cast<Item_cond_and *>(first_group);
    for (Item &and_arg : *and_group->argument_list()) {
      if (AlwaysPresent(or_item, &and_arg))  // 在每个 OR 分支中都存在？
        common_items.push_back(&and_arg);
    }
  } else {
    // 第一个分支是单个条件
    if (AlwaysPresent(or_item, first_group))
      common_items.push_back(first_group);
  }

  if (common_items.is_empty()) return cond;  // 无公共项

  // 从每个 OR 分支中移除公共项，构建剩余部分
  Item *remainder = OrGroupWithSomeRemoved(or_item, common_items);
  if (remainder != nullptr)
    common_items.push_back(remainder);  // 剩余部分作为新的 OR 条件

  // 返回 AND(公共项..., 剩余OR)
  return CreateConjunction(&common_items);
}
```

> **⚠️ 难点：此函数仅用于 Hypergraph 优化器**
>
> `CommonSubexpressionElimination()` 位于 `join_optimizer/` 目录下，是 MySQL 8.0 新增的 Hypergraph 优化器的一部分。传统优化器不调用此函数。传统优化器中，OR 条件的公共子表达式提取能力有限，主要依赖 `remove_eq_conds()` 中的简单折叠。

---

### D.11 make_cond_for_table() — 条件下推到单表

```cpp
// sql/sql_optimizer.cc:9496
// 从全局 WHERE 条件中提取属于特定表的子条件
Item *make_cond_for_table(THD *thd, Item *cond, table_map tables,
                          table_map used_table,
                          bool exclude_expensive_cond) {
  // ⚠️ 难点：used_table 为 0 和非 0 时行为完全不同
  //
  // used_table != 0：提取"恰好引用了 used_table"的条件
  //   → 用于为特定表生成过滤条件
  //
  // used_table == 0：提取"所有已可用表（tables）能求值"的条件
  //   → 用于提取可以提前求值的条件（如常量条件）

  // 跳过条件的三个条件（used_table != 0 时）：
  // 1. 正在为特定表提取条件
  // 2. 该条件不引用目标表
  // 3. 除非是昂贵的常量条件且 used_table == tables（第一个表）
  if (used_table &&                                     // 1
      !(cond->used_tables() & used_table) &&            // 2
      !(cond->is_expensive() && used_table == tables))  // 3
    return nullptr;

  if (cond->type() == Item::COND_ITEM) {
    if (((Item_cond *)cond)->functype() == Item_func::COND_AND_FUNC) {
      // AND：递归提取每个子项，组成新的 AND
      Item_cond_and *new_cond = new Item_cond_and;
      List_iterator<Item> li(*((Item_cond *)cond)->argument_list());
      Item *item;
      while ((item = li++)) {
        Item *fix = make_cond_for_table(thd, item, tables, used_table,
                                        exclude_expensive_cond);
        if (fix) new_cond->argument_list()->push_back(fix);
      }
      switch (new_cond->argument_list()->elements) {
        case 0: return nullptr;                          // 无匹配项
        case 1: return new_cond->argument_list()->head(); // 只有一项，提升
        default:
          new_cond->fix_fields(thd, nullptr);
          return new_cond;
      }
    } else {
      // ⚠️ OR：必须所有分支都能提取，否则整个 OR 不可用
      Item_cond_or *new_cond = new Item_cond_or;
      List_iterator<Item> li(*((Item_cond *)cond)->argument_list());
      Item *item;
      while ((item = li++)) {
        // 注意：OR 分支递归时 used_table 传 0
        // 因为 OR 的每个分支只要能在 tables 范围内求值即可
        Item *fix = make_cond_for_table(thd, item, tables, table_map(0),
                                        exclude_expensive_cond);
        if (!fix) return nullptr;  // 有一个分支不可提取 → 整个 OR 放弃
        new_cond->argument_list()->push_back(fix);
      }
      new_cond->fix_fields(thd, nullptr);
      return new_cond;
    }
  }

  // 叶子条件：检查是否所有引用的表都已可用
  if ((cond->used_tables() & ~tables) ||                                // 引用了不可用的表
      (!used_table && exclude_expensive_cond && cond->is_expensive()))  // 延迟昂贵条件
    return nullptr;

  return cond;
}
```

> **⚠️ 难点：AND 与 OR 递归时 `used_table` 参数的差异**
>
> | 节点类型 | 递归传入的 `used_table` | 原因 |
> |---------|----------------------|------|
> | AND 子项 | 保持原值（`used_table`） | AND 的每个子项独立，只提取引用目标表的 |
> | OR 子项 | 强制为 `0` | OR 要求所有分支都可求值，不限定特定表 |
>
> 这意味着 `WHERE t1.a=1 AND (t1.b=2 OR t2.c=3)` 在为 t1 提取条件时：
> - `t1.a=1` 被提取（AND 子项，引用 t1）
> - `(t1.b=2 OR t2.c=3)` 不被提取（OR 中 `t2.c=3` 引用了不可用的 t2）

---

### D.12 push_conditions_to_derived_tables() — 派生表条件下推

```cpp
// sql/sql_resolver.cc:620
// 将 WHERE 条件下推到物化派生表（子查询/视图）内部
bool Query_block::push_conditions_to_derived_tables(THD *thd) {
  // 仅当存在物化派生表时才处理
  if (materialized_derived_table_count > 0)
    for (Table_ref *tl = leaf_tables; tl; tl = tl->next_leaf) {
      if (tl->is_view_or_derived() &&          // 是派生表/视图
          tl->uses_materialization() &&          // 使用物化策略
          where_cond() &&                        // 有 WHERE 条件
          tl->can_push_condition_to_derived(thd)) {  // 可以下推
        Item **where = where_cond_ref();
        Condition_pushdown cp(*where, tl, thd, trace);
        // 构造可下推的条件
        if (cp.make_cond_for_derived()) return true;
        // 不可下推的部分留在外层 WHERE
        *where = cp.get_remainder_cond();
      }
    }

  // ⚠️ 难点：自顶向下递归
  // 先处理当前层，再递归处理内层子查询
  // 这样外层下推的条件到了内层后，可以继续被下推到更内层的派生表
  for (Query_expression *unit = first_inner_query_expression(); unit;
       unit = unit->next_query_expression()) {
    for (Query_block *sl = unit->first_query_block(); sl;
         sl = sl->next_query_block()) {
      if (sl->push_conditions_to_derived_tables(thd)) return true;
    }
  }
  return false;
}
```

> **⚠️ 难点：`can_push_condition_to_derived()` 的限制**
>
> 不是所有条件都能下推到派生表。以下情况不可下推：
> - 派生表含 `UNION`（条件不能穿过 UNION）
> - 派生表含 `LIMIT`（下推会改变语义）
> - 派生表含窗口函数（条件作用在窗口计算之前会改变结果）
> - 条件引用了派生表外部的列（相关子查询）

---

### D.13 push_index_cond() — Index Condition Pushdown (ICP)

```cpp
// sql/sql_select.cc:2905
// 将部分 WHERE 条件下推到存储引擎的索引扫描层
void QEP_TAB::push_index_cond(const JOIN_TAB *join_tab, uint keyno,
                               Opt_trace_object *trace_obj) {
  TABLE *const tbl = table();

  // ---- 前置检查：不适用 ICP 的情况 ----
  if (join_tab->reversed_access) return;           // 反向扫描不支持
  if (tbl->s->db_type() == innodb_hton &&
      tbl->s->tmp_table != NO_TMP_TABLE) return;  // InnoDB 内部临时表性能差
  if (tbl->vfield &&
      tbl->index_contains_some_virtual_gcol(keyno)) return;  // 虚拟列不支持

  // ⚠️ 难点：other_tbls_ok 决定是否允许引用其他表的列
  // 当使用 BNL join buffer 时，不允许引用其他表（因为 ICP 在引擎层执行）
  const bool other_tbls_ok =
      !((type() == JT_ALL || type() == JT_INDEX_SCAN ||
         type() == JT_RANGE || type() == JT_INDEX_MERGE) &&
        join_tab->use_join_cache() == JOIN_CACHE::ALG_BNL);

  // 7 个必要条件全部满足才能做 ICP
  if (condition() &&                                                    // 0. 有条件
      tbl->file->index_flags(keyno, 0, true) & HA_DO_INDEX_COND_PUSHDOWN && // 1. 引擎支持
      hint_key_state(thd, ..., ICP_HINT_ENUM, ...) &&                  // 2. 未被 hint 禁用
      thd->lex->sql_command != SQLCOM_UPDATE_MULTI &&                  // 3. 非多表更新
      !has_guarded_conds() &&                                           // 4. 非 NULL key 守卫
      type() != JT_CONST && type() != JT_SYSTEM &&                    // 5. 非 const 表
      !(keyno == tbl->s->primary_key &&
        tbl->file->primary_key_is_clustered()))                        // 6. 非聚簇主键
  {
    // 从完整条件中提取可用当前索引列求值的部分
    Item *idx_cond = make_cond_for_index(condition(), tbl, keyno, other_tbls_ok);

    if (idx_cond) {
      // 确认提取的条件确实引用了当前表的索引列
      idx_cond->update_used_tables();
      if ((idx_cond->used_tables() & table_ref->map()) == 0)
        return;  // 条件不引用本表索引列，放弃

      Item *idx_remainder_cond = nullptr;

      // ⚠️ 难点：BKA 场景下不推送，只记录
      if (join_tab->use_join_cache() && other_tbls_ok &&
          (idx_cond->used_tables() &
           ~(table_ref->map() | join_->const_table_map))) {
        idx_remainder_cond = idx_cond;  // BKA 时条件留在 server 层
      } else {
        // 核心：调用存储引擎的 idx_cond_push()
        // 引擎返回不能处理的剩余条件
        idx_remainder_cond = tbl->file->idx_cond_push(keyno, idx_cond);
      }

      if (idx_remainder_cond != idx_cond)
        ref().disable_cache = true;  // 推送成功，禁用 eq_ref 缓存

      // 将剩余条件（引擎不能处理的 + 非索引列条件）合并为新的表条件
      Item *row_cond = make_cond_remainder(condition(), true);
      if (row_cond) {
        and_conditions(&row_cond, idx_remainder_cond);
        idx_remainder_cond = row_cond;
      }
      set_condition(idx_remainder_cond);
    }
  }
}
```

> **⚠️ 难点：条件被拆分为三部分**
>
> ```
> 原始 WHERE 条件
>   ├── idx_cond：可用索引列求值 → 推送给存储引擎（ICP）
>   │     └── idx_remainder_cond：引擎不能处理的部分 → 留在 server 层
>   └── row_cond：不涉及索引列 → 必须在 server 层求值
>
> 最终 set_condition(row_cond AND idx_remainder_cond)
> ```
> 条件 6（非聚簇主键）的原因：聚簇索引的叶子节点就是完整行数据，ICP 没有意义（不能减少回表）。

---

### D.14 test_quick_select() — Range Optimizer 入口

```cpp
// sql/range_optimizer/range_optimizer.cc:524
// Range Optimizer 的总入口：评估是否值得使用范围扫描
int test_quick_select(THD *thd, MEM_ROOT *return_mem_root,
                      MEM_ROOT *temp_mem_root,
                      Key_map keys_to_use,
                      table_map prev_tables,   // join 序列中在本表之前的表
                      table_map read_tables,   // 已经读取过的表
                      ha_rows limit,
                      bool force_quick_range,
                      const enum_order interesting_order,
                      TABLE *table,
                      bool skip_records_in_range,
                      Item *cond,
                      Key_map *needed_reg,
                      bool ignore_table_scan,
                      Query_block *query_block,
                      AccessPath **path) {     // 输出：最优范围扫描路径
  *path = nullptr;
  if (keys_to_use.is_clear_all()) return 0;

  // ---- 基准：全表扫描代价 ----
  ha_rows records = table->file->stats.records;
  Cost_estimate cost_est = table->file->table_scan_cost();
  cost_est.add_io(1.1);
  cost_est.add_cpu(cost_model->row_evaluate_cost(records) + 1);
  if (ignore_table_scan) cost_est.set_max_cost();  // 强制使用索引

  // 代价太低且不强制 → 不需要 range scan
  if (cost_est.total_cost() <= 2.0 && !force_quick_range) return 0;

  // ---- 覆盖索引扫描代价 ----
  if (!table->covering_keys.is_clear_all()) {
    int key = find_shortest_key(table, &table->covering_keys);
    Cost_estimate key_read_time = table->file->index_scan_cost(key, 1, records);
    if (key_read_time < cost_est) cost_est = key_read_time;
  }

  double best_cost = cost_est.total_cost();
  AccessPath *best_path = nullptr;

  // ---- 构建范围树 ----
  SEL_TREE *tree = nullptr;
  if (cond) {
    // ⚠️ 难点：prev_tables 和 read_tables 都加上 INNER_TABLE_BIT
    // INNER_TABLE_BIT 表示"当前表自身"，使得引用当前表列的条件能被处理
    tree = get_mm_tree(thd, &param,
                       prev_tables | INNER_TABLE_BIT,   // ← 加上 INNER_TABLE_BIT
                       read_tables | INNER_TABLE_BIT,   // ← 加上 INNER_TABLE_BIT
                       table->pos_in_table_list->map(),
                       /*remove_jump_scans=*/true, cond);
    if (tree && tree->type == SEL_TREE::IMPOSSIBLE) return -1;  // 不可能条件
    if (!tree || tree->type != SEL_TREE::KEY) tree = nullptr;
  }

  // ---- 尝试 Group Index Skip Scan ----
  AccessPath *group_path = get_best_group_min_max(thd, &param, tree, ...);
  if (group_path && group_path->cost < best_cost) {
    best_path = group_path;
    best_cost = best_path->cost;
  }

  // ---- 尝试 Index Skip Scan ----
  AccessPath *skip_scan_path = get_best_skip_scan(thd, &param, tree, ...);

  // ---- 对每个索引评估范围扫描代价 ----
  if (tree) {
    for (uint idx = 0; idx < param.keys; idx++) {
      if (tree->keys[idx]) {
        // check_quick_select() 计算该索引上范围扫描的行数和代价
        ha_rows found_records = check_quick_select(thd, &param, idx, ...);
        if (found_records != HA_POS_ERROR && cost < best_cost) {
          best_path = /* 构建 INDEX_RANGE_SCAN AccessPath */;
          best_cost = cost;
        }
      }
    }
  }

  // ---- 尝试 Index Merge ----
  if (tree && !tree->merges.is_empty()) {
    // 评估 index merge union / intersection / sort-union
  }

  *path = best_path;
  return best_path ? 1 : 0;  // 1=找到更优路径, 0=全表扫描更优, -1=不可能条件
}
```

> **⚠️ 难点：`prev_tables` / `read_tables` / `INNER_TABLE_BIT` 的含义**
>
> | 参数 | 含义 | 示例 |
> |------|------|------|
> | `prev_tables` | join 序列中在当前表之前的所有表 | t1, t2 已在序列中 |
> | `read_tables` | 已经实际读取过数据的表（含 const 表） | const 表 t0 |
> | `INNER_TABLE_BIT` | 代表"当前表自身" | 使 `t3.col = 5` 能被 range 分析 |
>
> `get_mm_tree()` 内部用这些 bitmap 判断条件中的字段引用是否"可用"：只有引用 `prev_tables | read_tables | current_table` 范围内的表的条件才能参与范围分析。

---

### D.15 get_mm_tree() — 范围树构建（递归）

```cpp
// sql/range_optimizer/range_analysis.cc:845
// 递归遍历条件树，为每个可用索引构建 SEL_TREE（范围树）
SEL_TREE *get_mm_tree(THD *thd, RANGE_OPT_PARAM *param,
                      table_map prev_tables, table_map read_tables,
                      table_map current_table,
                      bool remove_jump_scans,  // 是否移除跳跃扫描
                      Item *cond) {
  if (param->has_errors()) return nullptr;

  // ---- 情况 1：AND / OR 复合条件 ----
  if (cond->type() == Item::COND_ITEM) {
    Item_func::Functype functype = down_cast<Item_cond *>(cond)->functype();
    SEL_TREE *tree = nullptr;
    bool first = true;
    for (Item &item : *down_cast<Item_cond *>(cond)->argument_list()) {
      SEL_TREE *new_tree = get_mm_tree(thd, param, prev_tables, read_tables,
                                       current_table, remove_jump_scans, &item);
      if (first) { tree = new_tree; first = false; continue; }

      // ⚠️ 难点：AND 用 tree_and()，OR 用 tree_or()
      if (functype == Item_func::COND_AND_FUNC) {
        tree = tree_and(param, tree, new_tree);
        if (tree && tree->type == SEL_TREE::IMPOSSIBLE) break;  // AND 中有不可能 → 整体不可能
      } else {
        tree = tree_or(param, remove_jump_scans, tree, new_tree);
        if (tree == nullptr || tree->type == SEL_TREE::ALWAYS) break;  // OR 中有 null/ALWAYS → 放弃
      }
    }
    return tree;
  }

  // ---- 情况 2：常量条件 ----
  if (cond->const_item() && !cond->is_expensive()) {
    return new SEL_TREE(cond->val_int() ? SEL_TREE::ALWAYS : SEL_TREE::IMPOSSIBLE, ...);
  }

  // ---- 情况 3：函数条件（叶子节点）----
  if (cond->type() != Item::FUNC_ITEM) return nullptr;
  Item_func *cond_func = (Item_func *)cond;

  switch (cond_func->functype()) {
    case Item_func::BETWEEN: {
      // a BETWEEN b AND c → 对 a、b、c 中的字段分别构建范围
      Item *arg_left = cond_func->arguments()[0];
      if (arg_left->real_item()->type() == Item::FIELD_ITEM)
        ftree = get_full_func_mm_tree(..., field_item, cond_func, nullptr, inv);
      // 对右侧参数也尝试构建（支持 const BETWEEN col1 AND col2）
      for (uint i = 1; i < cond_func->arg_count; i++) { ... }
      break;
    }
    case Item_func::IN_FUNC: {
      // col IN (v1, v2, ...) → 对 col 构建多个等值范围的 OR
      ftree = get_full_func_mm_tree(..., predicand, cond_func, nullptr, inv);
      break;
    }
    case Item_func::MULT_EQUAL_FUNC: {
      // ⚠️ 多重等式：对等式中每个字段都构建范围
      // MULT_EQUAL(a, b, 5) → a=5 的范围 AND b=5 的范围
      Item_equal *item_equal = down_cast<Item_equal *>(cond);
      Item *value = item_equal->const_arg();
      if (!value) return nullptr;
      for (Item_field &field_item : item_equal->get_fields()) {
        SEL_TREE *tree = get_mm_parts(thd, param, ..., field, Item_func::EQ_FUNC, value);
        ftree = !ftree ? tree : tree_and(param, ftree, tree);
      }
      return ftree;
    }
    default: {
      // 普通比较：col OP value
      if (arg_left->real_item()->type() == Item::FIELD_ITEM)
        ftree = get_full_func_mm_tree(..., field_item, cond_func, value, inv);
      // 也检查右侧是否是字段（支持 value OP col）
      if (cond_func->arg_count > 1 && arg_right->real_item()->type() == Item::FIELD_ITEM)
        ftree = tree_and(param, ftree, get_full_func_mm_tree(...));
      break;
    }
  }
  return ftree;
}
```

> **⚠️ 难点：AND 与 OR 合并时的中止条件相反**
>
> | 合并方式 | 中止条件 | 原因 |
> |---------|---------|------|
> | `tree_and()` | 结果为 `IMPOSSIBLE` | AND 中有不可能条件 → 整体不可能 |
> | `tree_or()` | 结果为 `nullptr` 或 `ALWAYS` | OR 中有无法分析的条件 → 必须放弃整个 OR（保守策略） |
>
> `MULT_EQUAL_FUNC` 的处理：多重等式 `MULT_EQUAL(a, b, 5)` 会为 a 和 b 分别生成 `=5` 的范围，然后用 `tree_and()` 合并。这使得多个等价列上的索引都能被考虑。

---

### D.16 get_mm_leaf() — 叶子范围节点构造

```cpp
// sql/range_optimizer/range_analysis.cc:1370
// 为单个 "field OP value" 条件构造 SEL_ARG 叶子节点
static SEL_ROOT *get_mm_leaf(THD *thd, RANGE_OPT_PARAM *param,
                             Item *cond_func, Field *field,
                             KEY_PART *key_part,
                             Item_func::Functype type,  // 比较类型
                             Item *value,               // 比较值（IS NULL 时为 nullptr）
                             bool *inexact) {           // 输出：范围是否精确
  const size_t null_bytes = field->is_nullable() ? 1 : 0;

  // ---- 路径 1：IS NULL / IS NOT NULL ----
  if (!value) {
    if (field->table->pos_in_table_list->outer_join)
      goto end;  // 外连接内表的 IS NULL 不能用 range scan
    if (!field->is_nullable()) {
      if (type == Item_func::ISNULL_FUNC)
        tree = new SEL_ROOT(SEL_ROOT::Type::IMPOSSIBLE);  // NOT NULL 列 IS NULL → 不可能
      goto end;
    }
    // 构造 NULL 值的范围节点
    uchar *null_string = alloc(key_part->store_length + 1);
    memcpy(null_string, is_null_string, sizeof(is_null_string));
    SEL_ARG *root = new SEL_ARG(field, null_string, null_string, ...);
    tree = new SEL_ROOT(root);
    if (type == Item_func::ISNOTNULL_FUNC) {
      root->min_flag = NEAR_MIN;      // IS NOT NULL → (NULL, +∞)
      root->max_flag = NO_MAX_RANGE;
    }
    goto end;
  }

  // 检查字段和值是否在索引中可比较
  if (!comparable_in_index(cond_func, field, key_part->image_type, type, value))
    goto end;

  // ---- 路径 2：LIKE 条件 ----
  if (type == Item_func::LIKE_FUNC) {
    if (!optimize_range) goto end;
    // 将 LIKE 'abc%' 转换为范围 ['abc', 'abd')
    my_like_range(field->charset(), pattern, ...,
                  min_str, max_str, &min_length, &max_length);
    // ⚠️ LIKE 范围总是不精确的（需要后续 WHERE 过滤验证）
    *inexact = true;
    SEL_ARG *root = new SEL_ARG(field, min_str, max_str, ...);
    tree = new SEL_ROOT(root);
    goto end;
  }

  // ---- 路径 3：普通比较 (=, <, <=, >, >=) ----
  if (!optimize_range && type != Item_func::EQ_FUNC &&
      type != Item_func::EQUAL_FUNC)
    goto end;

  // 将 value 存储为索引格式
  str = alloc(key_part->store_length + 1);
  field->get_key_image(str + null_bytes, key_part->length, key_part->image_type);

  switch (type) {
    case Item_func::LT_FUNC:    // field < value → (-∞, value)
      // ⚠️ 难点：对于 field < value，如果存储值被截断，
      // 需要调整为 field <= stored_value（因为截断后的值可能小于原值）
      if (stored_field_cmp_to_item(...) == 0)
        tree = new SEL_ARG(field, null_string, str);  // min=null, max=value
      else
        tree = new SEL_ARG(field, null_string, str);  // 调整 flag
      tree->max_flag = NEAR_MAX;  // 不含右端点
      break;

    case Item_func::GT_FUNC:    // field > value → (value, +∞)
      tree = new SEL_ARG(field, str, nullptr);
      tree->min_flag = NEAR_MIN;  // 不含左端点
      tree->max_flag = NO_MAX_RANGE;
      break;

    case Item_func::EQ_FUNC:
    case Item_func::EQUAL_FUNC: // field = value → [value, value]
      tree = new SEL_ARG(field, str, str);
      tree->min_flag = tree->max_flag = 0;  // 闭区间
      break;

    // LE_FUNC, GE_FUNC 类似...
  }
end:
  return tree;
}
```

> **⚠️ 难点：三条路径的选择逻辑**
>
> | 条件类型 | `value` 参数 | 处理路径 | `inexact` |
> |---------|-------------|---------|-----------|
> | `IS NULL` / `IS NOT NULL` | `nullptr` | 路径 1：直接构造 NULL 范围 | `false` |
> | `LIKE 'pattern'` | 非 null | 路径 2：`my_like_range()` 转换 | **`true`**（总是不精确） |
> | `=`, `<`, `<=`, `>`, `>=` | 非 null | 路径 3：直接存储值构造范围 | 取决于截断 |
>
> `inexact=true` 意味着范围扫描返回的行需要在 server 层用原始条件再次过滤。LIKE 由于 Unicode 排序规则的复杂性（收缩、扩展等），范围总是可能偏宽，因此始终标记为不精确。

---

### D.17 tree_and() — 范围树 AND 合并

```cpp
// sql/range_optimizer/tree.cc:503
// 将两棵 SEL_TREE 按 AND 语义合并：取各索引范围的交集
SEL_TREE *tree_and(RANGE_OPT_PARAM *param, SEL_TREE *tree1, SEL_TREE *tree2) {
  if (param->has_errors()) return nullptr;

  // ⚠️ 难点：null 树的处理——AND 中 null 表示"无法分析的条件"
  // 无法分析 AND X = X（保留可分析的部分，但标记为不精确）
  if (tree1 == nullptr) {
    if (tree2 != nullptr) tree2->inexact = true;
    return tree2;
  }
  if (tree2 == nullptr) {
    if (tree1 != nullptr) tree1->inexact = true;
    return tree1;
  }

  // ⚠️ 难点：IMPOSSIBLE / ALWAYS 的对称处理
  // IMPOSSIBLE AND X = IMPOSSIBLE（AND 中有不可能 → 整体不可能）
  if (tree1->type == SEL_TREE::IMPOSSIBLE) return tree1;
  if (tree2->type == SEL_TREE::IMPOSSIBLE) return tree2;
  // ALWAYS AND X = X（ALWAYS 是恒真，不影响 AND）
  if (tree2->type == SEL_TREE::ALWAYS) {
    tree1->inexact |= tree2->inexact;
    return tree1;
  }
  if (tree1->type == SEL_TREE::ALWAYS) {
    tree2->inexact |= tree1->inexact;
    return tree2;
  }

  Key_map result_keys;

  // 逐索引合并范围
  for (uint idx = 0; idx < param->keys; idx++) {
    SEL_ROOT *key1 = tree1->release_key(idx);
    SEL_ROOT *key2 = tree2->release_key(idx);

    if (key1 != nullptr || key2 != nullptr) {
      if (key1 == nullptr || key2 == nullptr)
        tree1->inexact = true;  // 一棵树有此索引范围，另一棵没有 → 不精确

      // key_and()：对同一索引的两组范围取交集
      SEL_ROOT *new_key = key_and(param, key1, key2);
      tree1->set_key(idx, new_key);
      if (new_key) {
        if (new_key->type == SEL_ROOT::Type::IMPOSSIBLE) {
          tree1->type = SEL_TREE::IMPOSSIBLE;  // 交集为空 → 不可能
          return tree1;
        }
        result_keys.set_bit(idx);
      }
    }
  }
  tree1->keys_map = result_keys;
  tree1->inexact |= tree2->inexact;

  // 合并 Index Merge 列表
  imerge_list_and_list(&tree1->merges, &tree2->merges);
  return tree1;
}
```

> **⚠️ 难点：`tree_and()` vs `tree_or()` 对特殊类型的处理对比**
>
> | 输入 | `tree_and()` 结果 | `tree_or()` 结果 |
> |------|------------------|-----------------|
> | `nullptr` AND/OR X | X（标记 inexact） | `nullptr`（放弃） |
> | IMPOSSIBLE AND/OR X | IMPOSSIBLE | X |
> | ALWAYS AND/OR X | X | ALWAYS |
>
> AND 语义：null 表示"不知道"，AND 不知道 = 保守地保留另一侧。
> OR 语义：null 表示"不知道"，OR 不知道 = 必须放弃整个 OR（因为不知道的部分可能匹配所有行）。

---

### D.18 tree_or() — 范围树 OR 合并

```cpp
// sql/range_optimizer/tree.cc:680
// 将两棵 SEL_TREE 按 OR 语义合并：取各索引范围的并集
SEL_TREE *tree_or(RANGE_OPT_PARAM *param, bool remove_jump_scans,
                  SEL_TREE *tree1, SEL_TREE *tree2) {
  if (param->has_errors()) return nullptr;

  // ⚠️ null 在 OR 中表示"无法分析" → 必须放弃
  if (!tree1 || !tree2) return nullptr;
  tree1->inexact = tree2->inexact = tree1->inexact | tree2->inexact;

  // IMPOSSIBLE OR X = X；ALWAYS OR X = ALWAYS
  if (tree1->type == SEL_TREE::IMPOSSIBLE || tree2->type == SEL_TREE::ALWAYS)
    return tree2;
  if (tree2->type == SEL_TREE::IMPOSSIBLE || tree1->type == SEL_TREE::ALWAYS)
    return tree1;

  // ⚠️ 难点：当树同时有 keys[] 和 merges 时，移除 merges
  // 因为 OR 操作在同时有两种类型时过于复杂
  if (!tree1->merges.is_empty()) {
    for (uint i = 0; i < param->keys; i++)
      if (tree1->keys[i] != NULL &&
          tree1->keys[i]->type == SEL_ROOT::Type::KEY_RANGE) {
        tree1->merges.clear();  // 有真实范围 → 丢弃 merges
        break;
      }
  }
  // tree2 同理...

  SEL_TREE *result = nullptr;
  Key_map result_keys;

  // ⚠️ 核心分支：能否直接按索引做 OR？
  if (sel_trees_can_be_ored(tree1, tree2, param)) {
    // 两棵树在相同索引上有范围 → 逐索引取并集
    for (uint idx = 0; idx < param->keys; idx++) {
      SEL_ROOT *key1 = tree1->release_key(idx);
      SEL_ROOT *key2 = tree2->release_key(idx);
      SEL_ROOT *new_key = key_or(param, key1, key2);
      tree1->set_key(idx, new_key);
      if (new_key) {
        result = tree1;
        result_keys.set_bit(idx);
      }
    }
    if (result) result->keys_map = result_keys;
  } else {
    // ⚠️ 不能直接 OR → 降级为 Index Merge
    if (tree1->merges.is_empty() && tree2->merges.is_empty()) {
      // 两棵都是纯范围树 → 创建新的 imerge
      SEL_IMERGE *merge = new SEL_IMERGE(...);
      result = new SEL_TREE(...);
      merge->or_sel_tree(tree1);
      merge->or_sel_tree(tree2);
      result->merges.push_back(merge);
    } else if (!tree1->merges.is_empty() && !tree2->merges.is_empty()) {
      // 两棵都是 imerge 树 → imerge 列表合并
      imerge_list_or_list(param, remove_jump_scans,
                          &tree1->merges, &tree2->merges);
      result = tree1;
    } else {
      // 一棵是范围树，一棵是 imerge 树
      if (tree1->merges.is_empty()) std::swap(tree1, tree2);
      imerge_list_or_tree(param, remove_jump_scans, &tree1->merges, tree2);
      result = tree1;
    }
  }
  return result;
}
```

> **⚠️ 难点：`sel_trees_can_be_ored()` 的判断条件**
>
> 两棵树能直接 OR 当且仅当：它们在**同一个索引**上都有范围。例如：
> - `idx1: [1,3]` OR `idx1: [5,7]` → 可以合并为 `idx1: [1,3] ∪ [5,7]`
> - `idx1: [1,3]` OR `idx2: [5,7]` → 不能直接合并，降级为 Index Merge
>
> Index Merge 降级意味着执行时需要分别扫描两个索引，然后对结果做 union/intersection。

---

### D.19 check_quick_select() — 范围扫描代价估算

```cpp
// sql/range_optimizer/index_range_scan_plan.cc:568
// 对单个索引上的范围树估算行数和代价
ha_rows check_quick_select(THD *thd, RANGE_OPT_PARAM *param, uint idx,
                           bool index_only,    // 是否覆盖索引
                           SEL_ROOT *tree,     // 该索引的范围树
                           bool update_tbl_stats,
                           enum_order order_direction,
                           bool skip_records_in_range,
                           uint *mrr_flags, uint *bufsize,
                           Cost_estimate *cost,
                           bool *is_ror_scan,     // 输出：是否 ROR 扫描
                           bool *is_imerge_scan) { // 输出：是否可用于 imerge
  if (!tree) return HA_POS_ERROR;
  if (tree->type == SEL_ROOT::Type::IMPOSSIBLE) return 0;
  if (tree->type != SEL_ROOT::Type::KEY_RANGE || tree->root->part != 0)
    return HA_POS_ERROR;  // 范围树不从第一个 key_part 开始 → 不可用

  uint keynr = param->real_keynr[idx];

  // ⚠️ 难点：eq_range_index_dive_limit 控制统计方式
  // 当等值范围数量超过此阈值时，从 index dive 切换为索引统计
  uint range_count = 0;
  param->use_index_statistics = eq_ranges_exceeds_limit(
      tree, &range_count, thd->variables.eq_range_index_dive_limit);

  *is_ror_scan = !(file->index_flags(keynr, 0, true) & HA_KEY_SCAN_NOT_ROR);

  // 设置 MRR 标志
  *mrr_flags = (order_direction == ORDER_DESC) ? HA_MRR_USE_DEFAULT_IMPL : 0;
  *mrr_flags |= HA_MRR_NO_ASSOCIATION;
  if (order_direction != ORDER_NOT_RELEVANT) *mrr_flags |= HA_MRR_SORTED;
  if (index_only && (file->index_flags(keynr, ...) & HA_KEYREAD_ONLY) &&
      !(pk_is_clustered && keynr == table->s->primary_key))
    *mrr_flags |= HA_MRR_INDEX_ONLY;

  // ⚠️ 核心：调用存储引擎的 multi_range_read_info_const()
  // 该函数遍历所有范围，对每个范围调用 records_in_range() 估算行数
  // 最终汇总返回总行数和总代价
  rows = file->multi_range_read_info_const(keynr, &seq_if, (void *)&seq, 0,
                                           bufsize, mrr_flags, &force_default_mrr,
                                           cost);
  if (rows != HA_POS_ERROR) {
    param->table->quick_rows[keynr] = rows;
    if (update_tbl_stats) {
      param->table->quick_keys.set_bit(keynr);
      param->table->quick_key_parts[keynr] = seq.max_key_part + 1;
      param->table->quick_n_ranges[keynr] = seq.range_count;
      param->table->quick_condition_rows =
          min(param->table->quick_condition_rows, rows);
    }
  }

  // ROR 扫描的额外检查：
  // 1. 索引算法必须是 BTREE 或 SE_SPECIFIC
  // 2. 索引不能有降序 key_part（除非是聚簇 PK）
  // 3. 索引不能包含虚拟生成列
  if ((key_alg != HA_KEY_ALG_BTREE && key_alg != HA_KEY_ALG_SE_SPECIFIC) ||
      (file->index_flags(keynr, 0, true) & HA_KEY_SCAN_NOT_ROR) ||
      param->table->index_contains_some_virtual_gcol(keynr))
    *is_ror_scan = false;
  else if (table->s->primary_key == keynr && pk_is_clustered)
    *is_ror_scan = true;  // 聚簇 PK 总是 ROR

  return rows;
}
```

> **⚠️ 难点：`eq_range_index_dive_limit` 的影响**
>
> | `IN` 列表大小 | 行为 | 精度 |
> |--------------|------|------|
> | < `eq_range_index_dive_limit`（默认 200） | 对每个值调用 `records_in_range()`（index dive） | 精确 |
> | >= `eq_range_index_dive_limit` | 使用 `rec_per_key` 索引统计 | 粗略 |
>
> ROR (Rowid-Ordered Retrieval) 扫描：返回的行按主键顺序排列，可用于 Index Merge Intersection（取交集时不需要排序）。

---

### D.20 get_ranges_from_tree() — 范围物化

```cpp
// sql/range_optimizer/index_range_scan_plan.cc:774
// 将 SEL_ROOT 范围树转换为 QUICK_RANGE 数组（物化）
// 这是"两阶段设计"中的第二阶段
bool get_ranges_from_tree(MEM_ROOT *return_mem_root, TABLE *table,
                          KEY_PART *key, uint keyno, SEL_ROOT *key_tree,
                          uint num_key_parts,
                          unsigned *used_key_parts,       // 输出：使用的键部分数
                          unsigned *num_exact_key_parts,  // 输出：精确匹配的键部分数
                          Quick_ranges *ranges) {         // 输出：QUICK_RANGE 数组
  *used_key_parts = 0;
  if (key_tree->type != SEL_ROOT::Type::KEY_RANGE)
    return false;

  // ⚠️ 记录第一个 key_part 的排序方向
  // 用于确定遍历 SEL_ARG 树的方向（升序/降序）
  const bool first_keypart_is_asc = key_tree->root->is_ascending;

  uchar min_key[MAX_KEY_LENGTH + MAX_FIELD_WIDTH];
  uchar max_key[MAX_KEY_LENGTH + MAX_FIELD_WIDTH];
  *num_exact_key_parts = num_key_parts;

  // 核心：递归遍历 SEL_ARG 树，生成 QUICK_RANGE
  // get_ranges_from_tree_given_base() 沿 SEL_ARG 的 left/right（同层 OR）
  // 和 next_key_part（下一键部分 AND）递归
  if (get_ranges_from_tree_given_base(
          current_thd, return_mem_root, &table->key_info[keyno], key,
          key_tree,
          min_key, min_key, 0,   // min_key 缓冲区及初始偏移
          max_key, max_key, 0,   // max_key 缓冲区及初始偏移
          first_keypart_is_asc,
          num_key_parts, used_key_parts, num_exact_key_parts, ranges))
    return true;

  // num_exact_key_parts 不超过 used_key_parts
  *num_exact_key_parts = std::min(*num_exact_key_parts, *used_key_parts);
  return false;
}
```

> **⚠️ 难点：两阶段设计回顾**
>
> | 阶段 | 函数 | 数据结构 | 目的 |
> |------|------|---------|------|
> | 阶段 1（分析） | `get_mm_tree()` | `SEL_TREE` / `SEL_ARG` | 构建范围树，支持 AND/OR 合并 |
> | 阶段 2（物化） | `get_ranges_from_tree()` | `QUICK_RANGE` 数组 | 转换为扁平的范围数组，供引擎使用 |
>
> `num_exact_key_parts` vs `used_key_parts`：如果范围条件只覆盖了索引前 2 个 key_part，但第 2 个是范围条件（非等值），则 `used_key_parts=2`，`num_exact_key_parts=1`（只有第 1 个是精确匹配）。

---

### D.21 make_min_endpoint() / make_max_endpoint() — QUICK_RANGE 转 key_range

```cpp
// sql/range_optimizer/range_optimizer.h:121
// 将 QUICK_RANGE 的 min 端点转换为存储引擎的 key_range 结构
void QUICK_RANGE::make_min_endpoint(key_range *kr) {
  kr->key = (const uchar *)min_key;
  kr->length = min_length;
  kr->keypart_map = min_keypart_map;
  // ⚠️ 难点：flag 到 ha_rkey_function 的映射
  kr->flag = ((flag & NEAR_MIN)   ? HA_READ_AFTER_KEY     // > value（开区间）
              : (flag & EQ_RANGE) ? HA_READ_KEY_EXACT      // = value（等值）
                                  : HA_READ_KEY_OR_NEXT);  // >= value（闭区间）
}

// sql/range_optimizer/range_optimizer.h:160
// 将 QUICK_RANGE 的 max 端点转换为存储引擎的 key_range 结构
void QUICK_RANGE::make_max_endpoint(key_range *kr) {
  kr->key = (const uchar *)max_key;
  kr->length = max_length;
  kr->keypart_map = max_keypart_map;
  // ⚠️ 难点：max 端点的映射规则与 min 不同
  // 这里用 READ_AFTER_KEY 表示"扫描到此为止"（不含/含端点都用它）
  // 而 NEAR_MAX（开区间）时用 READ_BEFORE_KEY
  kr->flag = (flag & NEAR_MAX ? HA_READ_BEFORE_KEY    // < value（开区间）
                               : HA_READ_AFTER_KEY);   // <= value（闭区间，扫描过端点再停）
}

// 重载版本：限制前缀长度（用于部分键比较）
void QUICK_RANGE::make_max_endpoint(key_range *kr, uint prefix_length,
                                     key_part_map keypart_map) {
  make_max_endpoint(kr);
  kr->length = std::min(kr->length, prefix_length);
  kr->keypart_map &= keypart_map;
}
```

> **⚠️ 难点：为什么闭区间 max 用 `HA_READ_AFTER_KEY`？**
>
> 这是存储引擎的约定：`HA_READ_AFTER_KEY` 在 max 端点的语义是"读到超过此 key 的第一条记录时停止"，也就是说会**包含**等于此 key 的记录。这与它在 min 端点的语义（"从此 key 之后开始读"，即不包含）是对称的。
>
> | 端点 | flag | ha_rkey_function | 语义 |
> |------|------|-----------------|------|
> | min, 闭 `[` | 无特殊 flag | `HA_READ_KEY_OR_NEXT` | >= value |
> | min, 开 `(` | `NEAR_MIN` | `HA_READ_AFTER_KEY` | > value |
> | min, 等值 `=` | `EQ_RANGE` | `HA_READ_KEY_EXACT` | = value |
> | max, 闭 `]` | 无特殊 flag | `HA_READ_AFTER_KEY` | 扫描到 > value 时停止（含 value） |
> | max, 开 `)` | `NEAR_MAX` | `HA_READ_BEFORE_KEY` | 扫描到 >= value 时停止（不含 value） |

---

### D.22 best_access_path() — 最优访问路径选择

```cpp
// sql/sql_planner.cc:981
// 为单个表选择最优访问方式（ref / range / scan）
void Optimize_table_order::best_access_path(
    JOIN_TAB *tab,
    const table_map remaining_tables,  // 尚未加入计划的表
    const uint idx,                    // 当前在 join->positions[] 中的位置
    bool disable_jbuf,                 // 是否禁用 join buffer
    const double prefix_rowcount,      // 前缀表的累计行数
    POSITION *pos) {                   // 输出：最优访问方式
  float filter_effect = 1.0;

  // ---- 第一步：评估 ref 访问 ----
  Key_use *best_ref = nullptr;
  if (tab->keyuse() != nullptr &&
      (table->file->ha_table_flags() & HA_NO_INDEX_ACCESS) == 0)
    best_ref = find_best_ref(tab, remaining_tables, idx, prefix_rowcount,
                             &found_condition, &ref_depend_map, &used_key_parts);

  double rows_fetched = best_ref ? best_ref->fanout : DBL_MAX;
  double best_read_cost = best_ref ? best_ref->read_cost : DBL_MAX;

  // ---- 第二步：决定是否需要评估 scan ----
  // ⚠️ 难点：四个跳过 scan 评估的条件
  if (rows_fetched < tab->found_records &&     // (1a) ref 行数 < scan 行数
      best_read_cost <= tab->read_time)         // (1b) ref 代价 < scan 代价
  {
    // 条件 1：ref 明显更优，跳过 scan
  }
  else if (tab->range_scan() && best_ref &&             // (2)
           used_index(tab->range_scan()) == best_ref->key &&
           used_key_parts >= table->quick_key_parts[best_ref->key] &&
           tab->range_scan()->type != AccessPath::GROUP_INDEX_SKIP_SCAN &&
           tab->range_scan()->type != AccessPath::INDEX_SKIP_SCAN)
  {
    // 条件 2：range 和 ref 用同一索引，且 ref 用了更多 key_part → ref 更优
  }
  else if ((table->file->ha_table_flags() & HA_TABLE_SCAN_ON_INDEX) &&  // (3)
           !table->covering_keys.is_clear_all() && best_ref &&
           (!tab->range_scan() ||
            (tab->range_scan()->type == AccessPath::ROWID_INTERSECTION &&
             best_ref->read_cost < tab->range_scan()->cost)))
  {
    // 条件 3：InnoDB 有覆盖索引且 ref 更优 → 不做全表扫描
  }
  else if (table->force_index && best_ref && !tab->range_scan())  // (4)
  {
    // 条件 4：FORCE INDEX 且有 ref 但无 range → 不做全表扫描
  }
  else {
    // ---- 第三步：评估 scan 代价 ----
    double rows_after_filtering;
    double scan_read_cost = calculate_scan_cost(
        tab, idx, best_ref, prefix_rowcount, found_condition,
        disable_jbuf, &rows_after_filtering, &trace_access_scan);

    const double scan_total_cost =
        scan_read_cost +
        cost_model->row_evaluate_cost(prefix_rowcount * rows_after_filtering);

    // ---- 第四步：比较 ref vs scan ----
    if (best_ref == nullptr ||
        scan_total_cost < best_read_cost +
            cost_model->row_evaluate_cost(prefix_rowcount * rows_fetched)) {
      // scan 更优（或没有可用的 ref）
      best_read_cost = scan_read_cost;
      rows_fetched = rows_after_filtering;
      best_ref = nullptr;
      // ...
    }
  }

  // ---- 第五步：计算条件过滤率 ----
  if (best_ref)
    filter_effect = calculate_condition_filter(tab, best_ref, ...);
  else
    filter_effect = calculate_condition_filter(tab, nullptr, ...);

  // ---- 填充 POSITION 结构 ----
  pos->rows_fetched = rows_fetched;
  pos->read_cost = best_read_cost;
  pos->filter_effect = filter_effect;
  pos->key = best_ref;
  // ...
}
```

> **⚠️ 难点：四个跳过 scan 的条件**
>
> | 条件 | 场景 | 原因 |
> |------|------|------|
> | (1) | ref 行数少且代价低 | ref 明显更优，无需评估 scan |
> | (2) | range 和 ref 用同一索引，ref 用更多 key_part | ref 更精确，range 不可能更好 |
> | (3) | InnoDB + 覆盖索引 + ref 可用 | InnoDB 全表扫描走聚簇索引，覆盖索引的 ref 总是更好 |
> | (4) | FORCE INDEX + 有 ref + 无 range | 用户强制用索引，全表扫描不被允许 |

---

### D.23 find_best_ref() — 评估所有 ref 访问

```cpp
// sql/sql_planner.cc:207
// 遍历所有可用的 Key_use，找到代价最低的 ref 访问方式
Key_use *Optimize_table_order::find_best_ref(
    const JOIN_TAB *tab,
    const table_map remaining_tables,
    const uint idx,
    const double prefix_rowcount,
    bool *found_condition,
    table_map *ref_depend_map,
    uint *used_key_parts) {
  Key_use *best_ref = nullptr;
  double best_ref_cost = DBL_MAX;

  enum idx_type { CLUSTERED_PK, UNIQUE, NOT_UNIQUE, FULLTEXT };
  enum idx_type best_found_keytype = NOT_UNIQUE;

  // 遍历该表的所有 Key_use 条目
  for (Key_use *keyuse = tab->keyuse(); keyuse->table_ref == tab->table_ref;) {
    key_part_map found_part = 0;     // 可用的 key_part bitmap
    key_part_map const_part = 0;     // 其中值为常量的
    key_part_map null_rejecting_part = 0;
    double cur_read_cost, cur_fanout;
    const uint key = keyuse->key;

    // 对每个 key_part，找最优的 keyuse
    while (keyuse->table_ref == tab->table_ref && keyuse->key == key) {
      const uint keypart = keyuse->keypart;

      // ⚠️ 同一个 keypart 可能有多个 keyuse（多个等值条件）
      // 例如 t1.col_x=4 AND t1.col_x=t2.col_y → 两个 keyuse
      for (; keyuse->keypart == keypart; ++keyuse) {
        // 跳过不可用的 keyuse：
        // 1) 跨 semi-join 边界的引用
        // 2) 引用了尚未加入计划的表
        // 3) 已有 ref_or_null 的 keypart 不能再加
        if ((excluded_tables & keyuse->used_tables) ||
            (remaining_tables & keyuse->used_tables) ||
            (ref_or_null_part && (keyuse->optimize & KEY_OPTIMIZE_REF_OR_NULL)))
          continue;

        if (!(keyuse->used_tables & ~join->const_table_map))
          const_part |= keyuse->keypart_map;  // 常量值

        found_part |= keyuse->keypart_map;
        if (keyuse->null_rejecting || !keyuse->val->is_nullable())
          null_rejecting_part |= keyuse->keypart_map;
      }
    }

    // ⚠️ 难点：判断 key_part 是否形成有效前缀
    // found_part 必须是从 part0 开始的连续位（前缀匹配）
    if (~found_part & make_prev_keypart_map(found_part)) continue;

    // 计算 ref 代价
    cur_fanout = ...;  // 通过 rec_per_key 估算
    cur_read_cost = find_cost_for_ref(thd, table, key, cur_fanout, worst_seeks);

    // 判断索引类型
    if (key == table->s->primary_key && table->file->primary_key_is_clustered())
      cur_keytype = CLUSTERED_PK;
    else if (found_part == PREV_BITS(uint, table->key_info[key].user_defined_key_parts)
             && null_rejecting_part == found_part)
      cur_keytype = UNIQUE;

    // 比较代价，选择最优
    double cur_ref_cost = cur_read_cost * prefix_rowcount +
                          cost_model->row_evaluate_cost(cur_fanout * prefix_rowcount);
    if (cur_ref_cost < best_ref_cost ||
        (cur_ref_cost == best_ref_cost && cur_keytype < best_found_keytype)) {
      best_ref = start_key;
      best_ref_cost = cur_ref_cost;
      best_found_keytype = cur_keytype;
      *used_key_parts = cur_used_keyparts;
      *ref_depend_map = table_deps;
    }
  }
  return best_ref;
}
```

> **⚠️ 难点：Key_use 的结构与遍历**
>
> `Key_use` 数组按 `(table, key, keypart)` 排序。对于 `WHERE t1.a=1 AND t1.a=t2.x AND t1.b=2`，索引 `idx(a,b)` 会有三个 Key_use：
> - `keyuse[0]: key=idx, keypart=0, val=1`（常量）
> - `keyuse[1]: key=idx, keypart=0, val=t2.x`（依赖 t2）
> - `keyuse[2]: key=idx, keypart=1, val=2`（常量）
>
> 对 keypart=0，选择使前缀组合更少的那个（`prev_record_reads()` 更小的）。

---

### D.24 find_cost_for_ref() — ref 访问代价计算

```cpp
// sql/sql_planner.cc:143
// 计算 ref 访问读取 num_rows 行的 I/O 代价
double find_cost_for_ref(const THD *thd, TABLE *table, unsigned keyno,
                         double num_rows, double worst_seeks) {
  // 限制行数不超过 max_seeks_for_key（防止代价过高）
  num_rows = std::min(num_rows, double(thd->variables.max_seeks_for_key));

  // 派生表未完成物化时无法计算代价
  if (table->pos_in_table_list->is_derived_unfinished_materialization())
    return worst_seeks;

  // ⚠️ 难点：三种不同的代价计算路径
  if (table->covering_keys.is_set(keyno)) {
    // 路径 1：覆盖索引 → 只需读索引树
    const Cost_estimate index_read_cost =
        table->file->index_scan_cost(keyno, 1, num_rows);
    return index_read_cost.total_cost();
  }
  if (keyno == table->s->primary_key &&
      table->file->primary_key_is_clustered()) {
    // 路径 2：聚簇主键 → 读索引即读数据
    const Cost_estimate table_read_cost =
        table->file->read_cost(keyno, 1, num_rows);
    return table_read_cost.total_cost();
  }
  // 路径 3：普通二级索引 → 需要回表
  // page_read_cost 包含了索引读取 + 回表的 I/O 代价
  return min(table->file->page_read_cost(keyno, num_rows), worst_seeks);
}
```

> **⚠️ 难点：三种路径的代价差异**
>
> | 路径 | 索引类型 | I/O 特征 | 代价来源 |
> |------|---------|---------|---------|
> | 1 | 覆盖索引 | 只读索引 B+ 树 | `index_scan_cost()` |
> | 2 | 聚簇主键 | 索引叶子 = 数据行 | `read_cost()` |
> | 3 | 二级索引 | 索引扫描 + 回表随机 I/O | `page_read_cost()` |
>
> `worst_seeks` 是一个上限值（`min(records/10, 文件大小/IO_SIZE * 2)`），防止代价估算爆炸。`page_read_cost()` 计算的代价如果超过 `worst_seeks`，则取 `worst_seeks`。

---

### D.25 calculate_scan_cost() — scan 代价计算

```cpp
// sql/sql_planner.cc:770
// 计算全表扫描 / 索引扫描 / 范围扫描的代价
double Optimize_table_order::calculate_scan_cost(
    const JOIN_TAB *tab, const uint idx,
    const Key_use *best_ref,
    const double prefix_rowcount,       // 前缀表的累计行数
    const bool found_condition,          // 是否有可用的过滤条件
    const bool disable_jbuf,             // 是否禁用 join buffer
    double *rows_after_filtering,        // 输出：过滤后的行数
    Opt_trace_object *trace_access_scan) {
  double scan_and_filter_cost;
  TABLE *const table = tab->table();
  *rows_after_filtering = static_cast<double>(tab->found_records);

  // ---- 计算过滤效果 ----
  if (thd->optimizer_switch_flag(OPTIMIZER_SWITCH_COND_FANOUT_FILTER)) {
    // 使用条件过滤率估算
    const float const_cond_filter = calculate_condition_filter(
        tab, nullptr, 0, tab->found_records, !disable_jbuf, true, ...);
    *rows_after_filtering = tab->found_records * const_cond_filter;
  } else if (table->quick_condition_rows != tab->found_records) {
    *rows_after_filtering = table->quick_condition_rows;
  } else if (found_condition) {
    // ⚠️ 条件过滤关闭时的启发式：假设过滤 25%
    *rows_after_filtering = tab->found_records * 0.75;
  }

  // ---- 分三种情况计算代价 ----
  if (tab->range_scan()) {
    // 情况 1：范围扫描
    scan_and_filter_cost =
        prefix_rowcount * (tab->range_scan()->cost +
            cost_model->row_evaluate_cost(tab->found_records - *rows_after_filtering));
  } else if (disable_jbuf) {
    // 情况 2：无 join buffer 的全表/索引扫描
    // 每次前缀行都要扫描一遍全表
    Cost_estimate scan_cost;
    if (table->force_index && !best_ref)
      scan_cost = table->file->read_cost(tab->ref().key, 1, tab->records());
    else
      scan_cost = table->file->table_scan_cost();

    scan_and_filter_cost =
        prefix_rowcount *
        (scan_cost.total_cost() +
         cost_model->row_evaluate_cost(tab->records() - *rows_after_filtering));
  } else {
    // 情况 3：有 join buffer 的全表/索引扫描
    // ⚠️ 难点：buffer_count 计算
    // join buffer 满一次就扫描一遍全表
    // buffer_count = 1 + (每行记录长度 × 前缀行数) / join_buffer_size
    const double buffer_count =
        1.0 + ((double)cache_record_length(join, idx) * prefix_rowcount /
               (double)thd->variables.join_buff_size);

    scan_and_filter_cost =
        buffer_count *
        (scan_cost.total_cost() +
         cost_model->row_evaluate_cost(tab->records() - *rows_after_filtering));
  }
  return scan_and_filter_cost;
}
```

> **⚠️ 难点：join buffer 场景下的代价模型**
>
> ```
> 无 join buffer：cost = prefix_rowcount × single_scan_cost
>   （前缀每一行都触发一次全表扫描）
>
> 有 join buffer：cost = buffer_count × single_scan_cost
>   （buffer_count ≈ prefix_rowcount / buffer_capacity）
>   （buffer 满了才触发一次全表扫描，大幅减少 I/O 次数）
> ```
>
> `buffer_count` 始终 >= 1.0（至少扫描一次）。当 `prefix_rowcount` 很大而 `join_buff_size` 很小时，`buffer_count` 接近 `prefix_rowcount`，退化为无 buffer 的情况。

---

### D.26 calculate_condition_filter() — 条件过滤率估算

```cpp
// sql/sql_planner.cc:1244
// 估算表的条件过滤率：ref/range 访问之后，还有多少比例的行满足剩余条件
float calculate_condition_filter(
    const JOIN_TAB *const tab,
    const Key_use *const keyuse,  // ref 访问用的 Key_use（scan 时为 nullptr）
    table_map used_tables,         // 已在计划中的表
    double fanout,                 // ref/scan 估计的行数
    bool is_join_buffering,
    bool write_to_trace,
    Opt_trace_object &parent_trace) {

  // ---- 前置检查：是否值得计算 ----
  // 仅当以下任一条件成立时才计算：
  // 2a) 使用 join buffering（常量条件过滤很重要）
  // 2b) 不是计划中最后一个表（过滤影响后续表）
  // 2c) 在子查询中（影响物化行数）
  // 2d) 有 semi-join（影响去重策略选择）
  // 2e) 有 ORDER BY/GROUP BY + LIMIT
  // 2f) EXPLAIN 语句
  if (!(condition_fanout_filter_enabled &&
        (is_join_buffering || remaining_tables != 0 || in_subquery ||
         has_semijoin || has_order_limit || is_explain)))
    return COND_FILTER_ALLPASS;  // 1.0

  if (fanout < 1.0 || tab->found_records < 1.0)
    return COND_FILTER_ALLPASS;

  if (bitmap_is_clear_all(&table->cond_set))
    return COND_FILTER_ALLPASS;  // 没有条件涉及此表

  float filter = COND_FILTER_ALLPASS;

  // ⚠️ 第一层：排除 ref/range 已覆盖的字段
  // 这些字段的过滤效果已包含在 ref/range 的行数估计中
  if (keyuse) {
    const KEY *key = table->key_info + keyuse->key;
    // 将 ref 访问覆盖的 keypart 对应字段加入排除集 tmp_set
    while (curr_ku->table_ref == tab->table_ref && curr_ku->key == keyuse->key &&
           curr_ku->keypart_map & keyuse->bound_keyparts) {
      bitmap_set_bit(&table->tmp_set, key->key_part[curr_ku->keypart].field->field_index());
      curr_ku++;
    }
  } else if (tab->range_scan()) {
    get_fields_used(tab->range_scan(), &table->tmp_set);
  }

  if (bitmap_is_subset(&table->cond_set, &table->tmp_set))
    goto cleanup;  // 所有条件字段都已被访问方法覆盖

  // ⚠️ 第二层：利用 range optimizer 的行数估计
  // 对未被 ref/range 覆盖的字段，如果 range optimizer 有估计值就用
  if (!table->quick_keys.is_clear_all()) {
    for (uint keyno = 0; keyno < table->s->keys; keyno++) {
      if (table->quick_keys.is_set(keyno)) {
        // 检查此索引的字段是否与已处理的字段重叠
        if (bitmap_is_overlapping(&table->tmp_set, &fields_current_quick))
          continue;  // 有重叠 → 跳过（避免重复计算）
        bitmap_union(&table->tmp_set, &fields_current_quick);
        const float selectivity =
            (float)table->quick_rows[keyno] / (float)tab->records();
        filter *= std::min(selectivity, 1.0f);  // 叠乘过滤率
      }
    }
  }

  // ⚠️ 第三层：利用 WHERE 条件的启发式估计
  // 对仍未覆盖的字段，用条件的 get_filtering_effect() 估计
  if (tab->join()->where_cond &&
      !bitmap_is_subset(&table->cond_set, &table->tmp_set)) {
    filter *= tab->join()->where_cond->get_filtering_effect(
        thd, tab->table_ref->map(), used_tables, &table->tmp_set, tab->records());
  }

  // 下限保护：至少匹配 1 行
  filter = max(filter, 1.0f / tab->records());
  // 防止 fan-out 过小
  if ((filter * fanout) < 0.05F)
    filter = 0.05F / static_cast<float>(fanout);

cleanup:
  bitmap_clear_all(&table->tmp_set);
  return filter;
}
```

> **⚠️ 难点：三层过滤率叠加**
>
> ```
> 第一层：ref/range 访问已覆盖的字段 → 排除（已体现在行数估计中）
>    ↓
> 第二层：range optimizer 有估计的其他索引字段 → 用 quick_rows/records
>    ↓
> 第三层：剩余字段 → 用 get_filtering_effect() 启发式估计
>
> 最终 filter = 第二层selectivity × 第三层selectivity
> ```
>
> 各层使用 `tmp_set` bitmap 跟踪已处理的字段，确保同一个字段不会被重复计算过滤效果。

---

### D.27 choose_table_order() — JOIN 顺序选择总入口

```cpp
// sql/sql_planner.cc:1951
// 选择最优的表连接顺序
bool Optimize_table_order::choose_table_order() {
  got_final_plan = false;

  // const 表的前缀代价初始化
  for (uint i = 0; i < join->const_tables; i++)
    (join->positions + i)->set_prefix_cost(0.0, 1.0);

  // 全是 const 表 → 无需优化
  if (join->const_tables == join->tables) {
    memcpy(join->best_positions, join->positions, ...);
    join->best_read = 1.0;
    join->best_rowcount = 1;
    got_final_plan = true;
    return false;
  }

  const bool straight_join = active_options() & SELECT_STRAIGHT_JOIN;
  table_map join_tables;

  if (emb_sjm_nest) {
    // Semi-join 物化嵌套：优先排序嵌套内的表
    merge_sort(..., Join_tab_compare_embedded_first(emb_sjm_nest));
    join_tables = emb_sjm_nest->sj_inner_tables;
  } else {
    // ⚠️ 难点：预排序策略
    if (straight_join)
      // STRAIGHT_JOIN：按依赖关系排序（保持用户指定顺序）
      merge_sort(..., Join_tab_compare_straight());
    else
      // 默认：按行数排序（少行的表优先，启发式）
      merge_sort(..., Join_tab_compare_default());

    join_tables = join->all_table_map & ~join->const_table_map;
  }

  // 设置 cond_set bitmap（用于后续 calculate_condition_filter）
  if (condition_fanout_filter_enabled && join->where_cond) {
    for (uint idx = join->const_tables; idx < join->tables; ++idx)
      bitmap_clear_all(&join->best_ref[idx]->table()->cond_set);
    join->where_cond->walk(&Item::add_field_to_cond_set_processor, ...);
  }

  // ⚠️ 核心：选择搜索策略
  if (straight_join)
    optimize_straight_join(join_tables);  // 按用户指定顺序，不搜索
  else
    greedy_search(join_tables);           // 贪心搜索

  got_final_plan = true;

  // 修正 semi-join 策略
  if (!emb_sjm_nest)
    fix_semijoin_strategies();

  return false;
}
```

> **⚠️ 难点：预排序的作用**
>
> `merge_sort()` 的预排序不是最终顺序，而是贪心搜索的**起始顺序**。排序依据：
> - `Join_tab_compare_default`：按 `found_records` 升序（行数少的表排前面）
> - 行数相同时，依赖其他表的排后面
> - 这个启发式使贪心搜索更可能先考虑小表，加速剪枝

---

### D.28 greedy_search() — 贪心搜索

```cpp
// sql/sql_planner.cc:2328
// 贪心搜索：每轮选出当前最优的表加入计划，直到所有表都加入
bool Optimize_table_order::greedy_search(table_map remaining_tables) {
  uint idx = join->const_tables;
  const uint n_tables = my_count_bits(remaining_tables);
  uint size_remain = n_tables;

  do {
    // ⚠️ 核心：调用有限深度搜索，找到当前最优的扩展
    join->best_read = DBL_MAX;
    join->best_rowcount = HA_POS_ERROR;
    if (best_extension_by_limited_search(remaining_tables, idx, search_depth))
      return true;

    // 搜索深度 >= 剩余表数 → best_positions 已包含完整最优计划
    if (size_remain <= search_depth || use_best_so_far)
      return false;

    // ⚠️ 难点：只取最优扩展的第一个表
    // best_extension_by_limited_search 可能找到了 search_depth 层深的最优路径
    // 但贪心策略只确定第一个表，然后重新搜索
    POSITION best_pos = join->best_positions[idx];
    JOIN_TAB *best_table = best_pos.table;
    join->positions[idx] = best_pos;

    // 维护 nested join 状态
    check_interleaving_with_nj(best_table);

    // 将 best_table 移到 best_ref[idx] 位置
    // （保持剩余表的 "#rows-sorted" 顺序）
    best_idx = /* find best_table in best_ref */;
    memmove(join->best_ref + idx + 1, join->best_ref + idx,
            sizeof(JOIN_TAB *) * (best_idx - idx));
    join->best_ref[idx] = best_table;

    remaining_tables &= ~(best_table->table_ref->map());
    --size_remain;
    ++idx;
  } while (true);
}
```

> **⚠️ 难点：`greedy_search()` 与 `best_extension_by_limited_search()` 的协作**
>
> ```
> greedy_search() 循环：
>   ┌──────────────────────────────────────────────┐
>   │ 第 1 轮：搜索深度 d 的最优扩展，确定表 T1      │
>   │ 第 2 轮：固定 T1，搜索深度 d 的最优扩展，确定 T2 │
>   │ ...                                          │
>   │ 第 n-d 轮：确定表 T_{n-d}                      │
>   │ 第 n-d+1 轮：剩余表 <= d，一次搜索得到完整计划   │
>   └──────────────────────────────────────────────┘
> ```
>
> `search_depth` 默认为 `min(表数, optimizer_search_depth)`。深度越大，搜索越精确但越慢。当 `search_depth >= 剩余表数` 时，等价于穷举搜索。

---

### D.29 best_extension_by_limited_search() — 有限深度搜索

```cpp
// sql/sql_planner.cc:2719
// 在给定深度内搜索最优的表连接顺序
bool Optimize_table_order::best_extension_by_limited_search(
    table_map remaining_tables,
    uint idx,                      // 当前计划长度
    uint current_search_depth) {   // 剩余搜索深度
  if (thd->killed) return true;

  double best_rowcount = DBL_MAX;
  double best_cost = DBL_MAX;
  table_map eq_ref_extended(0);

  // 保存 best_ref[] 以便回溯
  JOIN_TAB *saved_refs[MAX_TABLES];
  memcpy(saved_refs, join->best_ref + idx, ...);

  // 遍历所有候选表
  for (JOIN_TAB **pos = join->best_ref + idx; *pos && !use_best_so_far; pos++) {
    JOIN_TAB *const s = *pos;
    const table_map real_table_bit = s->table_ref->map();

    std::swap(join->best_ref[idx], *pos);

    // 检查该表是否可以加入（依赖关系、nested join 交错检查）
    if ((remaining_tables & real_table_bit) &&
        !(eq_ref_extended & real_table_bit) &&
        !(remaining_tables & s->dependent) &&
        (!idx || !check_interleaving_with_nj(s))) {

      POSITION *position = join->positions + idx;

      // ⚠️ 核心：为该表计算最优访问路径
      best_access_path(s, remaining_tables, idx, false,
                       idx ? (position - 1)->prefix_rowcount : 1.0, position);

      // 计算累计代价
      position->set_prefix_join_cost(idx, cost_model);

      // 处理 semi-join 策略
      if (has_sj) advance_sj_state(remaining_tables, s, idx);

      // ⚠️ 剪枝 1：代价剪枝
      if (position->prefix_cost >= join->best_read &&
          found_plan_with_allowed_sj) {
        backout_nj_state(remaining_tables, s);
        continue;  // 当前部分计划已比最优计划贵，剪掉
      }

      // ⚠️ 剪枝 2：启发式剪枝（prune_level=1 时启用）
      if (prune_level == 1) {
        if (best_rowcount > position->prefix_rowcount ||
            best_cost > position->prefix_cost ||
            (idx == join->const_tables && s->table() == join->sort_by_table)) {
          // 更新最优前缀估计
          if (best_rowcount >= position->prefix_rowcount &&
              best_cost >= position->prefix_cost)
          {
            best_rowcount = position->prefix_rowcount;
            best_cost = position->prefix_cost;
          }
        } else if (found_plan_with_allowed_sj) {
          backout_nj_state(remaining_tables, s);
          continue;  // 启发式剪枝
        }
      }

      // ---- 递归搜索更深层 ----
      const table_map remaining_after = remaining_tables & ~real_table_bit;
      if (current_search_depth > 1 && remaining_after) {
        // ⚠️ EQ_REF 优化：如果当前表是 eq_ref（1:1），
        // 尝试一次性扩展所有连续的 eq_ref 表
        if (prune_level == 1 && position->key != nullptr &&
            position->rows_fetched <= 1.0) {
          if (eq_ref_extended == 0) {
            eq_ref_extended = real_table_bit |
                eq_ref_extension_by_limited_search(
                    remaining_after, idx + 1, current_search_depth - 1);
            backout_nj_state(remaining_tables, s);
            if (eq_ref_extended == remaining_tables) goto done;
            continue;
          }
        } else {
          // 普通递归
          if (best_extension_by_limited_search(
                  remaining_after, idx + 1, current_search_depth - 1))
            return true;
        }
      } else {
        // 到达搜索深度底部或无剩余表 → 检查是否为最优完整计划
        if (position->prefix_cost < join->best_read ||
            (position->prefix_cost == join->best_read &&
             position->prefix_rowcount < join->best_rowcount)) {
          join->best_read = position->prefix_cost;
          join->best_rowcount = (ha_rows)position->prefix_rowcount;
          memcpy(join->best_positions, join->positions, sizeof(POSITION) * (idx + 1));
        }
      }
      backout_nj_state(remaining_tables, s);
    }
    std::swap(join->best_ref[idx], *pos);  // 回溯
  }
done:
  memcpy(join->best_ref + idx, saved_refs, ...);  // 恢复 best_ref[]
  return false;
}
```

> **⚠️ 难点：两种剪枝策略**
>
> | 剪枝 | 条件 | 效果 |
> |------|------|------|
> | 代价剪枝 | `prefix_cost >= best_read` | 当前部分计划已不如已知最优 → 跳过 |
> | 启发式剪枝 | 行数和代价都不优于已见最优前缀 | 可能错过最优解，但大幅加速搜索 |
>
> EQ_REF 优化：当连续多个表都是 eq_ref（一对一关系），它们的加入顺序不影响代价，所以可以一次性全部扩展，避免冗余搜索。

---

### D.30 Cost_model_server / Cost_model_table — 代价模型

```cpp
// sql/opt_costmodel.h:52
// Server 层代价模型：与表无关的代价常量
class Cost_model_server {
 public:
  // CPU 代价：处理 rows 行的求值代价
  double row_evaluate_cost(double rows) const {
    return rows * m_server_cost_constants->row_evaluate_cost();
    // 默认 row_evaluate_cost = 0.1
  }

  // CPU 代价：rows 次键比较
  double key_compare_cost(double keys) const {
    return keys * m_server_cost_constants->key_compare_cost();
    // 默认 key_compare_cost = 0.05
  }

  // 临时表代价
  double tmptable_create_cost(enum_tmptable_type type) const {
    // MEMORY: memory_temptable_create_cost = 1.0
    // DISK:   disk_temptable_create_cost = 20.0
  }
  double tmptable_readwrite_cost(enum_tmptable_type type,
                                 double write_rows, double read_rows) const {
    // MEMORY: (write + read) × memory_temptable_row_cost (= 0.1)
    // DISK:   (write + read) × disk_temptable_row_cost (= 0.5)
  }

 private:
  const Cost_model_constants *m_cost_constants;
  const Server_cost_constants *m_server_cost_constants;
};

// sql/opt_costmodel.h:240
// Table 层代价模型：与特定表/存储引擎相关的代价常量
class Cost_model_table {
 public:
  // 委托给 server 层（CPU 代价与表无关）
  double row_evaluate_cost(double rows) const {
    return m_cost_model_server->row_evaluate_cost(rows);
  }

  // ⚠️ I/O 代价：与存储引擎相关
  double io_block_read_cost(double blocks) const {
    return blocks * m_se_cost_constants->io_block_read_cost();
    // 默认 io_block_read_cost = 1.0（磁盘随机读）
  }

  double buffer_block_read_cost(double blocks) const {
    return blocks * m_se_cost_constants->memory_block_read_cost();
    // 默认 memory_block_read_cost = 0.25（内存读）
  }

  // 综合 I/O 代价：根据 buffer pool 命中率混合计算
  double page_read_cost(double pages) const;
  // ≈ pages_in_memory × memory_block_read_cost +
  //   pages_on_disk × io_block_read_cost

 private:
  const Cost_model_server *m_cost_model_server;
  const SE_cost_constants *m_se_cost_constants;
  const TABLE *m_table;
};
```

> **⚠️ 难点：代价常量的层次结构**
>
> ```
> mysql.server_cost 表          mysql.engine_cost 表
>       ↓                              ↓
> Server_cost_constants         SE_cost_constants
>       ↓                              ↓
> Cost_model_server ──────────→ Cost_model_table
>   row_evaluate_cost()           io_block_read_cost()
>   key_compare_cost()            buffer_block_read_cost()
>   tmptable_*_cost()             page_read_cost()
> ```
>
> 代价常量存储在 `mysql.server_cost` 和 `mysql.engine_cost` 系统表中，可以通过 `ALTER` 命令调整。修改后需 `FLUSH OPTIMIZER_COSTS` 生效。不同存储引擎可以有不同的 I/O 代价常量。

---

### D.31 read_range_first() — 存储引擎索引范围定位

```cpp
// sql/handler.cc:7314
// 定位到第一条满足范围条件的索引记录
int handler::read_range_first(const key_range *start_key,  // 起始键（null=从头开始）
                              const key_range *end_key,    // 结束键（null=到末尾）
                              bool eq_range_arg,           // start_key == end_key？
                              bool sorted) {
  eq_range = eq_range_arg;
  set_end_range(end_key, RANGE_SCAN_ASC);       // 保存结束键供后续 read_range_next 用
  range_key_part = table->key_info[active_index].key_part;

  int result;
  if (!start_key)
    // 无起始键 → 读索引第一条记录
    result = ha_index_first(table->record[0]);
  else
    // ⚠️ 核心：用 start_key->flag 决定定位方式
    // flag 来自 make_min_endpoint()，可能是：
    //   HA_READ_KEY_EXACT   → 精确匹配
    //   HA_READ_KEY_OR_NEXT → >= value
    //   HA_READ_AFTER_KEY   → > value
    result = ha_index_read_map(table->record[0],
                               start_key->key,
                               start_key->keypart_map,
                               start_key->flag);

  if (result) return (result == HA_ERR_KEY_NOT_FOUND) ? HA_ERR_END_OF_FILE : result;

  // 检查找到的记录是否在结束范围内
  if (compare_key(end_range) > 0) {
    unlock_row();                   // 超出范围，释放行锁
    result = HA_ERR_END_OF_FILE;
  }
  return result;
}
```

> **⚠️ 难点：`start_key->flag` 如何决定 `ha_index_read_map()` 的行为**
>
> `ha_index_read_map()` 将 flag 传给存储引擎（如 InnoDB），引擎据此在 B+ 树上定位：
>
> | flag | B+ 树操作 | 对应 SQL |
> |------|----------|---------|
> | `HA_READ_KEY_EXACT` | 找到精确匹配的第一条 | `WHERE col = 5` |
> | `HA_READ_KEY_OR_NEXT` | 找到 >= value 的第一条 | `WHERE col >= 5` |
> | `HA_READ_AFTER_KEY` | 找到 > value 的第一条 | `WHERE col > 5` |
>
> 之后通过 `read_range_next()` 循环读取后续记录，每次用 `compare_key(end_range)` 检查是否超出结束范围。

---

### D.32 convert_search_mode_to_innobase() — InnoDB B+ 树定位模式转换

```cpp
// storage/innobase/handler/ha_innodb.cc:10096
// 将 MySQL server 层的 ha_rkey_function 转换为 InnoDB 的 page_cur_mode_t
page_cur_mode_t convert_search_mode_to_innobase(ha_rkey_function find_flag) {
  switch (find_flag) {
    case HA_READ_KEY_EXACT:
      // ⚠️ 难点：EXACT 和 OR_NEXT 都映射到 PAGE_CUR_GE
      // 因为 InnoDB 的 B+ 树只有 GE/G/LE/L 四种定位模式
      // EXACT 在 GE 定位后，由上层（handler）检查是否精确匹配
    case HA_READ_KEY_OR_NEXT:
      return PAGE_CUR_GE;       // >= value

    case HA_READ_AFTER_KEY:
      return PAGE_CUR_G;        // > value

    case HA_READ_BEFORE_KEY:
      return PAGE_CUR_L;        // < value（反向扫描结束位置）

    case HA_READ_KEY_OR_PREV:
    case HA_READ_PREFIX_LAST:
    case HA_READ_PREFIX_LAST_OR_PREV:
      return PAGE_CUR_LE;       // <= value

    // 空间索引
    case HA_READ_MBR_CONTAIN:    return PAGE_CUR_CONTAIN;
    case HA_READ_MBR_INTERSECT:  return PAGE_CUR_INTERSECT;
    case HA_READ_MBR_WITHIN:     return PAGE_CUR_WITHIN;
    case HA_READ_MBR_DISJOINT:   return PAGE_CUR_DISJOINT;
    case HA_READ_MBR_EQUAL:      return PAGE_CUR_MBR_EQUAL;

    case HA_READ_PREFIX:
    case HA_READ_INVALID:
      return PAGE_CUR_UNSUPP;   // InnoDB 不支持
  }
  return PAGE_CUR_UNSUPP;
}
```

> **⚠️ 难点：为什么 `HA_READ_KEY_EXACT` 和 `HA_READ_KEY_OR_NEXT` 都映射到 `PAGE_CUR_GE`？**
>
> InnoDB 的 B+ 树游标定位只有四种基本模式（GE、G、LE、L），没有"精确匹配"模式。对于 `HA_READ_KEY_EXACT`：
>
> 1. InnoDB 先用 `PAGE_CUR_GE` 定位到 >= value 的第一条记录
> 2. 然后 server 层的 `handler::ha_index_read_map()` 检查找到的记录是否精确等于 value
> 3. 如果不等于，返回 `HA_ERR_KEY_NOT_FOUND`
>
> 这个两步过程对上层完全透明。`HA_READ_KEY_OR_NEXT`（`>= value`）则直接使用 GE 定位的结果，不需要额外检查。
>
> **完整的从 SQL 到 B+ 树的定位链路：**
>
> ```
> WHERE col >= 5
>   → make_min_endpoint(): flag = HA_READ_KEY_OR_NEXT
>     → read_range_first(): ha_index_read_map(HA_READ_KEY_OR_NEXT)
>       → convert_search_mode_to_innobase(): PAGE_CUR_GE
>         → btr_cur_search_to_nth_level(): 在 B+ 树中定位到 >= 5 的第一个叶子节点
> ```
