# `trx_rw_is_active` 深度分析

> 基于 MySQL 8.0.41 源码（`storage/innobase`）及官方 Worklog 整理

---

## 一、函数的核心作用

`trx_rw_is_active` 回答的问题只有一个：**给定一个 `trx_id`，对应的读写事务现在还活跃吗？如果活跃，返回 `trx_t*` 指针。**

**声明**（`storage/innobase/include/trx0sys.h`，第 166–178 行）：

```cpp
/** Checks if a rw transaction with the given id is active.
Please note, that positive result means only that the trx was active
at some moment during the call, but it might have already become
TRX_STATE_COMMITTED_IN_MEMORY before the call returns to the caller, as this
transition is protected by trx->mutex and Trx_shard's mutex, but it is
impossible for the caller to hold any of these mutexes when calling this
function as the function itself internally acquires Trx_shard's mutex which
would cause recurrent mutex acquisition if caller already had the same mutex,
or latching order violation in case of holding trx->mutex.
@param[in]      trx_id          trx id of the transaction
@param[in]      do_ref_count    if true then increment the trx_t::n_ref_count
@return transaction instance if active, or NULL; */
static inline trx_t *trx_rw_is_active(trx_id_t trx_id, bool do_ref_count);
```

**完整实现**（`storage/innobase/include/trx0sys.ic`，第 175–227 行）：

```cpp
static inline trx_t *trx_rw_is_active(trx_id_t trx_id, bool do_ref_count) {
  /* Fast checking without Trx_shard::mutex to see if trx_id can't be in the
  Trx_shard::active_rw_trxs because it's too small. This works, because
  whenever we add or remove an element from Trx_by_id_with_min::m_by_id under
  Trx_shard::mutex, we also modify the m_min_id to be lower or equal to
  each of ids inside Trx_by_id_with_min::m_by_id in the same critical section.
  ...
  NOTE: to appreciate why the assumption is important, observe, that if we call
  the trx_rw_is_active(trx_id,..) with trx_id for which trx_sys_rw_trx_add(trx)
  wasn't called yet, then it could happen, that trx_id+TRX_SHARDS_N was already
  assigned and added to Trx_shard::active_rw_trxs.m_by_id, and Trx_shard::min_id
  was set to larger than trx_id, so we decide to return nullptr. */
  ut_ad(trx_id < trx_sys_get_next_trx_id_or_no());
  auto &shard = trx_sys->get_shard_by_trx_id(trx_id);
  if (trx_id < shard.active_rw_trxs.peek().min_id()) {
    return nullptr;                  // 快路径：无锁原子读，直接返回
  }
  return trx_sys->latch_and_execute_with_active_trx(
      trx_id,
      [&](trx_t *trx) {
        if (trx != nullptr && do_ref_count) {
          trx_reference(trx);        // 增加引用计数，防止指针悬空
        }
        return trx;
      },
      UT_LOCATION_HERE);
}
```

函数内部有**两条执行路径**：

| 路径 | 机制 | 触发条件 |
|------|------|----------|
| **快路径** | 原子读 `m_min_id`，无锁返回 `nullptr` | `trx_id < shard.min_id`，事务已确定提交 |
| **慢路径** | 加 `Trx_shard::mutex`，O(1) 哈希表查找 | `trx_id >= shard.min_id`，需要精确确认 |

---

## 二、为什么需要这个函数？—— InnoDB 隐式锁机制

这个场景是理解该函数的关键。

InnoDB 的行锁默认是**隐式锁**：插入/修改一行时，**不会真正在内存中创建锁对象**，而是把 `trx_id` 写入行的 Clustered Index Record Header 中。只要该行上的 `trx_id` 所对应的事务还活跃，该行就被认为被那个事务隐式持有 X 锁。

当另一个事务需要访问这行时，就需要调用 `lock_rec_convert_impl_to_expl()`——把隐式锁"具体化"为显式锁。这个过程**必须确认**：行上记录的那个 `trx_id` 现在是否还活跃。这就是 `trx_rw_is_active` 最直接的触发场景。

源码注释（`storage/innobase/trx/trx0trx.cc`，第 1895–1912 行）明确说明：

```cpp
  /* (1) if you only know id of the trx, then you can obtain Trx_shard's mutex
  and check if trx is still in the Trx_shard's active_rw_trxs. This works,
      because the removal from the active_rw_trxs is also protected by the
      same mutex. We use this approach in lock_rec_convert_impl_to_expl() by
      using trx_rw_is_active() */
```

### 主要调用场景

