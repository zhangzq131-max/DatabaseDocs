# InnoDB 加锁机制：SELECT FOR UPDATE 与 Semi-Consistent Read

> 基于 MySQL 8.0.41 源码分析

---

## 第一部分：SELECT FOR UPDATE 的加锁实现

### 1. `impl` 参数是核心分水岭

`SELECT FOR UPDATE` 与 DML（UPDATE/DELETE）的加锁差异，集中体现在 `lock_rec_lock` 的 `impl` 参数上：

```cpp
// storage/innobase/lock/lock0lock.cc
static dberr_t lock_rec_lock(bool impl, select_mode sel_mode, ulint mode,
                             const buf_block_t *block, ulint heap_no,
                             dict_index_t *index, que_thr_t *thr) {
  ...
  /* Implicit locks are equivalent to LOCK_X|LOCK_REC_NOT_GAP, so we can omit
  creation of explicit lock only if the requested mode was LOCK_REC_NOT_GAP */
  ut_ad(!impl || ((mode & LOCK_REC_NOT_GAP) == LOCK_REC_NOT_GAP));
  ...
  switch (lock_rec_lock_fast(impl, mode, block, heap_no, index, thr)) {
    case LOCK_REC_SUCCESS:
      return (DB_SUCCESS);
    case LOCK_REC_SUCCESS_CREATED:
      return (DB_SUCCESS_LOCKED_REC);
    case LOCK_REC_FAIL:
      return (lock_rec_lock_slow(impl, sel_mode, mode, block, heap_no, index, thr));
  }
}
```

两条路径的调用方式：

| 来源 | impl 值 | 含义 |
|------|---------|------|
| DML（UPDATE/DELETE）| `true` | 无竞争时不创建显式锁，靠记录头 trx_id 做隐式锁 |
| `SELECT FOR UPDATE` | `false` | **无论有无竞争，都创建显式锁** |

`lock_rec_lock_fast` 中的关键判断：

```cpp
// storage/innobase/lock/lock0lock.cc: lock_rec_lock_fast (节选)
if (lock == nullptr) {
    if (!impl) {
      RecLock rec_lock(index, block, heap_no, mode);
      trx_mutex_enter(trx);
      rec_lock.create(trx);       // impl=false 时才创建锁对象
      trx_mutex_exit(trx);
      status = LOCK_REC_SUCCESS_CREATED;
    }
    // impl=true 时：什么都不做，走隐式锁
} else {
    ...
    if (other_lock != nullptr || lock->trx != trx || ...) {
      status = LOCK_REC_FAIL;     // 有冲突，走慢路径
    } else if (!impl) {
      if (!lock_rec_get_nth_bit(lock, heap_no)) {
        lock_rec_set_nth_bit(lock, heap_no);   // 复用已有锁对象，设置位图
        status = LOCK_REC_SUCCESS_CREATED;
      }
    }
}
```

慢路径同样：

```cpp
// storage/innobase/lock/lock0lock.cc: lock_rec_lock_slow (节选)
if (!impl || conflicting.bypassed) {
    /* Set the requested lock on the record. */
    lock_rec_add_to_queue(LOCK_REC | mode, block, heap_no, index, trx);
    return (DB_SUCCESS_LOCKED_REC);
}
return (DB_SUCCESS);  // impl=true 且无冲突 -> 直接返回，不建锁
```

---

### 2. SELECT FOR UPDATE 的完整加锁流程

```cpp
// storage/innobase/lock/lock0lock.cc: lock_clust_rec_read_check_and_lock
dberr_t lock_clust_rec_read_check_and_lock(...) {
  ...
  heap_no = page_rec_get_heap_no(rec);

  if (heap_no != PAGE_HEAP_NO_SUPREMUM) {
    // 第一步：将其他事务对该记录的隐式锁物化为显式锁
    lock_rec_convert_impl_to_expl(block, rec, index, offsets);
  }

  {
    locksys::Shard_latch_guard guard{...};
    // 第二步：impl=false，为自己创建显式锁（或进入等待队列）
    err = lock_rec_lock(false, sel_mode, mode | gap_mode, block, heap_no, index, thr);
  }
}
```

