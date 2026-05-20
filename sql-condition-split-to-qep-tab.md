# SQL 条件拆分到 QEP_TAB 的完整流程分析

## 一、整体流程概览

```
SQL 解析
    │
    ▼
WHERE 条件 + ON 条件（连接条件）
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│                make_join_select (核心入口)               │
│                                                         │
│  步骤1: 处理常量表条件                                   │
│  步骤2: 遍历非常量表，提取 WHERE 条件                    │
│  步骤3: 处理外连接条件 (attach_join_conditions)          │
│  步骤4: 可能重新检查索引使用                             │
└─────────────────────────────────────────────────────────┘
    │
    ▼
每个 QEP_TAB 的 condition() 字段
    │
    ▼
执行阶段：SplitConditions 进一步处理
```

## 二、核心函数调用链

### 1. 主入口：make_join_select

```cpp:sql/sql_optimizer.cc:9193-9194
static bool make_join_select(JOIN *join, Item *cond) {
```

### 2. 条件提取：make_cond_for_table

```cpp:sql/sql_optimizer.cc:9106-9107
Item *make_cond_for_table(THD *thd, Item *cond, table_map tables,
                          table_map used_table, bool exclude_expensive_cond) {
```

**参数含义**：
- `tables`: 当前可用的所有表（前缀表集合）
- `used_table`: 本次新加入的表（用于判断条件是否应该附加到当前表）

### 3. 外连接处理：attach_join_conditions

```cpp:sql/sql_optimizer.cc:8469
bool JOIN::attach_join_conditions(plan_idx last_tab) {
```

## 三、条件拆分详细流程

### Phase 1：常量表条件处理

```cpp:sql/sql_optimizer.cc:9208-9258
// 提取仅涉及常量表的条件
Item *const_cond = make_cond_for_table(thd, cond, join->const_table_map, table_map(0), true);

// 立即评估常量条件
if (const_cond != nullptr && evaluate_during_optimization(const_cond, join->select_lex)) {
  const bool const_cond_result = const_cond->val_int() != 0;
  if (!const_cond_result) {
    // 常量条件为 false，整个查询返回空结果
    return true;
  }
}
```

### Phase 2：遍历每个表，提取适用的 WHERE 条件

```cpp:sql/sql_optimizer.cc:9275-9325
for (uint i = join->const_tables; i < join->tables; i++) {
  JOIN_TAB *const tab = join->best_ref[i];
  
  const table_map used_tables = tab->prefix_tables();  // 当前连接前缀中的所有表
  const table_map current_map = tab->added_tables();   // 本次新加入的表
  
  // 从 WHERE 中提取可以在此表上评估的条件
  Item *tmp = make_cond_for_table(thd, cond, used_tables, current_map, false);
  
  // 如果是外连接的内表，需要添加 FOUND_MATCH 触发条件
  if (cond && tmp) {
    tmp = add_found_match_trig_cond(join, first_inner, tmp, NO_PLAN_IDX);
    tab->set_condition(tmp);
  }
}
```

### Phase 3：处理外连接的 ON 条件

```cpp:sql/sql_optimizer.cc:8469-8488
bool JOIN::attach_join_conditions(plan_idx last_tab) {
  // 当到达外连接的最后一个内表时，处理对应的 ON 条件
  for (plan_idx first_inner = lt->first_inner();
       first_inner != NO_PLAN_IDX &&
       best_ref[first_inner]->last_inner() == last_tab;
       first_inner = best_ref[first_inner]->first_upper()) {
    Item *const join_cond = best_ref[first_inner]->join_cond();
    attach_join_condition_to_nest(first_inner, last_tab, join_cond, false);
  }
}
```

## 四、make_cond_for_table 核心算法

