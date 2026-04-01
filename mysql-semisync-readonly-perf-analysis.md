# MySQL 8.0 半同步复制插件导致只读事务性能瓶颈深度分析

> 源码版本：MySQL 8.0.41  
> 分析依据：perf 火焰图 + 源码追踪  
> 报告日期：2026-03-31

---

## 一、问题现象

在安装了 `semisync_master.so` / `semisync_slave.so`（即 `rpl_semi_sync_source` / `rpl_semi_sync_replica`）插件的实例上，纯只读 SELECT 事务的吞吐量严重下降。perf 抓取的热点调用栈如下：

```
worker_main
  └─ do_command
       └─ dispatch_command
            └─ launch_hook_trans_begin          ← 占 38% CPU
                 └─ Trans_delegate::trans_begin
                      └─ plugin_lock            ← 占 23% CPU
                           └─ futex_wait        ← 占 17%
                                └─ native_queued_spin_lock_slowpath  ← 占 30%
```

关键数字：**每一条打到用户表的 SELECT，都因等待全局互斥锁 `LOCK_plugin` 而阻塞，且争用随并发线程数线性恶化。**

---

## 二、完整调用链源码追踪

### 2.1 入口：`dispatch_command` → `launch_hook_trans_begin`

每条 SQL 在完成解析后，`mysql_execute_command`（`sql/sql_parse.cc:3182`）会无条件调用：

```3182:3193:sql/sql_parse.cc
    int ret = launch_hook_trans_begin(thd, all_tables);
    if (ret) {
      my_error(ret, MYF(0));
      return -1;
    }

  } else {
    int ret = launch_hook_trans_begin(thd, all_tables);
    if (ret) {
      my_error(ret, MYF(0));
      return -1;
    }
```

**该函数对 slave 线程和普通连接线程均生效，没有针对只读路径的快速跳出。**

### 2.2 `launch_hook_trans_begin` 的过滤逻辑

```1385:1488:sql/rpl_handler.cc
int launch_hook_trans_begin(THD *thd, Table_ref *all_tables) {
  // ...
  // 已在同一事务内调用过则直接返回
  if (thd->get_transaction()->was_trans_begin_hook_invoked()) {
    return 0;
  }

  // 跳过纯管理命令：SHOW / SET / EMPTY / USE / KILL / SHUTDOWN ...
  if ((is_set || is_show || is_empty || is_use || is_stop_gr || is_shutdown ||
       is_reset_persist || is_kill) &&
      !lex->uses_stored_routines()) {
    return 0;
  }

  // 对于 SELECT / DO，只有无表或仅查 perf_schema/info_schema/sys 才可跳过
  if (is_query_block) {
    // ...
    if (!is_udf && all_tables == nullptr) {
      hold_command = false;   // 无表 SELECT 可跳过
    }
    if (is_process_list || is_perf_schema_table || is_sys_db) {
      hold_command = false;   // 系统库查询可跳过
    }
  }

  // 凡是 hold_command == true，就触发 hook
  if (hold_command) {
    RUN_HOOK(transaction, trans_begin, (thd, ret));
    // ...
  }
  return ret;
}
```

**结论：任何访问用户业务表的 `SELECT`（包括只读事务）都不会被提前跳过，`hold_command` 保持 `true`，`RUN_HOOK` 必然执行。**

### 2.3 `RUN_HOOK` 展开

```458:462:sql/rpl_handler.h
#define RUN_HOOK(group, hook, args) \
  (group##_delegate->is_empty() ? 0 : group##_delegate->hook args)
```

只要 `transaction_delegate` 的观察者列表非空（即 semisync 插件已注册），就会调用：

```854:876:sql/rpl_handler.cc
int Trans_delegate::trans_begin(THD *thd, int &out) {
  // ...
  Trans_param param;
  TRANS_PARAM_ZERO(param);
  param.server_uuid = server_uuid;
  param.thread_id   = thd->thread_id();
  // ...

  int ret = 0;
  FOREACH_OBSERVER_ERROR_OUT(ret, begin, &param, out);
  return ret;
}
```

