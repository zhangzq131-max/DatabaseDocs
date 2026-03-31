# MySQL 8.0.41 范围优化器：从 WHERE 条件到索引扫描起始位置

## 1. 概述

MySQL 的范围优化器（Range Optimizer）负责将 WHERE 子句中的范围条件转换为高效的索引扫描计划。这个过程涉及多个阶段：

1. **条件分析阶段**：将 WHERE 条件解析并转换为 `SEL_ARG` 树结构
2. **范围提取阶段**：从 `SEL_ARG` 树中提取有序的、不相交的键区间（`QUICK_RANGE`）
3. **执行阶段**：将 `QUICK_RANGE` 转换为存储引擎可理解的 `key_range`，确定 `ha_rkey_function` 值，调用存储引擎定位索引起始位置

整体调用链路：

```
WHERE 条件 (Item)
    │
    ▼
get_mm_tree()          ──→ SEL_TREE / SEL_ARG 树
    │
    ▼
get_ranges_from_tree() ──→ Quick_ranges (QUICK_RANGE 数组)
    │
    ▼
IndexRangeScanIterator::Read()
    │
    ▼
handler::multi_range_read_next()
    │
    ▼
handler::read_range_first()
    │
    ▼
handler::ha_index_read_map(buf, key, keypart_map, ha_rkey_function)
    │
    ▼
存储引擎 (InnoDB) 定位索引起始位置
```

---

## 2. 核心数据结构

### 2.1 key_range_flags —— 范围标志位

定义在 `include/my_base.h:1078-1122`：

```cpp
enum key_range_flags {
  NO_MIN_RANGE      = 1 << 0,   // 无下界，从 -∞ 开始
  NO_MAX_RANGE      = 1 << 1,   // 无上界，到 +∞ 结束
  NEAR_MIN          = 1 << 2,   // 不包含左端点（开区间），即 X < key
  NEAR_MAX          = 1 << 3,   // 不包含右端点（开区间），即 X > key
  UNIQUE_RANGE      = 1 << 4,   // 唯一索引上的等值条件（所有 keypart 非 NULL）
  EQ_RANGE          = 1 << 5,   // 等值条件（min_key == max_key）
  NULL_RANGE        = 1 << 6,   // 类似 UNIQUE_RANGE，但至少一个 keypart 为 NULL
  GEOM_FLAG         = 1 << 7,   // R-tree 空间索引范围
  SKIP_RANGE        = 1 << 8,   // 已弃用
  SKIP_RECORDS_IN_RANGE = 1 << 9, // 使用索引统计信息代替 index dive
  DESC_FLAG         = 1 << 10,  // 降序扫描
};
```

这些标志位是贯穿整个范围优化过程的关键：它们在 `SEL_ARG` 的 `min_flag`/`max_flag` 中设置，传递到 `QUICK_RANGE` 的 `flag` 中，最终决定 `ha_rkey_function` 的取值。

### 2.2 ha_rkey_function —— 索引读取函数枚举

定义在 `include/my_base.h:78-93`：

```cpp
enum ha_rkey_function {
  HA_READ_KEY_EXACT,             // 精确查找第一条匹配记录
  HA_READ_KEY_OR_NEXT,           // 查找匹配记录或下一条记录（>=）
  HA_READ_KEY_OR_PREV,           // 查找匹配记录或前一条记录（<=）
  HA_READ_AFTER_KEY,             // 查找严格大于给定键的下一条记录（>）
  HA_READ_BEFORE_KEY,            // 查找严格小于给定键的前一条记录（<）
  HA_READ_PREFIX,                // 具有相同前缀的键
  HA_READ_PREFIX_LAST,           // 具有相同前缀的最后一个键
  HA_READ_PREFIX_LAST_OR_PREV,   // 具有相同前缀的最后一个键或前一个
  HA_READ_MBR_CONTAIN,           // MBR 包含
  HA_READ_MBR_INTERSECT,         // MBR 相交
  HA_READ_MBR_WITHIN,            // MBR 内部
  HA_READ_MBR_DISJOINT,          // MBR 不相交
  HA_READ_MBR_EQUAL,             // MBR 相等
  HA_READ_INVALID = -1           // 无效值
};
```

| 枚举值 | 语义 | 对应 SQL 条件 |
|--------|------|--------------|
| `HA_READ_KEY_EXACT` | 精确定位（=） | `key = value` |
| `HA_READ_KEY_OR_NEXT` | 定位到 >= value 的第一条 | `key >= value` |
| `HA_READ_AFTER_KEY` | 定位到 > value 的第一条 | `key > value` |
| `HA_READ_BEFORE_KEY` | 定位到 < value 的最后一条（用于上界） | 上界 `key < value` |
| `HA_READ_KEY_OR_PREV` | 定位到 <= value 的最后一条 | 反向扫描用 |

