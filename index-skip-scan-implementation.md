# MySQL 8.0.41 索引 Skip Scan 实现调研

## 1. 结论摘要

- **有等值条件的前导 key part（A_1..A_k）**  
  枚举值来自 **WHERE 中的常量**（如 `kp1 IN (1,2,3)` 或 `kp1 = 1`），在规划阶段从范围树 SEL_TREE 中提取，存入 `eq_prefixes[].eq_key_prefixes`，执行时按序轮换。
- **被跳过的、无条件的前导 key part（B_1..B_m，如你说的 kp1）**  
  **不会**预先构造 kp1=1~n 的枚举；枚举值是在执行时 **沿索引顺序扫描，通过“下一个不同前缀”** 得到的，即 **来自 B-tree 中实际出现的前缀**。

因此：  
- “kp1=1~n” 若指 **有等值/IN 的 key part**，则 n 和每个值都来自 **查询条件**。  
- 若 kp1 是 **被跳过的、无条件的 key part**，则不存在事先的 1~n 枚举，而是 **边扫索引边用 `index_next_different` 得到下一个不同的 kp1**。

---

## 2. 索引结构与 key part 分类

Skip scan 适用的复合索引形式为（见 `get_best_skip_scan` 注释）：

```text
I = <A_1,...,A_k, B_1,..., B_m, C, [D_1,...,D_n]>
```

- **A_1..A_k**：有 **等值条件** 的 key part（EQ prefix），可来自 `=` 或 `IN`。
- **B_1..B_m**：**被跳过的** key part，**没有**等值/范围条件（规划时 `cur_range_root == nullptr`）。
- **C**：有 **范围条件** 的 key part（必须有）。
- **D_1..D_n**：可选的后续 key part。

对应到你的例子 `(kp1, kp2)`：

- 若 WHERE 只有 `kp2 BETWEEN ...`：  
  - kp1 = B_1（被跳过），  
  - kp2 = C（范围）。  
  - 此时 **kp1 的“枚举”来自索引扫描，见下**。
- 若 WHERE 有 `kp1 IN (1,2,3) AND kp2 BETWEEN ...`：  
  - kp1 = A_1，kp2 = C。  
  - 此时 **kp1 的枚举来自 WHERE 常量 1,2,3**，见第 3 节。

---

## 3. 有等值条件的前缀（A_1..A_k）：枚举来自 WHERE

规划阶段在 `sql/range_optimizer/index_skip_scan_plan.cc` 的 `get_best_skip_scan()` 中：

- 对每个 key part，从 `get_index_range_tree()` 得到的 `cur_index_range_tree`（SEL_TREE）中取该 key part 的 `SEL_ROOT`（`get_sel_root_for_keypart`）。
- 若该 key part 上有 **等值** 条件（`keypart_stage == EQUALITY_KEYPART`）：
  - 要求每个 SEL_ARG 的 `min_value == max_value`（及非 NEAR_MIN/NEAR_MAX 等），即严格等值。
  - 遍历 `cur_range->first()` 到 `cur_range->next`，把每个等值常量存入 **eq_prefixes**：

```342:379:sql/range_optimizer/index_skip_scan_plan.cc
    const SEL_ARG *cur_range = index_range_tree->root->first();
    ...
    for (uint i = 0; i < eq_prefix_key_parts;
         i++, cur_range = cur_range->next_key_part->root) {
      ...
      for (cur_range = first_range; cur_range;
           j++, cur_range = cur_range->next) {
        ...
        memcpy(eq_prefixes[i].eq_key_prefixes[pos], cur_range->min_value,
               field_length);
      }
    }
```

因此：**A_1..A_k 的枚举值 = WHERE 里该 key part 上的等值/IN 常量**，来自范围分析得到的 SEL_TREE，不是从统计信息或 1~n 构造的。

执行时在 `index_skip_scan.cc`：

- 当前等值前缀保存在 `eq_prefix`，由 `eq_prefixes` 组装。
- 换到“下一个”等值前缀用 `next_eq_prefix()`：在 `eq_prefixes[].eq_key_prefixes` 上按序轮换（类似多进制计数器），见 `IndexSkipScanIterator::next_eq_prefix()`。

---

