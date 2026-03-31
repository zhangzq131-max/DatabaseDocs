# MySQL 8.0.41 Loose Index Scan 实现调研

## 1. 概念区分

代码里“loose index scan”涉及两类机制：

| 类型 | 用途 | 实现路径 |
|------|------|----------|
| **单表 loose index scan（GROUP / MIN-MAX / agg distinct）** | 单表 `GROUP BY`、`MIN/MAX`、`SUM(DISTINCT)` 等 | **GROUP_INDEX_SKIP_SCAN**（range optimizer） |
| **Semijoin LooseScan** | 半连接去重：内表按索引 key 每组只取一条 | **LooseScan 策略** + **RemoveDuplicatesOnIndexIterator** |

二者“跳过不满足条件的记录”的方式不同，下面分开说明。

---

## 2. GROUP_INDEX_SKIP_SCAN：如何跳到“下一个 group”

单表的 loose index scan（GROUP BY、MIN/MAX、agg distinct）由 `GroupIndexSkipScanIterator` 执行，核心是 **next_prefix()**：确定下一个满足条件的 **group 前缀**，并让当前 record 指向该 group 的第一条。

### 2.1 无 prefix range（无前缀范围条件）

当没有对 group 前缀做范围约束时（`prefix_ranges->empty()`）：

- **第一次**：`ha_index_first(record)`，取索引第一条，再 `key_copy(group_prefix, record, ...)` 得到当前 group 前缀。
- **之后**：调用 **index_next_different(is_index_scan, file, key_part, record, group_prefix, group_prefix_len, group_key_parts)** 跳到下一个不同的前缀。

与 Index Skip Scan 相同，`index_next_different` 有两种行为（`sql/range_optimizer/range_optimizer.cc`）：

- **is_index_scan == true**：循环 `ha_index_next(record)`，直到 `key_cmp(key_part, group_prefix, group_prefix_len) != 0`，即**逐行 next 直到前缀不同**。
- **is_index_scan == false**：**一次** `ha_index_read_map(record, group_prefix, keypart_map, HA_READ_AFTER_KEY)`，在 B-tree 上**直接定位**到“严格大于当前 group_prefix”的第一条记录，即**直接跳到下一个 group**。

`is_index_scan` 在规划阶段设定（`group_index_skip_scan_plan.cc`）：仅在 **agg distinct 且该路径被选为更优** 时置为 true，其余情况为 false，即**默认用 HA_READ_AFTER_KEY 直接跳**，不是逐行 next。

```498:514:sql/range_optimizer/group_index_skip_scan.cc
  } else {
    if (!seen_first_key) {
      int result = table()->file->ha_index_first(table()->record[0]);
      ...
    } else {
      int result = index_next_different(
          is_index_scan, table()->file, index_info->key_part,
          table()->record[0], group_prefix, group_prefix_len, group_key_parts);
      ...
    }
  }
  key_copy(group_prefix, table()->record[0], index_info, group_prefix_len);
```

### 2.2 有 prefix range（有前缀范围条件）

当存在对 group 前缀的范围条件时，用 **get_next_prefix()**：

- 在**当前 range** 内：用 **ha_index_read_map(record, cur_prefix, keypart_map, HA_READ_AFTER_KEY)** 找“严格大于 cur_prefix”的下一条，得到下一个不同前缀；若已超出当前 range（compare_key 与 previous_endpoint），则返回成功前会先判定是否要切到下一个 range。
- 若当前 range 已耗尽或需要进入下一段：取 `prefix_ranges[cur_prefix_range_idx++]`，再 **ha_read_range_first(...)** 在该 range 内从头开始读。

因此：**有 prefix range 时，“跳过”到下一个 group 也是通过 HA_READ_AFTER_KEY 在索引上直接定位**，而不是在引擎层逐行 next。

```536:557:sql/range_optimizer/group_index_skip_scan.cc
int GroupIndexSkipScanIterator::get_next_prefix(...) {
  ...
  if (last_prefix_range != nullptr) {
    table()->file->set_end_range(nullptr, handler::RANGE_SCAN_ASC);
    int result = table()->file->ha_index_read_map(
        table()->record[0], cur_prefix, keypart_map, HA_READ_AFTER_KEY);
    ...
  }
  ...
  int result = table()->file->ha_read_range_first(...);
  ...
}
```

### 2.3 小结（GROUP_INDEX_SKIP_SCAN）