### 2.3 SEL_ARG —— 范围条件的基本单元

定义在 `sql/range_optimizer/tree.h:465`。

每个 `SEL_ARG` 对象表示一个 **"基本区间"**：

```
min_value <=?  table.keypartX  <=? max_value
```

核心字段：

| 字段 | 含义 |
|------|------|
| `min_flag` | 最小值标志（`NO_MIN_RANGE`, `NEAR_MIN` 等） |
| `max_flag` | 最大值标志（`NO_MAX_RANGE`, `NEAR_MAX` 等） |
| `part` | 属于索引的第几个 key part |
| `min_value`, `max_value` | 范围的最小/最大值指针 |
| `rkey_func_flag` | 空间索引的 R-tree 读取函数 |
| `next_key_part` | 指向下一个 key part 的 SEL_ROOT |
| `next`, `prev` | 同一 key part 上有序区间列表的双向链表 |
| `left`, `right`, `parent` | 红黑树子节点/父节点 |
| `is_ascending` | 是否为升序 key part |

`SEL_ARG` 对象以 **红黑树** 组织，同时通过 `next`/`prev` 链表维护有序的区间列表。不同 key part 之间通过 `next_key_part` 指针连接，形成一个图结构。

### 2.4 SEL_ROOT —— SEL_ARG 树的根节点包装

```cpp
class SEL_ROOT {
  enum class Type { IMPOSSIBLE, MAYBE_KEY, KEY_RANGE } type;
  SEL_ARG *root;  // 指向红黑树根节点
};
```

- `IMPOSSIBLE`：该索引上的条件永远为假
- `MAYBE_KEY`：条件引用了其他表，需要后续确定
- `KEY_RANGE`：有可用的范围条件

### 2.5 SEL_TREE —— 整张表的范围条件集合

```cpp
class SEL_TREE {
  enum Type { IMPOSSIBLE, ALWAYS, KEY } type;
  Mem_root_array<SEL_ROOT *> keys;  // 每个索引一个 SEL_ROOT
  bool inexact;                      // 是否为宽松表示（需要回表过滤）
};
```

`keys[i]` 对应表上第 `i` 个索引的范围条件。

### 2.6 QUICK_RANGE —— 物化后的索引范围

定义在 `sql/range_optimizer/range_optimizer.h:69-170`。

```cpp
class QUICK_RANGE {
  uchar *min_key, *max_key;       // 范围的最小/最大键值
  uint16 min_length, max_length;  // 键值长度
  uint16 flag;                    // key_range_flags 标志位的组合
  enum ha_rkey_function rkey_func_flag;  // 空间索引专用
  key_part_map min_keypart_map, max_keypart_map;  // 使用的 keypart 位图
};
```

`QUICK_RANGE` 是 `SEL_ARG` 树物化后的结果，包含了实际的键值数据和标志位，可直接用于存储引擎层的索引访问。

---

## 3. 阶段一：WHERE 条件 → SEL_ARG 树

### 3.1 入口函数 get_mm_tree()

源码位置：`sql/range_optimizer/range_analysis.cc:845`

```cpp
SEL_TREE *get_mm_tree(THD *thd, RANGE_OPT_PARAM *param,
                      table_map prev_tables, table_map read_tables,
                      table_map current_table, bool remove_jump_scans,
                      Item *cond);
```

此函数递归处理 WHERE 条件树：

1. **AND/OR 复合条件**：递归调用 `get_mm_tree()`，然后使用 `tree_and()` 或 `tree_or()` 合并结果
2. **常量条件**：直接返回 `ALWAYS` 或 `IMPOSSIBLE`
3. **函数条件**（`=`、`<`、`>`、`BETWEEN`、`IN`、`LIKE` 等）：调用 `get_full_func_mm_tree()`

### 3.2 条件分发 get_full_func_mm_tree()

源码位置：`sql/range_optimizer/range_analysis.cc:749`

该函数处理各种函数类型的条件：
- 找到条件中的字段（`Item_field`）
- 考虑等值传播（`Item_equal`），为同一等值集合中的所有字段生成范围
- 调用 `get_func_mm_tree()` → `get_mm_parts()` → `get_mm_leaf()` 逐层处理

### 3.3 叶子节点构造 get_mm_leaf()

源码位置：`sql/range_optimizer/range_analysis.cc:1370`

这是 **设置 `min_flag`/`max_flag` 的关键函数**，它根据比较运算符类型创建 `SEL_ARG` 节点并设置标志位。核心逻辑如下：

#### (a) IS NULL / IS NOT NULL

```cpp
if (type == Item_func::ISNOTNULL_FUNC) {
    root->min_flag = NEAR_MIN;     // IS NOT NULL → X > NULL
    root->max_flag = NO_MAX_RANGE;
}
// IS NULL: min_value = max_value = NULL（默认等值区间）
```