**`SELECT FOR UPDATE` 对每一条扫描到的记录都会创建显式锁对象。**

---

### 3. 默认锁类型：Next Key Lock

`SELECT FOR UPDATE` 在 REPEATABLE READ 下默认加的是 `LOCK_X | LOCK_ORDINARY`（Next Key Lock = 记录锁 + Gap 锁）：

```cpp
// storage/innobase/row/row0sel.cc: row_search_mvcc (节选)
if (set_also_gap_locks && !trx->skip_gap_locks() &&
    prebuilt->select_lock_type != LOCK_NONE &&
    !dict_index_is_spatial(index)) {
  err = sel_set_rec_lock(pcur, rec, index, offsets, prebuilt->select_mode,
                         prebuilt->select_lock_type, LOCK_ORDINARY, thr, &mtr);
```

只有唯一索引精确匹配时才退化为 `LOCK_REC_NOT_GAP`：

```cpp
// storage/innobase/row/row0sel.cc: Row_sel_get_clust_rec_for_mysql (节选)
/* we are searching the clust rec with a unique condition, hence
we set a LOCK_REC_NOT_GAP type lock */
err = lock_clust_rec_read_check_and_lock(
    ..., static_cast<lock_mode>(prebuilt->select_lock_type), LOCK_REC_NOT_GAP, thr);
```

---

### 4. 与 DML 的完整对比

```
SELECT FOR UPDATE 扫描记录 R：
  1. lock_rec_convert_impl_to_expl(R)    — 先物化其他事务的隐式锁
  2. lock_rec_lock(impl=false, LOCK_X | LOCK_ORDINARY)
       └── 无论有无竞争，都为自己创建显式 Next Key Lock 对象
       └── 有冲突 → 进入等待队列

DML 修改记录 R（UPDATE/DELETE）：
  1. lock_rec_convert_impl_to_expl(R)    — 先物化其他事务的隐式锁
  2. lock_rec_lock(impl=true, LOCK_X | LOCK_REC_NOT_GAP)
       └── 无竞争 → 不创建锁对象（写操作本身的 trx_id 充当隐式锁）
       └── 有冲突 → 进入等待队列，此时才创建显式锁
```

---

## 第二部分：Semi-Consistent Read

### 1. 一句话定义

Semi-Consistent Read（半一致读）是 InnoDB 在 **READ COMMITTED / READ UNCOMMITTED** 隔离级别下，专门为 **UPDATE/DELETE 语句**提供的一种优化：

> 当扫描到一条被其他事务锁住的记录时，不阻塞等待，而是**先读取该记录的最新已提交版本**来判断 WHERE 条件。如果不满足条件就直接跳过，**完全避免了不必要的锁等待**；只有满足条件、真正需要修改时，才回来加锁等待。

---

### 2. 生效条件

源码中直接把条件写死在 `allow_semi_consistent()` 里：

```cpp
// storage/innobase/include/trx0trx.h
bool skip_gap_locks() const {
    switch (isolation_level) {
      case READ_UNCOMMITTED:
      case READ_COMMITTED:
        return (true);        // 只有这两种级别才允许
      case REPEATABLE_READ:
      case SERIALIZABLE:
        return (false);
    }
}

bool allow_semi_consistent() const { return (skip_gap_locks()); }
bool releases_non_matching_rows() const { return skip_gap_locks(); }
```

**只在 RC / RU 隔离级别下生效**，RR 和 SERIALIZABLE 下完全不走这条路径。

---

### 3. 整体交互流程（Server 层 + InnoDB 层）

`handler.h` 中有官方给出的伪代码：