### 2.4 `FOREACH_OBSERVER_ERROR_OUT` 宏 — 锁的核心所在

```498:546:sql/rpl_handler.cc
#define FOREACH_OBSERVER_ERROR_OUT(r, f, args, out)                    \
  r = 0;                                                               \
  Prealloced_array<plugin_ref, 8> plugins(PSI_NOT_INSTRUMENTED);       \
  read_lock();          /* ① 获取 Delegate 的 OS rwlock 读锁 */        \
  Observer_info_iterator iter = observer_info_iter();                  \
  Observer_info *info = iter++;                                        \
  bool replication_optimize_for_static_plugin_config =                 \
      this->use_spin_lock_type();                                      \
  for (; info; info = iter++) {                                        \
    plugin_ref plugin = (replication_optimize_for_static_plugin_config \
                             ? info->plugin                            \
                             : my_plugin_lock(0, &info->plugin));      \
    /* ② 默认路径：调用 plugin_lock，内部加全局互斥锁 LOCK_plugin */   \
    // ...                                                             \
    hook_error = ((Observer *)info->observer)->f(args, error_out);     \
    /* ③ 调用 semisync 的 begin hook（见 2.5 节，是空函数！）*/         \
  }                                                                    \
  unlock();                                                            \
  if (!plugins.empty()) plugin_unlock_list(...);  /* ④ 再次加 LOCK_plugin */
```

### 2.5 semisync source 的 `begin` hook 是一个空函数

```124:124:plugin/semisync/semisync_source_plugin.cc
static int repl_semi_report_begin(Trans_param *, int &) { return 0; }
```

`repl_semi_report_begin` **什么业务逻辑都没有**，直接返回 0。  
但框架层在调用它之前，**必须先完成 `plugin_lock` 的引用计数增减操作**。

### 2.6 `plugin_lock` — 全局 `LOCK_plugin` 互斥锁

```975:983:sql/sql_plugin.cc
plugin_ref plugin_lock(THD *thd, plugin_ref *ptr) {
  LEX *lex = thd ? thd->lex : nullptr;
  plugin_ref rc;
  DBUG_TRACE;
  mysql_mutex_lock(&LOCK_plugin);   // ← 全局唯一互斥锁！
  rc = my_intern_plugin_lock_ci(lex, *ptr);
  mysql_mutex_unlock(&LOCK_plugin);
  return rc;
}
```

`LOCK_plugin` 是 MySQL 中**最重量级的全局互斥锁之一**，保护整个插件数组的引用计数和状态变更。它在以下场景都会被竞争：
- 任何调用 `plugin_lock` / `plugin_unlock` 的线程
- `INSTALL PLUGIN` / `UNINSTALL PLUGIN`
- `SHOW PLUGINS`
- 以及每一条触发 observer hook 的 SQL

---

## 三、锁竞争完整因果链

```
每条 SELECT（含只读事务）
  │
  ▼
launch_hook_trans_begin()              [sql/sql_parse.cc:3182]
  │  hold_command = true（访问用户表）
  ▼
RUN_HOOK(transaction, trans_begin)     [sql/rpl_handler.h:458]
  │  transaction_delegate 非空（semisync 已注册观察者）
  ▼
Trans_delegate::trans_begin()          [sql/rpl_handler.cc:854]
  │
  ▼
FOREACH_OBSERVER_ERROR_OUT 宏展开
  │
  ├─ read_lock()                        ← Delegate OS rwlock 读锁
  │     默认模式(opt_replication_optimize_for_static_plugin_config=false)
  │     → mysql_rwlock_rdlock(&lock)
  │     → 读写锁本身存在内核态竞争
  │
  └─ my_plugin_lock(0, &info->plugin)  ← 对每个 Observer 各调用一次
       │
       ▼
       plugin_lock()                   [sql/sql_plugin.cc:975]
         mysql_mutex_lock(&LOCK_plugin)  ← 全局唯一互斥锁
           │  高并发下大量线程堆积
           ▼
           futex_wait
             └─ native_queued_spin_lock_slowpath   ← perf 观测到的热点
                                 （CPU 在此空转/休眠等待锁）

调用 repl_semi_report_begin()          ← 空函数，什么都不做，return 0
  ↑
  └── 付出了上述全部锁开销，却得到零收益
```