#### (b) LIKE 条件

通过 `my_like_range()` 计算前缀范围的 `min_str` / `max_str`，创建一个半开区间的 `SEL_ARG`。

#### (c) 等值条件（EQ_FUNC, EQUAL_FUNC）

```cpp
// min_value = max_value = 字段的键图像（key image）
// min_flag = 0, max_flag = 0 → 表示精确的点区间
```

#### (d) 小于 / 小于等于（LT_FUNC, LE_FUNC）

```cpp
case Item_func::LT_FUNC:
case Item_func::LE_FUNC:
    // 根据存储值与原始值的比较结果，可能设置 NEAR_MAX
    if ((type == LT_FUNC && cmp_value >= 0) ||
        (type == LE_FUNC && cmp_value > 0))
        tree->root->max_flag = NEAR_MAX;  // 不含右端点
    
    if (!field->is_nullable())
        tree->root->min_flag = NO_MIN_RANGE;  // 从 -∞ 开始
    else
        tree->root->min_flag = NEAR_MIN;      // 从 NULL 之后开始
```

#### (e) 大于 / 大于等于（GT_FUNC, GE_FUNC）

```cpp
case Item_func::GT_FUNC:
case Item_func::GE_FUNC:
    // 根据存储值与原始值的比较结果，可能设置 NEAR_MIN
    if ((type == GT_FUNC && cmp_value <= 0) ||
        (type == GE_FUNC && cmp_value < 0))
        tree->root->min_flag = NEAR_MIN;  // 不含左端点
    
    tree->root->max_flag = NO_MAX_RANGE;  // 到 +∞ 结束
```

#### (f) 空间操作符

```cpp
case Item_func::SP_EQUALS_FUNC:
    tree->root->set_gis_index_read_function(HA_READ_MBR_EQUAL);
case Item_func::SP_WITHIN_FUNC:
    tree->root->set_gis_index_read_function(HA_READ_MBR_CONTAIN);
case Item_func::SP_CONTAINS_FUNC:
    tree->root->set_gis_index_read_function(HA_READ_MBR_WITHIN);
// ... 其他空间操作符类似
```

其中 `set_gis_index_read_function()` 的实现：

```cpp
void set_gis_index_read_function(const enum ha_rkey_function rkey_func) {
    min_flag = GEOM_FLAG;          // 标记为几何范围
    rkey_func_flag = rkey_func;    // 设置读取函数
    max_flag = NO_MAX_RANGE;       // 无上界
}
```

### 3.4 flag 含义总结

| SQL 条件 | min_flag | max_flag | 语义 |
|----------|----------|----------|------|
| `key = 5` | `0` | `0` | `[5, 5]` 精确等值 |
| `key > 5` | `NEAR_MIN` | `NO_MAX_RANGE` | `(5, +∞)` |
| `key >= 5` | `0` | `NO_MAX_RANGE` | `[5, +∞)` |
| `key < 5` | `NO_MIN_RANGE` | `NEAR_MAX` | `(-∞, 5)` |
| `key <= 5` | `NO_MIN_RANGE` | `0` | `(-∞, 5]` |
| `IS NOT NULL` | `NEAR_MIN` | `NO_MAX_RANGE` | `(NULL, +∞)` |
| `IS NULL` | `0` | `0` | `[NULL, NULL]` |
| `ST_Contains(...)` | `GEOM_FLAG` | `NO_MAX_RANGE` | 空间范围 |

---

## 4. 阶段二：SEL_ARG 树 → QUICK_RANGE 数组

### 4.1 两条路径

`SEL_ARG` 树到 `QUICK_RANGE` 数组有两条转换路径：

#### 路径 A：get_ranges_from_tree()（用于构建 AccessPath）

源码位置：`sql/range_optimizer/index_range_scan_plan.cc:774`

```
get_ranges_from_tree()
    └─ get_ranges_from_tree_given_base()   // 递归遍历 SEL_ARG 树
        └─ 创建 QUICK_RANGE 对象
```

#### 路径 B：sel_arg_range_seq_next()（用于行计数估算 check_quick_select）

源码位置：`sql/range_optimizer/index_range_scan_plan.cc:306`

```
check_quick_select()
    └─ handler::multi_range_read_info_const()
        └─ sel_arg_range_seq_next()
            └─ handler::records_in_range()   // 估算范围内的行数
```

### 4.2 get_ranges_from_tree_given_base() 详解

此函数递归遍历 `SEL_ARG` 树的每个节点，将多 keypart 的范围合并为连续的键值缓冲区。

关键步骤：

