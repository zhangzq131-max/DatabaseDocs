# MySQL 8.0.41 InnoDB 事务模块性能优化深度分析

## —— 相较于 8.0.22 的关键变化（基于源码）

> **分析基准**: MySQL 8.0.41 源码 (`mysql-8.0.41` 分支)
> **对比基线**: MySQL 8.0.22
> **分析范围**: `storage/innobase/trx/`, `storage/innobase/read/`, `storage/innobase/lock/`, `storage/innobase/include/trx0*.h`, `storage/innobase/include/read0*.h`
> **分析日期**: 2026-04-02

---

## 一、整体架构变化概览

MySQL 8.0.22 到 8.0.41 期间，InnoDB 事务模块经历了多项重要的性能优化，核心方向包括：

1. **活跃事务管理分片化**（Sharded Active RW Transaction Set）
2. **trx_sys_mutex 竞争缓解**（消除冗余 mutex 获取路径）
3. **锁系统并发度提升**（Lock-Sys Latch 粒度细化）
4. **MVCC 可见性判断优化**（消除冗余检查、快速路径）
5. **事务序列化与 purge 协调优化**
6. **锁调度公平性改进**（避免 S→X 升级导致的死锁）

以下逐项基于代码进行详细分析。

---

## 二、活跃事务管理分片化（Sharded `active_rw_trxs`）

### 2.1 变更背景

**关键 Commit**: `bf8fffd3dd0` — *BUG#32832196 SINGLE RW_TRX_SET LEADS TO CONTENTION ON TRX_SYS MUTEX*

在 8.0.22 中，活跃读写事务的查找通过 `trx_sys->rw_trx_set`（一个全局的 `TrxTrack` 集合）完成，所有对该集合的访问都需要获取 `trx_sys->mutex`。在高并发场景下，这个全局互斥锁成为严重的热点瓶颈。

### 2.2 8.0.41 中的实现

在 8.0.41 中，活跃读写事务被分散到 **256 个分片（shard）** 中管理：

```cpp
// storage/innobase/include/trx0sys.h

/** Number of shards created for transactions. */
constexpr size_t TRX_SHARDS_N = 256;

inline size_t trx_get_shard_no(trx_id_t trx_id) {
  ut_ad(trx_id != 0);
  return trx_id % TRX_SHARDS_N;
}
```

每个分片由独立的 `Trx_shard` 结构持有，内部通过 `ut::Guarded` 模板实现 **per-shard mutex** 保护：

```cpp
// storage/innobase/include/trx0sys.h

struct Trx_shard {
  ut::Cacheline_padded<ut::Guarded<Trx_by_id_with_min, LATCH_ID_TRX_SYS_SHARD>>
      active_rw_trxs;
};
```

`ut::Guarded` 模板（定义于 `ut0guarded.h`）将数据和 mutex 封装在一起，通过 `latch_and_execute()` 方法提供线程安全的访问接口：

```cpp
// storage/innobase/include/ut0guarded.h

template <typename Inner, latch_id_t latch_id>
class Guarded {
  Cacheline_padded<IB_mutex> mutex{latch_id};
  Inner inner;
public:
  template <typename F>
  auto latch_and_execute(F &&f, const ut::Location &loc) {
    IB_mutex_guard guard{&mutex, loc};
    return std::forward<F>(f)(inner);
  }
  const Inner &peek() const { return inner; }
};
```

### 2.3 `Trx_by_id_with_min` 数据结构

每个 shard 内部使用 `Trx_by_id_with_min` 管理该分片中的活跃事务：

```cpp
// storage/innobase/include/trx0sys.h

class Trx_by_id_with_min {
  struct Trx_track_hash {
    size_t operator()(const trx_id_t &key) const {
      return static_cast<size_t>(key / TRX_SHARDS_N);
    }
  };

  using By_id = std::unordered_map<trx_id_t, trx_t *, Trx_track_hash>;
  By_id m_by_id;
  std::atomic<trx_id_t> m_min_id{0};
  // ...
};
```

关键设计要点：

- **`m_min_id`** 使用 `std::atomic` 存储，允许**无锁快速路径**判断：如果 `trx_id < shard.min_id()`，可以直接确定该事务不在活跃列表中，无需获取 shard mutex。
- **hash 函数**使用 `key / TRX_SHARDS_N`，因为同一 shard 内的 trx_id 一定满足 `trx_id % TRX_SHARDS_N == shard_no`，除以 `TRX_SHARDS_N` 后得到的值可以直接用作 hash。