**perf 中占比 38% 的 `launch_hook_trans_begin` 开销几乎全部来自锁竞争，而不是 semisync 的业务逻辑。**

---

## 四、为什么只读事务受影响最严重

| 维度 | 写事务 | 只读事务 |
|------|--------|---------|
| 真正的 semisync 等待点 | `before_commit` → 等待 ACK，耗时 ms 级 | **无**（begin hook 是空函数） |
| `LOCK_plugin` 持有时间 | 极短（`plugin_lock` 本身很快） | 同上 |
| 锁竞争占总耗时比例 | 被 ACK 等待稀释，占比低 | **全部开销都是锁竞争，占比极高** |
| 事务本身执行时间 | 通常更长（有写入、redo log 等） | 通常极短（μs 级点查） |

只读事务越短，`LOCK_plugin` 的获取时间占该事务总耗时的比例就越高。在高并发场景下，大量短事务同时争抢同一把全局锁，形成**漏斗效应**，QPS 上不去。

---

## 五、两个相关优化参数的工作原理

MySQL 8.0 提供了两个系统变量来缓解此问题：

### 5.1 `replication_optimize_for_static_plugin_config`

```78:78:sql/rpl_handler.cc
bool opt_replication_optimize_for_static_plugin_config{false};
```

**默认值：`OFF`（即当前触发瓶颈的模式）**

| 状态 | 锁类型 | plugin_lock 调用 |
|------|--------|-----------------|
| OFF（默认） | OS `mysql_rwlock_t` | 每次 hook **都调用** `plugin_lock` → 争抢 `LOCK_plugin` |
| ON | 无锁化 `Shared_spin_lock`（原子操作） | **不调用** `plugin_lock`，提前持久化引用计数 |

开启后，`FOREACH_OBSERVER` 宏中：

```465:468:sql/rpl_handler.cc
  for (; info; info = iter++) {
    plugin_ref plugin = (replication_optimize_for_static_plugin_config
                             ? info->plugin                  // ← 直接用缓存引用，无锁
                             : my_plugin_lock(0, &info->plugin));  // ← 默认路径，加锁
```

**代价：** 开启后 `UNINSTALL PLUGIN` 会被推迟到变量重置为 OFF 之后才能执行，适合插件配置稳定的场景。

### 5.2 `replication_sender_observe_commit_only`

```79:79:sql/rpl_handler.cc
std::atomic<bool> opt_replication_sender_observe_commit_only{false};
```

**默认值：`OFF`**

开启后，复制 observer 的所有 hook 调用（除 `after_commit`）都被跳过。`trans_begin` hook 也在跳过范围内，可以从根源绕过此路径。

---

## 六、根本原因总结

| # | 根因 | 所在代码 |
|---|------|---------|
| 1 | `launch_hook_trans_begin` 对所有访问用户表的 SELECT 无差别触发，包括只读事务 | `sql/sql_parse.cc:3182` |
| 2 | `semisync_source` 注册了 `Trans_observer`，导致 `transaction_delegate` 非空，`RUN_HOOK` 必须执行 | `plugin/semisync/semisync_source_plugin.cc:403` |
| 3 | `repl_semi_report_begin` 是完全空的函数（`return 0`），毫无业务价值，却强迫框架走完整 hook 路径 | `plugin/semisync/semisync_source_plugin.cc:124` |
| 4 | 默认配置下 `FOREACH_OBSERVER` 为每个 observer 调用 `plugin_lock`，内部加全局互斥锁 `LOCK_plugin` | `sql/sql_plugin.cc:979` |
| 5 | `LOCK_plugin` 是全局单点锁，N 个并发只读线程对其产生 N 倍争用，退化为串行 | `sql/sql_plugin.cc:979` |