1. **存储当前 keypart 的值**：调用 `store_min_max_values()` 将 `min_value`/`max_value` 写入键缓冲区
2. **处理下一个 keypart**：
   - 如果当前是等值条件且有下一个 keypart → **递归**，将键值拼接
   - 如果不是等值但有下一个 keypart → 调用 `store_next_min_max_keys()` 做 **最后一搏优化**（缩窄扫描范围的起止位置）
3. **确定 flag**：合并 `min_flag | max_flag`，检查是否为 `EQ_RANGE`
4. **创建 QUICK_RANGE**：

```cpp
QUICK_RANGE *range = new QUICK_RANGE(
    return_mem_root,
    base_min_key, (uint)(tmp_min_key - base_min_key),
    min_part >= 0 ? make_keypart_map(min_part) : 0,
    base_max_key, (uint)(tmp_max_key - base_max_key),
    max_part >= 0 ? make_keypart_map(max_part) : 0,
    flag,
    node->rkey_func_flag);
```

### 4.3 sel_arg_range_seq_next() 中的 ha_rkey_function 确定

此函数在行计数估算路径中使用，它确定 `start_key.flag` 的方式：

**非空间索引**（`sql/range_optimizer/index_range_scan_plan.cc:494-501`）：

```cpp
// 起始键的 flag
range->start_key.flag =
    (cur->min_key_flag & NEAR_MIN ? HA_READ_AFTER_KEY : HA_READ_KEY_EXACT);

// 结束键的 flag
range->end_key.flag =
    (cur->max_key_flag & NEAR_MAX ? HA_READ_BEFORE_KEY : HA_READ_AFTER_KEY);
```

**空间索引**（`sql/range_optimizer/index_range_scan_plan.cc:481`）：

```cpp
range->start_key.flag = cur->rkey_func_flag;  // 直接使用空间操作函数
```

**等值范围检测**（`sql/range_optimizer/index_range_scan_plan.cc:517-518`）：

```cpp
if (is_eq_range_pred) {
    range->range_flag = EQ_RANGE;
}
```

---

## 5. 阶段三：QUICK_RANGE → 存储引擎索引定位

### 5.1 执行入口

`IndexRangeScanIterator::Read()` → `ha_multi_range_read_next()` → `multi_range_read_next()` → `read_range_first()`

### 5.2 quick_range_seq_next() —— QUICK_RANGE 到 key_range 的转换

源码位置：`sql/range_optimizer/index_range_scan.cc:160-188`

```cpp
uint quick_range_seq_next(range_seq_t rseq, KEY_MULTI_RANGE *range) {
    QUICK_RANGE *cur = *(ctx->cur);
    key_range *start_key = &range->start_key;
    key_range *end_key = &range->end_key;

    start_key->key = cur->min_key;
    start_key->length = cur->min_length;
    start_key->keypart_map = cur->min_keypart_map;
    start_key->flag = ((cur->flag & NEAR_MIN)   ? HA_READ_AFTER_KEY
                       : (cur->flag & EQ_RANGE) ? HA_READ_KEY_EXACT
                                                : HA_READ_KEY_OR_NEXT);

    end_key->key = cur->max_key;
    end_key->length = cur->max_length;
    end_key->keypart_map = cur->max_keypart_map;
    end_key->flag =
        (cur->flag & NEAR_MAX ? HA_READ_BEFORE_KEY : HA_READ_AFTER_KEY);
}
```

这与 `QUICK_RANGE::make_min_endpoint()` / `make_max_endpoint()` 的逻辑完全一致。

### 5.3 QUICK_RANGE::make_min_endpoint() —— 决定起始位置的 ha_rkey_function

源码位置：`sql/range_optimizer/range_optimizer.h:121-128`

```cpp
void make_min_endpoint(key_range *kr) {
    kr->key = (const uchar *)min_key;
    kr->length = min_length;
    kr->keypart_map = min_keypart_map;
    kr->flag = ((flag & NEAR_MIN)   ? HA_READ_AFTER_KEY
                : (flag & EQ_RANGE) ? HA_READ_KEY_EXACT
                                    : HA_READ_KEY_OR_NEXT);
}
```

**起始键 flag 的三路判断**：

| QUICK_RANGE::flag 条件 | ha_rkey_function | 含义 |
|------------------------|------------------|------|
| `flag & NEAR_MIN` | `HA_READ_AFTER_KEY` | 不含左端点，定位到严格大于 min_key 的第一条记录 |
| `flag & EQ_RANGE` | `HA_READ_KEY_EXACT` | 等值条件，精确定位到 min_key 对应的记录 |
| 其他 | `HA_READ_KEY_OR_NEXT` | 含左端点，定位到 >= min_key 的第一条记录 |

### 5.4 QUICK_RANGE::make_max_endpoint() —— 决定结束位置的 ha_rkey_function