```cpp:sql/sql_optimizer.cc:9106-9174
Item *make_cond_for_table(THD *thd, Item *cond, table_map tables,
                          table_map used_table, bool exclude_expensive_cond) {
  // 早期过滤：如果条件不引用当前表，跳过
  if (used_table && !(cond->used_tables() & used_table) &&
      !(cond->is_expensive() && used_table == tables))
    return nullptr;

  // 处理 AND 条件：递归提取每个子条件
  if (cond->type() == Item::COND_ITEM) {
    if (((Item_cond *)cond)->functype() == Item_func::COND_AND_FUNC) {
      // AND: 提取所有满足条件的子条件
      Item_cond_and *new_cond = new Item_cond_and;
      while ((item = li++)) {
        Item *fix = make_cond_for_table(thd, item, tables, used_table, ...);
        if (fix) new_cond->argument_list()->push_back(fix);
      }
      return new_cond;  // 或 nullptr（无满足条件的子条件）
    } else {
      // OR: 必须所有子条件都满足才能整体满足
      Item_cond_or *new_cond = new Item_cond_or;
      while ((item = li++)) {
        Item *fix = make_cond_for_table(thd, item, tables, table_map(0), ...);
        if (!fix) return nullptr;  // OR 中有子条件不满足，整体不满足
        new_cond->argument_list()->push_back(fix);
      }
      return new_cond;
    }
  }

  // 普通条件：检查是否所有引用的表都已可用
  if ((cond->used_tables() & ~tables) ||  // 有表还未读取
      (!used_table && exclude_expensive_cond && cond->is_expensive()))
    return nullptr;

  return cond;  // 条件可用，返回
}
```

## 五、条件分配的规则图解

```
原始条件: WHERE t1.a = 10 AND t2.b > 5 AND t1.c = t2.c AND t3.d < 100

表连接顺序: t1 → t2 → t3 (假设)

┌───────────────────────────────────────────────────────────────────┐
│                      条件拆分过程                                  │
├───────────────────────────────────────────────────────────────────┤
│                                                                   │
│  t1 (第一个表):                                                    │
│    prefix_tables = t1                                             │
│    added_tables = t1                                              │
│    提取: t1.a = 10                      → tab[0].condition()      │
│    不提取: t2.b > 5 (t2未读取)                                     │
│    不提取: t1.c = t2.c (t2未读取)                                  │
│    不提取: t3.d < 100 (t3未读取)                                   │
│                                                                   │
│  t2 (第二个表):                                                    │
│    prefix_tables = t1 | t2                                        │
│    added_tables = t2                                              │
│    提取: t2.b > 5                       → tab[1].condition()      │
│    提取: t1.c = t2.c (两表都已读取)     → tab[1].condition()      │
│    不提取: t3.d < 100 (t3未读取)                                   │
│                                                                   │
│  t3 (第三个表):                                                    │
│    prefix_tables = t1 | t2 | t3                                   │
│    added_tables = t3                                              │
│    提取: t3.d < 100                     → tab[2].condition()      │
│                                                                   │
├───────────────────────────────────────────────────────────────────┤
│  最终分配结果:                                                     │
│                                                                   │
│  tab[0].condition() = t1.a = 10                                   │
│  tab[1].condition() = t2.b > 5 AND t1.c = t2.c                    │
│  tab[2].condition() = t3.d < 100                                  │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

## 六、可能导致条件丢失的场景分析

基于以上分析，条件可能丢失的常见原因：

### 1. used_tables() 计算错误

```cpp:sql/sql_optimizer.cc:9120-9122
if (used_table && !(cond->used_tables() & used_table) &&
    !(cond->is_expensive() && used_table == tables))
  return nullptr;
```

如果条件的 `used_tables()` 没有正确标记引用的表，条件可能被错误跳过。

**调试方法**：检查条件对象的 `used_tables()` 返回值。

### 2. OR 条件处理问题

```cpp:sql/sql_optimizer.cc:9515-9514
// OR 条件要求所有子条件都满足才能保留
while ((item = li++)) {
  Item *fix = make_cond_for_table(thd, item, tables, table_map(0), ...);
  if (!fix) return nullptr;  // 任一子条件不满足，整个 OR 被丢弃
}
```

如果 OR 条件中某个子条件无法满足，整个 OR 条件会被丢弃。

**场景示例**：
```sql
WHERE t1.a > 10 OR t2.b < 5  -- 假设 t1 先读
```
在处理 t1 时，`t2.b < 5` 无法满足（t2 未读取），整个 OR 条件被丢弃，不会被分配到任何表！

### 3. 外连接触发条件处理

```cpp:sql/sql_optimizer.cc:9319-9321
if (!(tmp = add_found_match_trig_cond(join, first_inner, tmp, NO_PLAN_IDX)))
  return true;