| 调用位置 | 场景 | 重要性 |
|----------|------|--------|
| `lock0lock.cc:5330` | **隐式锁转显式锁**（`lock_rec_convert_impl_to_expl`） | ⭐⭐⭐ 最核心、最高频 |
| `row0vers.cc:498` | `row_vers_impl_x_locked_low`：判断行是否被隐式 X 锁保护 | ⭐⭐⭐ |
| `row0vers.cc:1388` | `row_vers_build_for_semi_consistent_read`：半一致性读版本回退 | ⭐⭐ |
| `clone0copy.cc:403,410` | Clone 操作等待某事务结束（循环轮询直到不再活跃） | ⭐⭐ |
| `trx0roll.cc:800` | 崩溃恢复时验证事务仍活跃再处理 | ⭐ |
| `lock0lock.cc:1063` | 调试断言：隐式锁持有者存活检查 | 仅 Debug |
| `row0row.cc:383` | 调试：外部列为 NULL 时断言事务仍活跃 | 仅 Debug |

---

## 三、256 Shards 的来龙去脉

### 历史问题：全局 `trx_sys_t::mutex` 是性能瓶颈

在 MySQL 5.6/5.7 时代，所有活跃读写事务存在一个全局链表 `trx_sys_t::rw_trx_list` 中，访问它需要持有全局 `trx_sys_t::mutex`。每次隐式锁转换的伪代码如下：

```
Acquire lock_sys_t::mutex
  Acquire trx_sys_t::mutex
    线性扫描 rw_trx_list（O(N)！N = 活跃 RW 事务数）
  Release trx_sys_t::mutex
  if 找到活跃事务:
    执行隐式锁 → 显式锁转换
Release lock_sys_t::mutex
```