源码位置：`sql/range_optimizer/range_optimizer.h:160-169`

```cpp
void make_max_endpoint(key_range *kr) {
    kr->key = (const uchar *)max_key;
    kr->length = max_length;
    kr->keypart_map = max_keypart_map;
    kr->flag = (flag & NEAR_MAX ? HA_READ_BEFORE_KEY : HA_READ_AFTER_KEY);
}
```

**结束键 flag 的二路判断**：

| QUICK_RANGE::flag 条件 | ha_rkey_function | 含义 |
|------------------------|------------------|------|
| `flag & NEAR_MAX` | `HA_READ_BEFORE_KEY` | 不含右端点，在严格小于 max_key 的记录处停止 |
| 其他 | `HA_READ_AFTER_KEY` | 包含右端点，扫描到 max_key 之后停止（用于键前缀匹配） |

注意：结束键使用 `HA_READ_AFTER_KEY` 而非 `HA_READ_KEY_EXACT` 的原因是 **键前缀匹配**——当使用复合索引的部分前缀时，需要找到具有该前缀的所有记录。

### 5.5 handler::read_range_first() —— 存储引擎层的索引定位

源码位置：`sql/handler.cc:7314-7342`

```cpp
int handler::read_range_first(const key_range *start_key,
                              const key_range *end_key,
                              bool eq_range_arg, bool sorted) {
    eq_range = eq_range_arg;
    set_end_range(end_key, RANGE_SCAN_ASC);
    range_key_part = table->key_info[active_index].key_part;

    if (!start_key)
        result = ha_index_first(table->record[0]);
    else
        result = ha_index_read_map(table->record[0], start_key->key,
                                   start_key->keypart_map, start_key->flag);
    // ...
}
```

最终调用 `ha_index_read_map()`，传递的 `start_key->flag` 就是之前确定的 `ha_rkey_function` 值。该函数将请求转发给具体的存储引擎（如 InnoDB）进行 B+ 树的查找定位。

### 5.6 handler::multi_range_read_next() 的范围迭代

源码位置：`sql/handler.cc:6438-6497`

该函数在 MRR（Multi-Range Read）框架下迭代多个范围：

```cpp
int handler::multi_range_read_next(char **range_info) {
    // 首次调用跳转到 start
    if (!mrr_have_range) { mrr_have_range = true; goto start; }

    do {
        // 对于非唯一等值范围，调用 read_range_next() 读取下一行
        if (!((mrr_cur_range.range_flag & UNIQUE_RANGE) &&
              (mrr_cur_range.range_flag & EQ_RANGE))) {
            result = read_range_next();
        }

    start:
        // 获取下一个范围并调用 read_range_first() 定位
        while (!(range_res = mrr_funcs.next(mrr_iter, &mrr_cur_range))) {
            result = read_range_first(
                mrr_cur_range.start_key.keypart_map ? &mrr_cur_range.start_key : nullptr,
                mrr_cur_range.end_key.keypart_map ? &mrr_cur_range.end_key : nullptr,
                mrr_cur_range.range_flag & EQ_RANGE,
                mrr_is_output_sorted);
        }
    } while (...);
}
```

其中 `mrr_funcs.next` 就是 `quick_range_seq_next()`，它从 `QUICK_RANGE` 数组中取出下一个范围并转换为 `key_range`。

---

## 6. 完整示例

### 示例 1：`WHERE key_col = 5`

```
get_mm_leaf(): 
  → SEL_ARG: min_value=5, max_value=5, min_flag=0, max_flag=0

get_ranges_from_tree_given_base():
  → flag = EQ_RANGE（因为 min_key == max_key 且无其他标志）
  → QUICK_RANGE: flag = EQ_RANGE

quick_range_seq_next():
  → start_key.flag = (EQ_RANGE → HA_READ_KEY_EXACT)
  → end_key.flag   = HA_READ_AFTER_KEY

handler::read_range_first():
  → ha_index_read_map(buf, key=5, keypart_map, HA_READ_KEY_EXACT)
  → InnoDB：精确定位到 key=5 的第一条记录
```

### 示例 2：`WHERE key_col > 10`

```
get_mm_leaf():
  → SEL_ARG: min_value=10, max_value=∞, min_flag=NEAR_MIN, max_flag=NO_MAX_RANGE

get_ranges_from_tree_given_base():
  → flag = NEAR_MIN | NO_MAX_RANGE
  → QUICK_RANGE: flag = NEAR_MIN | NO_MAX_RANGE

quick_range_seq_next():
  → start_key.flag = (NEAR_MIN → HA_READ_AFTER_KEY)
  → end_key 无效（NO_MAX_RANGE）

handler::read_range_first():
  → ha_index_read_map(buf, key=10, keypart_map, HA_READ_AFTER_KEY)
  → InnoDB：定位到严格大于 10 的第一条记录
```