### 2.4 `trx_rw_is_active()` 的快速路径

这是 MVCC 和隐式锁检测的热点函数。在 8.0.41 中：

```cpp
// storage/innobase/include/trx0sys.ic

static inline trx_t *trx_rw_is_active(trx_id_t trx_id, bool do_ref_count) {
  // 快速路径：无锁检查 m_min_id
  auto &shard = trx_sys->get_shard_by_trx_id(trx_id);
  if (trx_id < shard.active_rw_trxs.peek().min_id()) {
    return nullptr;
  }
  // 慢速路径：获取 shard mutex 进行精确查找
  return trx_sys->latch_and_execute_with_active_trx(
      trx_id, [&](trx_t *trx) { /* ... */ }, UT_LOCATION_HERE);
}
```

**性能影响**：在 8.0.22 中，`trx_rw_is_active()` 需要获取全局 `trx_sys->mutex`，这在行级别操作中频繁调用会产生严重的互斥竞争。8.0.41 的分片化设计将竞争分散到 256 个独立的 mutex，且大部分查询可以通过 `m_min_id` 的无锁快速路径直接返回。

### 2.5 `trx_sys_rw_trx_add()` 的分片化

```cpp
// storage/innobase/include/trx0sys.ic

static inline void trx_sys_rw_trx_add(trx_t *trx) {
  const trx_id_t trx_id = trx->id;
  const auto trx_shard_no = trx_get_shard_no(trx_id);
  trx_sys->shards[trx_shard_no].active_rw_trxs.latch_and_execute(
      [&](Trx_by_id_with_min &trx_by_id_with_min) {
        trx_by_id_with_min.insert(*trx);
      }, UT_LOCATION_HERE);
}
```

事务的注册和注销操作只需要获取对应 shard 的 mutex，而不是全局 `trx_sys->mutex`。

### 2.6 对内核开发的意义

如果要基于此进行进一步的事务管理优化：

- Shard 数量 `TRX_SHARDS_N = 256` 是编译时常量，可以根据 CPU 核数进行调优。
- `m_min_id` 的 `memory_order_acquire/release` 语义保证了快速路径的正确性，修改此处需要特别注意内存序。
- 事务从 `active_rw_trxs` 中删除和状态变更为 `TRX_STATE_COMMITTED_IN_MEMORY` 在**同一个临界区**内完成，这是保证 MVCC 一致性的关键不变量。

---

## 三、trx_sys_mutex 竞争的进一步消除

### 3.1 消除 `trx_rw_min_trx_id()` 中的冗余 mutex 获取

**关键 Commit**: `e6df317f8dd` / `0a0c0db97e9` — *Bug #33059387 TRX_RW_MIN_TRX_ID() CHECK IN LOCK_SEC_REC_READ_CHECK_AND_LOCK() IS REDUNDANT*

在 8.0.22 中，`lock_sec_rec_read_check_and_lock()` 在辅助索引加锁前会调用 `trx_rw_min_trx_id()` 作为启发式检查。这个函数需要获取 `trx_sys->mutex` 来计算全局最小活跃事务 ID。

8.0.41 的优化：

1. **移除了冗余检查**：`lock_rec_convert_impl_to_expl()` 内部已经会执行相同的检查，外层调用完全冗余。
2. **重构为 `can_older_trx_be_still_active()`**：不再需要精确的最小值，只需判断"是否存在某个更老的活跃事务"。
3. **利用 shard 的无锁 peek**：遍历 256 个 shard 的 `m_min_id`（无锁 `load`），而非获取全局 mutex。在 mutex 无竞争时使用原有路径，有竞争时退化为 256 次原子 `load`（在 x86 上等同于普通 `mov` 指令）。
4. **删除了 `trx_sys->rw_max_trx_id`**：这个原子变量的唯一用途是在 `trx_rw_min_trx_id()` 中处理空集合的情况，重构后不再需要。

### 3.2 trx_id / trx_no 分配的 mutex 拆分

在 8.0.41 中，`next_trx_id_or_no` 字段使用 `std::atomic<trx_id_t>` 存储：

