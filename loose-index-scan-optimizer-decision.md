# 优化器如何决定是否走 Loose Index Scan

## 1. 入口与整体流程

Loose index scan 的两种形式（**INDEX_SKIP_SCAN**、**GROUP_INDEX_SKIP_SCAN**）都在 **range 优化阶段** 被考虑，入口是 **test_quick_select()**（`sql/range_optimizer/range_optimizer.cc`）。

在单表或每张表选访问路径时，优化器会调用 `test_quick_select()`，在**同一张表**上尝试多种访问方式并比较代价，从中选出最优的 `AccessPath`（可能为全表扫描、索引 range、index merge、**group_min_max**、**skip_scan** 等）。

### 1.1 test_quick_select 内考虑顺序（简化）

1. **get_mm_tree()**：根据 WHERE 条件得到 SEL_TREE（可能为 NULL，例如条件无法形成单一 range 树）。
2. **get_best_group_min_max()**：尝试 **GROUP_INDEX_SKIP_SCAN**（单表 GROUP BY / MIN-MAX / agg distinct）。若得到 path 且 **cost < best_cost**，则选为当前 best_path。
3. **get_best_skip_scan()**：**仅当** `optimizer_switch` 的 **skip_scan=on** 或存在 **SKIP_SCAN hint** 时才调用；尝试 **INDEX_SKIP_SCAN**。若得到 path 且 **cost < best_cost**（或强制 hint），则选为 best_path。
4. **get_key_scans_params()**：普通索引 range 扫描。
5. **get_best_ror_intersect()** 等：index merge、ROR 等。

因此：**是否走 loose index scan = 先看“是否满足各自适用条件并生成 path”，再与当前 best_cost 比较；INDEX_SKIP_SCAN 还受开关和 hint 控制。**

---

## 2. INDEX_SKIP_SCAN 的决策条件

### 2.1 是否参与竞争（开关与 hint）

在 `test_quick_select()` 里，**只有**在下面条件为真时才会调用 `get_best_skip_scan()`：

```690:696:sql/range_optimizer/range_optimizer.cc
  bool force_skip_scan = hint_table_state(thd, param.table->pos_in_table_list,
                                          SKIP_SCAN_HINT_ENUM, 0);

  if (thd->optimizer_switch_flag(OPTIMIZER_SKIP_SCAN) || force_skip_scan) {
    AccessPath *skip_scan_path =
        get_best_skip_scan(thd, &param, tree, interesting_order,
                           skip_records_in_range, force_skip_scan);
```

- **optimizer_switch**：`optimizer_switch` 中的 **skip_scan** 必须为 on（默认 on，对应 `OPTIMIZER_SKIP_SCAN`）。
- **Hint**：若对该表使用了 **SKIP_SCAN** hint（如 `SKIP_SCAN(t)`），则 **force_skip_scan = true**，即使 switch 关闭也会尝试 skip scan，且若生成了 path 会**强制选用**（见下）。

若 **skip_scan=off** 且无 **SKIP_SCAN** hint，则**根本不会**尝试 INDEX_SKIP_SCAN。

### 2.2 是否生成 Skip Scan path（get_best_skip_scan 的适用条件）

在 `get_best_skip_scan()`（`sql/range_optimizer/index_skip_scan_plan.cc`）内，下列条件**全部**满足才会对某个索引生成 path，否则返回 NULL（不适用）：

- **单表**：`join->primary_tables == 1`。
- **无 GROUP BY**：`join->group_list.empty()`。
- **无 SELECT DISTINCT**：`!join->select_distinct`。
- **有 range 树**：`tree != nullptr`（否则 `cause = "disjuntive_predicate_present"`）。
- **需要正序**：`order_direction != ORDER_DESC`。
- **有可用索引**：`table->s->keys != 0`。
- **覆盖索引**：该索引在 `table->covering_keys` 中（查询用到的列都在该索引中）。
- **索引结构**：存在复合索引，满足 “等值前缀 A_1..A_k + 可跳过部分 B_1..B_m + 范围列 C” 的形式；且对 C 有 range 条件，B 段至多一个 key part 有谓词且较简单（见 `get_best_skip_scan` 注释）。
- **聚合限制**：若有聚合，则不允许 COUNT/SUM/AVG DISTINCT（否则直接 return nullptr）。
- **Hint**：若表上指定了 **NO_SKIP_SCAN** 等，该索引可能被排除（通过 `compound_hint_key_enabled(..., SKIP_SCAN_HINT_ENUM)` 等判断）。