### 示例 3：`WHERE key_col >= 10`

```
get_mm_leaf():
  → SEL_ARG: min_value=10, max_value=∞, min_flag=0, max_flag=NO_MAX_RANGE

get_ranges_from_tree_given_base():
  → flag = NO_MAX_RANGE
  → QUICK_RANGE: flag = NO_MAX_RANGE（无 NEAR_MIN，无 EQ_RANGE）

quick_range_seq_next():
  → start_key.flag = (无NEAR_MIN 且 无EQ_RANGE → HA_READ_KEY_OR_NEXT)
  → end_key 无效

handler::read_range_first():
  → ha_index_read_map(buf, key=10, keypart_map, HA_READ_KEY_OR_NEXT)
  → InnoDB：定位到 >= 10 的第一条记录
```

### 示例 4：`WHERE key_col < 20`

```
get_mm_leaf():
  → SEL_ARG: min_value=-∞, max_value=20, min_flag=NO_MIN_RANGE, max_flag=NEAR_MAX

get_ranges_from_tree_given_base():
  → flag = NO_MIN_RANGE | NEAR_MAX
  → QUICK_RANGE: flag = NO_MIN_RANGE | NEAR_MAX

quick_range_seq_next():
  → start_key 无效（NO_MIN_RANGE）
  → end_key.flag = (NEAR_MAX → HA_READ_BEFORE_KEY)

handler::read_range_first():
  → start_key 为空 → ha_index_first()
  → 从索引第一条记录开始扫描，到 < 20 的位置停止
```

### 示例 5：`WHERE a = 3 AND b >= 5`（复合索引 (a, b)）

```
get_mm_tree():
  → 对 a=3 创建 SEL_ARG: part=0, min=3, max=3, flags=0
  → 对 b>=5 创建 SEL_ARG: part=1, min=5, max=∞, min_flag=0, max_flag=NO_MAX_RANGE
  → tree_and() 将 b 的 SEL_ROOT 连接到 a 的 next_key_part

get_ranges_from_tree_given_base():
  → a=3 是等值 → 递归处理 b>=5
  → min_key = [3, 5], max_key = [3, +∞]
  → flag = NO_MAX_RANGE
  → QUICK_RANGE: min_key=[3,5], flag=NO_MAX_RANGE

quick_range_seq_next():
  → start_key.flag = HA_READ_KEY_OR_NEXT（无 NEAR_MIN 且无 EQ_RANGE）
  → end_key 无效

handler::read_range_first():
  → ha_index_read_map(buf, key=[3,5], keypart_map=0x03, HA_READ_KEY_OR_NEXT)
  → InnoDB：在复合索引中定位到 (a=3, b>=5) 的第一条记录
```

### 示例 6：`WHERE a >= 3 AND b IN (4, 9, 10)`（最后一搏优化）

当 key part a 是非等值条件（>= 3），后续 key part b 的条件无法形成子范围，但可以通过 `store_next_min_max_keys()` 缩窄起始位置：

```
→ 正常情况：范围 (a,b) >= (3, -∞)
→ 优化后：范围 (a,b) >= (3, 4)（取 b 的最小值 4 作为起始）
```

---

## 7. ha_rkey_function 确定规则总结

### 7.1 起始键（start_key）的 ha_rkey_function

```
if (flag & NEAR_MIN)
    → HA_READ_AFTER_KEY     // 开区间左端点：key > value
else if (flag & EQ_RANGE)
    → HA_READ_KEY_EXACT     // 等值条件：key = value
else
    → HA_READ_KEY_OR_NEXT   // 闭区间左端点：key >= value
```

决策流程图：

```
                  QUICK_RANGE::flag
                        │
                ┌───────┴───────┐
                │  NEAR_MIN ?   │
                └───────┬───────┘
               Yes ─────┼───── No
               │        │       │
      HA_READ_AFTER_KEY │  ┌────┴─────┐
                        │  │ EQ_RANGE?│
                        │  └────┬─────┘
                        │  Yes──┼──No
                        │  │    │   │
                        │ HA_READ_  HA_READ_
                        │ KEY_EXACT KEY_OR_NEXT
```

### 7.2 结束键（end_key）的 ha_rkey_function

```
if (flag & NEAR_MAX)
    → HA_READ_BEFORE_KEY   // 开区间右端点：key < value
else
    → HA_READ_AFTER_KEY    // 闭区间右端点（或前缀匹配）：扫描到 value 之后
```

### 7.3 空间索引的 ha_rkey_function

空间索引通过 `GEOM_FLAG` 标记，`rkey_func_flag` 直接传递为 `start_key.flag`：

