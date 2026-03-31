# MySQL 8.0.41 `Optimize_table_order::choose_table_order` 深度分析

## 目录

1. [总体架构概览](#1-总体架构概览)
2. [choose_table_order 主流程](#2-choose_table_order-主流程)
3. [表排序预处理](#3-表排序预处理)
4. [贪心搜索算法 greedy_search](#4-贪心搜索算法-greedy_search)
5. [有限深度搜索 best_extension_by_limited_search](#5-有限深度搜索-best_extension_by_limited_search)
6. [最佳访问路径 best_access_path](#6-最佳访问路径-best_access_path)
7. [索引选择核心 find_best_ref](#7-索引选择核心-find_best_ref)
8. [扫描代价计算 calculate_scan_cost](#8-扫描代价计算-calculate_scan_cost)
9. [WHERE/JOIN 条件如何转换为索引查找](#9-wherejoin-条件如何转换为索引查找)
10. [行数估算与条件过滤 calculate_condition_filter](#10-行数估算与条件过滤-calculate_condition_filter)
11. [代价模型与决策公式](#11-代价模型与决策公式)
12. [关键数据结构](#12-关键数据结构)
13. [完整调用链总结](#13-完整调用链总结)

---

## 1. 总体架构概览

`Optimize_table_order::choose_table_order()` 是 MySQL 查询优化器中**表连接顺序优化**的入口函数。它负责：

- 确定多表 JOIN 的**最优执行顺序**
- 为每张表选择**最优访问方法**（ref、range、全表扫描等）
- 根据 WHERE 条件和 JOIN 条件选择**最优索引**
- 估算每步的**行数和代价**，最终生成代价最低的查询执行计划（QEP）

### 在优化器中的位置

```
SQL解析 → 语义分析 → JOIN::optimize()
                         ├─ make_join_plan()     // 确定常量表、构建Key_use数组
                         ├─ choose_table_order() // ← 本文分析的核心
                         └─ make_join_readinfo() // 根据计划生成执行结构
```

### 源码位置

| 文件 | 说明 |
|------|------|
| `sql/sql_planner.h` | `Optimize_table_order` 类声明 |
| `sql/sql_planner.cc` | 核心实现（约4800行） |
| `sql/sql_select.h` | `POSITION`、`Key_use` 等数据结构定义 |
| `sql/sql_optimizer.cc` | 调用入口 (`JOIN::optimize()`) |

---

## 2. choose_table_order 主流程

```
choose_table_order()                    // sql_planner.cc:1951
├─ 1. 常量表处理：设置prefix_cost=0
├─ 2. 如果所有表都是常量表，直接返回
├─ 3. 表排序预处理（merge_sort）
├─ 4. 条件过滤位图初始化（cond_set）
├─ 5. 横向派生表依赖计算
├─ 6. 选择搜索策略：
│   ├─ STRAIGHT_JOIN → optimize_straight_join()
│   └─ 否则        → greedy_search()
└─ 7. fix_semijoin_strategies() 修正半连接策略
```

### 核心代码逻辑（简化版）

```cpp
bool Optimize_table_order::choose_table_order() {
    // 1. 常量表无需优化
    if (join->const_tables == join->tables) {
        join->best_read = 1.0;
        return false;
    }

    // 2. 预排序表的顺序
    if (straight_join)
        merge_sort(..., Join_tab_compare_straight());
    else
        merge_sort(..., Join_tab_compare_default());

    // 3. 初始化条件过滤位图
    if (OPTIMIZER_SWITCH_COND_FANOUT_FILTER && join->where_cond) {
        join->where_cond->walk(&Item::add_field_to_cond_set_processor, ...);
    }

    // 4. 执行搜索
    if (straight_join)
        optimize_straight_join(join_tables);
    else
        greedy_search(join_tables);

    // 5. 修正半连接策略
    fix_semijoin_strategies();
}
```

---

## 3. 表排序预处理

在贪心搜索之前，`choose_table_order()` 会对表进行**预排序**，这是优化器的启发式策略。

### 排序规则 (`Join_tab_compare_default`)

排序优先级从高到低：

1. **依赖关系**：被依赖的表排在前面（外连接、直连接的依赖）
2. **键依赖**：如果表B需要表A的列做索引查找，表A排在前面
3. **记录数**：记录数少的表排在前面
4. **内存地址**：作为最终tie-breaker

这个预排序的目的是给贪心搜索提供一个**良好的初始顺序**，使得搜索更快收敛到较优解。

### 条件过滤位图初始化

```cpp
// 为每张表的 cond_set 位图设置涉及到 WHERE 条件的列
join->where_cond->walk(&Item::add_field_to_cond_set_processor,
                       enum_walk::POSTFIX, nullptr);
```

这个步骤遍历 WHERE 条件树，对于每个涉及到的列，在对应表的 `cond_set` 位图中标记。后续在 `calculate_condition_filter()` 中使用，**如果 `cond_set` 为空则直接跳过过滤效果计算**，避免不必要的开销。

---

## 4. 贪心搜索算法 greedy_search

`greedy_search()` 是一种**混合贪心/穷举搜索**算法，通过 `search_depth` 控制搜索深度。

### 算法伪代码

```
greedy_search(remaining_tables):
    pplan = <空计划>
    while remaining_tables 非空:
        best_read = DBL_MAX
        // 搜索深度为 search_depth 的最优扩展
        best_extension_by_limited_search(remaining_tables, idx, search_depth)
        
        if size_remain <= search_depth:
            // 搜索深度足够，已找到完整最优计划
            return best_positions
        
        // 取最优扩展的第一张表作为下一张加入计划的表
        best_table = best_positions[idx].table
        positions[idx] = best_positions[idx]
        remaining_tables -= best_table
        idx++
```

### 搜索深度的确定

```cpp
uint determine_search_depth(uint search_depth, uint table_count) {
    if (search_depth > 0) return search_depth;  // 用户指定
    
    const uint max_tables_for_exhaustive_opt = 7;
    if (table_count <= 7)
        return table_count + 1;  // 小查询：穷举搜索
    else
        return 7;                // 大查询：限制深度
}
```

- **表数 ≤ 7**：使用穷举搜索（`search_depth = table_count + 1`）
- **表数 > 7**：限制搜索深度为 7，复杂度约 O(N * N^7 / 7)

---

## 5. 有限深度搜索 best_extension_by_limited_search

这是贪心搜索的**核心递归函数**，执行有限深度的深度优先搜索。

### 算法流程

```
best_extension_by_limited_search(remaining_tables, idx, depth):
    for each table s in remaining_tables:
        if s 不依赖于 remaining_tables 中的其他表:
            // 1. 计算最佳访问路径
            best_access_path(s, remaining_tables, idx, ...)
            
            // 2. 计算累积代价
            position.set_prefix_join_cost(idx, cost_model)
            
            // 3. 剪枝：如果已超过最优代价
            if position.prefix_cost >= join->best_read:
                continue  // 剪枝
            
            // 4. 启发式剪枝（prune_level=1）
            if prune_level == 1:
                // 根据行数和代价进行剪枝
                ...
            
            // 5. 递归搜索或记录计划
            if depth > 1 && 还有剩余表:
                if s 是 EQ_REF 连接（rows_fetched <= 1.0）:
                    eq_ref_extension_by_limited_search(...)  // EQ_REF优化
                else:
                    best_extension_by_limited_search(...)     // 继续递归
            else:
                consider_plan(idx)  // 评估完整计划
```

### 三级剪枝策略

| 剪枝级别 | 条件 | 说明 |
|----------|------|------|
| **代价剪枝** | `prefix_cost >= best_read` | 当前部分计划已比已知最优差 |
| **启发式剪枝** | `prune_level == 1` | 根据行数和代价趋势提前排除 |
| **EQ_REF优化** | `rows_fetched <= 1.0` | 1:1连接不需要穷举排列 |

### EQ_REF 优化

当发现一张表通过唯一键连接（`rows_fetched <= 1.0`）时，调用 `eq_ref_extension_by_limited_search()` 将所有连续的 EQ_REF 表一次性加入计划，因为它们的排列顺序不影响总代价。

---

## 6. 最佳访问路径 best_access_path

`best_access_path()` 是为单张表选择最优访问方法的核心函数。它比较 **ref 访问**和**扫描**（范围扫描/索引扫描/全表扫描）的代价。

### 补充说明：ref 与 scan 的分类

> **在 `best_access_path()` 的语境中，所有访问方式被分为两大阵营：ref 和 scan。**
>
> **ref 阵营**（由 `find_best_ref()` 处理）：
> - `eq_ref`：通过唯一索引的所有 keypart 做等值匹配，最多返回 1 行
> - `ref`：通过非唯一索引（或唯一索引的部分前缀）做等值匹配，可能返回多行
> - `ref_or_null`：同 ref，但额外匹配 `IS NULL`，即 `col = val OR col IS NULL`
> - `fulltext`：全文索引查找
>
> **scan 阵营**（由 `calculate_scan_cost()` 处理）：
> - `range`：索引范围扫描（如 `WHERE col > 10 AND col < 100`）
> - `index`：索引全扫描
> - `ALL`：全表扫描
> - `index_merge`：多索引合并扫描
>
> **关键点：range（索引范围扫描）属于 scan 阵营，不属于 ref。**
>
> 这是因为 ref 的本质是**等值查找**（`=` / `<=>`），由 `Key_use` 数组驱动；
> 而 range 是**范围条件**（`>`, `<`, `BETWEEN`, `IN` 等），由前期的 range 优化器
> 独立分析，结果存储在 `tab->range_scan()` 中。
>
> 举例说明：
> ```sql
> -- ref 访问：等值条件，通过 Key_use 表示
> WHERE customer_id = 100
>
> -- range 访问（属于 scan 阵营）：范围条件，由 range 优化器处理
> WHERE customer_id > 100 AND customer_id < 200
>
> -- 混合场景：customer_id=100 走 ref，order_date 范围条件贡献 filter_effect
> WHERE customer_id = 100 AND order_date > '2024-01-01'
> ```
>
> 在 `best_access_path()` 中，先通过 `find_best_ref()` 找到最优的 ref 访问，
> 再通过 `calculate_scan_cost()` 计算 range/index/ALL 扫描的代价，最后比较
> 两者选择更便宜的方案。

### 执行流程

```
best_access_path(tab, remaining_tables, idx, prefix_rowcount, pos):
    
    // 第一步：寻找最佳 ref 访问方法
    best_ref = find_best_ref(tab, remaining_tables, idx, prefix_rowcount, ...)
    rows_fetched = best_ref ? best_ref->fanout : DBL_MAX
    best_read_cost = best_ref ? best_ref->read_cost : DBL_MAX
    
    // 第二步：计算横向派生表的物化代价
    derived_mat_cost = lateral_derived_cost(...)
    
    // 第三步：决定是否需要比较扫描代价
    if ref 明显更好（行数少且代价低）:
        跳过扫描评估
    else if range 使用相同索引且 ref 能用更多 keypart:
        跳过扫描评估（启发式：ref 更好）
    else if InnoDB 表有覆盖索引且 ref 可用:
        跳过扫描评估
    else if FORCE INDEX 且有 ref:
        跳过扫描评估
    else:
        // 计算扫描代价并比较
        scan_cost = calculate_scan_cost(...)
        scan_total = scan_cost + row_evaluate_cost(prefix_rowcount * rows_after_filtering)
        
        if scan_total < ref_total:
            选择扫描
        else:
            选择 ref
    
    // 第四步：计算条件过滤效果
    filter_effect = calculate_condition_filter(tab, best_ref, ...)
    
    // 第五步：填充 POSITION 结构
    pos->rows_fetched = rows_fetched
    pos->read_cost = best_read_cost + derived_mat_cost
    pos->filter_effect = filter_effect
    pos->key = best_ref  // NULL 表示扫描
```

### ref vs scan 的比较公式

```
ref 总代价 = best_read_cost + row_evaluate_cost(prefix_rowcount * rows_fetched)

scan 总代价 = scan_read_cost + row_evaluate_cost(prefix_rowcount * rows_after_filtering)
```

其中 `scan_read_cost` 已经包含了被过滤掉的行的评估代价。

---

## 7. 索引选择核心 find_best_ref

`find_best_ref()` 遍历所有可用索引，选择代价最低的 ref 访问方法。

### 索引优先级

索引选择使用以下优先级（从高到低）：

| 类型 | 说明 | 条件 |
|------|------|------|
| `CLUSTERED_PK` | 聚簇主键 | 所有keypart都有等值谓词，且是聚簇主键 |
| `UNIQUE` | 非空唯一索引 | 所有keypart都有等值谓词，且索引唯一 |
| `NOT_UNIQUE` | 非唯一索引 | 部分keypart有等值谓词 |
| `FULLTEXT` | 全文索引 | keypart为FT_KEYPART |

**决策规则**：
1. 如果已找到 `CLUSTERED_PK`，立即返回（启发式：聚簇主键总是最优）
2. 已找到 `UNIQUE` 时，不再考虑 `NOT_UNIQUE`（除非 `test_all_ref_keys` 为真）
3. 同优先级的索引，选代价更低的

### WHERE/JOIN 条件转换为 Key_use

条件到 Key_use 的转换发生在 `find_best_ref()` **之前**的阶段（`update_ref_and_keys()`），但在 `find_best_ref()` 中被**使用和评估**。每个 Key_use 对象表示一个 `table.keypart = expr` 形式的等值条件。

#### Key_use 的构建过程（前置阶段，非本函数）

```
WHERE t1.a = t2.b AND t1.c = 5 AND t1.d > 10
                      ↓
对于表 t1 的索引 idx1(a, c):
  Key_use[0]: table=t1, key=idx1, keypart=0, val=t2.b, used_tables={t2}
  Key_use[1]: table=t1, key=idx1, keypart=1, val=5,    used_tables={}  (const)
```

#### find_best_ref 中的遍历逻辑

```cpp
for (Key_use *keyuse = tab->keyuse(); keyuse->table_ref == tab->table_ref;) {
    key_part_map found_part = 0;   // 可用的 keypart 位图
    key_part_map const_part = 0;   // 常量 keypart 位图
    
    // 对每个 keypart 遍历所有可能的等值条件
    while (keyuse->key == key) {
        // 过滤不可用的 keyuse：
        // 1. 引用了被排除的表（半连接外部表）
        // 2. 引用了尚未加入计划的表
        // 3. 已有 ref_or_null 的 keypart
        if (excluded_tables & keyuse->used_tables) continue;
        if (remaining_tables & keyuse->used_tables) continue;
        
        found_part |= keyuse->keypart_map;
        if (!(keyuse->used_tables & ~const_table_map))
            const_part |= keyuse->keypart_map;  // 可以const求值
        
        // 选择使 prefix 中 distinct 行数最少的 keyuse
        cur_distinct = prev_record_reads(join, idx, table_deps | keyuse->used_tables);
        if (cur_distinct < best_distinct)
            选择这个 keyuse;
    }
}
```

### 行数估算（Fanout 计算）

根据覆盖的 keypart 数量和统计信息，有不同的估算策略：

#### 情况1：完整唯一键匹配

```cpp
if (all_key_parts_covered && (keyinfo->flags & HA_NOSAME) && !nullable) {
    cur_fanout = 1.0;  // 唯一索引，恰好一行
    cur_read_cost = prev_record_reads() * page_read_cost(1.0);
}
```

#### 情况2：完整非唯一键匹配

```cpp
if (all_key_parts_covered && !ref_or_null_part) {
    // 优先使用 range 优化器的估算
    if (table->quick_keys.is_set(key))
        cur_fanout = table->quick_rows[key];
    // 否则使用 records_per_key 统计
    else if (keyinfo->has_records_per_key(...))
        cur_fanout = keyinfo->records_per_key(...);
    // 兜底：总行数 / 不同键值数
    else
        cur_fanout = tab->records() / distinct_keys_est;
}
```

#### 情况3：部分键匹配

```cpp
if ((found_part & 1) && ...) {  // 至少第一个 keypart 可用
    cur_used_keyparts = max_part_bit(found_part);
    
    // 优先尝试复用 range 优化器的估算
    if (table->quick_keys.is_set(key) && !table_deps &&
        table->quick_key_parts[key] == cur_used_keyparts &&
        table->quick_n_ranges[key] == 1) {
        cur_fanout = table->quick_rows[key];  // 复用 range 估算
    }
    // 使用 records_per_key
    else if (keyinfo->has_records_per_key(cur_used_keyparts - 1)) {
        cur_fanout = keyinfo->records_per_key(cur_used_keyparts - 1);
    }
    // 兜底公式：线性插值
    else {
        // records = (x * (b - a) + a*c - b) / (c - 1)
        // a = records * 0.01  (第一个keypart匹配1%的行)
        // b = rec_per_key     (完整key匹配的行数)
        // c = key_parts       (总keypart数)
        // x = used_key_parts  (使用的keypart数)
    }
}
```

#### Range 估算的四个复用场景

`find_best_ref()` 在多处尝试复用 range 优化器的行数估算，因为 range 优化器通常**更准确**：

| 场景 | 条件 | 说明 |
|------|------|------|
| **ReuseRange-1** | ref(const) 且 range 使用同一索引 | range 的单点区间等价于 ref(const) |
| **ReuseRange-2** | range 在 ref 的前缀上有估算 | 取较低值作为调整 |
| **ReuseRange-3** | ref(const) 在部分键上 | 满足 C1(const)、C2(相同keypart)、C3(区间数匹配) |
| **ReuseRange-4** | range 在部分键上有更低的估算 | 作为下限调整 |

### ref 代价计算

```cpp
// ref 的 IO 代价
cur_read_cost = prefix_rowcount * find_cost_for_ref(thd, table, key, cur_fanout, worst_seeks);

// find_cost_for_ref 的逻辑：
double find_cost_for_ref(thd, table, keyno, num_rows, worst_seeks) {
    num_rows = min(num_rows, max_seeks_for_key);
    
    if (covering_index)
        return index_scan_cost(keyno, 1, num_rows);  // 只读索引
    if (clustered_primary_key)
        return read_cost(keyno, 1, num_rows);         // 聚簇读
    return min(page_read_cost(keyno, num_rows), worst_seeks);
}

// ref 的总代价（含CPU）
cur_ref_cost = cur_read_cost + prefix_rowcount * row_evaluate_cost(cur_fanout);
```

---

## 8. 扫描代价计算 calculate_scan_cost

当 ref 访问不够好或不可用时，需要计算扫描（range/index/table scan）的代价。

### 行数过滤估算

```cpp
// 优先使用 condition filtering
if (OPTIMIZER_SWITCH_COND_FANOUT_FILTER) {
    const_cond_filter = calculate_condition_filter(tab, nullptr, 0, ...);
    rows_after_filtering = found_records * const_cond_filter;
}
// 其次使用 quick_condition_rows（来自 range 优化器）
else if (quick_condition_rows != found_records)
    rows_after_filtering = quick_condition_rows;
// 兜底：有条件时假设 25% 被过滤
else if (found_condition)
    rows_after_filtering = found_records * 0.75;
```

### Range 扫描代价

```cpp
if (tab->range_scan()) {
    scan_cost = prefix_rowcount * (
        range_scan_cost +
        row_evaluate_cost(found_records - rows_after_filtering)
    );
}
```

### 全表/索引扫描代价

```cpp
// 无 join buffering
if (disable_jbuf) {
    scan_cost = prefix_rowcount * (
        single_scan_cost +
        row_evaluate_cost(records - rows_after_filtering)
    );
}
// 有 join buffering
else {
    buffer_count = 1.0 + (cache_length * prefix_rowcount / join_buff_size);
    scan_cost = buffer_count * (
        single_scan_cost +
        row_evaluate_cost(records - rows_after_filtering)
    );
}
```

**关键点**：Join Buffer 可以显著减少扫描次数，从 `prefix_rowcount` 次降低到 `buffer_count` 次。

---

## 9. WHERE/JOIN 条件如何转换为索引查找

### 完整转换链路

```
SQL语句
  │
  ▼
WHERE t1.a = t2.b AND t1.c = 5 AND t1.d > 10
  │
  ▼ (解析阶段)
Item 表达式树
  │
  ▼ (JOIN::optimize → update_ref_and_keys)
Key_use 数组
  ├─ Key_use{table=t1, key=idx1, keypart=0, val=t2.b, used_tables={t2}}
  └─ Key_use{table=t1, key=idx1, keypart=1, val=5, used_tables={}}
  │
  ▼ (choose_table_order → find_best_ref)
根据已加入计划的表，确定哪些 Key_use 可用
  │
  ▼ (find_best_ref 遍历每个索引)
计算 found_part / const_part / ref_or_null_part
  │
  ▼ (行数估算)
使用 records_per_key / quick_rows / 公式估算 fanout
  │
  ▼ (代价计算)
ref_cost = read_cost + row_evaluate_cost
  │
  ▼ (与扫描比较)
best_access_path 选择最终访问方法
```

### 条件类型对应关系

| WHERE 条件形式 | 在 Key_use 中的表示 | 访问类型 |
|---------------|-------------------|---------|
| `t1.a = 5` (常量) | `const_part \|= keypart_map`, `used_tables = {}` | ref(const) |
| `t1.a = t2.b` (join条件) | `used_tables = {t2}` | eq_ref / ref |
| `t1.a = 5 OR t1.a IS NULL` | `ref_or_null_part \|= keypart_map` | ref_or_null |
| `t1.a > 10` (范围条件) | 不产生 Key_use，由 range 优化器处理 | range |
| `t1.a = 5 AND t1.b = 3` (多keypart) | 多个 Key_use，逐 keypart 合并 | ref (多keypart) |

### JOIN 条件的特殊处理

对于 `t1.a = t2.b` 这类 join 条件：

1. **表依赖检查**：`keyuse->used_tables & remaining_tables` 必须为 0，即 t2 已在计划前缀中
2. **不同 join 顺序下 ref 可用性不同**：如果 t2 在 t1 之前，则此 keyuse 可用于 t1 的 ref 访问；如果 t1 在 t2 之前，则可能用于 t2 的 ref 访问（如果也有对应的 keyuse）
3. **prev_record_reads 计算**：对于非 const 的 keyuse，需要估算前缀中有多少不同的查找值

```cpp
// 估算对 ref 的调用次数（不同查找值的数量）
double prev_record_reads(JOIN *join, uint idx, table_map found_ref) {
    double found = 1.0;
    for (each position in prefix) {
        fanout = pos->rows_fetched * pos->filter_effect;
        if (pos->table is in found_ref) {
            found_ref |= pos->ref_depend_map;
            found *= fanout;
        } else if (fanout < 1.0) {
            found *= fanout;  // 条件过滤可能小于1
        }
    }
    return found;
}
```

---

## 10. 条件过滤 calculate_condition_filter

`calculate_condition_filter()` 计算**不被访问方法覆盖的剩余条件**的过滤比例，返回 0~1.0 的小数。

### 三阶段计算流程

```
filter = 1.0

阶段1: 排除已用列
  - ref 访问用了哪些列 → 标记到 tmp_set
  - range 访问用了哪些列 → 标记到 tmp_set
  - 如果 cond_set ⊆ tmp_set（所有条件列都已被覆盖），直接返回 1.0

阶段2: range 优化器对其他索引的估算（精度较高）
  - 遍历 quick_keys 中的每个索引
  - 如果索引列与 tmp_set 无交集（不与已用列重叠）：
      filter *= quick_rows[keyno] / records    // 乘入选择率
      tmp_set |= 该索引的列                     // 标记为已用

阶段3: 递归计算 WHERE 条件树（get_filtering_effect）
  - 对 cond_set 中尚未被 tmp_set 覆盖的列上的谓词，逐个计算过滤效果
  - AND 条件：P(A AND B) = P(A) * P(B)          // 假设独立，连乘
  - OR 条件：P(A OR B) = P(A) + P(B) - P(A)*P(B) // 容斥原理

兜底: filter = max(filter, 1.0/records)          // 至少1行
      if (filter * fanout < 0.05) filter = 0.05/fanout  // 最低扇出
```

### 单个谓词的估算：直方图优先

每个比较运算符的 `get_filtering_effect()` 遵循相同模式：**先查直方图，没有再用硬编码默认值**。

以 `col = 5` 为例（`Item_func_eq::get_filtering_effect`）：

```cpp
// 第一优先：从直方图获取选择率
histogram = table_share->find_histogram(field->field_index());
if (histogram != nullptr)
    histogram->get_selectivity(args, EQUALS_TO, &selectivity);  // 成功则直接返回

// 第二优先（fallback）：硬编码默认值
return max(1.0 / rows_in_table, COND_FILTER_EQUALITY);  // 0.1
```

各运算符的直方图 operator 和默认值：

| 运算符 | 直方图 operator | 无直方图时的默认值 |
|-------|----------------|-------------------|
| `=` | `EQUALS_TO` | `0.1` |
| `!=` | — (用 `1 - 0.1`) | `0.9` |
| `>`, `>=` | `GREATER_THAN[_OR_EQUAL]` | `0.3333` |
| `<`, `<=` | `LESS_THAN[_OR_EQUAL]` | `0.3333` |
| `BETWEEN` | `BETWEEN` | `0.1111` |
| `IN (v1,..,vN)` | 逐值叠加 | `min(0.5, N * 0.1)` |
| `IS NULL` | — | `0.1` |

### 多列条件的组合示例

```sql
SELECT * FROM t1 WHERE a = 1 AND b > 10 AND c BETWEEN 5 AND 20
-- 假设 ref 访问用了列 a，无直方图，表有 10000 行
```

```
阶段1: a 标记到 tmp_set（ref 已用，不重复计算）
阶段2: 无其他 quick_keys → filter = 1.0
阶段3: AND 连乘：
    a = 1     → a 在 tmp_set 中，跳过 → 1.0
    b > 10    → 无直方图 → 0.3333
    c BETWEEN → 无直方图 → 0.1111
    filter = 1.0 × 0.3333 × 0.1111 ≈ 0.037
```

如果列 `b` 上有直方图且算出 `b > 10` 选择率为 0.6，则 `filter = 0.6 × 0.1111 ≈ 0.067`。

### 计算时机

`calculate_condition_filter()` 在 `best_access_path()` 中被调用，发生在**选定访问方法之后**：

- 选择 ref 访问 → 以 `keyuse` 参数调用，排除 ref 已覆盖的列
- 选择 scan 访问 → 以 `nullptr` 参数调用，排除 range 已覆盖的列

优化完成后如果访问方法被调整（如 `test_if_skip_sort_order()` 改变索引），`filter_effect` 会被标记为 `COND_FILTER_STALE`（-1.0）并重新计算。

### 过滤效果的使用

`filter_effect` 存储在 `POSITION` 结构中，影响 `prefix_rowcount`：

```cpp
prefix_rowcount = prev.prefix_rowcount * rows_fetched * filter_effect;
```

### 与 EXPLAIN 输出的对应

| 代码中的值 | EXPLAIN 中的列 | 示例 |
|-----------|---------------|------|
| `rows_fetched` = 100 | `rows` = 100 | 访问方法预估返回行数 |
| `filter_effect` = 0.037 | `filtered` = 3.70 | 百分比（代码值 × 100） |
| `rows_fetched × filter_effect` = 3.7 | （无直接列） | 实际传给下一张表的行数 |

**EXPLAIN 中 `rows × filtered% = 该表向后传递的行数`，即代码中 `prefix_rowcount` 的增长因子。**

---

## 11. 代价模型与决策公式

### 累积代价模型

对于一个 N 张表的 join 计划 `[T1, T2, ..., TN]`：

```
prefix_rowcount[i] = prefix_rowcount[i-1] × rows_fetched[i] × filter_effect[i]
prefix_cost[i]     = prefix_cost[i-1] + read_cost[i] + row_evaluate_cost(prefix_rowcount[i-1] × rows_fetched[i])
```

其中：
- `rows_fetched[i]`：表 Ti 每次查找返回的行数
- `filter_effect[i]`：非访问方法条件的过滤比例（0~1）
- `read_cost[i]`：IO代价 × 前缀行数（已经乘好了）
- `row_evaluate_cost(n) = n × 0.1`（MySQL 8.0 默认值）

### 总代价构成

```
总代价 = ∑(IO代价) + ∑(CPU代价:行评估) + 排序代价(可选) + 窗口函数代价(可选)
```

### 决策流程图

```
                    ┌─────────────────┐
                    │ 有可用的索引？  │
                    └───────┬─────────┘
                     是 │         │ 否
                        ▼         │
              ┌──────────────┐    │
              │ find_best_ref│    │
              │ 选最佳ref索引│    │
              └──────┬───────┘    │
                     ▼            │
         ┌───────────────────┐    │
         │ ref 行数 < 扫描行数│    │
         │   且 ref代价 < 扫描│    │
         └──┬────────────┬───┘    │
          是│            │否      │
            ▼            ▼        ▼
     选择ref访问    ┌──────────────────┐
                    │calculate_scan_cost│
                    │ 计算扫描代价      │
                    └────────┬─────────┘
                             ▼
                  ┌──────────────────────┐
                  │scan_total < ref_total?│
                  └───┬──────────┬───────┘
                    是│          │否
                      ▼          ▼
                选择扫描     选择ref访问
```

---

## 12. 关键数据结构

### POSITION 结构

```cpp
struct POSITION {
    double rows_fetched;    // 每次查找返回的行数
    double read_cost;       // IO代价（已乘前缀行数）
    float  filter_effect;   // 条件过滤比例 (0~1)
    double prefix_rowcount; // 累积输出行数
    double prefix_cost;     // 累积总代价
    
    JOIN_TAB *table;        // 对应的表
    Key_use  *key;          // 使用的索引（NULL=扫描）
    table_map ref_depend_map; // ref依赖的表
    bool use_join_buffer;   // 是否使用join buffer
    
    uint sj_strategy;       // 半连接策略
    table_map dups_producing_tables; // 产生重复的表
};
```

### Key_use 结构

```cpp
class Key_use {
    Table_ref *table_ref;  // 表引用
    Item *val;             // 查找值表达式
    table_map used_tables; // 值依赖的表
    uint key;              // 索引编号
    uint keypart;          // keypart 编号
    uint optimize;         // KEY_OPTIMIZE_* 标志
    key_part_map keypart_map; // keypart 位图
    ha_rows ref_table_rows;   // 引用表估算行数
    bool null_rejecting;      // 是否拒绝NULL
    
    // 以下在 find_best_ref 中动态设置：
    key_part_map bound_keyparts; // 已绑定的 keypart
    double fanout;               // 预估返回行数
    double read_cost;            // 读代价
};
```

### Optimize_table_order 类

```cpp
class Optimize_table_order {
    THD *const thd;
    JOIN *const join;
    const uint search_depth;    // 搜索深度
    const uint prune_level;     // 剪枝级别 (0=穷举, 1=启发式)
    nested_join_map cur_embedding_map;
    const Table_ref *const emb_sjm_nest; // 半连接物化嵌套
    const table_map excluded_tables;      // 排除的表
    const bool has_sj;                    // 是否有半连接
    bool test_all_ref_keys;               // 是否测试所有ref key
    bool found_plan_with_allowed_sj;      // 是否找到允许的sj计划
    bool got_final_plan;                  // 是否已有最终计划
    bool use_best_so_far;                 // 是否使用当前最优
};
```

---

## 13. 完整调用链总结

```
choose_table_order()
│
├─ merge_sort(Join_tab_compare_default)        // 预排序表
│
├─ where_cond->walk(add_field_to_cond_set)     // 条件位图初始化
│
├─ greedy_search(remaining_tables)              // 贪心搜索
│   │
│   └─ [循环] best_extension_by_limited_search(remaining, idx, depth)
│       │
│       └─ [递归] for each table s in remaining:
│           │
│           ├─ best_access_path(s, remaining, idx, prefix_rowcount, pos)
│           │   │
│           │   ├─ find_best_ref(tab, remaining, idx, prefix_rowcount)
│           │   │   │
│           │   │   ├─ 遍历每个索引的所有 Key_use
│           │   │   ├─ 确定可用 keypart (found_part/const_part)
│           │   │   ├─ 行数估算：
│           │   │   │   ├─ quick_rows (range优化器)
│           │   │   │   ├─ records_per_key (索引统计)
│           │   │   │   └─ 插值公式 (兜底)
│           │   │   ├─ 代价计算：find_cost_for_ref()
│           │   │   └─ 选择最优索引（优先级：CLUSTERED_PK > UNIQUE > NOT_UNIQUE）
│           │   │
│           │   ├─ calculate_scan_cost(tab, idx, best_ref, ...)
│           │   │   ├─ range 扫描代价
│           │   │   ├─ 全表扫描代价（含/不含 join buffer）
│           │   │   └─ 条件过滤行数
│           │   │
│           │   ├─ 比较 ref_cost vs scan_cost，选择更优的
│           │   │
│           │   └─ calculate_condition_filter(tab, best_ref, ...)
│           │       ├─ 标记已用列
│           │       ├─ 利用 range 优化器的其他索引估算
│           │       ├─ 利用 WHERE 条件的 get_filtering_effect
│           │       └─ 返回 filter_effect (0~1)
│           │
│           ├─ position.set_prefix_join_cost(idx, cost_model)
│           │   ├─ prefix_rowcount = prev.prefix_rowcount * rows_fetched * filter_effect
│           │   └─ prefix_cost = prev.prefix_cost + read_cost + row_evaluate_cost(...)
│           │
│           ├─ [剪枝] prefix_cost >= best_read ?
│           ├─ [剪枝] 启发式行数/代价剪枝
│           │
│           ├─ [递归] best_extension_by_limited_search(remaining-s, idx+1, depth-1)
│           │   或
│           ├─ [EQ_REF优化] eq_ref_extension_by_limited_search(...)
│           │   或
│           └─ consider_plan(idx) → 更新 best_positions / best_read
│
└─ fix_semijoin_strategies()                    // 修正半连接策略
```

### 性能复杂度

| 场景 | 搜索深度 | 复杂度 |
|------|---------|--------|
| 表数 ≤ 7 | N+1（穷举） | O(N!) |
| 表数 > 7 | 7（启发式） | O(N * N^7 / 7) |
| STRAIGHT_JOIN | 1（固定序） | O(N) |
| 有EQ_REF链 | 减少有效N | 显著降低 |

### 优化器开关的影响

| 开关 | 影响 |
|------|------|
| `optimizer_search_depth` | 控制搜索深度，0=自动 |
| `optimizer_prune_level` | 0=穷举搜索，1=启发式剪枝 |
| `condition_fanout_filter` | 是否启用条件过滤效果估算 |
| `block_nested_loop` | 是否考虑 join buffer 的代价优化 |
| `max_seeks_for_key` | 限制 ref 代价中的最大查找次数 |

---

## 总结

MySQL 8.0.41 的表连接顺序优化是一个**代价驱动的混合搜索算法**：

1. **搜索策略**：使用贪心搜索+有限深度穷举，在优化质量和性能之间取得平衡
2. **条件转换**：WHERE 和 JOIN 条件在前期被转换为 `Key_use` 数组，在 `find_best_ref()` 中根据当前部分计划的可用性动态评估
3. **索引选择**：遵循 `CLUSTERED_PK > UNIQUE > NOT_UNIQUE` 的优先级，同级选代价最低
4. **行数估算**：优先使用 range 优化器的精确估算，其次是索引统计，最后是启发式公式
5. **过滤效果**：`calculate_condition_filter()` 估算未被访问方法覆盖的条件的过滤比例，影响 fanout 计算
6. **代价模型**：累积模型，`prefix_cost[i] = prev_cost + IO_cost + CPU_cost`，最终选择总代价最低的完整计划
