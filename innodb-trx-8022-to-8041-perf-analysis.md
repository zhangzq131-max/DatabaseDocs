# InnoDB 事务模块变更分析：MySQL 8.0.22 → 8.0.41

> **文档性质**：本文档所有结论均直接来源于 `D:\codebase\mysql-server` 工程的源代码与 git 提交历史，每项结论附有具体的文件路径、行号和 commit hash，可供交叉查验。  
> **适用场景**：内核特性开发参考。

---

## 目录

1. [架构级变更总览](#1-架构级变更总览)
2. [活跃事务管理：256-Shard 分片架构](#2-活跃事务管理256-shard-分片架构)
3. [Read View 优化](#3-read-view-优化)
4. [提交序列化路径优化](#4-提交序列化路径优化)
5. [Purge 子系统优化](#5-purge-子系统优化)
6. [事务对象生命周期优化](#6-事务对象生命周期优化)
7. [锁子系统相关改进](#7-锁子系统相关改进)
8. [Redo Log 动态配置](#8-redo-log-动态配置)
9. [崩溃恢复与 XA 改进](#9-崩溃恢复与-xa-改进)
10. [哈希函数性能修复](#10-哈希函数性能修复)
11. [开发建议：基于本文档的内核特性扩展注意事项](#11-开发建议基于本文档的内核特性扩展注意事项)

---

## 1. 架构级变更总览

| 变更领域 | 核心 Bug/WL | 引入版本范围 | 性能影响 |
|---|---|---|---|
| 活跃事务集 256 分片 | BUG#32832196 | 8.0.27 附近 | 高并发下 `trx_sys->mutex` 争用大幅下降 |
| 删除冗余 `trx_rw_min_trx_id()` 调用 | BUG#33059387 | 8.0.29 附近 | 消除 secondary index 读路径上一次 `trx_sys->mutex` 获取 |
| Rollback Segment 并行初始化 | BUG#32170127 | 8.0.27 附近 | 服务启动/恢复速度提升 |
| Purge 线程识别改为 thread_local | BUG#35289390 | 8.0.35 附近 | 消除 `purge_sys->latch` 死锁场景 |
| 哈希函数性能回归修复 | BUG#34870256 | 8.0.32 附近 | 恢复 AHI/锁表哈希到原有性能水平 |
| `std::chrono::steady_clock` → `time()` 回退 | BUG#34352870 | 8.0.31 附近 | 消除因系统调用引入的 ~5% 事务启动回归 |
| Crash-safe XA + binary log | WL#11300 | 8.0.30 | XA 事务持久性语义重构 |
| Redo Log 动态配置 | WL#12527 | 8.0.30 | 支持运行时调整 redo log 大小 |
| data_lock_waits O(N³)→O(N) | BUG#36302624 | 8.0.40 附近 | P_S 锁等待查询大表场景下性能质变 |

---

## 2. 活跃事务管理：256-Shard 分片架构

### 2.1 背景问题

在 8.0.22 中，所有读写事务被维护在单一的 `trx_sys->rw_trx_set`（`std::set<TrxTrack>`）中，由全局 `trx_sys->mutex` 保护。在高并发场景（>64 线程）下，该 mutex 成为严重瓶颈。

**来源**：commit `bc95476c015`（后续 squash 为 `bf8fffd3dd0`），BUG#32832196，描述为 "SINGLE RW_TRX_SET LEADS TO CONTENTION ON TRX_SYS MUTEX"。

### 2.2 分片数据结构

**文件**：`storage/innobase/include/trx0sys.h`

```cpp
// trx0sys.h:368
constexpr size_t TRX_SHARDS_N = 256;

inline size_t trx_get_shard_no(trx_id_t trx_id) {
  ut_ad(trx_id != 0);
  return trx_id % TRX_SHARDS_N;
}
```

每个 shard 是一个独立的 `Trx_by_id_with_min`，由专属 mutex 保护，通过 `ut::Cacheline_padded` 避免伪共享：

```cpp
// trx0sys.h:443
struct Trx_shard {
  ut::Cacheline_padded<
    ut::Guarded<Trx_by_id_with_min, LATCH_ID_TRX_SYS_SHARD>>
      active_rw_trxs;
};
```

Shard 数组在 `trx_sys_t` 中（`trx0sys.h:567`）：
```cpp
Trx_shard shards[TRX_SHARDS_N];
```

### 2.3 关键优化：无锁最小事务 ID 检查

`Trx_by_id_with_min` 内部维护一个 `std::atomic<trx_id_t> m_min_id`：

```cpp
// trx0sys.h:395
std::atomic<trx_id_t> m_min_id{0};
```

`insert()` 与 `erase()` 均以 `std::memory_order_release` 更新 `m_min_id`。

**直接收益**：`trx_rw_is_active()` 的快速路径（`trx0sys.ic:175`）可以在**不持有任何 mutex** 的情况下，通过原子 load（`memory_order_acquire`）判断一个 `trx_id` 是否低于该 shard 的最小活跃事务 ID：

```cpp
// trx0sys.ic 逻辑（精简）
inline trx_t *trx_rw_is_active(trx_id_t trx_id, ...) {
  const auto shard_no = trx_get_shard_no(trx_id);
  // 无锁快速路径：
  if (trx_id < trx_sys->shards[shard_no]
                    .active_rw_trxs.peek().min_id()) {
    return nullptr;  // 不可能是活跃事务，直接返回
  }
  // 慢路径：获取 shard mutex 精确查找
  ...
}
```

同样的无锁检查也用于替换原来需要获取 `trx_sys->mutex` 的 `trx_rw_min_trx_id()` 调用（见下节）。

### 2.4 `m_min_id` 的 erase 逻辑

`m_min_id` 在 erase 时需要在同 shard 内找下一个最小的 ID：

```cpp
// trx0sys.h:420（Trx_by_id_with_min::erase）
// 因为同 shard 内所有 ID 的 trx_id % 256 相同，
// 所以步长为 TRX_SHARDS_N 的扫描是正确的。
trx_id_t new_min = erased_id + TRX_SHARDS_N;
while (new_min <= max_id_in_shard) {
  if (m_by_id.count(new_min)) break;
  new_min += TRX_SHARDS_N;
}
m_min_id.store(new_min, std::memory_order_release);
```

> **开发注意**：如果需要在 shard 结构上增加新字段（如统计计数），必须使用 `latch_and_execute()` 访问，不能绕过 shard mutex 直接读写除 `m_min_id` 以外的字段。

### 2.5 消除冗余的 `trx_rw_min_trx_id()` 调用

**文件**：`storage/innobase/lock/lock0lock.cc`  
**来源**：commit `0a0c0db97e9`，BUG#33059387

8.0.22 中 `lock_sec_rec_read_check_and_lock()` 在加锁前有一个启发式检查：

```cpp
// 8.0.22 旧代码（已删除）
if (page_get_max_trx_id(block->frame) >= trx_rw_min_trx_id() || ...) {
  lock_rec_convert_impl_to_expl(...);
}
```

`trx_rw_min_trx_id()` 需获取 `trx_sys->mutex`，这在 secondary index 读路径上是极高频操作。

**8.0.41 的做法**：
1. 移除该冗余的外层检查（`lock_rec_convert_impl_to_expl` 内部会做相同检查）。
2. 引入 `can_older_trx_be_still_active(max_trx_id)`，利用对 256 个 shard 的原子 load（无 mutex）替代原来的单次 mutex 获取：

```cpp
// lock0lock.cc（当前实现逻辑）
// 若 trx_sys->mutex 无争用则直接取，有争用则遍历 256 个 shard 的 m_min_id
// 256 次 acquire load 在 x86 上等于 256 次 mov，远优于等待 spinlock
```

---

## 3. Read View 优化

### 3.1 AC-NL-RO 事务视图重用（无 mutex）

**文件**：`storage/innobase/read/read0read.cc`，`MVCC::view_open()`

对于自动提交非锁定只读（Autocommit Non-Locking Read-Only）事务，若当前无活跃的 RW 事务且 `next_trx_id_or_no` 未推进，则可以**不获取 `trx_sys->mutex`** 直接复用旧视图：

```cpp
// read0read.cc:547（简化）
void MVCC::view_open(ReadView *&view, trx_t *trx) {
  if (view != nullptr && trx_is_autocommit_non_locking(trx) && view->empty()) {
    view->m_closed = false;
    if (view->m_low_limit_id == trx_sys_get_next_trx_id_or_no()) {
      return;  // 无任何新 RW 事务，视图有效，直接复用
    }
    view->m_closed = true;
  }
  // 慢路径：获取 trx_sys->mutex 重建视图
  trx_sys_mutex_enter();
  ...
}
```

`trx_sys_get_next_trx_id_or_no()` 是对 `trx_sys->next_trx_id_or_no` 的原子 load，无锁。

> **适用场景**：大量 SELECT 的 OLAP 负载、连接池空闲后第一条 SELECT。

### 3.2 视图关闭：低位指针标记（无 mutex）

**文件**：`storage/innobase/read/read0read.cc`，`MVCC::view_close()`

AC-NL-RO 事务关闭视图时不需要持有 `trx_sys->mutex`，通过对指针低位打标签实现：

```cpp
// read0read.cc:722（简化）
void MVCC::view_close(ReadView *&view, bool own_mutex) {
  if (!own_mutex) {
    // 仅设置 m_closed 标志，并在指针低位打 0x1 标记
    reinterpret_cast<ReadView *>(p & ~1)->m_closed = true;
    view = reinterpret_cast<ReadView *>(p | 0x1);  // 标记为 closed
  } else {
    // 完整删除，需要持有 mutex
    ...
  }
}
```

`MVCC::is_view_active()` 识别此标记：

```cpp
// read0read.h:89
static bool is_view_active(ReadView *view) {
  return (view != nullptr && !(intptr_t(view) & 0x1));
}
```

物理释放推迟到下次 `view_open()` 或 `clone_oldest_view()` 时在 mutex 内完成，将 closed 视图移至 `m_free` 链表。

> **开发注意**：若自行扩展 `ReadView` 字段，`m_closed` flag 的修改路径**不在 mutex 保护下**，若新字段需要在视图关闭时修改，需要特别处理并发安全性。

### 3.3 `copy_prepare` / `copy_complete` 两阶段拷贝

**文件**：`storage/innobase/read/read0read.cc`，`MVCC::clone_oldest_view()`

Purge 系统获取最旧视图时，将拷贝分为两个阶段以最小化持锁时间：

```cpp
// read0read.cc:674（简化）
void MVCC::clone_oldest_view(ReadView *view) {
  trx_sys_mutex_enter();
  ReadView *oldest = get_oldest_view();
  if (oldest) {
    view->copy_prepare(*oldest);   // 在 mutex 内拷贝元数据（短暂）
    trx_sys_mutex_exit();
    view->copy_complete();         // 在 mutex 外完成 m_ids 插入（可能较慢）
  } else {
    view->prepare(0);
    trx_sys_mutex_exit();
  }
  // GTID 持久化调整（只能收紧 low_limit，不能放宽）
  view->reduce_low_limit(gtid_persistor.get_oldest_trx_no());
}
```

### 3.4 `ReadView::changes_visible()` 热路径

**文件**：`storage/innobase/include/read0types.h:163`

三层水位检查 + 二分搜索，均为 inline：

```cpp
[[nodiscard]] bool changes_visible(trx_id_t id, const table_name_t &name) const {
  if (id < m_up_limit_id || id == m_creator_trx_id) return true;   // 低水位，直接可见
  check_trx_id_sanity(id, name);
  if (id >= m_low_limit_id) return false;                           // 高水位，直接不可见
  if (m_ids.empty()) return true;                                   // 无活跃事务，可见
  const ids_t::value_type *p = m_ids.data();
  return !std::binary_search(p, p + m_ids.size(), id);             // 二分搜索活跃集
}
```

`m_ids` 使用自定义数组（非 `std::vector`），避免 `resize()` 的不确定开销，`copy_trx_ids()` 使用 `memmove` 单次 filter-copy（`read0read.cc:354`）。

### 3.5 双 Mutex 分离：`mutex` 与 `serialisation_mutex`

**文件**：`storage/innobase/include/trx0sys.h:504–531`

8.0.22 中，事务启动（分配 `trx->id`，加入 `rw_trx_list`）和事务提交序列化（分配 `trx->no`，加入 `serialisation_list`）共用同一个 `trx_sys->mutex`。

8.0.41 将其拆分：

| Mutex | 保护内容 |
|---|---|
| `trx_sys->mutex` | `rw_trx_list`、`mysql_trx_list`、`rw_trx_ids`、`n_prepared_trx`、`trx->id` 分配 |
| `trx_sys->serialisation_mutex` | `serialisation_list`、`serialisation_min_trx_no`、`trx->no` 分配 |

两者可并发持有，允许事务启动与事务序列化提交在不同线程上同时进行。

`serialisation_min_trx_no` 是 `std::atomic<trx_id_t>`（`trx0sys.h:521`），供 `ReadView::prepare()` 以原子 load 方式读取，设置 `m_low_limit_no` 而无需额外加锁。

---

## 4. 提交序列化路径优化

### 4.1 `trx_add_to_serialisation_list` / `trx_erase_from_serialisation_list_low`

**文件**：`storage/innobase/trx/trx0trx.cc:1469`

序列化链表维护 `serialisation_min_trx_no` 原子值，用于 Read View 的 `m_low_limit_no` 设置，防止 purge 删除仍被活跃视图引用的 undo 记录。

### 4.2 `trx_release_impl_and_expl_locks` — 状态转换与 Shard 删除的原子性

**文件**：`storage/innobase/trx/trx0trx.cc:1856`

关键约束（代码注释原文，`trx0trx.cc:1931`）：

> "It is important to remove the transaction from the serialisation list **after** it is erased from the rw_trx_ids / rw_trx_list (not before!). Otherwise a read-view could be created, which could still pretend that changes of this transaction are invisible, but related undo records could become purged."

状态转换为 `COMMITTED_IN_MEMORY` 在 shard mutex 内与 shard erase 原子完成：

```cpp
trx_sys->get_shard_by_trx_id(trx->id).active_rw_trxs.latch_and_execute(
    [&](Trx_by_id_with_min &by_id) {
      state_transition();      // 在 shard mutex 内设置 COMMITTED_IN_MEMORY
      by_id.erase(trx->id);   // 同时从 shard 删除
    }, UT_LOCATION_HERE);
```

确保任何通过 `trx_rw_is_active()` 检查的代码都不会看到"已提交但仍在 shard 中"的中间状态。

### 4.3 提交 LSN 与事务号顺序

**文件**：`storage/innobase/trx/trx0trx.cc:2207`，注释原文：

> "NOTE that transaction numbers...do not necessarily come in exactly the same order as commit lsn's, if the transactions have different rollback segments. To get exactly the same order we should hold the kernel mutex up to this point, adding to the contention of the kernel mutex."

这是一个显式的**性能 vs 顺序**权衡：接受不同 rollback segment 的事务提交 LSN 与 `trx_no` 可能轻微乱序，换取更低的 mutex 争用。

> **开发注意**：如果你的特性需要按严格提交顺序处理事务（如 binlog 序列化相关），不能依赖 `trx_no` 与 LSN 完全对应。

---

## 5. Purge 子系统优化

### 5.1 Purge 线程识别：`thread_local bool` 替代全局集合

**文件**：`storage/innobase/include/trx0purge.h:1084`，`storage/innobase/handler/ha_innodb.cc`  
**来源**：commit `98c11a5dc92`，BUG#35289390

8.0.22 中，purge 线程集合存储在 `purge_sys->thds`（`std::unordered_set`），由 `purge_sys->latch` 保护。`innobase_trx_allocate()` 在每次分配事务时会 S-latch `purge_sys->latch` 以判断当前线程是否是 purge 线程。

**问题**：当 purge coordinator 持有 `purge_sys->latch`（X-lock）并等待 `trx_sys->mutex` 以拷贝 oldest read view 时，其他线程的 `innobase_trx_allocate()` 因需要 S-latch 而阻塞，造成"long semaphore wait"。

**8.0.41 方案**：改用 `thread_local bool is_this_a_purge_thread`：

```cpp
// trx0purge.h:1084
static inline thread_local bool is_this_a_purge_thread{false};
```

- 零争用：`thread_local` 无需任何同步。
- purge 线程在 `srv_do_purge()` 入口设置为 `true`，`innobase_trx_allocate()` 直接读取，不需要任何 latch。

### 5.2 Purge 并行工作负载分组：`Purge_groups_t`

**文件**：`storage/innobase/trx/trx0purge.cc:1997`

```cpp
struct Purge_groups_t {
  GroupBy m_grpid_umap;        // table_id -> group_id（unordered_map）
  std::vector<purge_node_t::Recs *> m_groups;  // per-thread 记录列表
  std::size_t m_total_rec;
};
```

同一 `table_id` 的 undo 记录被分配到同一个 purge worker 线程，**避免多个 purge 线程并发修改同一张表的 B-tree** 导致的锁冲突。

`distribute_if_needed()`（`trx0purge.cc:2183`）仅在 `rseg_history_len > srv_max_purge_lag` 时触发重分配，正常负载下使用"最小负载组"分配策略，无重分配开销。

`distribute()` 使用 `std::list::splice()`（O(1)）在组间移动记录，最多 2 次 pass。

### 5.3 LOB 一级页延迟释放

**文件**：`storage/innobase/trx/trx0purge.cc:2472`

多个 purge worker 在处理同一 LOB 时存在竞争：worker A 正在读 LOB 一级页，worker B 可能将其释放。

**解决方案**：LOB 一级页的释放延迟到整个 purge batch 所有 worker 完成之后统一处理，由 coordinator 串行完成。

### 5.4 DML 节流：`trx_purge_dml_delay()`

**文件**：`storage/innobase/trx/trx0purge.cc:2315`

```cpp
float ratio = float(rseg_history_len) / srv_max_purge_lag;
if (ratio > 1.0) {
  delay = (ulint)((ratio - 0.9995) * 10000);  // 微秒延迟
}
```

DML 线程以与 history list 长度成比例的方式被节流。0.9995 的偏移确保从约 5µs 开始平滑斜坡，避免突变。`rseg_history_len` 以"脏读"方式读取（无 mutex），注释中已标注此为有意行为（`trx0purge.cc:2323`）。

### 5.5 Rollback Segment 并行初始化

**文件**：`storage/innobase/trx/trx0sys.cc:432`  
**来源**：commit `587a9e8120f` / `ee1bf602e4f`，BUG#32170127

```cpp
// trx0sys.cc:432
constexpr size_t max_rseg_init_threads = 4;
srv_rseg_init_threads = std::min(hardware_concurrency(), max_rseg_init_threads);
if (srv_rseg_init_threads > 1) {
  trx_rsegs_parallel_init(purge_queue);
}
```

最多 4 个线程并行扫描并初始化 rollback segments，加速崩溃恢复后的启动时间。使用 `std::chrono::high_resolution_clock` 计时并记录到 error log。

### 5.6 Purge Worker 自适应等待

**文件**：`storage/innobase/trx/trx0purge.cc:2348`

```cpp
while (purge_sys->n_completed.load() != n_submitted) {
  if (++i < 10) {
    std::this_thread::yield();         // 前 10 次：协作让步
  } else {
    srv_release_threads(SRV_WORKER, 1); // 唤醒 worker（若 task queue 非空）
    std::this_thread::sleep_for(std::chrono::microseconds(20));
    i = 0;
  }
}
```

`n_completed` 为 `std::atomic<ulint>`，`n_submitted` 为 `volatile ulint`，两者配合确保 coordinator 能正确感知所有 worker 完成。

---

## 6. 事务对象生命周期优化

### 6.1 `trx_t` 对象池（4MB 块）

**文件**：`storage/innobase/trx/trx0trx.cc:381`

```cpp
static const ulint MAX_TRX_BLOCK_SIZE = 1024 * 1024 * 4;
typedef Pool<trx_t, TrxFactory, TrxPoolLock> trx_pool_t;
typedef PoolManager<trx_pool_t, TrxPoolManagerLock> trx_pools_t;
```

`trx_t` 对象以 4MB 为单位批量分配，避免高频 `malloc/free`。`TrxFactory::init()` 使用 placement-new（`trx0trx.cc:499`），对已置零的内存调用构造函数。

已知局限（代码注释，`trx0trx.cc:499`）：`autoinc_locks` 向量的堆内存仍在每次 commit 时释放并重分配（FIXME 待解决）。

### 6.2 `trx->in_depth`：嵌套调用深度无锁计数

**文件**：`storage/innobase/include/trx0trx.h:709`

代码注释原文：

> "in_depth was split from in_innodb for fixing a RO performance issue. Acquiring the trx_t::mutex for each row costs ~3% in performance. It is not required for correctness."

`in_depth` 在不持任何 mutex 的情况下追踪 InnoDB 调用嵌套深度。只有 `in_depth` 从 0 升为 1 时才获取 `trx_mutex` 修改 `in_innodb`，节省了嵌套调用路径上的 mutex 开销。

### 6.3 `trx->start_time`：从 `std::chrono` 改回 `time(nullptr)`

**文件**：`storage/innobase/trx/trx0trx.cc`  
**来源**：commit `0568caadd89`，BUG#34352870

8.0.30 附近将计时改为 `std::chrono::steady_clock::now()`，在某些旧版 Linux 上引入了因 syscall 导致的 ~5% 性能回归（`std::chrono` 无法使用 vDSO 的系统调用路径）。

**8.0.41**：`trx->start_time`、`thd_start_time`、`trx->lock.wait_started` 统一改回使用低精度时钟 `time(nullptr)`，确保无 syscall 开销。

> **开发注意**：若需要记录精确的事务启动时间戳（如用于监控或审计），注意 `trx->start_time` 是秒精度的，不适合高精度计时。

### 6.4 `trx->state` 与 `trx->killed_by`：`std::atomic` 化

**来源**：BUG#32832196

`trx->state` 由 `trx_state_t` 改为 `std::atomic<trx_state_t>`（`trx0trx.h:800`），防止在非 x86 平台上的撕裂读（torn read）。大多数状态转换使用 `memory_order_relaxed`，仅在语义上需要顺序性的地方使用 `memory_order_release`/`acquire`（在代码注释中逐一说明）。

`trx->lock.schedule_weight`（`trx0trx.h:566`）同样是 `std::atomic`，注释明确说明接受 stale 读以换取性能：

> "Values are updated and read without any synchronization beyond that provided by atomics, as slightly stale values do not hurt correctness, just the performance."

---

## 7. 锁子系统相关改进

### 7.1 `performance_schema.data_lock_waits` 查询优化

**来源**：commit `96ccd8dc9f9`，BUG#36302624

8.0.22 中该表的生成复杂度为 O(N³)（N = 锁数量）。8.0.41 中降至 O(N)，原理是为每个等待事务直接维护 `blocking_trx` 引用，而不是对所有锁对做笛卡尔积扫描。

在大表高并发锁等待场景（如数百个并发事务等待锁），该优化可将该 P_S 查询从秒级变为毫秒级。

**文件**：`storage/innobase/include/trx0trx.h:439`

```cpp
std::atomic<trx_t *> blocking_trx;  // 当前阻塞本事务的事务指针
```

### 7.2 锁优先级：S-lock 持有者可获 X-lock（BUG#11745929）

**来源**：commit `7037a0bdc83`

持有 S-lock 的事务请求 X-lock 时获得优先权，避免被后续事务饿死。此变更调整了 `lock_rec_has_to_wait()` 的等待判断逻辑。

---

## 8. Redo Log 动态配置

**来源**：WL#12527，commit `b7a22c741f6`

8.0.30 开始支持在不关闭 MySQL 的情况下动态调整 redo log 文件大小（`innodb_redo_log_capacity`）。这对事务吞吐量的影响体现在：

- 减少因 redo log 写满造成的 stall（可在线扩容）
- 更精确地匹配不同工作负载的 redo log 需求

相关结构在 `storage/innobase/log/` 目录下重构较大，运行时 resize 通过独立的 log resize 线程完成，对事务提交路径透明。

---

## 9. 崩溃恢复与 XA 改进

### 9.1 Crash-safe XA + Binary Log（WL#11300）

**来源**：commit `c1401ada9b1`，8.0.30

重构了 XA 事务与 binary log 的一致性协议，确保在 `PREPARE` 后 `COMMIT` 前崩溃时的正确恢复行为。关键改动在 `trx_t` 的 XA state 机器与 binlog 的 XA one-phase 优化路径上。

> **开发注意**：若开发需要与 XA 语义交互的特性，需要理解新的状态机：`NOT_STARTED → ACTIVE → PREPARED → COMMITTED`，其中 `PREPARED` 状态需要特别处理以保证 crash-safe。

### 9.2 恢复期间事务复活进度日志

**来源**：commit `6337405f71b`，WL#15387

崩溃恢复时，InnoDB 会在 error log 中周期性输出正在 resurrect 的事务进度信息，方便运维定位长时间恢复的原因。

---

## 10. 哈希函数性能修复

### 10.1 全局哈希函数重构（8.0.28）与回退（8.0.32）

**来源**：commit `b11a1759241`（BUG#16739204，8.0.28），随后 `624f5847ef5`（BUG#34870256，8.0.32 修复回归）

8.0.28 将 InnoDB 内部所有哈希/随机函数统一为基于 64KB 查找表的 Tabulation Hashing，提升随机分布质量，但引入了对 L1 cache 的强烈污染，在锁表、AHI、buf pool 等高频哈希场景造成显著性能回归。

**8.0.32 修复**：
- `page_id_t::hash()`：改回轻量 64-bit 计算（不用查找表）
- `ut_delay()` 等随机延迟：改为基于 `my_timer_cycles()` 的 TSC 低位
- 其余高频哈希（锁表分区等）：使用 CPU 原生 CRC32 指令（`ut0crc32.h`）

```cpp
// include/ut0crc32.h（8.0.32 新增）
// 若 CPU 支持 CRC32 硬件指令则走 intrinsic，否则 fallback 到软件实现
inline uint64_t ut_crc32_64(uint64_t val);
```

> **开发注意**：在 InnoDB 事务/锁代码中引入新的哈希需求时，请使用 `ut_crc32_64()` 或 `page_id_t::hash()` 的同类轻量实现，**不要**使用 8.0.28 引入的 `ut::hash_binary()`（适合低频场景）。

---

## 11. 开发建议：基于本文档的内核特性扩展注意事项

### A. 读写活跃事务集（最常被触及）

- **读取**：通过 `trx_rw_is_active()` 或 `trx_sys->shards[n].active_rw_trxs`，不要直接访问 `trx_sys->rw_trx_list`（后者需持有 `trx_sys->mutex`）。
- **遍历所有活跃事务**：必须持有 `trx_sys->mutex`，遍历 `rw_trx_list`；或者逐 shard 调用 `latch_and_execute()` 遍历每个 shard 的 `active_rw_trxs`。
- **写入**：insert 和 erase 分别在事务启动/提交时进行，入口已固定，避免在其他地方绕过标准路径修改。

### B. Read View 扩展

- `ReadView::prepare()` 在 `trx_sys->mutex` 保护下执行，可以在此处安全读取 `trx_sys` 状态。
- `changes_visible()` 是极热路径，任何新增字段检查都需要评估对单行读性能的影响。
- `m_ids` 使用自定义数组，扩展其容量或格式时注意 `copy_trx_ids()` 的 filter-copy 逻辑同步修改。

### C. Purge 子系统扩展

- 新增 purge 相关线程时，需设置 `is_this_a_purge_thread = true`，否则 `innobase_trx_allocate()` 无法正确识别。
- purge worker 的工作分发通过 `Purge_groups_t` 按 `table_id` 做，分组逻辑扩展需注意分组键的意义（当前以表为粒度，防止 B-tree 争用）。

### D. 事务状态机

当前完整状态转换链（`trx0trx.h:739`）：

```
NOT_STARTED
  ↓ (trx_start_low with rsegs) 
ACTIVE                    ←→  PREPARED (XA)
  ↓ (trx_commit_in_memory, under shard mutex)
COMMITTED_IN_MEMORY
  ↓ (cleanup)
NOT_STARTED
```

新增事务状态（如引入 Two-Phase Locking 的特殊状态）需要同时处理：shard mutex 内的状态变更原子性、`trx_rw_is_active()` 的正确性、以及 Read View 的可见性判断。

### E. `trx_sys->mutex` vs `serialisation_mutex` 的持有规则

| 操作 | 需要持有的 mutex |
|---|---|
| 分配 `trx->id`，读写 `rw_trx_ids`，读写 `rw_trx_list` | `trx_sys->mutex` |
| 分配 `trx->no`，读写 `serialisation_list`，读 `serialisation_min_trx_no`（写） | `serialisation_mutex` |
| 读取 `serialisation_min_trx_no`（读） | 无（原子 load） |
| 读写 shard 内的 `active_rw_trxs` | 对应 shard 的 latch（via `latch_and_execute`） |
| 开/关 Read View | `trx_sys->mutex`（AC-NL-RO 关闭除外） |

---

*文档生成日期：2026-04-03*  
*分析基础：MySQL 8.0.41 源码工程，分支 `mysql-8.0.41`，git HEAD `14ba93991ba`*