| 空间函数 | rkey_func_flag |
|---------|---------------|
| `ST_Equals` | `HA_READ_MBR_EQUAL` |
| `ST_Disjoint` | `HA_READ_MBR_DISJOINT` |
| `ST_Intersects` / `ST_Touches` / `ST_Crosses` / `ST_Overlaps` | `HA_READ_MBR_INTERSECT` |
| `ST_Within` | `HA_READ_MBR_CONTAIN`（注意反转） |
| `ST_Contains` | `HA_READ_MBR_WITHIN`（注意反转） |

---

## 8. 降序索引（DESC）的特殊处理

对于降序（DESC）key part，`SEL_ARG` 中 `min_value` / `max_value` 的语义发生反转：

- `store_min_max_values()` 中：ASC 时 min→min_key, max→max_key；DESC 时 min→max_key, max→min_key
- `get_min_flag()` / `get_max_flag()` 进行标志位的翻转
- `get_ranges_from_tree_given_base()` 中，DESC key part 遍历顺序从 `last()` 到 `prev`

---

## 9. 为什么需要 ha_rkey_function —— 两套标志位的设计考量

读者可能会疑惑：既然 `key_range_flags` 已经描述了范围的特征，为什么还需要 `ha_rkey_function`？能不能让存储引擎直接理解 `key_range_flags` 来完成索引定位？

答案是：**不能。`ha_rkey_function` 和 `key_range_flags` 服务于完全不同的抽象层次，不可相互替代。**

### 9.1 本质区别：区间描述 vs. 定位指令

**`key_range_flags`** 是范围优化器的 **内部语言**，以 bitmask 形式描述一个 **完整区间的形状**（两个端点的特征）：

```cpp
// 一个 flag 同时刻画了整个区间的多个特征，可以组合
flag = NEAR_MIN | NO_MAX_RANGE;           // (value, +∞)
flag = EQ_RANGE | UNIQUE_RANGE;           // 唯一索引上的精确匹配
flag = EQ_RANGE | SKIP_RECORDS_IN_RANGE;  // 等值 + 跳过 index dive
```

**`ha_rkey_function`** 是存储引擎 handler 接口的 **操作指令**，以枚举值形式表达一次 **单点索引定位操作** 的语义——"给你一个 key，从哪里开始读"：

```cpp
// 每次调用只传一个值，不可组合
int handler::index_read_map(uchar *buf, const uchar *key,
                            key_part_map keypart_map,
                            enum ha_rkey_function find_flag);
```

一个区间有起始和结束 **两个端点**，每个端点独立需要一个 `ha_rkey_function`。转换逻辑清晰地展示了这种"一拆为二"的过程：

```cpp
// 同一个 flag 的不同 bit 分别映射到不同的 ha_rkey_function
start_key->flag = ((flag & NEAR_MIN)   ? HA_READ_AFTER_KEY
                   : (flag & EQ_RANGE) ? HA_READ_KEY_EXACT
                                       : HA_READ_KEY_OR_NEXT);

end_key->flag = (flag & NEAR_MAX ? HA_READ_BEFORE_KEY : HA_READ_AFTER_KEY);
```

### 9.2 ha_rkey_function 的使用场景远不止范围扫描

`ha_rkey_function` 是存储引擎层 **通用的索引定位接口参数**，被多种场景使用，许多场景根本不经过范围优化器：

#### (a) HANDLER 语句 —— SQL 解析器直接生成

MySQL 的 `HANDLER ... READ` 语句允许用户直接指定读取模式（`sql/sql_handler.cc:676-735`）：

```sql
HANDLER t1 READ idx_a (>=, 5);   -- 直接传入 HA_READ_KEY_OR_NEXT
HANDLER t1 READ idx_a (>, 5);    -- 直接传入 HA_READ_AFTER_KEY
HANDLER t1 READ idx_a (=, 5);    -- 直接传入 HA_READ_KEY_EXACT
```

此时 `ha_rkey_function` 由 **SQL 解析器直接从语法生成**，没有任何 `key_range_flags` 参与。

#### (b) ref access / eq_ref 访问

`JOIN` 中的等值关联（如 `t1.a = t2.b`）通过 `ha_index_read_map()` + `HA_READ_KEY_EXACT` 定位，同样不经过范围优化器的 flag 体系。

#### (c) PREFIX 系列 —— flag 中无对应物

`HA_READ_PREFIX`、`HA_READ_PREFIX_LAST`、`HA_READ_PREFIX_LAST_OR_PREV` 这几个值用于 GROUP BY 优化、松散索引扫描等场景，在 `key_range_flags` 中 **完全没有对应的 bit**。

#### (d) 空间索引

空间索引的 `HA_READ_MBR_*` 系列值不是从 flag 的 bit 运算得来的，而是从 `SEL_ARG::rkey_func_flag` 直接传递。

### 9.3 ha_rkey_function 是跨存储引擎的抽象契约