---

## 七、优化建议

### 方案 A：开启静态插件配置优化（推荐，立竿见影）

```sql
SET GLOBAL replication_optimize_for_static_plugin_config = ON;
```

- **效果：** 彻底消除 `plugin_lock` 对 `LOCK_plugin` 的竞争，改用无锁的原子共享自旋锁
- **适用场景：** semisync 插件长期稳定加载，不动态卸载的生产环境
- **副作用：** `UNINSTALL PLUGIN` 被 defer，需先将此变量设回 OFF

### 方案 B：跳过非 commit hook（适合主库）

```sql
SET GLOBAL replication_sender_observe_commit_only = ON;
```

- **效果：** `trans_begin` hook 完全不触发，`launch_hook_trans_begin` 路径提前返回
- **适用场景：** 主库场景，只关心 commit 时的半同步等待

### 方案 C：两者同时开启（最优）

```sql
SET GLOBAL replication_optimize_for_static_plugin_config = ON;
SET GLOBAL replication_sender_observe_commit_only = ON;
```

对只读事务来说，`replication_sender_observe_commit_only` 可以让 `launch_hook_trans_begin` 提前返回（跳过整个 observer 遍历），`replication_optimize_for_static_plugin_config` 保障写路径也不受 `LOCK_plugin` 拖累。

### 方案 D：业务层——区分读写实例

- 只读实例（Replica）不需要加载 `semisync_source`（master 插件），应只加载 `semisync_replica`
- `semisync_replica` 不注册 `Trans_observer`，`transaction_delegate->is_empty()` 为 true，`RUN_HOOK` 直接返回 0，整条 hook 路径开销接近零

### 方案 E（长期）：内核侧优化

在 `launch_hook_trans_begin` 的 `is_query_block` 分支中增加只读事务快速路径：
- 若事务为只读（`thd->tx_read_only` 或 `autocommit=1`）且 semisync 仅关注写事务，可在此处提前 `return 0`
- 或者在 `FOREACH_OBSERVER` 宏中，先通过无锁方式检查 observer 的 `begin` 函数指针是否为默认空实现再决定是否调用 `plugin_lock`

---

## 八、各方案效果对比

| 方案 | 消除 `LOCK_plugin` 争用 | 消除 `Delegate rwlock` 争用 | 需要重启 | 适用范围 |
|------|------------------------|---------------------------|---------|---------|
| A（静态配置优化） | ✅ 完全消除 | ✅ 改为无锁自旋 | 否 | 主/备均可 |
| B（只观测 commit） | ✅ hook 不调用 | ✅ hook 不调用 | 否 | 主库 |
| C（A+B） | ✅ | ✅ | 否 | 主库最优 |
| D（备库不装 source 插件） | ✅ 天然避免 | ✅ 天然避免 | 是（卸载插件） | 只读备库 |
| 不装 semisync 插件 | ✅ | ✅ | 是 | 无半同步需求时 |

---

## 九、附：关键源码文件索引

| 文件 | 关键内容 |
|------|---------|
| `sql/sql_parse.cc:3182` | `launch_hook_trans_begin` 调用点 |
| `sql/rpl_handler.cc:1385` | `launch_hook_trans_begin` 完整实现，含 SELECT 过滤逻辑 |
| `sql/rpl_handler.cc:854` | `Trans_delegate::trans_begin` 实现 |
| `sql/rpl_handler.cc:498` | `FOREACH_OBSERVER_ERROR_OUT` 宏，`my_plugin_lock` 调用点 |
| `sql/rpl_handler.h:56-268` | `Delegate` 基类，两种锁类型的切换逻辑 |
| `sql/sql_plugin.cc:975` | `plugin_lock` → `LOCK_plugin` 互斥锁 |
| `plugin/semisync/semisync_source_plugin.cc:124` | `repl_semi_report_begin` 空函数 |
| `plugin/semisync/semisync_source_plugin.cc:403` | `Trans_observer` 注册表 |

---

## 十、补充：`Trans_observer` 详解