## 4. 被跳过的 key part（B_1..B_m）：枚举来自索引上的“下一个不同前缀”

被跳过的 key part 在规划阶段 **没有** 任何枚举列表：只是确定“从第几个 key part 到第几个是 distinct_prefix（A_1..A_k + B_1..B_m）”，以及哪个是 range key part C。

执行时在 `IndexSkipScanIterator::Read()` 中（`sql/range_optimizer/index_skip_scan.cc`）：

1. **取“当前 distinct 前缀”的第一条**  
   - 若 `eq_prefix_key_parts == 0`：`ha_index_first(record)`，即索引第一条。  
   - 若 `eq_prefix_key_parts > 0`：用当前 `eq_prefix` 做 `ha_index_read_map(..., HA_READ_KEY_OR_NEXT)` 定位。
2. **把当前行的 (A_1..A_k, B_1..B_m) 拷到 `distinct_prefix`**（即“当前 kp1 的值”来自当前记录）：

```291:293:sql/range_optimizer/index_skip_scan.cc
      // Save the prefix of this group for subsequent calls.
      key_copy(distinct_prefix, table()->record[0], index_info,
               distinct_prefix_len);
```

3. **在 (distinct_prefix, range_on_C) 上做一次 range 扫描**（`ha_read_range_first` / `ha_read_range_next`）。
4. **当前前缀扫完后，要“下一个 kp1”时**：不查表、不查统计，而是调 **index_next_different**：

```283:287:sql/range_optimizer/index_skip_scan.cc
        result = index_next_different(false /* is_index_scan */, table()->file,
                                      index_info->key_part, table()->record[0],
                                      distinct_prefix, distinct_prefix_len,
                                      distinct_prefix_key_parts);
```

`index_next_different` 在 `sql/range_optimizer/range_optimizer.cc` 中实现（skip scan 走 `is_index_scan == false` 分支）：

```1401:1420:sql/range_optimizer/range_optimizer.cc
int index_next_different(bool is_index_scan, handler *file,
                         KEY_PART_INFO *key_part, uchar *record,
                         const uchar *group_prefix, uint group_prefix_len,
                         uint group_key_parts) {
  file->set_end_range(nullptr, handler::RANGE_SCAN_ASC);
  if (is_index_scan) {
    // ... 逐行 ha_index_next 直到前缀不同
  } else
    return file->ha_index_read_map(record, group_prefix,
                                   make_prev_keypart_map(group_key_parts),
                                   HA_READ_AFTER_KEY);
}
```

即：用当前 **distinct_prefix**（当前 kp1 的值）作为 key，用 **HA_READ_AFTER_KEY** 读“严格大于该 key”的**下一条**索引记录。这样一次就得到 **下一个不同的 (kp1) 前缀**，从而得到“下一个 kp1 的枚举值”。

因此：**被跳过的 kp1 的“枚举” = 索引 B-tree 中按序出现的、不同的 kp1 值**，是执行时 **边扫边跳** 得到的，不是事先从 1~n 或统计信息构造的。

---

## 5. 成本估算里“组数”与枚举的关系

成本在 `cost_skip_scan()`（`index_skip_scan_plan.cc`）中计算：

- `num_groups` 用 **索引统计** `rec_per_key(distinct_key_parts - 1)` 或 `guess_rec_per_key` 估算“每个 (A_1..A_k, B_1..B_m) 前缀有多少行”，再结合 `quick_prefix_records` 得到组数，用于 IO/CPU 成本。
- 这里 **没有** 为 kp1 构造 1~n 的枚举列表；只是用统计量估计“会有多少组”，从而估计代价。真正执行时组（即 kp1 的取值）仍由上面的 **index_first / index_read_map + index_next_different** 逐组产生。

---

## 6. 小结表

| 前导 key part 类型 | 是否“跳过” | 枚举值来源 | 实现位置 |
|--------------------|------------|------------|----------|
| 有等值/IN（A_1..A_k） | 否，用于定位 | WHERE 常量，从 SEL_TREE 提取 | `get_best_skip_scan` → `eq_prefixes`；执行时 `next_eq_prefix()` |
| 无条件（B_1..B_m）   | 是         | 索引中实际存在的前缀，执行时“下一个不同前缀” | `Read()` → `key_copy(distinct_prefix)` + `index_next_different` → `ha_index_read_map(..., HA_READ_AFTER_KEY)` |