```cpp
// storage/innobase/include/trx0sys.h

/** The smallest number not yet assigned as a transaction id
or transaction number. This is declared as atomic because it
can be accessed without holding any mutex during AC-NL-RO
view creation. */
std::atomic<trx_id_t> next_trx_id_or_no;
```

trx_id 的分配受 `trx_sys_t::mutex` 保护，trx_no 的分配受 `trx_sys_t::serialisation_mutex` 保护，两者可以并行操作：

```cpp
// storage/innobase/include/trx0sys.ic

inline trx_id_t trx_sys_allocate_trx_id() {
  ut_ad(trx_sys_mutex_own());
  return trx_sys_allocate_trx_id_or_no();
}

inline trx_id_t trx_sys_allocate_trx_no() {
  ut_ad(trx_sys_serialisation_mutex_own());
  return trx_sys_allocate_trx_id_or_no();
}
```

`serialisation_mutex` 和 `mutex` 的分离使得事务 ID 分配和序列化号分配可以在不同的临界区中并行进行。

### 3.3 purge 线程竞争消除

**关键 Commit**: `98c11a5dc92` — *Bug#35289390 purge_sys->thds.find(thd) is causing congestion*

8.0.22 中 `innobase_trx_allocate()` 需要获取 `purge_sys->latch` 的 S-锁来判断当前线程是否是 purge 线程。当 purge coordinator 持有 X-锁（例如在复制最老的 ReadView 时需要获取 `trx_sys->mutex`）时，所有新事务的分配都会被阻塞。

8.0.41 使用 `thread_local bool` 替代了 `unordered_set` + latch 的方案，完全消除了这个竞争点。

---

## 四、ReadView 与 MVCC 优化

### 4.1 ReadView 的结构（8.0.41）

ReadView 的核心数据结构在 8.0.22 到 8.0.41 之间保持了结构稳定性，关键字段：

```cpp
// storage/innobase/include/read0types.h

class ReadView {
  trx_id_t m_low_limit_id;   // 高水位：>= 此值的事务不可见
  trx_id_t m_up_limit_id;    // 低水位：< 此值的事务一定可见
  trx_id_t m_creator_trx_id; // 创建者事务 ID
  ids_t m_ids;                // 活跃 RW 事务 ID 快照（有序）
  trx_id_t m_low_limit_no;   // purge 边界
  bool m_closed;              // AC-NL-RO 关闭标记
};
```

### 4.2 `changes_visible()` 可见性判断的优化

```cpp
// storage/innobase/include/read0types.h

bool changes_visible(trx_id_t id, const table_name_t &name) const {
  if (id < m_up_limit_id || id == m_creator_trx_id) {
    return true;    // 快速路径1：低水位以下或自身事务
  }
  check_trx_id_sanity(id, name);
  if (id >= m_low_limit_id) {
    return false;   // 快速路径2：高水位以上
  } else if (m_ids.empty()) {
    return true;    // 快速路径3：无活跃事务
  }
  // 慢速路径：二分搜索
  const ids_t::value_type *p = m_ids.data();
  return !std::binary_search(p, p + m_ids.size(), id);
}
```

此函数在 8.0.22 与 8.0.41 之间的核心逻辑未变，但其依赖的 **`m_ids` 快照的生成效率**因分片化而得到间接优化。

### 4.3 ReadView 创建的优化（AC-NL-RO 快速路径）

```cpp
// storage/innobase/read/read0read.cc

void MVCC::view_open(ReadView *&view, trx_t *trx) {
  if (view != nullptr) {
    // AC-NL-RO 优化：如果没有新的 RW 事务启动，复用已有 view
    if (trx_is_autocommit_non_locking(trx) && view->empty()) {
      view->m_closed = false;
      if (view->m_low_limit_id == trx_sys_get_next_trx_id_or_no()) {
        return;  // 无需获取 trx_sys->mutex，直接返回
      } else {
        view->m_closed = true;
      }
    }
  }
  // 慢速路径：获取 trx_sys->mutex 创建新的 view
  trx_sys_mutex_enter();
  // ...
}
```

`trx_sys_get_next_trx_id_or_no()` 直接读取原子变量，不需要任何 mutex：

```cpp
inline trx_id_t trx_sys_get_next_trx_id_or_no() {
  return trx_sys->next_trx_id_or_no.load();
}
```