```cpp
// sql/handler.h
file->try_semi_consistent_read(true);     // 开启半一致读模式
file->ha_rnd_init(true);
while (file->ha_rnd_next(table->record[0]) == 0) {
  if (row is filtered...) {
    file->unlock_row();                   // 不满足 WHERE -> 释放锁
    continue;
  }
  if (file->was_semi_consistent_read()) {
    // 这行是乐观读来的（未加锁），需要重新加锁读
    // 下一次 ha_rnd_next() 会自动对同一行加锁重读
    continue;
  }
  // 正常处理这行（加锁读成功）
}
file->ha_rnd_end();
file->try_semi_consistent_read(false);    // 关闭半一致读模式
```

---

### 4. InnoDB 内部执行逻辑

#### 4.1 触发入口：遇到锁冲突时不等待

```cpp
// storage/innobase/row/row0sel.cc: row_search_mvcc (节选)

/* in case of semi-consistent read, we use SELECT_SKIP_LOCKED,
   so we don't waste time on creating a WAITING lock */
const bool use_semi_consistent =
    prebuilt->row_read_type == ROW_READ_TRY_SEMI_CONSISTENT &&
    !unique_search && index == clust_index && !trx_is_high_priority(trx);

err = sel_set_rec_lock(
    pcur, rec, index, offsets,
    use_semi_consistent ? SELECT_SKIP_LOCKED : prebuilt->select_mode,
    prebuilt->select_lock_type, lock_type, thr, &mtr);

switch (err) {
  case DB_SKIP_LOCKED:          // 加锁失败（记录被锁）且是半一致读模式
    row_sel_build_committed_vers_for_mysql(
        clust_index, prebuilt, rec, &offsets, &heap, &old_vers, ...);

    if (old_vers == nullptr) {
      goto next_rec;   // 该记录还没有已提交版本（刚插入）-> 跳过
    }

    did_semi_consistent_read = true;
    rec = old_vers;    // 用最新已提交版本替换当前记录，继续判断 WHERE
```

#### 4.2 构建最新已提交版本：`row_vers_build_for_semi_consistent_read`

```cpp
// storage/innobase/row/row0vers.cc
void row_vers_build_for_semi_consistent_read(...) {
  version = rec;
  for (;;) {
    version_trx_id = row_get_rec_trx_id(version, index, *offsets);

    if (!trx_rw_is_active(version_trx_id, false)) {
    committed_version_trx:
      /* 找到了一个属于已提交事务的版本，直接返回它 */
      *old_vers = rec_copy(buf, version, *offsets);
      break;
    }

    // 该版本的事务还活跃，继续往 undo log 里找更老的版本
    if (!trx_undo_prev_version_build(..., &prev_version, ...)) {
      goto committed_version_trx;   // undo log 已无更老版本，用当前版本
    }

    if (prev_version == nullptr) {
      *old_vers = nullptr;  // 该记录是被活跃事务新插入的，还没有已提交版本
      break;
    }

    version = prev_version;  // 继续向前追溯
  }
}
```

本质就是**沿 undo log 链向前遍历，直到找到 `trx_id` 对应的事务已提交的那个版本**。

#### 4.3 三种结果的后续处理

| 情况 | 处理方式 |
|------|----------|
| 加锁成功（`DB_SUCCESS_LOCKED_REC`） | 正常返回行数据，标记 `new_rec_lock=true` |
| 被锁 + 最新提交版本不满足 WHERE | `goto next_rec`，跳过该行，**零锁等待** |
| 被锁 + 最新提交版本满足 WHERE | 返回 `old_vers` 给上层，标记 `did_semi_consistent_read=true` |

#### 4.4 满足 WHERE 后的"二次确认"加锁

当 `was_semi_consistent_read()` 返回 `true` 时，Server 层会对**同一行**发起第二次读取，这次会真正加锁等待：

```cpp
// storage/innobase/handler/ha_innodb.cc
bool ha_innobase::was_semi_consistent_read(void) {
  return (m_prebuilt->row_read_type == ROW_READ_DID_SEMI_CONSISTENT);
}

void ha_innobase::try_semi_consistent_read(bool yes) {
  if (yes && m_prebuilt->trx->allow_semi_consistent()) {
    m_prebuilt->row_read_type = ROW_READ_TRY_SEMI_CONSISTENT;
  } else {
    m_prebuilt->row_read_type = ROW_READ_WITH_LOCKS;
  }
}
```