所以：**复合索引 (kp1, kp2) 下，若 kp1 无条件、kp2 有范围**，则 kp1 的取值 **不是** 预先构造的 1~n，而是 **沿索引顺序，用“当前前缀 + HA_READ_AFTER_KEY” 逐次得到下一个不同的 kp1**。

---

## 7. InnoDB：`ha_index_read_map(..., HA_READ_AFTER_KEY)` 是直接定位，不是逐行 next

Skip scan 在取“下一个不同前缀”时调用的是：

```c
file->ha_index_read_map(record, group_prefix, keypart_map, HA_READ_AFTER_KEY);
```

在 InnoDB 里，这是一次 **按 key 在 B-tree 上直接定位到“严格大于该 key 的第一条记录”**，**不是** 先定位到等于 key 再逐行 `ha_index_next` 直到前缀不同。

### 7.1 调用链

1. **handler 层**  
   `ha_index_read_map()` 会根据 `keypart_map` 算出使用的 key 长度，然后调用引擎的 `index_read(buf, key, key_len, find_flag)`。这里 `find_flag == HA_READ_AFTER_KEY`。

2. **ha_innobase::index_read**（`storage/innobase/handler/ha_innodb.cc`）  
   - 把 MySQL 的 key 转成 InnoDB 的 `search_tuple`（`row_sel_convert_mysql_key_to_innobase`）。  
   - 把 MySQL 的查找方式转成 InnoDB 的 page cursor 模式：

     ```c
     page_cur_mode_t mode = convert_search_mode_to_innobase(find_flag);
     // HA_READ_AFTER_KEY -> PAGE_CUR_G
     ```

     （`HA_READ_AFTER_KEY` 对应 `PAGE_CUR_G`，即 “strictly greater than”。）  
   - 调用 `row_search_mvcc(buf, mode, m_prebuilt, match_mode, 0)`，其中 `direction == 0` 表示**首次定位**，不是“下一次”。

3. **row_search_mvcc**（`storage/innobase/row/row0sel.cc`）  
   当 `direction == 0` 且 `dtuple_get_n_fields(search_tuple) > 0` 时，用 **当前 search_tuple 和 mode** 做一次 B-tree 定位：

   ```c
   pcur->open_no_init(index, search_tuple, mode, BTR_SEARCH_LEAF, 0, &mtr, UT_LOCATION_HERE);
   ```

   这里 `mode == PAGE_CUR_G`，即“找第一条 **严格大于** search_tuple 的索引记录”。

4. **btr_pcur_t::open_no_init**  
   内部会走到 **btr_cur_search_to_nth_level**，在 B-tree 上从根一路下到叶子，再在叶子页上做 **page_cur_search_with_match(..., PAGE_CUR_G, ...)**。

5. **叶子页上的语义**（`storage/innobase/page/page0cur.cc`）  
   注释明确说明：

   - **PAGE_CUR_G**：要回答的是 “tuple < X” 的查询，即把 cursor 放在**第一个满足 X > tuple** 的物理记录 X 上。  
   - 实现是在**当前叶子页上做二分查找**（先按 page directory 二分，再在 slot 内线性），找到第一个 “rec > tuple” 的位置，**没有**“先定位到等于再 next”的循环。

因此：**一次 `ha_index_read_map(record, distinct_prefix, keypart_map, HA_READ_AFTER_KEY)` = 一次 B-tree 搜索（根到叶 + 叶内二分）直接得到“严格大于 distinct_prefix 的第一条记录”**，不是逐行 next。

### 7.2 与“逐行 next”的对比

- **index_next_different(is_index_scan == true)** 时才会用“先 next 再比较前缀”的循环（`ha_index_next` 直到 `key_cmp(...) != 0`）。  
- Skip scan 用的是 **index_next_different(is_index_scan == false)**，走的是 **ha_index_read_map(..., HA_READ_AFTER_KEY)** 这条路径，即上面这条“直接按 key 定位到严格大于”的路径。

所以：**InnoDB 对 HA_READ_AFTER_KEY 的实现是“直接定位到下一个严格大于该 key 的记录”，而不是逐行 next。**
