# InnoDB 行锁隐式锁实现原理

> 基于 MySQL 8.0.41 源码分析

---

## 一、核心设计思想

**隐式锁（Implicit Lock）是一种"无锁对象"的锁**：当一个事务插入或修改了某行记录，在其提交之前，InnoDB 并不为该行在内存 lock 系统中创建任何锁对象。只有当另一个事务试图对该行加锁、并发现潜在冲突时，才会"物化"出一个显式锁对象。

这个设计避免了大量无竞争场景下的锁内存分配，是 InnoDB 高并发的关键优化之一。

---

## 二、隐式锁的物理载体

隐式锁的"存在"依赖于记录本身携带的事务 ID，而非 lock 系统中的锁对象。

### 聚簇索引（主键索引）

每条聚簇索引记录的系统列中存储了最后修改该行的 `trx_id`。

```cpp
// storage/innobase/include/row0row.ic: row_get_rec_trx_id
static inline trx_id_t row_get_rec_trx_id(const rec_t *rec,
                                          const dict_index_t *index,
                                          const ulint *offsets) {
  ulint offset;

  ut_ad(index->is_clustered());
  ut_ad(rec_offs_validate(rec, index, offsets));

  offset = index->trx_id_offset;

  if (!offset) {
    offset = row_get_trx_id_offset(index, offsets);
  }

  return (trx_read_trx_id(rec + offset));  // 读取记录中的 6 字节事务 ID
}
```

读取逻辑：找到 `DATA_TRX_ID` 系统列偏移，调用 `mach_read_from_6` 从记录物理位置读取 6 字节事务 ID。

### 二级索引

二级索引记录本身不存储 `trx_id`，但每个 Page 头部维护了 `PAGE_MAX_TRX_ID`，即**写入该 Page 的最大事务 ID**，作为快速排除手段。

---

## 三、隐式锁的检测

检测是否存在隐式锁，分聚簇索引和二级索引两条路径。

### 3.1 聚簇索引检测：`lock_clust_rec_some_has_impl`

```cpp
// storage/innobase/include/lock0priv.ic
static inline trx_id_t lock_clust_rec_some_has_impl(const rec_t *rec,
                                                    const dict_index_t *index,
                                                    const ulint *offsets) {
  ut_ad(index->is_clustered());
  ut_ad(page_rec_is_user_rec(rec));

  return (row_get_rec_trx_id(rec, index, offsets));
}
```

极其简单：**直接读取记录头中的 `trx_id`**，返回给上层。上层再调用 `trx_rw_is_active(trx_id, true)` 判断该事务是否仍处于活跃状态：

```cpp
// storage/innobase/lock/lock0lock.cc: lock_rec_convert_impl_to_expl (节选)
if (index->is_clustered()) {
    trx_id_t trx_id;

    trx_id = lock_clust_rec_some_has_impl(rec, index, offsets);

    trx = trx_rw_is_active(trx_id, true);  // 确认事务是否仍活跃
}
```

### 3.2 二级索引检测：`lock_sec_rec_some_has_impl`

二级索引的检测要复杂得多，分三层过滤：

```cpp
// storage/innobase/lock/lock0lock.cc
static trx_t *lock_sec_rec_some_has_impl(const rec_t *rec, dict_index_t *index,
                                         const ulint *offsets) {
  trx_t *trx;
  trx_id_t max_trx_id;
  const page_t *page = page_align(rec);
  ...
  max_trx_id = page_get_max_trx_id(page);

  if (!recv_recovery_is_on() && !can_older_trx_be_still_active(max_trx_id)) {
    // 第一层：Page 级别快速排除
    trx = nullptr;

  } else if (!lock_check_trx_id_sanity(max_trx_id, rec, index, offsets)) {
    // 第二层：Page 损坏检查
    trx = nullptr;

  } else {
    // 第三层：回溯 undo log 精确匹配
    trx = row_vers_impl_x_locked(rec, index, offsets);
  }

  return (trx);
}
```

**三层过滤逻辑：**

| 阶段 | 检查项 | 目的 |
|------|--------|------|
| 第一层 | `page_max_trx_id < 当前活跃最小 trx_id` | 快速排除：Page 上不可能有活跃隐式锁 |
| 第二层 | `lock_check_trx_id_sanity` | 检查 Page 是否损坏，防崩溃 |
| 第三层 | `row_vers_impl_x_locked` | 回溯 undo log 精确匹配 |

### 3.3 二级索引的 undo log 回溯：`row_vers_impl_x_locked_low`

二级索引无法直接读 `trx_id`，核心思路是通过 undo log 找到哪个事务版本"创造"了当前可见的二级索引记录。

