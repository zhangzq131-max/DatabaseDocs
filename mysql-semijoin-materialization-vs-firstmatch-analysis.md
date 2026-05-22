# MySQL 8.0.41 Semi-Join 策略选择分析：物化 vs FirstMatch（合并）

## 概述

MySQL 的 semi-join 共有 5 种可选策略（定义在 `sql/sql_select.h`）：

```c
#define SJ_OPT_NONE             0  // 无策略
#define SJ_OPT_DUPS_WEEDOUT     1  // 重复消除
#define SJ_OPT_LOOSE_SCAN       2  // 松散扫描
#define SJ_OPT_FIRST_MATCH      3  // 首次匹配（合并策略）
#define SJ_OPT_MATERIALIZE_LOOKUP 4  // 物化+索引查找
#define SJ_OPT_MATERIALIZE_SCAN    5  // 物化+全表扫描
```

本文所说的 **"合并策略"** 对应的是 `SJ_OPT_FIRST_MATCH`（FirstMatch），**"物化策略"** 对应的是 `SJ_OPT_MATERIALIZE_LOOKUP` 和 `SJ_OPT_MATERIALIZE_SCAN`。

策略的选择发生在两个阶段：**前提条件筛选** 和 **基于成本的比较**。

---

## 一、前提条件筛选阶段（决定某策略是否可用）

### 1. 物化策略的前提条件

物化策略是否可用由 `semijoin_types_allow_materialization()` 函数（`sql/sql_optimizer.cc:5697`）判定，它设置 `sjm.lookup_allowed` 和 `sjm.scan_allowed` 两个标志。

#### (a) 优化器开关

`optimizer_switch` 中 `OPTIMIZER_SWITCH_MATERIALIZATION` 必须开启（`sql/sql_optimizer.cc:6194-6195`）。

#### (b) 非平凡相关（non-trivially correlated）限制

如果 semi-join nest 有 `sj_corr_tables != 0`（即存在非平凡相关的外部表），则**物化策略完全不可用**，直接跳过：

```c
// sql/sql_optimizer.cc:6207
if (sj_nest->nested_join->sj_corr_tables) continue;
```

这是最关键的限制条件：**当子查询引用了外部查询的列且无法通过 table pullout 消除时，物化被排除**。

#### (c) 比较表达式数量限制

semi-join 的内外比较表达式数量不能超过 `MAX_REF_PARTS`，也不能为 0：

```c
// sql/sql_optimizer.cc:5703-5708
if (sj_nest->nested_join->sj_outer_exprs.size() > MAX_REF_PARTS ||
    sj_nest->nested_join->sj_outer_exprs.size() == 0) {
  // building an index is impossible
  sj_nest->nested_join->sjm.scan_allowed = false;
  sj_nest->nested_join->sjm.lookup_allowed = false;
  return;
}
```

#### (d) 数据类型兼容性

由 `types_allow_materialization()`（`sql/sql_select.cc:1149`）检查：

- 不能是 `ROW_RESULT` 类型
- 如果内层是数值类型（INT/REAL/DECIMAL），允许任何数值类型外层进行查找
- 如果内层是非数值类型，外层不能是数值类型
- 字符串类型要求**相同排序规则（collation）**
- 时间类型有特殊规则（内层含日期可匹配任何时间类型，否则要求严格匹配）

#### (e) Lookup 策略的额外限制

`MaterializeLookup` 比 `MaterializeScan` 更严格，有以下额外限制：

- **BLOB/GEOMETRY 列不允许** Lookup（`sql/sql_optimizer.cc:5746`）：
  ```c
  if (blobs_involved) sj_nest->nested_join->sjm.lookup_allowed = false;
  ```
- **单键部分长度**不能超过临时表引擎的 `max_key_part_length`（`sql/sql_optimizer.cc:5739-5740`）
- **总索引长度**不能超过 `max_key_length`（`sql/sql_optimizer.cc:5743-5744`）

**结论**：`MaterializeScan` 比 `MaterializeLookup` 的限制更少（允许 BLOB 列），因为 Scan 不需要建立索引。

---

### 2. FirstMatch（合并策略）的前提条件

FirstMatch 的可用性在贪心搜索过程中逐表判定（`sql/sql_planner.cc:4154-4233`）。

#### (a) 优化器开关

`OPTIMIZER_SWITCH_FIRSTMATCH` 必须开启（`sql/sql_planner.cc:4154-4155`）。