**这意味着在纯读场景下（只有 AC-NL-RO 事务），ReadView 的创建可以完全绕过 `trx_sys->mutex`。**

### 4.4 ReadView::prepare() 中的验证优化

在 8.0.41 的 debug 模式下，`copy_trx_ids()` 中增加了概率性验证，通过 shard 机制验证 `rw_trx_ids` 中的事务确实都是活跃的：

```cpp
// storage/innobase/read/read0read.cc

#ifdef UNIV_DEBUG
  if (ut::random_from_interval_fast(0, 99) == 0) {
    for (auto trx_id : trx_ids) {
      while (trx_sys->latch_and_execute_with_active_trx(
          trx_id, [](trx_t *trx) { /* ... */ }, UT_LOCATION_HERE)) {
        std::this_thread::sleep_for(std::chrono::microseconds(10));
      }
    }
  }
#endif
```

这个验证使用的是 per-shard 的 mutex 而非全局 mutex，体现了分片化对调试能力的改善。

### 4.5 serialisation_min_trx_no 的原子化

```cpp
// storage/innobase/include/trx0sys.h

/* The minimum trx->no inside the serialisation_list. Protected by
the serialisation_mutex. Might be read without the mutex. */
std::atomic<trx_id_t> serialisation_min_trx_no;
```

ReadView 创建时通过 `trx_get_serialisation_min_trx_no()` 读取这个原子变量来设置 `m_low_limit_no`，无需获取 `serialisation_mutex`。

---

## 五、锁系统（Lock-Sys）性能优化

### 5.1 Lock-Sys 全局 Latch 持有时间限制

**关键 Commit**: `5a6fd9936d8` — *Bug #33502610 Global Lock-Sys latch can be held too long and cause long semaphore wait*

8.0.41 中增加了全局 lock-sys latch 持有时间的限制：线程在持有全局 latch 超过一定时间后会主动释放并重新获取，避免"long semaphore wait"。

### 5.2 锁优先级优化：避免 S→X 升级死锁

**关键 Commit**: `7037a0bdc83` — *Bug #11745929 CHANGE LOCK PRIORITY SO THAT THE TRANSACTION HOLDING S-LOCK GETS X-LOCK FIRST*

这是一个存在 15 年的 bug。核心场景：

```
1. trx1 获得 S-lock（GRANTED）
2. trx2 请求 X-lock，需要等待 trx1
3. trx1 请求升级为 X-lock，按原规则需等待 trx2 的 WAITING 请求
→ 死锁！
```

8.0.41 的解决方案：当 trx1 的锁请求与 trx2 的 WAITING 请求冲突，且 trx2 已经被 trx1 阻塞时，trx1 可以**忽略 trx2 的 WAITING 请求**直接获取锁。这在代码中通过检查已有的 GRANTED 锁关系实现：

- 适用于 `X` 或 `X|REC_NOT_GAP` 请求与同类型 WAITING 请求之间
- 不适用于 `INSERT_INTENTION` 锁（防止拆分 GAP 锁的不变量被违反）

附带的性能优化：
- **`lock_rec_find_set_bit()`**：从逐 bit 循环改为跳跃式搜索（64→32→16→8 bits），提升约 13 倍
- **`lock_rec_has_expl()`**：利用队列中 GRANTED 锁在 WAITING 锁之前的有序性，提前终止搜索

### 5.3 HP 事务超时机制

**关键 Commit**: `5ccf484a4fc` — *Bug #33856332 InnoDB should let HP transaction timeout while waiting on a lock*

8.0.22 中，高优先级（HP）事务在等待锁时不会超时，导致在未检测到的死锁场景下系统可能永久卡死。

8.0.41 允许 HP 事务在以下条件下超时：

- 阻塞者也是 HP 事务
- 当前事务被 KILL QUERY 中断

实现上重构了超时判断逻辑，将 `error_state` 的设置职责从事务线程本身转移到 `lock_wait_timeout_thread`。

### 5.4 `performance_schema.data_lock_waits` 的 O(N³) → O(N) 优化

**关键 Commit**: `96ccd8dc9f9` — *Bug#36302624 performance_schema.data_lock_waits is generated with O(N^3) complexity*