### 10.1 本质：插件钩子函数表（Vtable）

`Trans_observer` 是 MySQL 插件系统暴露给外部插件的**事务生命周期观察者接口**，本质上是一张**函数指针表**，定义在 `sql/replication.h`：

```c
typedef struct Trans_observer {
  uint32 len;

  before_dml_t    before_dml;       // DML 执行前校验
  before_commit_t before_commit;    // binlog 写入前
  before_rollback_t before_rollback;// 回滚前
  after_commit_t  after_commit;     // 存储引擎提交后
  after_rollback_t after_rollback;  // 存储引擎回滚后
  begin_t         begin;            // SQL 开始执行前
} Trans_observer;
```

插件初始化时填入自己的回调函数，注册到全局 `transaction_delegate`。此后 MySQL 在每个事务的关键生命周期节点，都会遍历所有已注册的 observer，依次调用对应的钩子。

---

### 10.2 六个钩子的触发时机与语义

#### `begin` — 事务开始前

**触发点**：每条 SQL 执行前，在 `launch_hook_trans_begin` 中调用。

**语义**：SQL 命令即将开始执行，可在此处**拦截或延迟事务**。`out_val` 是输出参数，可向调用方传递错误码。

**实际用途对比**：

| 插件 | 实现 | 作用 |
|------|------|------|
| semisync_source | `return 0`（空函数） | 什么都不做，却引发全局锁争用（本文核心问题所在） |
| Group Replication | `before_transaction_begin(...)` | 检查 `group_replication_consistency` 级别，若设置了 `BEFORE` 或 `BEFORE_AND_AFTER` 一致性，则**在此挂起等待**组内所有已提交事务都被 apply 完毕后才放行 |

---

#### `before_dml` — DML 执行前校验

**触发点**：INSERT/UPDATE/DELETE 语句执行前，通过 `RUN_HOOK(transaction, before_dml, ...)` 调用。

**语义**：在行数据实际修改之前，对本次 DML 涉及的**表结构进行合规性校验**。参数 `param->tables_info` 包含所有涉及表的元信息。

**Group Replication 的实现**是最典型的用法，会逐项检查：
- binlog 必须开启，且格式必须为 ROW
- 必须开启 `transaction_write_set_extraction`
- 所有涉及的表必须是 InnoDB 引擎
- 所有涉及的表必须有主键
- 不允许含 `ON DELETE/UPDATE CASCADE` 的外键

任一检查失败则 `out++`，事务被中止。

---

#### `before_commit` — binlog 写入前

**触发点**：事务 binlog 缓存即将落盘（写入 binary log 文件）之前。

**这是整个 `Trans_observer` 体系中最重量级的钩子。**

- **semisync_source**：此钩子为空函数（`return 0`），真正的等待在 `Binlog_storage_observer` 的 `after_flush` / `after_sync` 里
- **Group Replication**：此处是 GR 的核心路径：
  1. 将事务的 `write_set`（行哈希集合）+ binlog 缓存序列化成 `Transaction_message`
  2. 通过 GCS（Paxos/Raft 层）**广播给所有组成员**
  3. 调用 `transactions_latch->waitTicket()` **阻塞等待**认证结果（冲突检测通过才放行）
  4. 认证失败则回滚整个事务

---

#### `before_rollback` — 回滚前

**触发点**：事务向存储引擎发起 rollback 之前。

当前 semisync 和 GR 均为空实现或只做简单清理。

---

#### `after_commit` — 存储引擎提交后

**触发点**：事务提交到存储引擎（InnoDB redo 落盘）之后。此时 `param->log_file` 和 `param->log_pos` 已有效。

**semisync 的关键用法**（`WAIT_AFTER_COMMIT` 模式）：

```c
static int repl_semi_report_commit(Trans_param *param) {
  bool is_real_trans = param->flags & TRANS_IS_REAL_TRANS;
  if (rpl_semi_sync_source_wait_point == WAIT_AFTER_COMMIT
      && is_real_trans && param->log_pos) {
    return repl_semisync->commitTrx(param->log_file, param->log_pos);
    // ↑ 阻塞等待至少一个 replica 回复 ACK
  }
  return 0;
}
```