- **跳过方式**：与 Index Skip Scan 一致——通过 **index_next_different**，在默认（is_index_scan=false）或 get_next_prefix 内使用 **ha_index_read_map(..., HA_READ_AFTER_KEY)**，在 B-tree 上**直接定位**到下一个不同前缀，而不是逐行 next。
- 仅在对 agg distinct 做特殊成本选择时才会设 `is_index_scan=true`，此时才退化为“逐行 ha_index_next 直到 key_cmp 不同”。

---

## 3. Semijoin LooseScan：如何“跳过”重复 key 的行

Semijoin LooseScan 策略要求：内表访问按 **loosescan_key** 有序（即按用于去重的索引 key 有序），这样相同 key 前缀会连续出现。**“跳过”不满足条件的记录**不是在存储引擎层跳 key，而是在迭代器层**按 key 前缀去重**。

### 3.1 策略设置（排序与 key 长度）

- 选用 LooseScan 时，会 **set_need_sorted_output(tab->range_scan())**，保证内表按 **pos->loosescan_key** 有序。
- **loosescan_key_len** = 前 **loosescan_parts** 个 key part 的总长度，用于“按前缀视为同一组”。

```1543:1565:sql/sql_select.cc
        /* For LooseScan, duplicate elimination is based on rows being sorted
           on key. ... */
        if (tab->range_scan()) {
          set_need_sorted_output(tab->range_scan());
        }
        ...
        for (uint kp = 0; kp < pos->loosescan_parts; kp++)
          keylen += tab->table()->key_info[keyno].key_part[kp].store_length;
        tab->loosescan_key_len = keylen;
```

### 3.2 RemoveDuplicatesOnIndexIterator：在迭代器层过滤

执行时，在产生内表行的 AccessPath 之上会包一层 **RemoveDuplicatesOnIndexIterator**（`sql/iterators/composite_iterators.cc`），用 **loosescan_key_len** 作为比较长度：

- **Init()**：`m_first_row = true`。
- **Read()**：
  1. 从下层 source（如 index range scan）读一行。
  2. 若**不是第一行**且**当前行索引前缀与上一行相同**（`key_cmp(m_key->key_part, m_key_buf, m_key_len) == 0`），则 **continue**，即**不返回本行，继续读下一行**。
  3. 否则：把当前行 key 前缀拷到 `m_key_buf`，返回 0（输出本行）。

因此：**Semijoin LooseScan 的“跳过” = 在迭代器层比较当前行与上一行的 key 前缀；若相同则丢弃当前行并继续读下一行**。依赖底层按索引有序返回，所以相同前缀会连续出现，读到第一个“新前缀”就输出并更新 key_buf，后续同前缀行在 Read() 里被 continue 掉。

```2098:2119:sql/iterators/composite_iterators.cc
int RemoveDuplicatesOnIndexIterator::Read() {
  for (;;) {
    int err = m_source->Read();
    if (err != 0) return err;
    ...
    if (!m_first_row && key_cmp(m_key->key_part, m_key_buf, m_key_len) == 0) {
      // Same as previous row, so keep scanning.
      continue;
    }
    m_first_row = false;
    key_copy(m_key_buf, m_table->record[0], m_key, m_key_len);
    return 0;
  }
}
```

### 3.3 小结（Semijoin LooseScan）

- **不**在存储引擎层用 HA_READ_AFTER_KEY 跳 key。
- **在迭代器层**：用上一行的 key 前缀（长度 loosescan_key_len）与当前行比较，相同则 **continue** 读下一行，实现“每组 key 前缀只保留一条”的跳过。

---

## 4. 对比总结

| 类型 | 跳过不满足条件记录的方式 | 是否在引擎层“跳 key” |
|------|--------------------------|----------------------|
| **GROUP_INDEX_SKIP_SCAN** | 通过 **index_next_different** → 默认 **ha_index_read_map(..., HA_READ_AFTER_KEY)** 直接定位到下一个不同 group 前缀；有 prefix range 时在 get_next_prefix 内同样用 HA_READ_AFTER_KEY。 | 是（B-tree 直接定位） |
| **Semijoin LooseScan** | 上层 **RemoveDuplicatesOnIndexIterator**：若当前行 key 前缀与上一行相同则 **continue**（不返回，继续读下一行）。 | 否（顺序读 + 内存比较） |

两者都依赖**索引有序**：GROUP_INDEX_SKIP_SCAN 用有序性做“按前缀跳”；Semijoin LooseScan 用有序性保证同前缀连续，便于在迭代器里按前缀去重。