这是一个大规模重构（+2194/-2292 行）：

- **旧方式**：按事务遍历锁，报告每个事务的所有锁
- **新方式**：按表和 hash bucket 遍历，每次只 latch 一个 shard

关键数据结构变化：
- 引入 `Lock_hashtable` 结构，将 `hash_table_t` 与 `ut::Sharded_bitset` 捆绑
- `Sharded_bitset` 支持快速跳过空 bucket
- `locksys::find_blockers(wait_lock, visitor)` 使用 `Lock_hashtable::find_on_record` 在 O(N) 时间内找到所有阻塞者

### 5.5 `lock_rec_convert_impl_to_expl()` 路径优化

与 §3.1 联动：移除了 `lock_sec_rec_read_check_and_lock()` 中的冗余 `trx_rw_min_trx_id()` 调用，避免了在每次辅助索引记录加锁时获取全局 mutex 的开销。

---

## 六、事务结构体与生命周期管理优化

### 6.1 `trx_t::state` 和 `trx_t::start_time` 的原子化

**关键 Commit**: `bf8fffd3dd0`

```cpp
// storage/innobase/include/trx0trx.h

std::atomic<trx_state_t> state;
std::atomic<std::chrono::system_clock::time_point> start_time{
    std::chrono::system_clock::time_point{}};
static_assert(decltype(start_time)::is_always_lock_free);
```

在 8.0.22 中 `trx_t::state` 不是原子类型。8.0.41 将其改为 `std::atomic`，消除了在某些平台上的 torn-read 风险，同时允许在不持有 mutex 的情况下安全读取状态（例如 AC-NL-RO 事务的状态转换）。

### 6.2 时钟函数性能回退修复

**关键 Commit**: `0568caadd89` — *Bug #34352870 [InnoDB] Bug #33460092 introduced 5% performance regression*

在 8.0.30 左右，`time(nullptr)` 被替换为 `std::chrono::steady_clock::now()`，导致了 5% 的性能回退。原因是 `steady_clock::now()` 在某些旧版 Linux/devtoolset 上会触发系统调用，比 `time(nullptr)` 慢 1000 倍。

8.0.41 对关键路径回退到使用 `time(nullptr)`（低精度但无系统调用）：
- `trx->start_time`
- `trx->lock.wait_started`
- `thd_start_time`

### 6.3 事务提交路径中的 shard 化释放

```cpp
// storage/innobase/trx/trx0trx.cc

static void trx_release_impl_and_expl_locks(trx_t *trx, bool serialised) {
  // ...
  if (trx->id > 0) {
    trx_erase_lists(trx);  // 从 rw_trx_ids 和 rw_trx_list 中移除
  }
  // 状态转换和 shard 操作
  auto state_transition = [&]() {
    trx_mutex_enter(trx);
    // 此处将隐式锁释放和状态变更原子化
    // 受 Trx_shard mutex 和 trx->mutex 双重保护
  };
}
```

事务从 `active_rw_trxs` 的移除操作仅获取对应 shard 的 mutex（而非全局 trx_sys mutex），发生在状态变更为 `TRX_STATE_COMMITTED_IN_MEMORY` 的同一临界区内。

### 6.4 `trx_guid_t` 的引入

8.0.41 引入了 `trx_guid_t` 结构来唯一标识事务：

```cpp
// storage/innobase/include/trx0types.h

struct trx_guid_t {
  uint64_t m_immutable_id{};  // trx_t 对象的不可变 ID
  uint64_t m_version{};       // 版本号，trx_t 对象复用时递增
};
```

由于 `trx_t` 对象通过对象池复用，单独的 `trx->id` 不足以跨时间唯一标识一个事务实例。`trx_guid_t` 解决了 ABA 问题。

---

## 七、Hash 函数与随机数优化

**关键 Commit**: `624f5847ef5` — *Bug#34870256: New hash function causes performance regression*

8.0.30 统一使用的新 hash 函数依赖 64KB 查找表，导致频繁使用时的 cache 污染和性能回退。

8.0.41 的修复策略：
- `page_id_t::hash()` 回退到原有的计算方式（但扩展为 64 位）
- 轻量级随机数生成使用 `my_timer_cycles()` 替代重量级 hash
- 其他场景优先使用 **CRC32 指令**（如果 CPU 支持）