```cpp
// storage/innobase/row/row0vers.cc: row_vers_impl_x_locked_low (节选)
trx_id = row_get_rec_trx_id(clust_rec, clust_index, clust_offsets);

trx_t *trx = trx_rw_is_active(trx_id, true);

if (trx == nullptr) {
    // 聚簇索引最新版本的修改事务已不活跃，则不可能有隐式锁
    lock_check_trx_id_sanity(trx_id, clust_rec, clust_index, clust_offsets);
    mem_heap_free(heap);
    return nullptr;
}

bool looking_for_match = rec_get_deleted_flag(sec_rec, comp);

if (!row_vers_find_matching(looking_for_match, clust_index, clust_rec,
                            clust_offsets, sec_index, sec_rec, sec_offsets,
                            comp, trx_id, mtr, heap)) {
    trx_release_reference(trx);
    trx = nullptr;
}
```

**算法核心推导（源码注释中有严谨的形式化论证）：**

代码中定义了如下概念链：

- `S matches C[t]`：二级索引记录 S 与聚簇索引记录第 t 版本字段值一致且未被删除标记
- `S was-authored-by C[t]`：版本 t 是最近一次让 S 状态发生对应变化的版本

算法步骤：

1. 读取聚簇索引当前版本的 `trx_id`，若该事务已不活跃，则**不可能有隐式锁**（比它更老的事务肯定也已提交）
2. 若 `trx_id` 活跃，则通过 undo log 向历史版本遍历，寻找"最后一次让二级索引记录状态发生变化的版本 t"
3. 找到则说明该活跃事务持有隐式 X 锁

> **注意**：该函数可能返回假阳性（false positive），但**绝不返回假阴性**，调用方负责二次确认。

---

## 四、隐式锁转换为显式锁

当另一个事务试图对某行加锁时，必须先将可能存在的隐式锁物化为显式锁，以便正确排队等待。这由 `lock_rec_convert_impl_to_expl` 完成：

```cpp
// storage/innobase/lock/lock0lock.cc
void lock_rec_convert_impl_to_expl(const buf_block_t *block, const rec_t *rec,
                                   dict_index_t *index, const ulint *offsets) {
  trx_t *trx;

  if (index->is_clustered()) {
    trx_id_t trx_id;
    trx_id = lock_clust_rec_some_has_impl(rec, index, offsets);
    trx = trx_rw_is_active(trx_id, true);
  } else {
    trx = lock_sec_rec_some_has_impl(rec, index, offsets);
  }

  if (trx != nullptr) {
    ulint heap_no = page_rec_get_heap_no(rec);
    lock_rec_convert_impl_to_expl_for_trx(block, rec, index, offsets, trx,
                                          heap_no);
  }
}
```

### 核心转换函数：`lock_rec_convert_impl_to_expl_for_trx`

```cpp
// storage/innobase/lock/lock0lock.cc
static void lock_rec_convert_impl_to_expl_for_trx(
    const buf_block_t *block, const rec_t *rec,
    dict_index_t *index, const ulint *offsets,
    trx_t *trx, ulint heap_no)
{
  ut_ad(trx_is_referenced(trx));
  {
    locksys::Shard_latch_guard guard{UT_LOCATION_HERE, block->get_page_id()};
    trx_mutex_enter(trx);

    if (!trx_state_eq(trx, TRX_STATE_COMMITTED_IN_MEMORY) &&
        !lock_rec_has_expl(LOCK_X | LOCK_REC_NOT_GAP, block, heap_no, trx)) {
      // 事务未提交且尚无显式锁，则为其创建一条显式记录锁
      ulint type_mode = (LOCK_REC | LOCK_X | LOCK_REC_NOT_GAP);
      lock_rec_add_to_queue(type_mode, block, heap_no, index, trx, true);
    }

    trx_mutex_exit(trx);
  }

  trx_release_reference(trx);  // 释放引用计数
}
```

**关键细节：**

- 持有 `Shard_latch`（保护 lock 系统中该页的分片）和 `trx->mutex` 双重保护
- 检查：若事务未提交且尚无显式锁，则为其创建一条 `LOCK_X | LOCK_REC_NOT_GAP` 记录锁加入队列
- 最后释放引用计数，允许事务正常提交与释放

---

## 五、触发隐式锁转换的场景

| 调用函数 | 场景 |
|----------|------|
| `lock_clust_rec_modify_check_and_lock` | 修改聚簇索引记录（UPDATE/DELETE）前 |
| `lock_sec_rec_read_check_and_lock` | 对二级索引记录加读锁前 |
| `lock_clust_rec_read_check_and_lock` | 对聚簇索引记录加读锁前 |

以修改聚簇索引记录为例：