#### (b) 外部相关表必须在 join 前缀中

所有 `sj_depends_on`（semi-join 依赖的外部表）必须已经在已处理的 join 前缀中，不能在剩余表中：

```c
// sql/sql_planner.cc:4168-4169
if (pos->dups_producing_tables == 0 &&        // (2)
    !(remaining_tables & outer_corr_tables))  // (3)
```

如果某个相关外表仍在 `remaining_tables` 中，FirstMatch 就不能使用：

```c
// sql/sql_planner.cc:4183-4188
if (outer_corr_tables & pos->first_firstmatch_rtbl) {
  // FirstMatch can't be used.
  pos->first_firstmatch_table = MAX_TABLES;
}
```

#### (c) 不能处于其他去重策略的范围中

`dups_producing_tables == 0`（`sql/sql_planner.cc:4168`），即当前没有其他半连接策略正在处理重复行。

#### (d) 嵌套结构约束

FirstMatch 的跳转目标不能破坏嵌套层次结构（`sql/sql_planner.cc:4198-4205`）。例如：

```sql
B LEFT JOIN (C SEMIJOIN D ON B.X=D.Y)
```

在表顺序 B-D-C 中，不能从 D 跳回 B，因为这会导致非层次化的连接。

#### (e) Anti-join 兼容性

如果是 anti-join nest，FirstMatch 是允许的策略之一（`sql/sql_lex.cc:4476-4481`）：

```c
if (sj_nest->is_aj_nest()) {
  sj_nest->nested_join->sj_enabled_strategies &=
      OPTIMIZER_SWITCH_FIRSTMATCH | OPTIMIZER_SWITCH_MATERIALIZATION |
      OPTIMIZER_SWITCH_DUPSWEEDOUT;
}
```

---

## 二、基于成本的比较阶段（选择最优策略）

所有满足前提条件的策略会在贪心搜索中（`best_extension_by_limited_search` 的策略评估部分）进行**成本比较**。核心代码在 `sql/sql_planner.cc:4366-4535`。

### 各策略的成本计算

#### FirstMatch 成本

由 `semijoin_firstmatch_loosescan_access_paths()` 计算（`sql/sql_planner.cc:3635`）：

```
cost = 前缀成本 + 内外表逐表连接成本（含 row evaluate cost）
rowcount = 前缀行数 × outer_fanout
```

FirstMatch 的特点是：
- 当只有1个内部表时，可以使用 join buffering
- 当有多个内部表时，不能使用 join buffering（`sql/sql_planner.cc:3673-3674`）

#### MaterializeLookup 成本

由 `semijoin_mat_lookup_access_paths()` 计算（`sql/sql_planner.cc:3856-3877`）：

```
cost = 前缀成本 + materialization_cost + rowcount × lookup_cost
rowcount = 前缀行数（不变）
```

Lookup 策略的特点是输出行数等于前缀行数（因为物化表通过索引查找去重后不增加行数）。

#### MaterializeScan 成本

由 `semijoin_mat_scan_access_paths()` 计算（`sql/sql_planner.cc:3775-3838`）：

```
cost = 前缀成本 + materialization_cost + rowcount × scan_cost
       + 外表重新计算的成本（逐表 best_access_path）
rowcount = 前缀行数 × outer_fanout
```

Scan 策略的特点是：
- 需要重新计算外表的访问路径（因为物化表扫描后行数减少）
- 物化表的行数为 `expected_rowcount`（去重后的行数）

### 最终选择逻辑

所有候选策略在同一位置被逐一评估，最终选择**成本最低的策略**。但有一个重要例外（`sql/sql_planner.cc:4415` 和 `4464`）：

```c
// MaterializeLookup 选择条件
if (cost < best_cost || pos->dups_producing_tables) {
  sj_strategy = SJ_OPT_MATERIALIZE_LOOKUP;
  ...
}

// MaterializeScan 选择条件
if (cost < best_cost || pos->dups_producing_tables) {
  sj_strategy = SJ_OPT_MATERIALIZE_SCAN;
  ...
}
```

即：**如果还没有任何其他 semi-join 策略被选中（`dups_producing_tables != 0`），则即使物化成本更高也会被选中**，因为必须消除重复行，不能留有未被处理的重复产出表。

### 物化策略内部的子选择：Lookup vs Scan

在 `semijoin_order_allows_materialization()`（`sql/sql_planner.cc:2178-2214`）中，决定使用 MaterializeLookup 还是 MaterializeScan：