---

### 5. `row_read_type` 状态机

```
                  ┌─────────────────────────────────────────┐
                  │                                         │
          try_semi_consistent_read(true)               加锁重读
                  │                                         │
                  ▼                                         │
       ROW_READ_TRY_SEMI_CONSISTENT ──────────────────────►─┘
                  │
          遇到被锁记录
          读最新提交版本
          版本满足 WHERE
                  │
                  ▼
       ROW_READ_DID_SEMI_CONSISTENT
         (was_semi_consistent_read() = true)
                  │
         Server 层检测到，
         重置状态，再次读同一行
                  │
                  ▼
       ROW_READ_TRY_SEMI_CONSISTENT（等待加锁）
```

---

### 6. 与普通 UPDATE 加锁的对比

```
普通 UPDATE（RR 隔离级别）扫描记录 R，R 被 T1 锁住：
  lock_rec_lock(LOCK_X) → 冲突 → 进入等待队列 → 阻塞 → T1 提交后唤醒

Semi-Consistent Read（RC 隔离级别）扫描记录 R，R 被 T1 锁住：
  lock_rec_lock(SELECT_SKIP_LOCKED) → 跳过，不等待
    └─ 读 R 的最新已提交版本 R'
          ├─ R' 不满足 WHERE → 直接跳过（0 锁等待，0 锁对象）
          └─ R' 满足 WHERE   → 返回 R' 给 Server 层
                                Server 发现 was_semi_consistent_read()
                                对 R 发起第二次读 → 正常加锁等待 T1
```

---

### 7. 设计意义

Semi-Consistent Read 本质上是**将"是否需要等锁"这个决策延迟到 WHERE 条件判断之后**。

在 UPDATE 大表时（全表扫描），大量记录可能被其他事务锁住，但其中大多数根本不满足 WHERE 条件。没有这个优化，这些行都会造成不必要的锁等待；有了它，只有真正需要修改的行才会等锁，大幅降低并发争用。

代价是需要**读两次满足条件的行**（一次乐观读，一次加锁确认读），但这通常远比等待锁要快。

---

## 附：核心源码文件索引

| 文件 | 关键函数 | 作用 |
|------|----------|------|
| `storage/innobase/lock/lock0lock.cc` | `lock_rec_lock` | 加锁总入口，`impl` 参数决定是否创建显式锁 |
| `storage/innobase/lock/lock0lock.cc` | `lock_rec_lock_fast` | 快速加锁路径（无竞争场景） |
| `storage/innobase/lock/lock0lock.cc` | `lock_rec_lock_slow` | 慢速加锁路径（处理竞争/等待） |
| `storage/innobase/lock/lock0lock.cc` | `lock_clust_rec_read_check_and_lock` | `SELECT FOR UPDATE` 聚簇索引加锁入口 |
| `storage/innobase/lock/lock0lock.cc` | `lock_sec_rec_read_check_and_lock` | `SELECT FOR UPDATE` 二级索引加锁入口 |
| `storage/innobase/row/row0sel.cc` | `row_search_mvcc` | 扫描行主循环，含半一致读逻辑 |
| `storage/innobase/row/row0sel.cc` | `row_sel_build_committed_vers_for_mysql` | 构建最新已提交版本（半一致读） |
| `storage/innobase/row/row0vers.cc` | `row_vers_build_for_semi_consistent_read` | 沿 undo log 回溯已提交版本 |
| `storage/innobase/handler/ha_innodb.cc` | `was_semi_consistent_read` | Server 层检测是否走了半一致读 |
| `storage/innobase/handler/ha_innodb.cc` | `try_semi_consistent_read` | 开启/关闭半一致读模式 |
| `storage/innobase/include/trx0trx.h` | `allow_semi_consistent` | 判断当前隔离级别是否允许半一致读 |