```cpp
// storage/innobase/lock/lock0lock.cc: lock_clust_rec_modify_check_and_lock (节选)

// 第一步：将可能存在的隐式锁物化为显式锁
lock_rec_convert_impl_to_expl(block, rec, index, offsets);

// 第二步：尝试为当前事务加 X 锁，若与物化后的显式锁冲突则进入等待队列
{
    locksys::Shard_latch_guard guard{UT_LOCATION_HERE, block->get_page_id()};
    err = lock_rec_lock(true, SELECT_ORDINARY, LOCK_X | LOCK_REC_NOT_GAP,
                        block, heap_no, index, thr);
}
```

---

## 六、活跃事务检测：`trx_rw_is_active`

检测事务是否仍活跃的核心是 `Trx_shard` 分片结构，采用两级检查：

```cpp
// storage/innobase/include/trx0sys.ic
static inline trx_t *trx_rw_is_active(trx_id_t trx_id, bool do_ref_count) {
  auto &shard = trx_sys->get_shard_by_trx_id(trx_id);

  // 快速路径：原子读取 min_id，无锁判断
  if (trx_id < shard.active_rw_trxs.peek().min_id()) {
    return nullptr;
  }

  // 慢速路径：加 Trx_shard mutex 查询活跃事务哈希表
  return trx_sys->latch_and_execute_with_active_trx(
      trx_id,
      [&](trx_t *trx) {
        if (trx != nullptr && do_ref_count) {
          trx_reference(trx);  // 增加引用计数，防止事务提交时被释放
        }
        return trx;
      },
      UT_LOCATION_HERE);
}
```

**两级检查机制：**

| 路径 | 方式 | 说明 |
|------|------|------|
| 快速路径 | 原子读 `min_id`，无锁 | 利用内存序保证正确性，绝大多数情况走此路径 |
| 慢速路径 | 加 `Trx_shard` mutex 查哈希表 | 精确查找，并增加引用计数确保线程安全 |

---

## 七、整体流程总结

```
事务 T1 执行 INSERT / UPDATE 记录 R
  └── 不创建任何锁对象
      ├── [聚簇索引] 将 T1.id 写入 R 的记录头系统列
      └── [二级索引] 更新 Page 头的 PAGE_MAX_TRX_ID

事务 T2 尝试对记录 R 加锁
  └── lock_rec_convert_impl_to_expl(R)
        ├── [聚簇索引路径]
        │     读 R.trx_id → trx_rw_is_active(T1.id)
        │     └── T1 活跃 → 为 T1 创建显式 LOCK_X | REC_NOT_GAP
        │
        └── [二级索引路径]
              page_max_trx_id 预检（快速排除）
              └── row_vers_impl_x_locked → undo log 回溯确认
                    └── 确认 T1 持有隐式锁 → 为 T1 创建显式 LOCK_X | REC_NOT_GAP

  └── lock_rec_lock(T2 的加锁请求)
        └── 发现 T1 的显式锁 → T2 进入等待队列
```

---

## 八、设计亮点与权衡

| 特性 | 说明 |
|------|------|
| **零开销写路径** | 写事务不需要创建锁对象，INSERT/UPDATE 性能极高 |
| **懒转换** | 只有真正发生竞争时才物化显式锁，节省内存 |
| **聚簇索引 O(1) 检测** | 直接读记录中的 `trx_id`，极其高效 |
| **二级索引代价较高** | 需先过滤 page max trx id，再回溯 undo log，复杂度更高 |
| **引用计数保护** | `trx_reference / trx_release_reference` 确保检测期间事务对象不被销毁 |
| **允许假阳性** | 二级索引检测允许假阳性，由调用方二次确认，确保安全的同时避免漏判 |

---

## 九、核心源码文件索引

| 文件 | 关键函数 | 作用 |
|------|----------|------|
| `storage/innobase/include/lock0priv.ic` | `lock_clust_rec_some_has_impl` | 聚簇索引隐式锁检测（读 trx_id） |
| `storage/innobase/lock/lock0lock.cc` | `lock_sec_rec_some_has_impl` | 二级索引隐式锁检测（三层过滤） |
| `storage/innobase/lock/lock0lock.cc` | `lock_rec_convert_impl_to_expl` | 隐式锁转换为显式锁入口 |
| `storage/innobase/lock/lock0lock.cc` | `lock_rec_convert_impl_to_expl_for_trx` | 为指定事务创建显式锁对象 |
| `storage/innobase/row/row0vers.cc` | `row_vers_impl_x_locked_low` | 二级索引 undo log 回溯精确匹配 |
| `storage/innobase/include/trx0sys.ic` | `trx_rw_is_active` | 活跃事务检测（两级无锁/加锁路径） |
| `storage/innobase/include/row0row.ic` | `row_get_rec_trx_id` | 从记录物理字节读取事务 ID |