```

外连接的条件需要正确的触发条件包装，如果触发条件添加失败，可能导致条件丢失。

### 4. 触发条件的嵌套层级

```cpp:sql/sql_optimizer.cc:8408-8411
if (!(cond = add_found_match_trig_cond(
          this, best_ref[i]->first_inner(), cond,
          is_sj_mat_cond ? NO_PLAN_IDX : first_inner)))
  return true;
```

多层嵌套外连接时，触发条件需要正确包装到对应的层级。

## 七、调试建议

### 1. 检查条件是否正确分配到 QEP_TAB

在 `make_join_select` 循环后检查：
```cpp
for (uint i = 0; i < join->tables; i++) {
  JOIN_TAB *tab = join->best_ref[i];
  if (tab->condition() != nullptr) {
    // 检查每个表的 condition
  }
}
```

### 2. 检查 make_cond_for_table 的返回值

```cpp:sql/sql_optimizer.cc:9289
tmp = make_cond_for_table(thd, cond, used_tables, current_map, false);
```

如果期望的条件没有被提取，检查：
- `used_tables` 是否包含条件引用的所有表
- `current_map` 是否正确标记新加入的表
- `cond->used_tables()` 是否返回正确的值

### 3. 检查 OR 条件的子条件

对于 OR 条件，确保每个子条件都能在某个表上被评估。如果某个子条件引用的表永远不会成为"新加入的表"，整个 OR 条件会丢失。

### 4. 使用 OPT_TRACE

```cpp:sql/sql_optimizer.cc:9270-9273
Opt_trace_object trace_conditions(trace, "attaching_conditions_to_tables");
trace_conditions.add("original_condition", cond);
Opt_trace_array trace_attached_comp(trace, "attached_conditions_computation");
```

开启 optimizer trace 可以看到条件分配的详细过程：
```sql
SET optimizer_trace='enabled=on';
-- 执行查询
SELECT * FROM INFORMATION_SCHEMA.OPTIMIZER_TRACE;
```

## 八、关键函数调用图

```
make_join_select(cond)
    │
    ├─► make_cond_for_table(..., const_table_map, 0)
    │       提取常量条件
    │
    ├─► for each table i:
    │       │
    │       ├─► make_cond_for_table(..., prefix_tables, added_tables)
    │       │       提取 WHERE 条件中适用的部分
    │       │
    │       ├─► add_found_match_trig_cond()
    │       │       为外连接内表添加触发条件
    │       │
    │       └─► attach_join_conditions(i)
    │               │
    │               └─► attach_join_condition_to_nest()
    │                       │
    │                       ├─► make_cond_for_table()
    │                       │       提取 ON 条件的部分
    │                       │
    │                       ├─► add_found_match_trig_cond()
    │                       │       添加嵌套外连接的触发条件
    │                       │
    │                       └─► Item_func_trig_cond(IS_NOT_NULL_COMPL)
    │                               包装为"不应用于NULL补充行"
```

## 九、相关源码位置

| 函数 | 文件位置 |
|------|----------|
| `make_join_select` | `sql/sql_optimizer.cc:9193` |
| `make_cond_for_table` | `sql/sql_optimizer.cc:9106` |
| `attach_join_conditions` | `sql/sql_optimizer.cc:8469` |
| `attach_join_condition_to_nest` | `sql/sql_optimizer.cc:8358` |
| `SplitConditions` | `sql/sql_executor.cc:1143` |

---

*文档生成时间: 2026-05-20*