对**每个**满足上述条件的索引会估算 cost；在**所有**候选索引中选 cost 最小的一个作为 skip_scan_path（若存在）。

### 2.3 是否被选为最终 path（代价与强制）

即使生成了 `skip_scan_path`，还要和当前 best_path 比代价：

```705:711:sql/range_optimizer/range_optimizer.cc
      if (skip_scan_path->cost < best_cost || force_skip_scan) {
        summary.add("chosen", true);
        best_path = skip_scan_path;
        best_cost = best_path->cost;
      } else
        summary.add("chosen", false).add_alnum("cause", "cost");
```

- **正常**：仅当 `skip_scan_path->cost < best_cost` 时，才用 skip scan 替换当前 best_path。
- **强制**：若 **force_skip_scan == true**（即用了 SKIP_SCAN hint），则**不论代价**都会选用 skip scan（`cost < best_cost || force_skip_scan`）。

小结：**INDEX_SKIP_SCAN 是否被选用 = (skip_scan 开关或 SKIP_SCAN hint) + get_best_skip_scan() 能生成 path + (cost 更优 或 强制 hint)。**

---

## 3. GROUP_INDEX_SKIP_SCAN 的决策条件

### 3.1 是否尝试（无开关，仅适用条件）

**没有**单独的 optimizer_switch 控制 GROUP_INDEX_SKIP_SCAN。只要进入 `test_quick_select()`，就会**始终**调用 `get_best_group_min_max()`（不依赖 tree 是否为 NULL）：

```667:673:sql/range_optimizer/range_optimizer.cc
  /*
    Try to construct a GroupIndexSkipScanIterator.
    Notice that it can be constructed no matter if there is a range tree.
  */
  AccessPath *group_path = get_best_group_min_max(
      thd, &param, tree, interesting_order, skip_records_in_range, best_cost);
```

是否走 group loose index scan 完全由 **get_best_group_min_max() 是否返回 path** 以及 **cost 是否更优** 决定。

### 3.2 是否生成 path（get_best_group_min_max 的适用条件）

在 `get_best_group_min_max()`（`sql/range_optimizer/group_index_skip_scan_plan.cc`）里，会先做一批“廉价”检查，不满足则直接返回 NULL：

- **单表**：`param->query_block->original_tables_map == 1`，且 `table->pos_in_table_list->map() == 1`（原始查询只涉及这一张表，或满足文档中 (B0) 的例外）。
- **无 ROLLUP**：`join->query_block->olap != ROLLUP_TYPE`。
- **需要 GROUP BY / DISTINCT / 聚合**：  
  `!join->group_list.empty() || join->select_distinct || is_agg_distinct`；  
  否则 `cause = "not_group_by_or_distinct"`。
- **聚合类型**：若有聚合，只允许 MIN、MAX，或（在 is_agg_distinct 时）COUNT/SUM/AVG DISTINCT；且 MIN/MAX 参数为同一列（且为字段），不能同时有 MIN/MAX 与 agg distinct。
- **多表 + 聚合**：若多表且存在聚合，直接不允许（`cause = "Multi_table_with_aggregate"`）。
- **ORDER**：`order_direction != ORDER_DESC`。
- **无索引**：`table->s->keys == 0` 则不用。
- **析取条件**：若 tree 为 merge（多棵析取 range 树）或 tree 为 NULL 且 WHERE 里同时涉及 min_max 列与其它列，可能 `cause = "disjuntive_predicate_present"` 或 `"minmax_keypart_in_disjunctive_query"`。
- **GROUP BY 列均为字段**：不能是表达式（否则 `cause = "group_field_is_expression"`）。
- **SELECT DISTINCT**：若存在，则要求 select 的字段均为 Item_field 等（否则 return nullptr）。

然后对**每个候选索引**检查（概要）：