`ha_rkey_function` 是 MySQL Server 与 **所有** 存储引擎之间的稳定 API 契约。每个存储引擎在内部再做一次转换，例如 InnoDB（`storage/innobase/handler/ha_innodb.cc:10096`）：

```cpp
page_cur_mode_t convert_search_mode_to_innobase(ha_rkey_function find_flag) {
    switch (find_flag) {
        case HA_READ_KEY_EXACT:
        case HA_READ_KEY_OR_NEXT:  return PAGE_CUR_GE;   // B+ 树 >=
        case HA_READ_AFTER_KEY:    return PAGE_CUR_G;     // B+ 树 >
        case HA_READ_BEFORE_KEY:   return PAGE_CUR_L;     // B+ 树 <
        case HA_READ_KEY_OR_PREV:
        case HA_READ_PREFIX_LAST:
        case HA_READ_PREFIX_LAST_OR_PREV:
                                   return PAGE_CUR_LE;    // B+ 树 <=
        // HA_READ_MBR_* → PAGE_CUR_CONTAIN / INTERSECT / WITHIN ...
    }
}
```

如果把 `key_range_flags` 直接暴露给存储引擎，会带来以下问题：

- 每个存储引擎都需要理解 bitmask 的组合语义（`NEAR_MIN | EQ_RANGE` 代表什么？）
- 范围优化器内部的 flag 变动会直接破坏存储引擎接口稳定性
- 不同场景（HANDLER、ref access 等）需要各自拼凑 flag，增加不必要的复杂度

### 9.4 语义精度的差异

`ha_rkey_function` 在某些地方提供了比 flag 更精确的语义：

- `HA_READ_KEY_EXACT` 和 `HA_READ_KEY_OR_NEXT` 在 InnoDB 中都映射到 `PAGE_CUR_GE`，但在 Server 层行为不同——前者在 key 不完全匹配时返回 `HA_ERR_KEY_NOT_FOUND`，后者会继续读取下一条
- `HA_READ_PREFIX_LAST` 语义为"找到匹配前缀的最后一条"，这在 flag 体系中根本无法表达（flag 没有"最后一条"的概念）

### 9.5 层次架构总结

```
  ┌────────────────────────────────────┐
  │   SQL 条件 / HANDLER 语句 / ref     │  ← 多种输入来源
  └──────────────┬─────────────────────┘
                 │
  ┌──────────────▼─────────────────────┐
  │   key_range_flags (bitmask)         │  ← 范围优化器内部表示
  │   描述"区间的形状"                    │     仅用于范围扫描路径
  │   一个 flag 同时描述两个端点          │
  └──────────────┬─────────────────────┘
                 │  转换（拆分为两个端点）
  ┌──────────────▼─────────────────────┐
  │   ha_rkey_function (enum)           │  ← 存储引擎 API 的通用契约
  │   描述"一次索引定位操作的语义"        │     服务所有索引访问方式
  └──────────────┬─────────────────────┘
                 │  各引擎自行转换
  ┌──────────────▼─────────────────────┐
  │   PAGE_CUR_GE / PAGE_CUR_G         │  ← InnoDB 内部 B+ 树游标模式
  │   PAGE_CUR_LE / PAGE_CUR_L         │
  └────────────────────────────────────┘
```

本质上，这是经典的 **关注点分离** 设计：`key_range_flags` 是优化器内部的"区间描述语言"，`ha_rkey_function` 是跨越 Server/Engine 边界的"索引操作指令集"。前者只在范围扫描路径中存在，后者是所有索引访问方式的通用接口。两者各自演化、互不影响。

---

## 10. 关键源码文件索引

| 文件 | 作用 |
|------|------|
| `include/my_base.h` | `ha_rkey_function` 枚举、`key_range_flags` 定义 |
| `sql/range_optimizer/tree.h` | `SEL_ARG`、`SEL_ROOT`、`SEL_TREE` 类定义 |
| `sql/range_optimizer/tree.cc` | `SEL_ARG` 红黑树操作、`tree_and()`、`tree_or()` |
| `sql/range_optimizer/range_analysis.cc` | `get_mm_tree()`、`get_mm_leaf()` —— WHERE 到 SEL_ARG 的转换 |
| `sql/range_optimizer/range_optimizer.h` | `QUICK_RANGE` 类、`make_min_endpoint()`、`make_max_endpoint()` |
| `sql/range_optimizer/index_range_scan_plan.cc` | `get_ranges_from_tree()`、`sel_arg_range_seq_next()` |
| `sql/range_optimizer/index_range_scan.cc` | `IndexRangeScanIterator`、`quick_range_seq_next()` |
| `sql/handler.cc` | `read_range_first()`、`ha_index_read_map()`、`multi_range_read_next()` |