**GR 的用法**：通知内部 `Group_transaction_listener` 更新 GTID 追踪、释放一致性等待。

---

#### `after_rollback` — 存储引擎回滚后

**触发点**：事务在存储引擎中回滚完成之后。

semisync 将此钩子复用 `repl_semi_report_commit` 的实现。原因：XA 事务等边界场景下，rollback 也可能已经写入了 binlog，需要触发 ACK 等待流程。

---

### 10.3 传递给钩子的数据：`Trans_param`

所有钩子都接收同一个 `Trans_param` 结构体，各字段在不同钩子时机下有效性不同：

| 字段 | 类型 | 有效时机 | 用途 |
|------|------|---------|------|
| `server_id` / `server_uuid` | 标识 | 全程 | 标识事务来源 |
| `thread_id` | 标识 | 全程 | 关联会话 |
| `flags` | 位标志 | 全程 | `TRANS_IS_REAL_TRANS` 等 |
| `tables_info` / `number_of_tables` | 表元信息 | `before_dml` | 表名、引擎类型、主键数、是否有 CASCADE 外键 |
| `trans_ctx_info` | 系统变量快照 | `before_dml` / `before_commit` | binlog_format、隔离级别、write_set 算法等 |
| `trx_cache_log` / `stmt_cache_log` | binlog 缓存 | `before_commit` | GR 用于序列化广播消息 |
| `log_file` / `log_pos` | binlog 坐标 | `after_commit` | semisync 用于等待 ACK |
| `gtid_info` | GTID | `after_commit` | sidno + gno，GR 用于一致性追踪 |
| `group_replication_consistency` | 枚举 | `begin` | GR 用于决定是否需要在 begin 处等待 |
| `hold_timeout` | 超时 | `begin` | GR 事务最大等待时间 |

---

### 10.4 整体架构图

```
MySQL Server 事务生命周期
       │
       ├─ SQL 开始执行          → begin()
       │    ↓                       semisync: 空函数（但触发 LOCK_plugin 竞争）
       │                            GR: 检查一致性级别，可能挂起等待
       ├─ DML 行操作前          → before_dml()
       │    ↓                       semisync: 空函数
       │                            GR: 校验表引擎/主键/外键合规性
       │  [事务执行中 ...]
       │    ↓
       ├─ 写 binlog 缓存前      → before_commit()
       │    ↓                       semisync: 空函数
       │                            GR: 序列化广播 + 等待冲突检测认证
       ├─ binlog fsync 后       → [Binlog_storage_observer::after_sync]
       │    ↓                       semisync: WAIT_AFTER_SYNC 模式下等待 replica ACK
       ├─ 存储引擎 commit 后    → after_commit()
       │    ↓                       semisync: WAIT_AFTER_COMMIT 模式下等待 replica ACK
       │                            GR: 释放一致性等待，更新 GTID 追踪
       └─ 存储引擎 rollback 后  → after_rollback()
                                    semisync: 同 after_commit 处理逻辑
                                    GR: 通知内部观察者清理
```

---

### 10.5 设计哲学：可插拔的事务扩展点

`Trans_observer` 是 MySQL 为第三方插件开放的**事务扩展钩子**，将事务流程的每个关键节点暴露出来，允许插件在不修改 Server 核心代码的前提下：

- **拦截**（`before_*`）：返回非 0 可中止事务
- **扩展**（`after_*`）：通知外部系统（如等待 replica ACK）
- **校验**（`before_dml`）：在数据变更前检查合规性

正因为这套机制足够通用，semisync、Group Replication 以及任何第三方复制插件都可以复用同一套框架，Server 核心对具体实现一无所知。

**本文性能问题的根源**正在于此：即使 semisync 的 `begin` 钩子是空函数，框架层仍须为每个注册的 observer 执行完整的 `plugin_lock` 流程。"空钩子不等于零开销"——插件注册的存在本身就是开销的来源。