官方 [WL#6899](https://dev.mysql.com/worklog/task/?id=6899) 对此有明确描述：

> "The above pseudo code should make it clear that as the `trx_sys_t::rw_trx_list` grows it has a proportional cost on the `lock_sys_t::mutex` and **that causes a sharp drop in TPS at higher concurrency e.g., 1K RW threads in Sysbench**."

问题的关键在于：`lock_rec_convert_impl_to_expl()` 持有 `lock_sys_t::mutex` 的同时，还要去扫描 `rw_trx_list`，导致两把热锁同时被长时间持有，高并发下 TPS 出现断崖式下跌。

生产环境中可以直接观测到这个瓶颈：

```
| InnoDB | &trx_sys-mutex | os_waits=19963 |
| InnoDB | &lock_mutex    | os_waits=8231  |
```

### 256 分片的设计

`TRX_SHARDS_N = 256` 定义在 `storage/innobase/include/trx0sys.h`：

```cpp
/** Number of shards created for transactions. */
constexpr size_t TRX_SHARDS_N = 256;

/** Computes shard number for a given trx_id. */
inline size_t trx_get_shard_no(trx_id_t trx_id) {
  ut_ad(trx_id != 0);
  return trx_id % TRX_SHARDS_N;   // 简单取模，保证同一事务始终落在同一 shard
}
```

`trx_sys_t` 中的核心结构：

```cpp
/** Mapping from transaction id to transaction instance. */
Trx_shard shards[TRX_SHARDS_N];

Trx_shard &get_shard_by_trx_id(trx_id_t trx_id) {
  return trx_sys->shards[trx_get_shard_no(trx_id)];
}

template <typename F>
auto latch_and_execute_with_active_trx(trx_id_t trx_id, F &&f,
                                       const ut::Location &loc) {
  return get_shard_by_trx_id(trx_id).active_rw_trxs.latch_and_execute(
      [&](Trx_by_id_with_min &trx_by_id_with_min) {
        return std::forward<F>(f)(trx_by_id_with_min.get(trx_id));
      },
      loc);
}
```

每个 `Trx_shard` 包含一个 `Trx_by_id_with_min`，其内部结构如下：

- `m_by_id`：`std::map<trx_id_t, trx_t*>`，O(1) 哈希查找
- `m_min_id`：`std::atomic<trx_id_t>`，该 shard 中最小的活跃 `trx_id`

`m_min_id` 的维护保证了快路径的正确性：删除元素时更新为下一个同模 256 的候选 id：

```cpp
void erase(trx_id_t trx_id) {
  if (m_min_id.load(std::memory_order_relaxed) == trx_id) {
    trx_id_t new_min = trx_id + TRX_SHARDS_N;   // 下一个同 shard 的候选 id
    while (m_by_id.count(new_min) == 0) {
      new_min += TRX_SHARDS_N;
    }
    m_min_id.store(new_min, std::memory_order_release);
  }
}
```

此外，每个 shard 的 mutex 还使用 **cache line padding** 防止伪共享：

```cpp
ut::Cacheline_padded<ut::Guarded<Trx_by_id_with_min, LATCH_ID_TRX_SYS_SHARD>>
    active_rw_trxs;
```

### 三层优化效果

| 优化点 | 旧设计 | 新设计 |
|--------|--------|--------|
| 查找复杂度 | O(N) 链表扫描 | O(1) 哈希查找 |
| 并发锁粒度 | 1 把全局 mutex | 256 把分片 mutex |
| 无锁快路径 | 无 | `m_min_id` 原子读，绝大多数"已提交"判断直接返回 |

---

## 四、256 分片是否主要为了这个场景？

**是，但不完全是。** 隐式锁转换（`trx_rw_is_active` 的最高频调用）是最主要的受益场景，但分片架构同时服务于多条热路径：

1. **隐式锁转显式锁** — 最高频，是 WL#6899 的核心动机
2. **MVCC 行版本可见性判断** — `row0vers.cc` 中多处调用
3. **事务提交时从 `active_rw_trxs` 删除自身** — 高频写操作，同样受益于分片（不同事务提交时操作不同 shard，互不干扰）
4. **Clone、恢复等低频路径** — 顺带受益

> **注意**：普通 MVCC 快照可见性判断（`ReadView::changes_visible()`）走的是另一套独立机制——基于 `m_ids` 数组的二分查找，与 `trx_rw_is_active` **相互独立**。前者回答"我建立快照时哪些事务未提交"，后者回答"此刻这个 trx_id 还在活跃集合里吗"，两者职责分离。

---

## 五、性能收益评估

官方没有为"256 shard"单独发布 benchmark 数字，但可以通过多方证据综合判断：

### 证据 1：官方 WL#6899 描述（定性）

> "causes a **sharp drop in TPS** at higher concurrency e.g., 1K RW threads in Sysbench"

在旧设计下，高并发（1000 RW 线程）时 TPS 会出现断崖式下跌，分片优化是解决这一问题的根本手段。

### 证据 2：`trx_sys-mutex` 实测争用数据

```
| InnoDB | &trx_sys-mutex | os_waits=19963 |   ← 旧版本，单 mutex 极热
```

256 分片后，单个 shard mutex 的理论 os_waits 降为原来的 **1/256**（假设 trx_id 均匀分布）。

### 证据 3：相关 Sysbench 数据参考

社区测试（MySQL 5.6 vs 8.0，sysbench update-index）显示 8.0 在写密集型负载下有 **~1.6x** 的 TPS 提升（QPS ratio 1.61），其中分片优化是重要贡献因子之一。

### 收益场景分析

| 场景 | 预期收益 | 原因 |
|------|----------|------|
| 写密集型，>128 并发线程 | **显著**，可能消除 TPS 断崖 | 隐式锁转换频繁，分片直接减少锁竞争 |
| 混合读写，中等并发 | **中等** | 写操作受益，读操作间接受益 |
| 读多写少，低并发 | **有限** | `trx_rw_is_active` 调用频次本身低 |
| NUMA 多路服务器 | **额外加成** | 分片后每个 shard 的 cache line 可以局部化，减少跨 NUMA node 的 cache line 拉取 |

### 如何在当前实例验证

在 MySQL 8.0.41 上可直接查询分片 mutex 的等待分布：

```sql
SELECT EVENT_NAME, COUNT_STAR, SUM_TIMER_WAIT / 1000000000 AS WAIT_MS
FROM performance_schema.events_waits_summary_global_by_event_name
WHERE EVENT_NAME LIKE '%trx_sys%'
ORDER BY WAIT_MS DESC;
```

新版本中你会看到 `wait/synch/mutex/innodb/trx_sys_shard`（256 个）代替了旧的单个 `trx_sys_mutex`，等待时间分散在 256 个 bucket 中。

---

## 六、参考资料

| 来源 | 链接 |
|------|------|
| 官方 WL#6899（隐式锁转换优化） | https://dev.mysql.com/worklog/task/?id=6899 |
| 官方 WL#6578（Read View 创建优化） | https://dev.mysql.com/worklog/task/?id=6578 |
| 官方 WL#10314（Lock-sys 分片 mutex） | https://dev.mysql.com/worklog/task/?id=10314 |
| MariaDB MDEV-22680（trx_sys 改进跟踪） | https://jira.mariadb.org/browse/MDEV-22680 |
| MySQL 官方 trx0sys.h 源码文档 | https://dev.mysql.com/doc/dev/mysql-server/9.6.0/trx0sys_8h_source.html |

---

## 七、关键源码路径速查

| 内容 | 文件路径 | 行号 |
|------|----------|------|
| `trx_rw_is_active` 实现（含长注释） | `storage/innobase/include/trx0sys.ic` | 175–227 |
| `trx_rw_is_active` 声明 | `storage/innobase/include/trx0sys.h` | 166–178 |
| `TRX_SHARDS_N = 256` 及分片函数 | `storage/innobase/include/trx0sys.h` | 368–377 |
| `trx_sys_t` 结构体（含 `shards[]`） | `storage/innobase/include/trx0sys.h` | 525–580 |
| 隐式锁释放与 `trx_rw_is_active` 关系注释 | `storage/innobase/trx/trx0trx.cc` | 1895–1912 |
| 隐式锁转显式锁调用点 | `storage/innobase/lock/lock0lock.cc` | 5330 |
| MVCC 可见性判断（独立路径） | `storage/innobase/include/read0types.h` | 163–183 |