- **覆盖索引**：该索引必须 covering（且若引擎在二级索引后附加主键，所有 read_set 中的列都须在该索引中）。
- **GROUP BY 与索引前缀**（GA1）：GROUP BY 的列必须**按顺序**对应索引的**前若干列**（group 前缀）；否则 `cause = "group_attribute_not_prefix_in_index"`。
- **MIN/MAX 列**：MIN/MAX 的参数列必须出现在该索引的某个 key part 上（且满足 (SA2) 等条件）。
- **key infix / range**：若有范围条件等，需能形成合法 prefix_ranges / key_infix_ranges / min_max_ranges。
- **索引 hint**：若表上有禁止该索引或只允许某些索引的 hint，会通过 `param->keys` 等过滤。

对满足条件的索引会算 cost；在**所有**这类索引里选 cost 最小的，若小于传入的 **cost_est**（当前 best_cost）才返回该 path，否则返回 NULL。

### 3.3 是否被选为最终 path（代价）

```674:687:sql/range_optimizer/range_optimizer.cc
  if (group_path) {
    ...
    if (group_path->cost < best_cost) {
      grp_summary.add("chosen", true);
      best_path = group_path;
      best_cost = best_path->cost;
    } else
      grp_summary.add("chosen", false).add_alnum("cause", "cost");
  }
```

- 只有 **group_path->cost < best_cost** 时，才会用 GROUP_INDEX_SKIP_SCAN 作为 best_path。  
- best_cost 的初值是表扫描（或 covering index 扫描）的 cost，所以 group path 必须比“全表/覆盖扫描”更便宜才会被选。

小结：**GROUP_INDEX_SKIP_SCAN 是否被选用 = get_best_group_min_max() 能生成 path（满足单表、GROUP/DISTINCT/聚合、索引结构、覆盖、GROUP 前缀与 MIN/MAX 等条件）+ cost 低于当前 best_cost。**

---

## 4. 与其它访问方式的竞争

- **GROUP_INDEX_SKIP_SCAN** 在 **INDEX_SKIP_SCAN** 和 **普通 index range** 之前被考虑；若其 path 的 cost 优于当时的 best_cost，就会先占住 best_path。
- **INDEX_SKIP_SCAN** 在 **get_key_scans_params()（普通 range）** 之前被考虑，但**只有**在 skip_scan 开关或 SKIP_SCAN hint 打开时才会参与。
- 随后 **get_key_scans_params()**、**get_best_ror_intersect()** 等会继续用各自的 path 与当前 best_path 比 cost，可能把 best_path 再替换成普通 range 或 index merge 等。

因此：**是否“走 loose index scan” = 在 range 优化阶段，对应 path 被生成且 cost 优于当前 best_cost（或 INDEX_SKIP_SCAN 被 hint 强制）。**

---

## 5. 总结表

| 项目 | INDEX_SKIP_SCAN | GROUP_INDEX_SKIP_SCAN |
|------|------------------|------------------------|
| **开关 / Hint** | 需要 `optimizer_switch skip_scan=on` 或表级 **SKIP_SCAN** hint；否则不尝试。 | 无单独开关，始终尝试（若进入 test_quick_select）。 |
| **查询形状** | 单表；无 GROUP BY；无 SELECT DISTINCT；有 range 树；等值前缀 + 可跳过 key part + 范围列。 | 单表（或符合 (B0)）；GROUP BY 或 SELECT DISTINCT 或 MIN/MAX 或 agg distinct。 |
| **索引** | 覆盖索引；结构为 等值前缀 + 跳过段 + 范围列，且范围列有 range 条件。 | 覆盖索引；GROUP BY 列 = 索引前缀；MIN/MAX 或 distinct 列在索引中。 |
| **是否选用** | 生成 path 且 **(cost < best_cost 或 SKIP_SCAN 强制)**。 | 生成 path 且 **cost < best_cost**。 |

优化器**决定是否走 loose index scan** 的步骤可以概括为：  
1）对 INDEX_SKIP_SCAN 先看开关/hint 是否允许尝试；  
2）在各自函数内根据查询形状和索引结构判断是否生成 path；  
3）用生成的 path 的 cost 与当前 best_cost 比较（INDEX_SKIP_SCAN 在强制 hint 下可忽略 cost），更优或强制则选为 best_path，否则继续与后续的 range/index merge 等比较。