```c
// sql/sql_planner.cc:2203-2213
/*
  Must use MaterializeScan strategy if there are outer correlated tables
  among the remaining tables, otherwise, if possible, use MaterializeLookup.
*/
if ((remaining_tables & emb_sj_nest->nested_join->sj_depends_on) ||
    !emb_sj_nest->nested_join->sjm.lookup_allowed) {
  if (emb_sj_nest->nested_join->sjm.scan_allowed)
    return SJ_OPT_MATERIALIZE_SCAN;
  return SJ_OPT_NONE;
}
return SJ_OPT_MATERIALIZE_LOOKUP;
```

**逻辑**：
1. 如果外部相关表还在剩余表中 → 必须用 Scan（因为 Lookup 需要外表在前面才能做索引查找）
2. 如果 `lookup_allowed` 为 false（BLOB列、键长度超限等） → 降级为 Scan
3. 否则 → 默认优先选择 Lookup（这是一个启发式决策，注释中明确说"based on a heuristic decision"）

---

## 三、关键决策因素总结

| 决策因素 | 物化策略可用？ | FirstMatch 可用？ |
|---|---|---|
| 子查询**非平凡相关**（sj_corr_tables != 0） | **不可用** | 可能可用（需相关表在prefix中） |
| 比较**表达式数量** > MAX_REF_PARTS 或 == 0 | **不可用** | 不受此限制 |
| 比较表达式类型不兼容 | Lookup 不可用，Scan 也可能不可用 | 不受此限制 |
| 内层含 **BLOB/GEOMETRY** 列 | Lookup 不可用，Scan 可用 | 不受此限制 |
| 索引键长度超限 | Lookup 不可用 | 不受此限制 |
| 外部相关表不在join prefix中 | Lookup 不可用，**Scan 可用** | **不可用** |
| optimizer_switch 开关 | SUBSEMIMAT | FIRSTMATCH |
| 最终选择 | **成本最低者胜出** | **成本最低者胜出** |

### 核心区别

1. **相关性是最大分歧点**：非平凡相关的子查询**不能用物化**，但可能用 FirstMatch（只要相关表在 join 前缀中）。这是最常见的场景——当子查询引用了外部查询的列且无法通过 table pullout 消除时，物化被排除。

2. **FirstMatch 要求所有相关外表在 join 前缀中**，而 `MaterializeScan` 仅要求相关外表不在剩余表中时使用 Lookup，否则可降级为 Scan。因此 Scan 对表顺序的约束更灵活。

3. **物化策略对数据类型有严格要求**（尤其是 Lookup），FirstMatch 不受这些限制。

4. **最终决策以成本比较为准**，但保证至少有一个策略处理重复行。

---

## 四、关键源码位置索引

| 功能 | 文件 | 行号 |
|---|---|---|
| 策略常量定义 | `sql/sql_select.h` | 301-310 |
| 物化前提条件检查 | `sql/sql_optimizer.cc` | 5697-5749 (`semijoin_types_allow_materialization`) |
| 数据类型兼容性检查 | `sql/sql_select.cc` | 1149-1195 (`types_allow_materialization`) |
| 物化成本计算(前置) | `sql/sql_optimizer.cc` | 6185-6235 (`optimize_semijoin_nests_for_materialization`) |
| 物化成本估算 | `sql/sql_optimizer.cc` | 10533-10630 (`calculate_materialization_costs`) |
| Lookup/Scan 启发式选择 | `sql/sql_planner.cc` | 2178-2214 (`semijoin_order_allows_materialization`) |
| FirstMatch 前提条件 | `sql/sql_planner.cc` | 4154-4233 |
| MaterializeLookup 成本 | `sql/sql_planner.cc` | 3856-3877 (`semijoin_mat_lookup_access_paths`) |
| MaterializeScan 成本 | `sql/sql_planner.cc` | 3775-3838 (`semijoin_mat_scan_access_paths`) |
| FirstMatch 成本 | `sql/sql_planner.cc` | 3635-3753 (`semijoin_firstmatch_loosescan_access_paths`) |
| 最终策略选择（成本比较） | `sql/sql_planner.cc` | 4366-4535 |
| sj_corr_tables / sj_depends_on 定义 | `sql/nested_join.h` | 122-128 |
| sj_enabled_strategies 设置 | `sql/sql_lex.cc` | 4472-4481 |