这直接影响了 lock system 中的 hash bucket 分配和 adaptive hash index 的性能。

---

## 八、Purge 子系统优化

### 8.1 Undo 截断时的锁降级

**关键 Commit**: `db3d364d1f1` — *Bug #32353863 WORKLOAD STALLS WHILE RUNNING TRUNCATION WITH LARGE BUFFER POOL*

将 undo 空间截断操作中对 undo space 的 X-latch 降级为 S-latch，且持有时间更短，只用于访问 spaces 列表。

### 8.2 Rollback Segment 并行初始化

**关键 Commit**: `587a9e8120f` / `08c30638d22` — *Bug#32170127: Rollback segment can be initialized parallel*

允许回滚段在恢复阶段并行初始化，缩短了崩溃恢复时间。

### 8.3 `TrxUndoRsegs` 的堆分配消除

**关键 Commit**: `bf8fffd3dd0` (同 §2)

```cpp
// storage/innobase/include/trx0types.h

class TrxUndoRsegs {
  Rsegs_array<2> m_rsegs;  // 内联 2 元素数组，无堆分配
  size_t m_rsegs_n{};
};
```

将原来基于 `std::vector` 的实现改为固定大小的栈数组（最多 2 个回滚段：redo + noredo），消除了 purge 队列操作中的堆分配开销。

---

## 九、事务恢复相关优化

### 9.1 恢复进度日志

**关键 Commit**: `6337405f71b` — *WL #15387: Innodb: Log progress information while resurrecting transaction during recovery*

恢复大事务时每 30 秒输出进度日志，帮助 DBA 评估恢复时间：

```cpp
// storage/innobase/trx/trx0trx.cc

auto const progress_log_interval =
    DBUG_EVALUATE_IF("resurrect_logs", 1s, 30s);

if (time_diff >= progress_log_interval) {
  ib::info(ER_IB_RESURRECT_RECORD_PROGRESS, n_undo_recs, n_undo_pages);
  last_progress_log_time = now;
}
```

### 9.2 Crash-safe XA + Binary Log

**关键 Commit**: `c1401ada9b1` — *WL#11300 Crash-safe XA + binary log*

增强了 XA 事务在崩溃场景下的安全性，确保 XA PREPARE 与 binlog 的一致性。

---

## 十、Cache Line 对齐与内存布局优化

`trx_sys_t` 结构中大量使用 cache line padding 来避免 false sharing：

```cpp
// storage/innobase/include/trx0sys.h

struct trx_sys_t {
  char pad0[ut::INNODB_CACHE_LINE_SIZE];
  MVCC *mvcc;
  // ...
  char pad1[ut::INNODB_CACHE_LINE_SIZE];
  std::atomic<trx_id_t> next_trx_id_or_no;
  char pad2[ut::INNODB_CACHE_LINE_SIZE];
  TrxSysMutex serialisation_mutex;
  // ...
  char pad4[ut::INNODB_CACHE_LINE_SIZE];
  TrxSysMutex mutex;
  char pad5[ut::INNODB_CACHE_LINE_SIZE];
  UT_LIST_BASE_NODE_T(trx_t, trx_list) rw_trx_list;
  char pad6[ut::INNODB_CACHE_LINE_SIZE];
  // ...
  char pad7[ut::INNODB_CACHE_LINE_SIZE];
  Trx_shard shards[TRX_SHARDS_N];
};
```

每个关键字段组之间都有 cache line 大小的 padding，确保不同的热点字段不会共享同一 cache line。`Trx_shard` 中的 `active_rw_trxs` 也使用了 `Cacheline_padded` 包装。

---

## 十一、关键 Commit 汇总表

