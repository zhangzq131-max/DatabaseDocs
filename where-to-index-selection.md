# MySQL 8.0.41 WHERE 条件到最优索引选择全流程解析

> 本文基于 MySQL 8.0.41 源码，深入分析 `SELECT` 语句中 `WHERE` 条件从解析到最终选定最优索引的完整过程。

## 目录

- [1. 总览：优化流程全景图](#1-总览优化流程全景图)
- [2. WHERE 条件的内部表示（Item 体系）](#2-where-条件的内部表示item-体系)
- [3. 条件规范化与等值传播](#3-条件规范化与等值传播)
- [4. 条件下推到各表](#4-条件下推到各表)
- [5. Range Optimizer：从条件构建范围树](#5-range-optimizer从条件构建范围树)
- [6. Range 分析的两阶段设计](#6-range-分析的两阶段设计)
- [7. 候选索引评估与代价估算](#7-候选索引评估与代价估算)
- [8. 最优访问路径选择（best_access_path）](#8-最优访问路径选择best_access_path)
- [9. Join 顺序优化与贪心搜索](#9-join-顺序优化与贪心搜索)
- [10. 关键数据结构速查](#10-关键数据结构速查)
- [11. 端到端示例](#11-端到端示例)

---

## 1. 总览：优化流程全景图

```
SQL 文本
  │
  ▼
Parser（解析器）
  │  生成 Item 表达式树
  ▼
JOIN::optimize()                           ← sql/sql_optimizer.cc:337
  │
  ├─ optimize_cond()                       ← 条件规范化（等值传播、常量传播、消除恒真/假）
  │   ├─ build_equal_items()               ← 构建多重等值谓词 Item_equal
  │   ├─ propagate_cond_constants()        ← 常量传播 (field=field → field=const)
  │   └─ remove_eq_conds()                 ← 移除恒真/恒假条件
  │
  ├─ make_join_plan()                      ← 制定 JOIN 计划
  │   ├─ update_ref_and_keys()             ← 收集可用于 ref 访问的 Key_use 结构
  │   │
  │   ├─ estimate_rowcount()               ← 【第一阶段】Range 分析（仅常量条件）
  │   │   └─ get_quick_record_count()
  │   │       └─ test_quick_select(prev_tables=0, read_tables=0)
  │   │           ├─ get_mm_tree()         ← 跨表条件被跳过或标记为 MAYBE_KEY
  │   │           ├─ get_key_scans_params()← 评估各索引的范围扫描代价
  │   │           └─ → 输出: tab->range_scan(), tab->found_records, tab->needed_reg
  │   │
  │   └─ choose_table_order()              ← 确定表的 JOIN 顺序（复用第一阶段结果）
  │       └─ greedy_search()
  │           └─ best_extension_by_limited_search()
  │               └─ best_access_path()    ← 为每个表选择最优访问方式
  │                   ├─ find_best_ref()    ← 评估 ref 访问（用 rec_per_key，轻量）
  │                   └─ calculate_scan_cost()  ← 直接读取 tab->range_scan() 的代价
  │
  ├─ make_join_query_block()               ← 将条件下推到各表
  │   ├─ make_cond_for_table()             ← 提取适用于特定表的条件
  │   └─ 【第二阶段】若 needed_reg 非空，重新运行 Range 分析
  │       └─ test_quick_select(prev_tables=已确定的前置表, ...)
  │           └─ → 此时跨表条件可被利用，产生更精确的范围扫描
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
| `sql/opt_costmodel.h` / `opt_costmodel.cc` | 代价模型 |
| `sql/item_cmpfunc.h` | 条件 Item 类层次 |
| `sql/handler.h` / `handler.cc` | 存储引擎接口（代价估算、统计信息） |

---

## 2. WHERE 条件的内部表示（Item 体系）

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

### 2.2 Item_cond：AND/OR 条件

`Item_cond` 类是 AND 和 OR 逻辑运算的基类，它持有一个子 Item 列表（`List<Item> list`）。

```cpp
// sql/item_cmpfunc.h:2427
class Item_cond : public Item_bool_func {
  List<Item> list;   // 子条件列表
  ...
};
```

- `Item_cond_and`：表示 `cond1 AND cond2 AND ...`
- `Item_cond_or`：表示 `cond1 OR cond2 OR ...`

嵌套的 AND/OR 会在 `fix_fields()` 阶段被**扁平化** —— 例如 `(a AND (b AND c))` 变为 `(a AND b AND c)`。

### 2.3 Item_equal：多重等值谓词

`Item_equal` 是 MySQL 独有的一种特殊 Item，用于表示**多重等值关系**。例如 `a=b AND b=c` 会被合并为一个 `Item_equal(a, b, c)`。

```cpp
// sql/item_cmpfunc.h:2567
class Item_equal final : public Item_bool_func {
  List<Item_field> fields;   // 等值的字段集合
  Item *m_const_arg;         // 若有常量，存在这里
  ...
};
```

### 2.4 COND_EQUAL：等值条件容器

```cpp
// sql/item_cmpfunc.h:2707
struct COND_EQUAL {
  List<Item_equal> current_level;  // 当前 AND 层的 Item_equal 列表
  COND_EQUAL *upper_levels;        // 外层 AND 的 COND_EQUAL 指针
};
```

`Item_cond_and` 内嵌了一个 `COND_EQUAL`，其中 `current_level` 存储在该 AND 层级提取出来的所有 `Item_equal` 对象。

---

## 3. 条件规范化与等值传播

### 3.1 入口：optimize_cond()

```cpp
// sql/sql_optimizer.cc:10338
bool optimize_cond(THD *thd, Item **cond, COND_EQUAL **cond_equal,
                   mem_root_deque<Table_ref *> *join_list,
                   Item::cond_result *cond_value);
```

该函数执行三个关键转换步骤：

#### 步骤 1：等值传播（equality_propagation）

```cpp
// sql/sql_optimizer.cc:10374
build_equal_items(thd, *cond, cond, nullptr, true, join_list, cond_equal);
```

**目标**：识别形如 `a=b` 的等值条件，将它们合并为 `Item_equal` 对象。

**过程**（`build_equal_items_for_cond()`）：
1. 递归遍历条件树
2. 在每个 **AND 层级**，通过 `check_equality()` 检测等值谓词
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

#### 步骤 2：常量传播（constant_propagation）

```cpp
// sql/sql_optimizer.cc:10390
propagate_cond_constants(thd, nullptr, *cond, *cond);
```

将 `field = const` 的知识传播到其他使用该 field 的条件中。例如：
- `a = 5 AND a + b > 10` → `5 + b > 10`（简化后续条件）

#### 步骤 3：消除恒真/恒假条件（trivial_condition_removal）

```cpp
// sql/sql_optimizer.cc:10408
remove_eq_conds(thd, *cond, cond, cond_value);
```

- 移除 `1=1` 这种恒真条件
- 如果检测到 `1=0` 这种恒假条件，标记整个查询返回空集

### 3.2 后续：字段替换

在确定 JOIN 顺序后，`substitute_for_best_equal_field()` 会将 `Item_equal` 对象转换回普通的等值谓词 (`Item_func_eq`)，并选择最优的字段作为比较基准（优先选择已经通过索引定位的表的字段）。

```cpp
// sql/sql_optimizer.cc:4774
static Item *substitute_for_best_equal_field(THD *thd, Item *cond,
                                             COND_EQUAL *cond_equal,
                                             JOIN_TAB **table_join_idx);
```

---

## 4. 条件下推到各表

### 4.1 make_cond_for_table()

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

### 4.2 下推流程

```
make_join_query_block()                     ← sql/sql_optimizer.cc:9583
  │
  ├─ 提取常量条件（不依赖任何表的条件）
  │
  └─ 对 JOIN 顺序中的每个表：
      └─ make_cond_for_table(cond, available_tables, current_table)
          → 提取可以在该表评估的条件
```

---

## 5. Range Optimizer：从条件构建范围树

Range Optimizer 是索引选择的核心组件，它将 WHERE 条件转换为索引上的**区间范围**，并评估各种范围扫描的代价。

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
  │     scan_time = row_evaluate_cost(records) + 1
  │     cost_est = table_scan_cost() + 1.1 (IO) + scan_time (CPU)
  │
  ├─ 2. 若有覆盖索引，计算最短覆盖索引扫描代价
  │     key_read_time = index_scan_cost(key, 1, records) + row_evaluate_cost(records)
  │     若更优则更新 cost_est
  │
  ├─ 3. 调用 get_mm_tree() 构建范围树
  │     tree = get_mm_tree(thd, &param, ..., cond)
  │
  ├─ 4. 若 tree->type == IMPOSSIBLE → 返回 -1（不可能有结果）
  │     若 tree->type != KEY → 置 tree = nullptr（无法用于范围扫描）
  │
  ├─ 5. 评估 Group Index Skip Scan → get_best_group_min_max()
  │
  ├─ 6. 评估 Index Skip Scan → get_best_skip_scan()
  │
  ├─ 7. 评估各索引的范围扫描 → get_key_scans_params()
  │     对每个索引调用 check_quick_select() 估算行数和代价
  │
  ├─ 8. 评估 ROR 索引交集 → get_best_ror_intersect()
  │
  ├─ 9. 评估 Index Merge Union → get_best_disjunct_quick()
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
      ├─ BETWEEN: get_full_func_mm_tree()
      ├─ IN:      get_full_func_mm_tree()
      ├─ EQ/LT/LE/GT/GE/NE: get_full_func_mm_tree()
      │   └─ get_mm_parts()
      │       └─ get_mm_leaf()  ← 创建单个 SEL_ARG
      └─ 其他: return nullptr
```

### 5.3 核心函数链

#### get_mm_parts() — 为字段找到所有匹配的索引键部分

```cpp
// sql/range_optimizer/range_analysis.cc:~1077
static SEL_TREE *get_mm_parts(THD *thd, RANGE_OPT_PARAM *param,
                              table_map prev_tables, table_map read_tables,
                              Item_func *cond_func, Field *field,
                              Item_func::Functype type, Item *value);
```

遍历所有候选索引，找到包含该字段的索引键部分（key part），为每个匹配的索引键部分调用 `get_mm_leaf()` 创建范围条件。

#### get_mm_leaf() — 创建单个范围节点

```cpp
// sql/range_optimizer/range_analysis.cc:~1138
static SEL_ROOT *get_mm_leaf(THD *thd, RANGE_OPT_PARAM *param,
                             Item *cond_func, Field *field,
                             KEY_PART *key_part,
                             Item_func::Functype type, Item *value,
                             bool *inexact);
```

根据比较操作类型创建对应的 `SEL_ARG` 节点：

| 操作 | 生成的范围 |
|------|-----------|
| `col = 5` | `[5, 5]` (EQ_RANGE) |
| `col > 5` | `(5, +∞)` |
| `col >= 5` | `[5, +∞)` |
| `col < 5` | `(-∞, 5)` |
| `col BETWEEN 3 AND 7` | `[3, 7]` |
| `col IN (1,3,5)` | `[1,1] ∪ [3,3] ∪ [5,5]` |
| `col IS NULL` | `[NULL, NULL]` |

### 5.4 SEL_TREE / SEL_ARG 数据结构

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
  Field *field;                    // 对应字段
  uchar *min_value, *max_value;   // 范围端点值

  SEL_ARG *left, *right;          // 红黑树子节点
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

#### tree_and() — AND 合并

```cpp
// sql/range_optimizer/tree.cc:503
SEL_TREE *tree_and(RANGE_OPT_PARAM *param, SEL_TREE *tree1, SEL_TREE *tree2);
```

- 对每个索引：调用 `key_and()` 合并对应的 SEL_ROOT
- `key_and()` 在**相同键部分**上做**区间交集**，在**不同键部分**上通过 `next_key_part` 做 AND 连接

#### tree_or() — OR 合并

```cpp
// sql/range_optimizer/tree.cc:680
SEL_TREE *tree_or(RANGE_OPT_PARAM *param, bool remove_jump_scans,
                  SEL_TREE *tree1, SEL_TREE *tree2);
```

- 对每个**共有索引**：调用 `key_or()` 合并对应的 SEL_ROOT
- `key_or()` 在**相同键部分**上做**区间并集**
- 不共有的索引会被丢弃（OR 要求两边都满足，不能只用一边的索引）

**关键区别**：
- AND：可以保留所有索引的范围（交集越多越精确）
- OR：只能保留两边都有范围的索引（并集可能退化为全表扫描）

---

## 6. Range 分析的两阶段设计

Range Optimizer（`test_quick_select`）在 `make_join_plan` 中被调用时，`choose_table_order` 尚未执行，join 顺序未知。这意味着形如 `t1.a = t2.b` 的跨表条件无法确定依赖关系是否满足。MySQL 采用**"预计算 + 延迟精化"**的两阶段设计来解决这一循环依赖。

### 6.1 第一阶段：仅常量条件的 Range 分析

**调用位置**：`make_join_plan()` → `estimate_rowcount()` → `get_quick_record_count()`

```cpp
// sql/sql_optimizer.cc:6236
int error = test_quick_select(
    thd, thd->mem_root, &temp_mem_root, keys_to_use,
    0,    // prev_tables = 0：不假设任何表已被读取
    0,    // read_tables = 0：不假设任何表的值可用
    limit, false, ORDER_NOT_RELEVANT, tab->table(), ...
    condition, &tab->needed_reg, ...);
```

`prev_tables` 和 `read_tables` 都传 **0**，这决定了本阶段只能处理**不依赖任何其他表**的条件。

**跨表条件的过滤机制**在 `get_mm_parts()` 中实现：

```cpp
// sql/range_optimizer/range_analysis.cc:1087
// value 引用了不可用的表 → 直接跳过该条件
if (value && value->used_tables() & ~(prev_tables | read_tables))
    return nullptr;
```

由于 `prev_tables | read_tables = 0`，任何 `value` 引用了其他表的条件（如 `t1.a = t2.b` 中的 `t2.b`）都会被跳过。

**MAYBE_KEY 机制**：对于"字段属于当前表但值依赖其他表"的条件，Range Optimizer 不会直接丢弃，而是标记为 `MAYBE_KEY`：

```cpp
// sql/range_optimizer/range_analysis.cc:1103-1124
if (!value || !(value->used_tables() & ~read_tables)) {
    // 值是常量或只引用已读取的表 → 创建真正的范围节点
    sel_root = get_mm_leaf(...);
} else {
    // 值引用了尚未读取的表 → 标记为 MAYBE_KEY（将来可能可用）
    sel_root = new SEL_ROOT(param->temp_mem_root, SEL_ROOT::Type::MAYBE_KEY);
}
```

`MAYBE_KEY` 随后在 `get_key_scans_params()` 中被记录到 `needed_reg` 位图：

```cpp
// sql/range_optimizer/index_range_scan_plan.cc:859
if (key->type == SEL_ROOT::Type::MAYBE_KEY || key->root->maybe_flag)
    needed_reg->set_bit(keynr);  // 通知后续阶段：该索引可能在确定 join 顺序后变得可用
```

**本阶段的输出**保存在 `JOIN_TAB` 上，供后续 `choose_table_order` 直接读取：
- `tab->range_scan()`：最优范围扫描的 AccessPath（含代价）
- `tab->found_records`：估算的行数
- `tab->needed_reg`：依赖其他表的候选索引位图

### 6.2 choose_table_order 如何复用第一阶段结果

`choose_table_order()` 的搜索过程通过 `best_access_path()` 为每个候选位置的表评估代价。在搜索循环内部，`best_access_path` **直接读取**第一阶段的预计算结果，而非重新运行 Range Optimizer：

```cpp
// sql/sql_planner.cc:821-836  calculate_scan_cost() 内部
if (tab->range_scan()) {
    // 直接使用第一阶段存储的 range 代价和行数，零额外 I/O
    scan_and_filter_cost = prefix_rowcount * (tab->range_scan()->cost + ...);
}
```

对于跨表条件（如 `t1.a = t2.b`），搜索过程通过 `find_best_ref()` 评估 ref 访问。ref 访问使用内存中的 `rec_per_key` 统计信息估算行数，无需调用存储引擎，因此可以在搜索循环中反复执行。

### 6.3 第二阶段：join 顺序确定后的 Range 精化

**调用位置**：`make_join_query_block()`，在 join 顺序已确定之后。

首先检查是否需要重新分析。若 `needed_reg` 为空（无跨表依赖），保留第一阶段结果：

```cpp
// sql/sql_optimizer.cc:9720-9729
if (tab->range_scan()) {
    if (tab->needed_reg.is_clear_all() && tab->type() != JT_CONST) {
        // needed_reg 为空 → 保留第一阶段的 range_scan，无需重新分析
    } else {
        destroy(tab->range_scan());  // 丢弃旧结果，准备重新分析
        tab->set_range_scan(nullptr);
    }
}
```

若存在非常量条件的索引（`tab->keys() != tab->const_keys`）且不是 join 中第一个表（`i > 0`），重新运行 Range Optimizer：

```cpp
// sql/sql_optimizer.cc:9905-9916
test_quick_select(
    thd, thd->mem_root, &temp_mem_root, usable_keys,
    used_tables & ~tab->table_ref->map(),  // prev_tables = 当前表之前的所有表
    0, limit, false, interesting_order, tab->table(), ...
    tab->condition(), &tab->needed_reg, ...);
```

此时 `prev_tables` 包含了 join 顺序中**当前表之前的所有表**。若 `t2` 排在 `t1` 前面，条件 `t1.a = t2.b` 中的 `t2.b` 在运行时将可用，Range Optimizer 就能为 `t1.a` 构建有效的范围节点。

### 6.4 两阶段对比

| | 第一阶段（`estimate_rowcount`） | 第二阶段（`make_join_query_block`） |
|---|---|---|
| **时机** | `choose_table_order` **之前** | `choose_table_order` **之后** |
| **`prev_tables`** | `0`（无前置表） | 已确定的前置表集合 |
| **可用条件** | 仅常量条件（如 `col = 5`） | 含跨表条件（如 `t1.a = t2.b`） |
| **跨表条件处理** | 跳过或标记为 `MAYBE_KEY` | 可构建真正的范围节点 |
| **调用次数** | 每个非 const 表各 1 次 | 仅对 `needed_reg` 非空的表调用 |
| **结果用途** | 被 `best_access_path` 在搜索中反复引用 | 直接用于最终执行计划 |

### 6.5 为何不合并为一次

将 Range 分析推迟到 `choose_table_order` 内部（即在搜索循环中按需运行）理论上可行，但工程代价极高：

**1. 搜索循环中 `best_access_path` 的调用次数极多**

`choose_table_order` 的搜索复杂度为：
- N ≤ 7 个表：穷举搜索，O(N!) 次调用 — 7 个表 = **5,040 次**
- N > 7 个表：贪心搜索（depth=7），O(N^7) 量级

每次调用中若重新运行完整的 Range 分析，包括 `get_mm_tree` 构建范围树、对每个候选索引调用 `check_quick_select`，开销不可承受。

**2. Range 分析涉及存储引擎 I/O**

Range Optimizer 的核心代价估算依赖 `handler::records_in_range()`，这是一个 InnoDB B+ 树 dive 操作：

```cpp
// sql/range_optimizer/rowid_ordered_retrieval_plan.cc:540
records = table->file->records_in_range(scan->keynr, &min_range, &max_range);
```

这不是纯内存计算，而是需要实际遍历索引页。假设 10 个表、每表 5 个候选索引、每索引 3 个范围区间：
- 两阶段设计：10 × 15 = **150 次** B+ 树 dive（第一阶段）+ 少量重新分析
- 合并为一次：3,628,800 × 15 = **~5400 万次** B+ 树 dive（10! 排列 × 15 dive/表）

**3. ref 访问已覆盖跨表条件的评估**

跨表等值条件（`t1.a = t2.b`）在搜索过程中主要通过 `find_best_ref()` 评估 ref 访问方式。ref 评估使用内存中的 `rec_per_key` 统计信息，无需调用存储引擎，可以在 O(N!) 循环内高效执行。Range Optimizer 对跨表条件的增量价值有限，不值得在搜索循环中重复运行。

**4. 第一阶段的粗略估算已足够引导搜索**

纯常量条件（如 `col = 5`、`col BETWEEN 3 AND 7`）通常已足以区分一个表是"大表"还是"小表"，某个索引是否高效。搜索算法只需要**相对准确的行数估算**来做出合理的排序决策，而非精确值。

---

## 7. 候选索引评估与代价估算

### 7.1 代价模型

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
  double buffer_block_read_cost(double blocks);   // IO: 从缓冲池读取代价
};
```

**代价组成** = **IO 代价**（从磁盘/缓冲池读取页面）+ **CPU 代价**（评估条件、比较 key）

### 7.2 索引统计信息：rec_per_key

```cpp
// sql/key.h:214-249
class KEY {
  bool has_records_per_key(uint key_part_no);
  rec_per_key_t records_per_key(uint key_part_no);  // 每个不同 key 值对应的平均行数
};
```

`rec_per_key` 是选择率估算的核心统计量：
- `records_per_key(0)` = 匹配第一个键部分的平均行数
- `records_per_key(N-1)` = 匹配全部 N 个键部分的平均行数
- 例如：索引 `(city, name)` 上 `records_per_key(0) = 1000`（每个 city 约 1000 行），`records_per_key(1) = 2`（每个 city+name 约 2 行）

### 7.3 Range 扫描行数估算

```cpp
// sql/range_optimizer/index_range_scan_plan.cc:568
ha_rows check_quick_select(THD *thd, RANGE_OPT_PARAM *param, uint idx,
                           bool index_only, SEL_ROOT *tree,
                           bool update_tbl_stats, enum_order order_direction,
                           bool skip_records_in_range, uint *mrr_flags,
                           uint *bufsize, Cost_estimate *cost,
                           bool *is_ror_scan, bool *is_imerge_scan);
```

1. 将 SEL_ARG 图转换为连续的 key 区间序列
2. 调用存储引擎的 `multi_range_read_info_const()` 估算每个区间的行数
3. 底层调用 `handler::records_in_range()` → InnoDB 通过 B+树 dive 估算
4. 汇总所有区间的行数和 IO 代价

### 7.4 Ref 访问代价估算

```cpp
// sql/sql_planner.cc:143
double find_cost_for_ref(const THD *thd, TABLE *table, unsigned keyno,
                         double num_rows, double worst_seeks);
```

根据索引类型使用不同的代价公式：

| 场景 | 代价计算 |
|------|---------|
| 覆盖索引 | `index_scan_cost(keyno, 1, num_rows)` |
| 聚簇主键 | `read_cost(keyno, 1, num_rows)` |
| 普通二级索引 | `min(page_read_cost(keyno, num_rows), worst_seeks)` |

### 7.5 Ref 访问行数估算（fanout）

`find_best_ref()` 中行数估算的优先级：

```
1. 若索引为 UNIQUE 且所有键部分都有等值条件 → fanout = 1
2. 尝试复用 Range Optimizer 的估算（ReuseRangeEstimateForRef-1/2/3/4）
3. 使用 rec_per_key 统计信息
4. 若无统计信息，使用启发式公式：
   records = (x * (b-a) + a*c - b) / (c-1)
   其中：
   b = 全键匹配的行数
   a = 首键部分匹配的行数（总行数的 1%）
   c = 索引键部分总数
   x = 实际使用的键部分数
```

---

## 8. 最优访问路径选择（best_access_path）

### 8.1 函数签名

```cpp
// sql/sql_planner.cc:981
void Optimize_table_order::best_access_path(
    JOIN_TAB *tab,
    const table_map remaining_tables,
    const uint idx,
    bool disable_jbuf,
    const double prefix_rowcount,  // 前缀行数（之前表的组合数）
    POSITION *pos                  // 输出：选定的访问方式
);
```

### 8.2 决策流程

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
  ├─ 第二阶段：决定是否需要评估 scan
  │   通过 4 个启发式规则判断：
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
  │   scan_total = scan_read_cost + row_evaluate_cost(prefix_rows * scan_rows)
  │   ref_total  = best_read_cost + row_evaluate_cost(prefix_rows * ref_rows)
  │   选择代价更低的方案
  │
  └─ 输出到 POSITION:
      pos->rows_fetched    = 预估行数
      pos->read_cost       = 访问代价（不含行评估代价）
      pos->filter_effect   = 非访问条件的过滤率
      pos->key             = 所用 Key_use（scan 为 nullptr）
```

### 8.3 条件过滤率（Condition Filtering）

在确定访问方式后，还会计算**条件过滤效果**：

```cpp
filter_effect = calculate_condition_filter(
    tab, best_ref, available_tables, rows_fetched, false, false, trace);
```

这量化了通过索引访问获得的行中，还有多少比例能通过其余 WHERE 条件。例如：
- 索引访问通过 `city='Beijing'` 选出 1000 行
- WHERE 还有 `AND age > 30`，假设通过 30% 的行
- 则 `filter_effect = 0.3`，最终预估行数 = `1000 * 0.3 = 300`

---

## 9. Join 顺序优化与贪心搜索

### 9.1 入口

```cpp
// sql/sql_planner.cc:1951
bool Optimize_table_order::choose_table_order();
```

- 若使用 `STRAIGHT_JOIN`：按 FROM 指定顺序，调用 `optimize_straight_join()`
- 否则：调用 `greedy_search()` 进行代价驱动的搜索

### 9.2 贪心搜索算法

```cpp
// sql/sql_planner.cc:2328
bool Optimize_table_order::greedy_search(table_map remaining_tables);
```

**核心逻辑**：

```
greedy_search():
  while (还有剩余表):
    // 从当前位置向前搜索 search_depth 层
    best_extension_by_limited_search(remaining, idx, search_depth)

    // 选择搜索到的最优下一个表
    best_table = best_positions[idx]
    remaining -= best_table

    // 固定这个选择，继续下一轮
    idx++
```

### 9.3 深度优先搜索（带剪枝）

```cpp
// sql/sql_planner.cc:2719
bool Optimize_table_order::best_extension_by_limited_search(
    table_map remaining_tables, uint idx, uint current_search_depth);
```

```
best_extension_by_limited_search(remaining, idx, depth):
  for each table T in remaining:
    // 为 T 选择最优访问路径
    best_access_path(T, remaining, idx, ...)

    // 代价剪枝：若累积代价已超过当前最优，跳过
    if (current_cost > best_cost_so_far) continue;

    // 启发式剪枝 (prune_level=1)：
    // 若已有更好的表在此位置，跳过
    if (current_cost > best_at_this_position) continue;

    if (depth > 1 && remaining has more tables):
      // 递归搜索更深层
      best_extension_by_limited_search(remaining - T, idx+1, depth-1)
    else:
      // 到达搜索深度限制，评估当前完整计划
      consider_plan(idx)
```

### 9.4 搜索深度

```cpp
// sql/sql_planner.cc:2078
uint Optimize_table_order::determine_search_depth();
```

- **≤ 7 个表**：穷举搜索（`depth = table_count + 1`）
- **> 7 个表**：贪心搜索（`depth = 7`）

---

## 10. 关键数据结构速查

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
  // 由 find_best_ref 设置：
  key_part_map bound_keyparts;  // 已绑定的键部分位图
  double fanout;                // 估算的匹配行数
  double read_cost;             // 估算的读取代价
};
```

### RANGE_OPT_PARAM — Range Optimizer 的参数

```cpp
struct RANGE_OPT_PARAM {
  TABLE *table;
  Query_block *query_block;
  KEY_PART *key_parts;      // 所有候选键部分的数组
  uint *real_keynr;         // param 索引号到真实索引号的映射
  uint keys;                // 候选索引数
  MEM_ROOT *temp_mem_root;  // 临时内存分配器
  MEM_ROOT *return_mem_root;// 返回结果用的内存分配器
};
```

### AccessPath — 统一的访问路径描述

Range Optimizer 返回 `AccessPath` 对象来描述选定的范围扫描：
- `AccessPath::INDEX_RANGE_SCAN` — 单索引范围扫描
- `AccessPath::INDEX_MERGE` — Index Merge
- `AccessPath::ROWID_INTERSECTION` — ROR 交集
- `AccessPath::ROWID_UNION` — ROR 并集
- `AccessPath::GROUP_INDEX_SKIP_SCAN` — Group 松散索引扫描
- `AccessPath::INDEX_SKIP_SCAN` — 索引跳跃扫描

---

## 11. 端到端示例

假设有表：

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
```

查询：

```sql
SELECT * FROM orders
WHERE customer_id = 100
  AND status = 'paid'
  AND amount > 500
  AND created_at > '2024-01-01';
```

### 阶段 1：条件规范化

`optimize_cond()` 处理后：
- 四个简单等值/比较条件，无需等值传播
- 条件保持原样（无恒真/恒假条件）

### 阶段 2：Range Optimizer 构建范围树

`get_mm_tree()` 递归处理 AND 中的每个条件：

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

### 阶段 3：评估各索引范围扫描代价

`get_key_scans_params()` 对每个非 NULL 的索引调用 `check_quick_select()`：

| 索引 | 估算行数 | 代价 |
|------|---------|------|
| `idx_customer` | 50 (InnoDB dive) | 51.1 |
| `idx_status_amount` | 20 (InnoDB dive) | 21.1 |
| `idx_created` | 5000 (InnoDB dive) | 2501.1 |

Range 最优选择：**`idx_status_amount`**（代价 21.1）

### 阶段 4：评估 ref 访问

`find_best_ref()` 通过 `Key_use` 列表评估：

| 索引 | 可用键部分 | fanout | read_cost |
|------|-----------|--------|-----------|
| `idx_customer`（kp0 = 100） | 1 | 50 (rec_per_key) | 25.5 |
| `idx_status_amount`（kp0 = 'paid'） | 1 | 200 (rec_per_key) | 101.1 |

ref 最优选择：**`idx_customer`**（fanout=50, cost=25.5）

### 阶段 5：best_access_path 最终对比

```
ref 总代价 = 25.5 + row_evaluate_cost(1 * 50) = 25.5 + 5.0 = 30.5
range 总代价 (idx_status_amount) = 21.1 + row_evaluate_cost(1 * 20) = 21.1 + 2.0 = 23.1
```

**最终选择：`idx_status_amount` 的范围扫描**（代价 23.1 < 30.5）

EXPLAIN 输出将显示：
```
type: range, key: idx_status_amount, rows: 20, Extra: Using index condition
```

剩余条件 `customer_id = 100 AND created_at > '2024-01-01'` 作为 **filter condition** 在获取行后评估。

---

## 附录：关键源码位置速查

| 功能 | 文件 | 函数 | 行号 |
|------|------|------|------|
| JOIN 优化入口 | `sql/sql_optimizer.cc` | `JOIN::optimize()` | 337 |
| 条件规范化 | `sql/sql_optimizer.cc` | `optimize_cond()` | 10338 |
| 等值传播 | `sql/sql_optimizer.cc` | `build_equal_items()` | 4468 |
| 条件下推 | `sql/sql_optimizer.cc` | `make_cond_for_table()` | 9496 |
| Range Optimizer 入口 | `sql/range_optimizer/range_optimizer.cc` | `test_quick_select()` | 524 |
| 范围树构建 | `sql/range_optimizer/range_analysis.cc` | `get_mm_tree()` | 845 |
| 范围树合并 (AND) | `sql/range_optimizer/tree.cc` | `tree_and()` | 503 |
| 范围树合并 (OR) | `sql/range_optimizer/tree.cc` | `tree_or()` | 680 |
| 范围扫描代价估算 | `sql/range_optimizer/index_range_scan_plan.cc` | `check_quick_select()` | 568 |
| 最优访问路径 | `sql/sql_planner.cc` | `best_access_path()` | 981 |
| ref 访问评估 | `sql/sql_planner.cc` | `find_best_ref()` | 207 |
| ref 代价计算 | `sql/sql_planner.cc` | `find_cost_for_ref()` | 143 |
| scan 代价计算 | `sql/sql_planner.cc` | `calculate_scan_cost()` | 770 |
| 贪心搜索 | `sql/sql_planner.cc` | `greedy_search()` | 2328 |
| 深度搜索 | `sql/sql_planner.cc` | `best_extension_by_limited_search()` | 2719 |
| 代价模型 | `sql/opt_costmodel.h` | `Cost_model_server/Table` | 52/240 |
| SEL_TREE 定义 | `sql/range_optimizer/tree.h` | `SEL_TREE` | 871 |
| SEL_ARG 定义 | `sql/range_optimizer/tree.h` | `SEL_ARG` | 465 |