| Commit | Bug/WL | 核心变化 | 影响范围 |
|--------|--------|----------|----------|
| `bf8fffd3dd0` | BUG#32832196 | 活跃事务管理 256 分片化 | trx_sys, ReadView, MVCC |
| `e6df317f8dd` | Bug#33059387 | 消除 `trx_rw_min_trx_id()` 冗余 mutex 获取 | lock_sec_rec, trx_sys |
| `7037a0bdc83` | Bug#11745929 | S→X 升级优先级优化，避免死锁 | lock0lock |
| `5ccf484a4fc` | Bug#33856332 | HP 事务超时机制 | lock0wait |
| `96ccd8dc9f9` | Bug#36302624 | data_lock_waits O(N³)→O(N) | p_s, lock0lock, lock0iter |
| `5a6fd9936d8` | Bug#33502610 | Global lock-sys latch 持有时间限制 | lock0lock |
| `624f5847ef5` | Bug#34870256 | Hash 函数性能回退修复 | ut0rnd, lock, AHI |
| `0568caadd89` | Bug#34352870 | 时钟函数性能回退修复 | trx0trx, lock0wait |
| `98c11a5dc92` | Bug#35289390 | purge_sys latch 竞争消除 | ha_innodb, trx0purge |
| `db3d364d1f1` | Bug#32353863 | Undo 截断锁降级 | trx0purge |
| `587a9e8120f` | Bug#32170127 | Rollback segment 并行初始化 | trx0rseg |
| `c1401ada9b1` | WL#11300 | Crash-safe XA + binary log | trx0trx |
| `6337405f71b` | WL#15387 | 恢复事务进度日志 | trx0trx |

---

## 十二、对内核特性开发的建议

### 12.1 如果要进一步优化活跃事务管理

- 当前 ReadView 创建仍需要获取全局 `trx_sys->mutex` 来复制 `rw_trx_ids`。可以考虑基于 epoch-based 的无锁快照方案。
- `Trx_by_id_with_min::erase()` 中更新 `m_min_id` 的循环在极端场景下可能较慢（当 shard 内事务 ID 跨度很大时）。

### 12.2 如果要优化 ReadView

- 当前 `m_ids` 使用自定义数组 + `binary_search`，可以考虑基于 bitmap 或布隆过滤器的方案来加速 `changes_visible()` 的慢速路径。
- AC-NL-RO 的快速路径依赖 `next_trx_id_or_no` 的原子读，这意味着任何 RW 事务的启动都会使所有 AC-NL-RO view 失效。

### 12.3 如果要修改锁系统

- 8.0.41 的锁系统已经完成了从全局 mutex 到分片 latch 的演进，新的锁特性需要注意 per-shard 的 latch 协议。
- `lock_rec_find_set_bit()` 的跳跃式搜索是性能关键路径，修改 lock bitmap 结构时需要维持或改进这个优化。
- `data_locks` / `data_lock_waits` 的报告路径已经完全重写为 per-table/per-bucket 遍历，新增的锁类型需要适配这个遍历框架。

### 12.4 关键不变量（修改代码时必须维护）

1. 事务从 `active_rw_trxs` 移除和状态变更为 `TRX_STATE_COMMITTED_IN_MEMORY` 必须在同一个 `Trx_shard::mutex` 临界区内完成。
2. `trx_sys->rw_trx_ids` 的修改必须在 `trx_sys->mutex` 保护下进行。
3. `m_min_id` 的 `store` 使用 `memory_order_release`，`load` 使用 `memory_order_acquire`，保证了快速路径的正确性。
4. ReadView 的 `m_ids` 必须是有序的，`changes_visible()` 依赖二分搜索。

---

## 附录：关键源码文件索引

| 文件 | 行数(约) | 职责 |
|------|----------|------|
| `include/trx0sys.h` | 630 | trx_sys_t 结构定义，Trx_shard, Trx_by_id_with_min |
| `include/trx0sys.ic` | 313 | trx_rw_is_active(), trx_sys_rw_trx_add() 内联实现 |
| `include/trx0trx.h` | 1609 | trx_t 结构定义，trx_lock_t, TrxInInnoDB |
| `include/read0types.h` | 322 | ReadView 类定义，changes_visible() |
| `include/read0read.h` | 138 | MVCC 管理器定义 |
| `include/trx0types.h` | 643 | 基本类型定义，TrxUndoRsegs, trx_guid_t |
| `include/ut0guarded.h` | 61 | Guarded 模板（per-shard mutex 封装） |
| `trx/trx0trx.cc` | 3720 | 事务生命周期管理实现 |
| `read/read0read.cc` | 748 | MVCC view_open/close/clone 实现 |
| `trx/trx0sys.cc` | 779 | 事务系统初始化与关闭 |
| `lock/lock0lock.cc` | 6302 | 锁系统核心实现 |
| `lock/lock0wait.cc` | 1460 | 锁等待与超时处理 |
